# Database Schema & Order State Machine (Multi-Tenant / Zero-Trust Architecture)

## Architecture Overview

- **Multi-tenancy**: every tenant-owned table (`optimization_sessions`,
  `bulk_batches`, `vehicles`, `batch_vehicles`, `routes`, `route_legs`,
  `drivers`) carries a mandatory `tenant_id` column, `NOT NULL`, `FOREIGN
  KEY → companies.id ON DELETE CASCADE`, and declares `TenantScopedMixin`
  (see `apps/api/app/models/mixins.py`). An automatic SQLAlchemy event
  listener (`apps/api/app/db/tenant_filter.py`) injects `tenant_id ==
  <current tenant>` into **every** ORM `select`/`update`/`delete` against
  these tables, so a developer forgetting an explicit filter can no longer
  leak cross-tenant rows. Queries with no tenant bound to the current
  context fail closed (`MissingTenantContextError`) rather than running
  unfiltered.
- **Normalized vehicle split (Multi-Leg Relay)**: a physical vehicle (VIN)
  and a specific relocation TASK for it are now two separate tables,
  fixing a bug where the old single flat `vehicles` table's
  `UNIQUE(vin)` constraint made it impossible for the same vehicle to
  participate in more than one leg/batch:
  - **`vehicles`** (the **global registry**) — one row per unique
    physical vehicle per tenant (`vin`, `make_model`, `license_plate`
    only). Enforced unique via a **partial** unique index,
    `uq_vehicles_tenant_vin_known`, on `(tenant_id, vin) WHERE vin <>
    'UNKNOWN'` — deliberately excluding the `"UNKNOWN"` placeholder VIN
    (see `apps/api/app/utils/vehicle_normalization.py::UNKNOWN_VIN`) so
    manifests with an unresolved VIN never collide with each other.
  - **`batch_vehicles`** (the **transactional task/link table**, the
    direct successor of the old flat `vehicles` table) — one row per
    relocation TASK: a specific leg, in a specific batch, with its own
    pickup/dropoff window, status, `driver_hash`, and routing metadata.
    `vehicle_id` (FK → `vehicles.id`) carries **no uniqueness
    constraint**, so the exact same physical vehicle can have many
    `batch_vehicles` rows — one per leg of a Multi-Leg Relay chain —
    without ever raising `UniqueViolationError`.
  - Every import path (Excel/CSV, OCR/LLM manifest pipeline, single-PDF
    orders, direct API `POST /api/v1/vehicles/`) resolves/creates the
    registry row via `VehicleRegistryService.get_or_create_vehicle`
    (`apps/api/app/services/vehicle_registry_service.py`, an atomic
    Postgres `INSERT ... ON CONFLICT (tenant_id, vin) WHERE vin <>
    'UNKNOWN' DO UPDATE` upsert) and then creates a fresh,
    unconstrained `batch_vehicles` row for the task.
- **Parallel optimization sessions**: `optimization_sessions` is the parent
  grouping that lets one dispatcher run two or more independent,
  concurrent routing/chaining streams (e.g. a Ketzin→Berlin stream and a
  Frankfurt→Hamburg stream) without cross-contamination. `bulk_batches`
  and `route_legs` both carry a `session_id`, and `route_legs` additionally
  enforces `session_id` as mandatory (`NOT NULL`) so every segment is
  strictly scoped to one stream.
- **Multi-file manifest uploads**: a single `optimization_sessions` row can
  own many `bulk_batches` rows (one per uploaded file/manifest) via the
  `bulk_batches.session_id` foreign key, allowing concurrent multi-file
  uploads to be grouped and tracked together while each file's own
  import/parse outcome (`status`, `successful_rows`, `failed_rows`,
  `payload`) remains independently auditable per batch.
