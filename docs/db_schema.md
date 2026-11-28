# Database Schema & Order State Machine (Multi-Tenant / Zero-Trust Architecture)

> **Synthetic data:** all example values in this document (companies, addresses, vehicles, VINs, plates, phone numbers, hashes and IDs) are synthetic, generated for R&D demonstration only. See the synthetic data notice in the [README](../README.md).

The PostgreSQL schema as defined by the application's SQLAlchemy models and
Alembic migrations; a test runs `alembic check`, so the two can't silently
differ. Last checked against the code: 2026-11-28.

Keycloak (dashboard login) keeps its own data in a separate database with
its own user in production; it shares no tables or credentials with this
schema. The manifest job queue (procrastinate) keeps its own
`procrastinate_*` tables in this database; its jobs carry only ids and
random storage keys, never file names or manifest content.

## Architecture Overview

- **Multi-tenancy**: every tenant-owned table (`optimization_sessions`,
  `bulk_batches`, `vehicles`, `batch_vehicles`, `route_legs`, `drivers`,
  `tenant_drivers`, `outbox_messages`, `idempotency_keys`,
  `uploaded_file_hashes`, `manifest_file_results`) carries a mandatory
  `tenant_id` column, `NOT NULL`, `FOREIGN KEY → companies.id ON DELETE
  CASCADE`, and declares `TenantScopedMixin`. An automatic SQLAlchemy event
  listener injects `tenant_id == <current tenant>` into **every** ORM
  `select`/`update`/`delete` against these tables, so a forgotten explicit
  filter can't leak cross-tenant rows. Queries with no tenant bound to the
  current context fail closed (`MissingTenantContextError`) rather than
  running unfiltered.
- **Vehicle registry and vehicle tasks (Multi-Leg Relay)**: a physical
  vehicle and a relocation *task* for it live in two tables, so the same
  vehicle can take part in several legs and batches:
  - **`vehicles`** (the **registry**) — one row per physical vehicle per
    tenant (`vin`, `make_model`, `license_plate` only). Unique per
    `(tenant_id, vin)` through the partial index
    `uq_vehicles_tenant_vin_known`, which excludes the `"UNKNOWN"`
    placeholder VIN so manifests with an unresolved VIN never collide.
  - **`batch_vehicles`** (the **tasks**) — one row per relocation task: a
    specific leg in a specific batch, with its own pickup/dropoff window,
    status, `driver_hash` and routing metadata. `vehicle_id` (FK →
    `vehicles.id`) is not unique: a vehicle has one task per leg.
  - Every import path (`POST /api/v1/batches/upload` for JSON/Excel/CSV and
    the OCR/LLM pipeline, `POST /api/v1/vehicles/`) resolves the registry row
    with an atomic `INSERT ... ON CONFLICT (tenant_id, vin) WHERE vin <>
    'UNKNOWN' DO UPDATE` upsert, then creates a new `batch_vehicles` row.
- **Parallel optimization sessions**: `optimization_sessions` groups the
  batches and route legs of one dispatch stream, so a dispatcher can run
  independent streams at once (e.g. Riyadh→Jeddah and Dubai→Abu Dhabi).
  `bulk_batches` and `route_legs` both carry a `session_id`; on `route_legs`
  it is `NOT NULL`.