- **Route leg state machine**: `route_legs.status` is constrained by a
  Postgres `CHECK` constraint (`ck_route_legs_status`) to exactly:
  `pending → claimed_driver | rejected`, and `pending → passenger_offered →
  claimed_passenger` for round-trip backhaul chaining offered after a
  driver claims the outbound leg. See the state diagram in
  [Route Leg State Machine](#route-leg-state-machine-route_legsstatus) below.
- **Zero-PII**: no table in this schema ever stores a raw phone number.
  `batch_vehicles.driver_hash`, `drivers.driver_hash`, and
  `route_legs.driver_phone` store only an HMAC-SHA256 hash derived via
  `apps/api/app/core/security.py::generate_driver_hash`. Raw phone numbers
  live transiently (24h TTL) only in Redis — see
  `apps/api/app/services/driver_session_service.py`.

## PostgreSQL Schema (Live DDL, reconstructed from `information_schema`)

```sql
-- 0. COMPANIES (Tenants) -----------------------------------------------
-- Every tenant-scoped table below has a mandatory FK into this table.
CREATE TABLE companies (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    stripe_customer_id TEXT,
    stripe_subscription_id TEXT,
    plan_tier TEXT NOT NULL DEFAULT 'free',          -- free | starter | pro
    messages_used INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 1. OPTIMIZATION_SESSIONS (Parallel dispatcher streams) -----------------
-- Parent grouping allowing a dispatcher to run 2+ independent, concurrent
-- routing/chaining streams without cross-contaminating their batches or
-- route legs. See apps/api/app/models/session.py.
CREATE TABLE optimization_sessions (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    dispatcher_id UUID,
    name TEXT,
    status VARCHAR NOT NULL DEFAULT 'active',        -- active | completed | cancelled | archived
    payload JSONB,                                   -- freeform session-level metadata (backward-compat bag)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_optimization_sessions_tenant_id ON optimization_sessions(tenant_id);
CREATE INDEX ix_optimization_sessions_tenant_status ON optimization_sessions(tenant_id, status);

-- 2. BULK_BATCHES (Manifest Imports — one row per uploaded file) --------
-- Multi-file manifest uploads: many bulk_batches rows (one per file) can
-- belong to the SAME optimization_sessions row via session_id, letting a
-- dispatcher upload several files concurrently into one grouped session
-- while each file's parse/import outcome stays independently tracked.
CREATE TABLE bulk_batches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    -- Nullable for backward compatibility with batches created before the
    -- OptimizationSession concept existed; new dispatcher flows always set
    -- this to isolate parallel streams from each other.
    session_id UUID REFERENCES optimization_sessions(id) ON DELETE SET NULL,
    -- Multi-Leg Relay: this batch's strict leg position (1, 2, 3...)
    -- within its session's chained-relocation sequence. Drives sequential
    -- wave broadcasting (Step 2 only offered to drivers who cleared Step
    -- 1, etc.) and the dispatcher-facing "Leg N" label in the UI.
    sequence_order INTEGER NOT NULL DEFAULT 1,
    dispatcher_id UUID,
    file_name TEXT NOT NULL,
    total_vehicles INTEGER NOT NULL,
    successful_rows INTEGER NOT NULL DEFAULT 0,
    failed_rows INTEGER NOT NULL DEFAULT 0,
    claimed_count INTEGER NOT NULL DEFAULT 0,
    status VARCHAR NOT NULL DEFAULT 'processing',    -- processing | ready | broadcasting | completed | expired | manual_review
    payload JSONB,                                   -- parsed vehicles/waypoints/locations/drivers (OCR/manifest output)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_bulk_batches_tenant_id ON bulk_batches(tenant_id);
CREATE INDEX ix_bulk_batches_tenant_status ON bulk_batches(tenant_id, status);
CREATE INDEX ix_bulk_batches_tenant_session ON bulk_batches(tenant_id, session_id);
CREATE INDEX ix_bulk_batches_tenant_session_sequence ON bulk_batches(tenant_id, session_id, sequence_order);

-- 3a. VEHICLES (Global Vehicle Registry -- one row per unique physical
-- vehicle per tenant) -----------------------------------------------------
-- Holds ONLY immutable/slow-changing attributes. The real uniqueness
-- guarantee is the PARTIAL unique index below: unique per (tenant_id,
-- vin), but explicitly excluding the "UNKNOWN" placeholder VIN so
-- unresolved-VIN vehicles never collide with each other. This replaces
-- the OLD `vehicles_vin_key` GLOBAL UNIQUE(vin) constraint, which made it
-- impossible for the same vehicle to appear in more than one leg/batch.
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
-- THE fix: unique per (tenant, vin), but only for KNOWN vins.
CREATE UNIQUE INDEX uq_vehicles_tenant_vin_known
    ON vehicles (tenant_id, vin)
    WHERE vin <> 'UNKNOWN';

-- 3b. BATCH_VEHICLES (Transactional relocation TASKS, Zero-Data-Retention
-- State) -- one row per (vehicle, leg, batch) combination -----------------
-- Direct successor of the pre-refactor flat `vehicles` table (identical
-- pickup/dropoff/status/driver_hash/routing columns) -- only the name
-- changed and vin/make_model/license_plate became FK + denormalized
-- read-copies of the registry row (`vehicles` above) instead of the
-- source of truth. `vehicle_id` carries NO uniqueness constraint: the
-- same physical vehicle can have many `batch_vehicles` rows (one per
-- leg/batch in a Multi-Leg Relay chain).
CREATE TABLE batch_vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE CASCADE,
    vehicle_id UUID NOT NULL REFERENCES vehicles(id) ON DELETE CASCADE,
    sequence_order INTEGER NOT NULL DEFAULT 1,  -- Multi-Leg Relay: strict leg order (1, 2, 3...)

    -- Denormalized read-copies of the registry row, snapshotted at task
    -- creation time -- the registry row remains the source of truth.
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

    -- State & Hashed/Anonymized Assignment (Zero Trust: no raw phone/name stored)
    status VARCHAR DEFAULT 'unassigned'
        CHECK (status IN ('unassigned', 'locked', 'assigned', 'in_transit', 'delivered', 'cancelled')),
    driver_hash VARCHAR,     -- HMAC-SHA256 hashed phone/session token (Zero PII retention)

    locked_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Real-road routing metadata (Mapbox/OSRM; replaces straight-line
    -- visualization). route_provider == "fallback" flags a degraded
    -- straight-line approximation when every real provider was unreachable.
    distance_km DOUBLE PRECISION,
    duration_minutes DOUBLE PRECISION,
    route_geometry JSONB,          -- ordered [lon, lat] polyline (GeoJSON LineString convention)
    route_provider VARCHAR         -- "mapbox" | "osrm" | "fallback"
);

CREATE INDEX ix_batch_vehicles_tenant_id ON batch_vehicles(tenant_id);
CREATE INDEX ix_batch_vehicles_tenant_batch ON batch_vehicles(tenant_id, batch_id);
CREATE INDEX ix_batch_vehicles_tenant_vehicle ON batch_vehicles(tenant_id, vehicle_id);
CREATE INDEX ix_batch_vehicles_tenant_status ON batch_vehicles(tenant_id, status);

-- 4. ROUTES (Computed/optimized route per batch) ------------------------
CREATE TABLE routes (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    assigned_driver_hash VARCHAR,   -- driver_hash, never a raw phone number
    distance_km DOUBLE PRECISION,
    duration_minutes DOUBLE PRECISION,
    waypoints JSONB,
    status VARCHAR NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_routes_tenant_id ON routes(tenant_id);
CREATE INDEX ix_routes_tenant_batch ON routes(tenant_id, batch_id);

-- 5. ROUTE_LEGS (Multi-leg route segments + state machine) --------------
-- A single segment of a (possibly multi-leg/chained) route, e.g. an
-- outbound Ketzin->Berlin leg and its round-trip Berlin->Ketzin backhaul
-- leg. Mandatory tenant_id AND session_id: every leg is strictly isolated
-- to one tenant and one parallel dispatcher stream.
CREATE TABLE route_legs (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    session_id UUID NOT NULL REFERENCES optimization_sessions(id) ON DELETE CASCADE,
    batch_id UUID REFERENCES bulk_batches(id) ON DELETE SET NULL,
    route_id UUID REFERENCES routes(id) ON DELETE SET NULL,

    origin VARCHAR NOT NULL,
    destination VARCHAR NOT NULL,

    -- Multi-Leg Relay: mirrors the owning bulk_batches.sequence_order so
    -- this leg's chain position survives even if batch_id is later
    -- nulled out (ON DELETE SET NULL above).
    sequence_order INTEGER NOT NULL DEFAULT 1,

    -- Route leg state machine -- see state diagram below.
    status VARCHAR NOT NULL DEFAULT 'pending'
        CONSTRAINT ck_route_legs_status
        CHECK (status IN ('pending', 'claimed_driver', 'rejected', 'passenger_offered', 'claimed_passenger')),

    -- Zero-PII: driver_hash only, never a raw phone number.
    driver_phone VARCHAR,

    -- Co-driver / Passenger mode (Multi-Leg Relay): driver_hash values of
    -- drivers who opted in as passengers riding along in an
    -- already-assigned vehicle for this leg, once its primary-vehicle
    -- pool was exhausted. Zero-PII: driver_hash only.
    co_driver_hashes VARCHAR[] NOT NULL DEFAULT '{}',

    -- Real-road routing metadata (same convention as vehicles above).
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

-- 6. DRIVERS (FCFS claim state per batch, Zero-PII) ----------------------
CREATE TABLE drivers (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    batch_id UUID NOT NULL REFERENCES bulk_batches(id) ON DELETE CASCADE,
    driver_hash VARCHAR NOT NULL,   -- HMAC-SHA256, never a raw phone number
    status VARCHAR NOT NULL DEFAULT 'pending',  -- pending | claimed | expired | cancelled
    claimed_at TIMESTAMPTZ,
    expired_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_drivers_tenant_batch_hash UNIQUE (tenant_id, batch_id, driver_hash)
);

CREATE INDEX ix_drivers_tenant_id ON drivers(tenant_id);
CREATE INDEX ix_drivers_tenant_batch ON drivers(tenant_id, batch_id);
CREATE INDEX ix_drivers_tenant_batch_status ON drivers(tenant_id, batch_id, status);

-- 7. BROADCAST_WAVES (Iterative driver broadcast tracking) --------------
-- NOT tenant-scoped at the column level; isolation is transitive via
-- batch_id -> bulk_batches.tenant_id. Stores ONLY driver_hash identifiers.
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
```

## Entity Relationship Summary

```
companies (tenant)
  ├─< vehicles (tenant_id)                        -- GLOBAL REGISTRY: one row per unique physical vehicle
  │     └─< batch_vehicles (tenant_id, vehicle_id) -- one row per relocation TASK (leg); vehicle_id NOT unique
  └─< optimization_sessions (tenant_id)           -- parallel dispatcher streams
        ├─< bulk_batches (tenant_id, session_id)  -- one row per uploaded file/manifest
        │     ├─< batch_vehicles (tenant_id, batch_id)  -- this batch's relocation tasks
        │     ├─< routes (tenant_id, batch_id)
        │     ├─< drivers (tenant_id, batch_id)   -- FCFS claim bookkeeping, zero-PII
        │     └─< broadcast_waves (batch_id)
        └─< route_legs (tenant_id, session_id, batch_id?, route_id?)  -- multi-leg segments
```

- `optimization_sessions` is the top-level parallel-stream grouping per tenant.
- `vehicles` (the global registry) is reachable from `batch_vehicles` via
  `vehicle_id`, independently of any one `bulk_batches`/`optimization_sessions`
  row -- the same registry vehicle can be referenced by `batch_vehicles`
  rows belonging to many different batches/sessions over time (Multi-Leg
  Relay: the same physical vehicle doing leg 1 in one batch, then leg 2 in
  another).
- `bulk_batches.session_id` is **nullable** (backward compatibility with
  pre-session batches) but **always set** by current upload flows.
- `route_legs.session_id` is **mandatory** (`NOT NULL`) — every segment
  belongs to exactly one session, and `route_legs.batch_id` /
  `route_legs.route_id` are optional links back to the originating file
  and computed route respectively (`ON DELETE SET NULL`, so a leg survives
  batch/route cleanup for audit purposes).

## Route Leg State Machine (`route_legs.status`)

Enforced at the database level by the `ck_route_legs_status` CHECK
constraint — only these five values are ever valid:

```
                 ┌──────────┐
   driver claim  │          │  dispatcher "Retry"
   ┌────────────▶│ pending  │◀────────────────────────────┐
   │             │          │                              │
   │             └────┬─────┘                              │
   │                  │                                    │
   │       driver claims outbound leg                      │
   │                  │                                    │
   │                  ▼                                    │
   │          ┌────────────────┐        driver rejects     │
   │          │ claimed_driver │───────────────────────────┤
   │          └───────┬────────┘                           │
   │                  │                                    │
   │     round-trip backhaul auto-offered                  │
   │     (same tenant_id + session_id only)                │
   │                  ▼                                    │
   │       ┌────────────────────┐                          │
   │       │ passenger_offered  │                           │
   │       └──────────┬─────────┘                          │
   │                  │                                    │
   │       passenger/driver claims backhaul                │
   │                  ▼                                    │
   │       ┌─────────────────────┐                         │
   └───────│ claimed_passenger   │                          │
           └─────────────────────┘                         │
                                                             │
                 ┌──────────┐                               │
                 │ rejected │───────────────────────────────┘
                 └──────────┘   dispatcher "Retry" resets to pending
```

- **`pending`** — default state on leg creation; eligible for FCFS
  claiming or backhaul offering.
- **`claimed_driver`** — set when a driver claims the outbound leg via the
  Telegram webhook (`apps/api/app/api/v1/endpoints/telegram.py`), strictly
  scoped to `(tenant_id, batch_id)`; triggers an automatic lookup for a
  matching round-trip backhaul leg within the **same** `(tenant_id,
  session_id)`.
- **`rejected`** — set when a driver explicitly declines the leg via the
  Telegram webhook; a dispatcher can reset it back to `pending` via
  `POST /api/v1/batches/{batch_id}/segments/{leg_id}/retry`.
- **`passenger_offered`** — set automatically on the matching backhaul leg
  (destination → origin) once the outbound leg is claimed, so it enters
  the round-trip chaining offer pool for a return passenger/driver.
- **`claimed_passenger`** — terminal state once the backhaul leg itself is
  claimed.
- Dispatcher manual controls (`apps/api/app/api/v1/endpoints/batches.py`):
  - `POST /api/v1/batches/{batch_id}/segments/{leg_id}/retry` — resets a
    leg to `pending` and clears `driver_phone`, strictly verified against
    `(tenant_id, batch_id, leg_id)` before mutation.
  - `DELETE /api/v1/batches/{batch_id}/segments/{leg_id}/driver` —
    unassigns the current `driver_phone` and resets to `pending`, under
    the same strict tenant/batch/leg verification.


## Multi-Tenant Isolation Enforcement

1. **Column-level**: `tenant_id` is `NOT NULL` with a `FOREIGN KEY →
   companies(id) ON DELETE CASCADE` on every `TenantScopedMixin` table
   (`optimization_sessions`, `bulk_batches`, `vehicles`, `batch_vehicles`,
   `routes`, `route_legs`, `drivers`).
2. **Query-level (automatic)**: `apps/api/app/db/tenant_filter.py` installs
   a `do_orm_execute` SQLAlchemy event listener that injects
   `with_loader_criteria(TenantScopedMixin, tenant_id == <current
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
   row (`_get_tenant_scoped_batch` / `_get_tenant_scoped_leg` in
   `apps/api/app/api/v1/endpoints/batches.py`), so tenant isolation does
   not depend solely on the global ORM hook.
5. **Webhook context (no bound tenant)**: inbound Telegram webhooks carry
   no JWT/tenant header, so the automatic ORM filter has nothing to bind.
   These code paths explicitly resolve `tenant_id` from the owning
   `bulk_batches`/`route_legs` row first (via a narrowly-scoped
   `tenant_isolation_bypass()`), then apply that `tenant_id` as an
   explicit `WHERE` predicate on every subsequent query — the bypass never
   substitutes for isolation, it only unblocks the lookup that determines
   which tenant to isolate by. The legacy WhatsApp booking flow
   (`apps/api/app/services/booking_service.py::claim_vehicle`) follows the
   same pattern for `batch_vehicles` lookups, since it predates
   multi-tenancy and has no `tenant_id` of its own to scope by.
6. **Fail-closed default**: any query against a `TenantScopedMixin` table
   executed with no tenant bound to the current context raises
   `MissingTenantContextError` rather than running unfiltered.
7. **Cron/background jobs (explicit, audited bypass)**:
   `BookingExpirationService.release_expired_locks` intentionally releases
   expired `batch_vehicles` locks **across every tenant** in one pass (it
   is not a per-request operation), so it wraps its query in
   `tenant_isolation_bypass()` rather than running under one tenant's
   context — the only sanctioned way to perform a genuinely cross-tenant
   operation.