- **Multi-file manifest uploads**: a session owns one `bulk_batches` row
  **per leg** (files tagged with the same leg number are merged into that
  leg's batch). Each batch's outcome (`status`, `successful_rows`,
  `failed_rows`, `payload`) is tracked per leg.
- **Route leg state machine**: `route_legs.status` is constrained by a
  Postgres `CHECK` constraint (`ck_route_legs_status`) to five values; the
  code uses `pending`, `claimed_driver`, `claimed_passenger` and `rejected`.
  See [Route Leg State Machine](#route-leg-state-machine-route_legsstatus).
- **Batch row lock**: every change to a batch's `payload`, `claimed_count`
  or `status` first takes `SELECT … FOR UPDATE` on the `bulk_batches` row and
  commits soon after, so concurrent claims, offers and edits can't overwrite
  each other's changes.
- **Zero-PII**: no table in this schema stores a raw phone number.
  `batch_vehicles.driver_hash`, `drivers.driver_hash`,
  `outbox_messages.driver_hash` and `route_legs.driver_phone` store an opaque
  driver key: either the key of a driver from the dispatcher's list
  (`drv_…`, table `tenant_drivers`, nothing derived from a phone), or, for
  Telegram drivers, a keyed HMAC-SHA256 hash of the phone. The claim logic
  treats both the same. Redis holds hashes too (chat ↔ driver_hash links,
  pending offers): raw phone numbers are stored nowhere.
- **Retention**: a batch and everything that refers to it is deleted on
  request (`DELETE /api/v1/batches/{batch_id}`) or by the worker's hourly
  sweep once it has been inactive longer than the tenant's retention period
  (`companies.batch_retention_days`, default `BATCH_RETENTION_DAYS` 90).
  Working tables (`outbox_messages`, `idempotency_keys`,
  `uploaded_file_hashes`, `manifest_file_results`, `telegram_updates`)
  expire sooner; see each table.
- **Not tenant-scoped:** `telegram_updates` holds only Telegram's
  `update_id` and processing state, no tenant data, chat id or text; the
  tenant of a tap is resolved from the offered task (see
  [Multi-Tenant Isolation Enforcement](#multi-tenant-isolation-enforcement),
  point 5).

## PostgreSQL Schema (DDL)

```sql
-- 0. COMPANIES (Tenants) -----------------------------------------------
-- Every tenant-scoped table below has a mandatory FK into this table.
CREATE TABLE companies (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    -- (billing / plan columns omitted from this public preview)
    -- Days a batch is kept after its last activity, as agreed with this
    -- customer; NULL: the BATCH_RETENTION_DAYS default. Set by the
    -- operator, not through the API.
    batch_retention_days INTEGER,
    created_at TIMESTAMPTZ DEFAULT now(),

    CONSTRAINT ck_companies_batch_retention_days
        CHECK (batch_retention_days IS NULL OR batch_retention_days > 0)
);

-- 1. OPTIMIZATION_SESSIONS (Parallel dispatcher streams) -----------------
CREATE TABLE optimization_sessions (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    dispatcher_id UUID,
    name TEXT,
    status VARCHAR NOT NULL DEFAULT 'active',        -- active | completed | cancelled | archived
    payload JSONB,                                   -- freeform session-level metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_optimization_sessions_tenant_id ON optimization_sessions(tenant_id);
CREATE INDEX ix_optimization_sessions_tenant_status ON optimization_sessions(tenant_id, status);

-- 2. BULK_BATCHES (Manifest imports — one row per leg) -------------------
CREATE TABLE bulk_batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    -- Nullable for batches created before sessions existed; current upload
    -- flows always set it.
    session_id UUID REFERENCES optimization_sessions(id) ON DELETE SET NULL,
    -- Multi-Leg Relay: this batch's leg position (1, 2, 3...) within its
    -- session. Drives sequential wave broadcasting (leg 2 is only offered to
    -- drivers who cleared leg 1, etc.) and the "Leg N" label in the UI.
    sequence_order INTEGER NOT NULL DEFAULT 1,
    dispatcher_id UUID,
    file_name TEXT NOT NULL,
    total_vehicles INTEGER NOT NULL DEFAULT 1,
    successful_rows INTEGER NOT NULL DEFAULT 0,
    failed_rows INTEGER NOT NULL DEFAULT 0,
    claimed_count INTEGER NOT NULL DEFAULT 0,
    status VARCHAR NOT NULL DEFAULT 'ready',         -- processing | ready | manual_review | failed | broadcasting | completed
    -- The dashboard's view of the batch: parsed vehicles (with their claim
    -- status mirrored from batch_vehicles), waypoints, offered drivers
    -- (driver key + masked phone or name, offer status) and, for a batch in
    -- manual_review, review_reasons (code, vehicle, leg, value). The
    -- batch_vehicles rows stay the source of truth for who has which car.
    payload JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Target of batch_vehicles' same-tenant foreign key (below).
    CONSTRAINT uq_bulk_batches_id_tenant UNIQUE (id, tenant_id)
);

CREATE INDEX ix_bulk_batches_tenant_id ON bulk_batches(tenant_id);
CREATE INDEX ix_bulk_batches_tenant_status ON bulk_batches(tenant_id, status);
CREATE INDEX ix_bulk_batches_tenant_session ON bulk_batches(tenant_id, session_id);
CREATE INDEX ix_bulk_batches_tenant_session_sequence ON bulk_batches(tenant_id, session_id, sequence_order);

-- 3a. VEHICLES (Registry — one row per physical vehicle per tenant) ------
-- Only slow-changing attributes.
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    vin VARCHAR NOT NULL,
    make_model VARCHAR NOT NULL,
    license_plate VARCHAR,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_vehicles_tenant_id ON vehicles(tenant_id);
CREATE INDEX ix_vehicles_tenant_vin ON vehicles(tenant_id, vin);
-- Unique per (tenant, vin), for known VINs only.
CREATE UNIQUE INDEX uq_vehicles_tenant_vin_known
    ON vehicles (tenant_id, vin)
    WHERE vin <> 'UNKNOWN';

-- 3b. BATCH_VEHICLES (Relocation tasks — one row per vehicle, leg, batch) -
-- vehicle_id is NOT unique: a vehicle has one task per leg of a relay.
CREATE TABLE batch_vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE CASCADE,
    sequence_order INTEGER NOT NULL DEFAULT 1,  -- Multi-Leg Relay: leg order (1, 2, 3...)

    -- Copies of the registry row taken when the task is created; the
    -- registry row stays the source of truth.
    vin VARCHAR NOT NULL,
    make_model VARCHAR NOT NULL,
    license_plate VARCHAR,

    -- Origin / Destination
    pickup_address VARCHAR NOT NULL,
    pickup_lat DOUBLE PRECISION,
    pickup_lng DOUBLE PRECISION,
    pickup_window_start TIMESTAMPTZ NOT NULL,
    pickup_window_end TIMESTAMPTZ NOT NULL,

    dropoff_address VARCHAR NOT NULL,
    dropoff_lat DOUBLE PRECISION,
    dropoff_lng DOUBLE PRECISION,
    dropoff_deadline TIMESTAMPTZ NOT NULL,

    -- State & hashed assignment (no raw phone or name stored)
    status VARCHAR NOT NULL DEFAULT 'unassigned'
        CONSTRAINT vehicles_status_check
        CHECK (status IN ('unassigned', 'locked', 'assigned', 'in_transit', 'delivered', 'cancelled')),
    driver_hash VARCHAR,     -- opaque driver key: dispatcher-list key or HMAC-SHA256 of the phone

    locked_until TIMESTAMPTZ,   -- unused by the current claim flow (atomic UPDATE, no locks)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Real-road routing metadata (Mapbox/OSRM). route_provider = 'fallback'
    -- marks a straight-line approximation used when no provider answered.
    distance_km DOUBLE PRECISION,
    duration_minutes DOUBLE PRECISION,
    route_geometry JSONB,          -- ordered [lon, lat] polyline (GeoJSON LineString convention)
    route_provider VARCHAR,        -- 'mapbox' | 'osrm' | 'fallback'

    -- A task's batch must belong to the task's tenant (NULL batch_id is not
    -- checked): cross-tenant task -> batch links are impossible.
    CONSTRAINT fk_batch_vehicles_batch_same_tenant
        FOREIGN KEY (batch_id, tenant_id) REFERENCES bulk_batches(id, tenant_id) ON DELETE CASCADE
);

CREATE INDEX ix_batch_vehicles_tenant_id ON batch_vehicles(tenant_id);
CREATE INDEX ix_batch_vehicles_tenant_batch ON batch_vehicles(tenant_id, batch_id);
CREATE INDEX ix_batch_vehicles_tenant_vehicle ON batch_vehicles(tenant_id, vehicle_id);
CREATE INDEX ix_batch_vehicles_tenant_status ON batch_vehicles(tenant_id, status);
CREATE INDEX ix_batch_vehicles_batch_id ON batch_vehicles(batch_id);

-- 4. ROUTE_LEGS (Multi-leg route segments + state machine) --------------
-- One segment of a (possibly chained) route, e.g. an outbound
-- Riyadh->Jeddah leg and its Jeddah->Riyadh backhaul. Mandatory tenant_id
-- and session_id.
CREATE TABLE route_legs (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    session_id UUID NOT NULL REFERENCES optimization_sessions(id) ON DELETE CASCADE,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,

    origin VARCHAR NOT NULL,
    destination VARCHAR NOT NULL,

    -- Mirrors bulk_batches.sequence_order.
    sequence_order INTEGER NOT NULL DEFAULT 1,

    -- See the state diagram below.
    status VARCHAR NOT NULL DEFAULT 'pending'
        CONSTRAINT ck_route_legs_status
        CHECK (status IN ('pending', 'claimed_driver', 'rejected', 'passenger_offered', 'claimed_passenger')),

    -- Despite the name, an opaque driver key (as batch_vehicles.driver_hash),
    -- never a raw phone number.
    driver_phone VARCHAR,

    -- Passenger mode: driver keys of drivers riding along in an assigned
    -- vehicle on this leg once every vehicle was taken.
    co_driver_hashes VARCHAR[] NOT NULL DEFAULT '{}',

    -- Real-road routing metadata (same convention as batch_vehicles).
    distance_km DOUBLE PRECISION,
    duration_minutes DOUBLE PRECISION,
    route_geometry JSONB,
    route_provider VARCHAR,

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_route_legs_tenant_id ON route_legs(tenant_id);
CREATE INDEX ix_route_legs_tenant_session ON route_legs(tenant_id, session_id);
CREATE INDEX ix_route_legs_tenant_session_status ON route_legs(tenant_id, session_id, status);
CREATE INDEX ix_route_legs_tenant_session_sequence ON route_legs(tenant_id, session_id, sequence_order);
CREATE INDEX ix_route_legs_batch_id ON route_legs(batch_id);

-- 5. DRIVERS (claim state per batch and driver) --------------------------
CREATE TABLE drivers (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    driver_hash VARCHAR NOT NULL,   -- opaque driver key, never a raw phone number
    status VARCHAR NOT NULL DEFAULT 'pending',  -- pending | claimed | passenger | expired | cancelled
    claimed_at TIMESTAMPTZ,
    expired_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_drivers_tenant_batch_hash UNIQUE (tenant_id, batch_id, driver_hash)
);

CREATE INDEX ix_drivers_tenant_id ON drivers(tenant_id);
CREATE INDEX ix_drivers_tenant_batch ON drivers(tenant_id, batch_id);
CREATE INDEX ix_drivers_tenant_batch_status ON drivers(tenant_id, batch_id, status);
CREATE INDEX ix_drivers_batch_id ON drivers(batch_id);

-- 6. TENANT_DRIVERS (the dispatcher's driver list) -----------------------
-- No phone number and nothing derived from one: an internal ID, a name or
-- alias the dispatcher recognises, and the language offer texts are written
-- in. A driver's key in claims, route legs and Redis is derived from the id
-- ('drv_' + the id without dashes).
CREATE TABLE tenant_drivers (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    display_name VARCHAR(80) NOT NULL,   -- name or alias; phone-like values rejected by the API
    language VARCHAR(2) NOT NULL,        -- de | ar | en: language of offer texts
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_tenant_drivers_tenant_name UNIQUE (tenant_id, display_name),
    CONSTRAINT ck_tenant_drivers_language CHECK (language IN ('de', 'ar', 'en'))
);

CREATE INDEX ix_tenant_drivers_tenant_id ON tenant_drivers(tenant_id);

-- 7. BROADCAST_WAVES (not used by the dispatch flow yet) -----------------
-- Not tenant-scoped at the column level; isolation is transitive via
-- batch_id -> bulk_batches.tenant_id. Stores only driver keys.
CREATE TABLE broadcast_waves (
    id UUID PRIMARY KEY,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    wave_number INTEGER NOT NULL,
    status VARCHAR NOT NULL DEFAULT 'pending',   -- pending | sent | expired
    target_driver_hashes VARCHAR[] NOT NULL,
    sent_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_broadcast_waves_batch_id ON broadcast_waves(batch_id);

-- 8. OUTBOX_MESSAGES (driver messages waiting to be sent) ----------------
-- Written in the same transaction as the change it reports (an offer, a
-- claim, a reject), so it exists if and only if that change committed. The
-- relay service sends pending rows in per-driver order and retries
-- transient failures. The recipient is the driver key; the relay looks up
-- the Telegram chat when it sends, so no messenger identifier is stored.
-- Sent rows are deleted after 24 h, failed ones after 14 days.
CREATE TABLE outbox_messages (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- per-driver send order
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,
    driver_hash VARCHAR NOT NULL,
    kind VARCHAR(30) NOT NULL,
    payload JSONB NOT NULL,                -- message text and buttons; for offers the offered task id
    status VARCHAR(10) NOT NULL DEFAULT 'pending',
    attempts INTEGER NOT NULL DEFAULT 0,
    next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ,                -- an offer not delivered by then fails
    counts_against_quota BOOLEAN NOT NULL DEFAULT false,  -- refunded if the message fails
    last_error TEXT,                       -- HTTP status and Telegram's description only
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ,

    CONSTRAINT ck_outbox_messages_status CHECK (status IN ('pending', 'sent', 'failed')),
    CONSTRAINT ck_outbox_messages_kind
        CHECK (kind IN ('offer', 'claim_confirmation', 'passenger_added', 'declined', 'removed_by_dispatcher'))
);

CREATE INDEX ix_outbox_messages_due ON outbox_messages(next_attempt_at) WHERE status = 'pending';
CREATE INDEX ix_outbox_messages_driver_pending ON outbox_messages(driver_hash, id) WHERE status = 'pending';
CREATE INDEX ix_outbox_messages_batch_id ON outbox_messages(batch_id);

-- 9. IDEMPOTENCY_KEYS (safe upload retries) ------------------------------
-- One row per tenant and Idempotency-Key. 'pending' while the owning
-- request runs (it holds an advisory lock on the key, so a crashed owner
-- can be told from a running one); 'done' with the response that a retry
-- gets back. Deleted after IDEMPOTENCY_KEY_TTL_HOURS (24 h).
CREATE TABLE idempotency_keys (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,
    fingerprint VARCHAR(64) NOT NULL,      -- SHA-256 of the file names, contents and form fields
    status VARCHAR(10) NOT NULL,
    owner UUID NOT NULL,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,
    response_status INTEGER,
    response_body JSONB,                   -- batch summary and warnings, no vehicle data
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT uq_idempotency_keys_tenant_key UNIQUE (tenant_id, key),
    CONSTRAINT ck_idempotency_keys_status CHECK (status IN ('pending', 'done'))
);

CREATE INDEX ix_idempotency_keys_expires_at ON idempotency_keys(expires_at);
CREATE INDEX ix_idempotency_keys_batch_id ON idempotency_keys(batch_id);

-- 10. UPLOADED_FILE_HASHES (duplicate upload warning) --------------------
-- SHA-256 of each uploaded file's bytes, so an upload can warn that the
-- same content already went into another batch of the tenant. Deleted
-- after DUPLICATE_UPLOAD_WINDOW_DAYS (30 days).
CREATE TABLE uploaded_file_hashes (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    sha256 VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_uploaded_file_hashes_tenant_sha256 ON uploaded_file_hashes(tenant_id, sha256);
CREATE INDEX ix_uploaded_file_hashes_batch_id ON uploaded_file_hashes(batch_id);
CREATE INDEX ix_uploaded_file_hashes_created_at ON uploaded_file_hashes(created_at);

-- 11. MANIFEST_FILE_RESULTS (worker progress per scanned file) -----------
-- One row per stored scan of a batch being read, written by the upload in
-- the same transaction as the batch and its queue job. After a worker
-- restart, files already read are not read again and the interrupted one
-- is retried (it fails after PARSE_MAX_ATTEMPTS). A working copy, not a
-- record: deleted when the batch reaches its final state.
CREATE TABLE manifest_file_results (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    file_key VARCHAR NOT NULL UNIQUE,      -- random storage key; file bytes are never stored in PostgreSQL
    file_name TEXT NOT NULL,
    leg INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL,           -- queued | started | ok | low_confidence | failed
    attempts INTEGER NOT NULL DEFAULT 0,
    result JSONB,                          -- the parser's output, until the batch is final
    confidence DOUBLE PRECISION,
    error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT ck_manifest_file_results_status
        CHECK (status IN ('queued', 'started', 'ok', 'low_confidence', 'failed'))
);

CREATE INDEX ix_manifest_file_results_batch_id ON manifest_file_results(batch_id);

-- 12. TELEGRAM_UPDATES (webhook updates: at least once, decided at most once)
-- One row per webhook update. 'received' (attempts counted) in its own
-- commit before processing, while the request holds a PostgreSQL advisory
-- lock on the update (a free lock means the earlier request crashed);
-- 'done' in the handler's first commit, i.e. together with the decision
-- (claim, decline, passenger); 'failed' after TELEGRAM_UPDATE_MAX_ATTEMPTS
-- (5). No chat id or text. Deleted after TELEGRAM_UPDATE_RETENTION_HOURS (24).
CREATE TABLE telegram_updates (
    update_id BIGINT PRIMARY KEY,           -- Telegram's, per bot
    status VARCHAR(10) NOT NULL,            -- received | done | failed
    attempts INTEGER NOT NULL,
    last_error VARCHAR(200),                -- exception class name only
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ,

    CONSTRAINT ck_telegram_updates_status CHECK (status IN ('received', 'done', 'failed'))
);

CREATE INDEX ix_telegram_updates_received_at ON telegram_updates(received_at);
```

## Entity Relationship Summary

```
companies (tenant)
  ├─< tenant_drivers (tenant_id)                  -- the dispatcher's driver list, no phone numbers
  ├─< vehicles (tenant_id)                        -- REGISTRY: one row per physical vehicle
  │     └─< batch_vehicles (tenant_id, vehicle_id) -- one row per relocation TASK (leg); vehicle_id NOT unique
  └─< optimization_sessions (tenant_id)           -- parallel dispatcher streams
        ├─< bulk_batches (tenant_id, session_id)  -- one row per leg
        │     ├─< batch_vehicles (tenant_id, batch_id)        -- this batch's relocation tasks
        │     ├─< drivers (tenant_id, batch_id)               -- claim bookkeeping, zero-PII
        │     ├─< outbox_messages (tenant_id, batch_id?)      -- driver messages waiting to be sent
        │     ├─< idempotency_keys (tenant_id, batch_id?)     -- upload retries
        │     ├─< uploaded_file_hashes (tenant_id, batch_id)  -- duplicate upload warning
        │     ├─< manifest_file_results (tenant_id, batch_id) -- worker progress, until the batch is final
        │     └─< broadcast_waves (batch_id)                  -- not used by the dispatch flow yet
        └─< route_legs (tenant_id, session_id, batch_id?)  -- multi-leg segments

telegram_updates                                  -- standalone: webhook processing state, no tenant data
```

- `optimization_sessions` is the top-level parallel-stream grouping per tenant.
- `vehicles` (the registry) is reached from `batch_vehicles` via
  `vehicle_id`, independently of any one batch or session: the same
  vehicle can do leg 1 in one batch and leg 2 in another.
- `bulk_batches.session_id` is **nullable** (batches from before sessions)
  but **always set** by current upload flows.
- `route_legs.session_id` is **mandatory** (`NOT NULL`) — every segment
  belongs to exactly one session, and `route_legs.batch_id` links it to its
  batch. One leg exists per (batch, `sequence_order`).
- Deleting a batch cascades to its tasks, driver rows, waves, route legs,
  outbox messages, idempotency keys, file hashes and file results; the
  application also deletes registry vehicles nothing else uses, and the
  session once its last batch is gone.

## Route Leg State Machine (`route_legs.status`)

The `ck_route_legs_status` CHECK constraint allows five values; four are
used:

```
           every offered driver declines
   pending ─────────────────────────────▶ rejected
     │  ▲                                    │
     │  └──── dispatcher "Retry" ◀───────────┘
     │  ▲
     │  └──── dispatcher "Remove driver" (last driver removed)
     │
     ├─ a driver claims a vehicle ─────────▶ claimed_driver
     └─ a passenger joins (nobody claimed) ─▶ claimed_passenger
```

| Status | Set by |
|---|---|
| `pending` | leg creation; "Retry" (`POST /batches/{batch_id}/segments/{leg_id}/retry`, 409 while the leg has drivers); removing the last driver |
| `claimed_driver` | a vehicle claim on the leg — Telegram or `POST /jobs/claim`; `driver_phone` holds the driver key |
| `claimed_passenger` | a passenger joining a still-pending leg — Telegram or `POST /jobs/claim` with `mode: passenger`; passengers are listed in `co_driver_hashes` |
| `rejected` | the last offered driver declining (no vehicle taken) — Telegram or `POST /jobs/claim` with `mode: reject` |
| `passenger_offered` | allowed by the constraint, not set by the current code |

After a claim, the driver is offered the next relay leg, else an open
return-trip (backhaul) leg (destination → origin) in the **same**
`(tenant_id, session_id)`, as a normal vehicle offer.

"Remove driver" (`DELETE /batches/{batch_id}/segments/{leg_id}/driver`)
releases every vehicle the driver holds on the leg (back to `unassigned`),
notifies them, and keeps the leg `claimed_driver` if other drivers remain.


## Multi-Tenant Isolation Enforcement

1. **Column-level**: `tenant_id` is `NOT NULL` with a `FOREIGN KEY →
   companies(id) ON DELETE CASCADE` on every `TenantScopedMixin` table
   (listed under [Architecture Overview](#architecture-overview)).
2. **Query-level (automatic)**: a `do_orm_execute` SQLAlchemy event listener
   injects `with_loader_criteria(TenantScopedMixin, tenant_id == <current
   tenant>)` into every ORM `select`/`update`/`delete`, including
   relationship-joined loads — a forgotten `.where(tenant_id == ...)`
   cannot leak rows across tenants.
3. **Write-level (automatic)**: a `before_flush` listener verifies every
   pending INSERT/UPDATE of a `TenantScopedMixin` row has `tenant_id`
   matching the current request's tenant context, auto-stamping it if
   omitted and raising `CrossTenantWriteError` on mismatch.
4. **Endpoint-level (explicit, redundant)**: dispatcher-control endpoints
   additionally re-verify `(tenant_id, batch_id)` and `(tenant_id,
   batch_id, leg_id)` triples explicitly before returning or mutating any
   row, so tenant isolation does not depend solely on the global ORM hook.
5. **Webhook context (no bound tenant)**: inbound Telegram webhooks carry
   no JWT/tenant header, so the automatic ORM filter has nothing to bind.
   These code paths resolve `tenant_id` from the owning
   `bulk_batches`/`route_legs` row first (via a narrowly scoped
   `tenant_isolation_bypass()`), then apply that `tenant_id` as an
   explicit `WHERE` predicate on every subsequent query — the bypass never
   substitutes for isolation, it only unblocks the lookup that determines
   which tenant to isolate by.
6. **Fail-closed default**: any query against a `TenantScopedMixin` table
   executed with no tenant bound to the current context raises
   `MissingTenantContextError` rather than running unfiltered.
7. **Background processes:** the manifest worker (OCR pipeline, retention
   sweep) and the outbox relay run outside any request, inside
   `tenant_isolation_bypass()`, and write every row with the batch's or
   message's `tenant_id` passed explicitly. The bypass is the only
   sanctioned way to run without a bound tenant.
8. **Database-level:** `fk_batch_vehicles_batch_same_tenant` rejects a
   vehicle task whose batch belongs to another tenant.
