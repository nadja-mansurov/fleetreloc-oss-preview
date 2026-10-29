# FleetReloc API & Integration Protocols

> **Synthetic data:** all example values in this document (companies, addresses, vehicles, VINs, plates, phone numbers, hashes and IDs) are synthetic, generated for R&D demonstration only. See the synthetic data notice in the [README](../README.md).

The contracts between the dispatcher dashboard, the API, drivers' messengers
and the on-premise parsing pipeline — **as built**. Last checked against the
code: 2026-10-18. Base path: `/api/v1`. Planned work: see the Roadmap in the
[README](../README.md#roadmap).

```text
  Dispatcher dashboard (Next.js) ──Bearer (OIDC)──►  FastAPI  ──► PostgreSQL (tenant-filtered ORM)
        ▲   login: Keycloak (OIDC, PKCE)              │  ▲      ──► Redis (offers, relay chain, dedupe)
        └──────────── polls every 10–20 s ────────────┘  │      ──► Ollama + Tesseract/OpenCV (on-prem)
  Drivers: Telegram bot ──webhook (secret header)────────┤      ──► Nominatim (geocoding), OSRM / Mapbox (routing)
           WhatsApp via Bird ──webhook (shared secret)───┘      ──► Telegram Bot API, Bird API (outbound messages)
```

---

## 1. Authentication

| Caller | Credentials | Resolved tenant |
|---|---|---|
| Dashboard user | `Authorization: Bearer <access token>` from the OIDC provider (Keycloak, authorization code + PKCE) | the token's tenant claim; `sub` = dispatcher id |
| Internal service | shared internal secret plus a tenant header (constant-time compare) | the tenant header |
| Telegram | Telegram's webhook secret-token header | from the offered task |
| Bird (WhatsApp) | shared webhook secret | from the offered task |

- **OIDC tokens** are verified against the provider's JWKS (RS256 only),
  with issuer, audience and expiry checks; keys are cached and refetched on
  rotation.
- **Every tenant endpoint requires credentials** — there is no default
  tenant. The tenant must exist in `companies` (403 otherwise). Every ORM
  query is then filtered by that tenant automatically.
- **Webhooks fail closed:** with the secret unset, every call is rejected.
- **Production** refuses to start with placeholder/short secrets or without a
  login method.

## 2. Conventions

- **Errors:** `{"detail": "<message>"}`. Unhandled errors return
  `500 {"detail": "Internal server error", "request_id": "<id>"}` — the
  traceback is logged under that id, never returned. Validation errors: 422
  (FastAPI format).
- **Status codes used:** 401 no/invalid credentials · 402 message quota used
  up · 403 unknown tenant / bad webhook token · 404 not found *for this
  tenant* · 409 state conflict (see each endpoint) · 411 body without
  Content-Length · 413 request/file too large · 422 invalid input ·
  503 OIDC provider unreachable.
- **Limits** (settings): request body `MAX_REQUEST_BYTES` 100 MB (checked from
  Content-Length before parsing); per file `MAX_UPLOAD_BYTES` 40 MB; files per
  upload `MAX_UPLOAD_FILES` 20; rows/vehicles per file `MAX_SPREADSHEET_ROWS`
  5000; unpacked `.xlsx` `MAX_SPREADSHEET_UNCOMPRESSED_BYTES` 100 MB plus a
  zip-bomb ratio check; phones per broadcast `MAX_BROADCAST_DRIVERS` 50.
- **Zero-PII:** phone numbers only enter request *bodies* (never URLs) and
  are reduced to `driver_hash` (keyed HMAC-SHA256) plus a masked display
  form; access logs redact secret and phone values.

## 3. Endpoints (dashboard / internal)

All require credentials (section 1).

### Sessions and batches

| Method & path | Does |
|---|---|
| `GET /sessions/` | the tenant's dispatch sessions (`OptimizationSession`) |
| `GET /sessions/{session_id}/batches` | the session's batches — one per leg; 404 if not the tenant's |
| `POST /batches/upload` | upload manifests (below) → `202` + `BulkBatchRead` |
| `GET /batches/` | the tenant's batches |
| `GET /batches/{batch_id}` | batch with `payload` (vehicles, waypoints), `drivers` (hash, masked phone, status), `paths`, `routing_provider`, `stats` |
| `PUT /batches/{batch_id}/payload` | manifest review editor, "Save & approve" (below) |
| `POST /batches/{batch_id}/broadcast` | offer the leg's unclaimed vehicles to drivers (below) |
| `POST /batches/{batch_id}/next-wave` | same, for the next wave of drivers |
| `GET /batches/{batch_id}/segments` | the batch's route legs with status and driver (hash) |
| `POST /batches/{batch_id}/segments/{leg_id}/retry` | reopen a leg every driver declined (`pending`, driver cleared); 409 if the leg has drivers |
| `DELETE /batches/{batch_id}/segments/{leg_id}/driver` | remove the leg's driver: their vehicles go back to `unassigned`, they are told, dashboard status `removed` |

**Upload** — `multipart/form-data`:

| Field | |
|---|---|
| `files` | one or more of `.json .xlsx .xls .csv .pdf .png .jpg .jpeg` |
| `sequence_orders` | optional, one leg number per file (one batch per leg) |
| `sequence_order` | optional leg number for all files |
| `session_id` | optional: add to this session (404 if not the tenant's) |
| `batch_id` | optional: append to this batch (409 once offers were sent or while processing) |
| `region` | optional OCR language profile |

Structured files (JSON, Excel, CSV) are imported in the request: vehicles
are geocoded and routed, the batch is `ready`. PDFs and images return at once
with status `processing`; OCR → LLM → geocoding → routing runs in the
background. Clients poll `GET /batches/{batch_id}` (the dashboard every 20 s).

**Batch status:**

| Status | Meaning | Offers / claims |
|---|---|---|
| `processing` | OCR/LLM pipeline running | blocked (409) |
| `manual_review` | parsed with low confidence; needs "Save & approve" | blocked (409) |
| `failed` | pipeline failed | blocked (409) |
| `ready` | parsed, can be dispatched | allowed |
| `broadcasting` | offers sent, not every vehicle claimed | allowed |
| `completed` | every vehicle claimed | — |

**Payload editor** (`PUT /batches/{batch_id}/payload`, body
`{"vehicles": [...]}`, same count and order as the batch): only `vin`,
`model`, `plate`, `origin`, `originTitle`, `destination`,
`destinationTitle` can change. Edits reach the vehicle tasks (changed
addresses are re-geocoded and re-routed). 409 while `processing` or once
offers were sent. A `manual_review` batch becomes `ready`.

**Broadcast / next wave** — body `{"driver_phones": ["+49…", …]}` (1–50,
normalized, duplicates dropped):
- 409 if the batch isn't dispatchable or no vehicle on the leg is unclaimed.
- Drivers are spread over the unclaimed vehicles round-robin; each offer is
  a Telegram message with `[✅ Claim Driving]`, `[👥 Claim as Passenger]`
  (leg 2+), `[❌ Reject]`, valid for 1 hour.
- Offers are subject to the tenant's message quota.
- Response: `offered`, `pending`, `unreachable` (not registered with the
  bot / not delivered), `ineligible` (hasn't cleared the previous leg),
  `quota_exceeded`, `vehicles`; 402 if the quota stopped every offer. The same
  statuses appear per driver in `GET /batches/{batch_id}` → `drivers`.

### Vehicles, offers, claims

| Method & path | Does |
|---|---|
| `GET /vehicles/?limit=&offset=` | vehicle tasks, newest first; `limit` 1–500 (default 100) |
| `GET /vehicles/{vehicle_id}` | one vehicle task |
| `POST /vehicles/` | create a task; with `batch_id` the batch must be the tenant's (404) and accept vehicles (409); it goes through the upload ingestion (geocoded, routed, added to the batch) |
| `POST /orders/{vehicle_id}/send-offer` | body `{"driver_phone": "…"}`: offer one vehicle through `NOTIFICATION_CHANNEL` (Telegram or Bird); 402 quota, 409 if taken or the batch needs review |
| `POST /jobs/claim` | body `{"batch_id", "phone_number"}`: claim the first unclaimed vehicle of the batch's leg for that driver; 409 if the batch isn't dispatchable, the driver hasn't cleared the previous leg, or nothing is left |
| `POST /vrp/batches/{batch_id}/optimize-fleet` | real-road routes for the batch's vehicles |
| `POST /vrp/recalculate-driver-vector` | route metrics for a driver and co-drivers on a batch |

**Claiming is atomic:** one conditional `UPDATE … WHERE status = 'unassigned'`
per vehicle. When two drivers claim the
same vehicle, exactly one wins; the same driver claiming twice is a no-op.
No Redis lock is used: the database row is the single source of truth, so
there is no lock to expire or get out of sync with the data.
Vehicle task statuses: `unassigned`, `locked`, `assigned`, `in_transit`,
`delivered`, `cancelled` (database CHECK constraint).

## 4. Driver messaging

### 4.1 Telegram — `POST /telegram/webhook`

Always answers 200 (a 5xx would make Telegram redeliver); each `update_id` is
processed once.

| Update | Effect |
|---|---|
| `/start` | asks the driver to share their phone number (contact button) |
| contact message | registers the chat — only the sender's **own** contact (`contact.user_id == from.id`); only the `driver_hash` is stored |
| **Drive** button or text such as "CLAIM" | claim the offered vehicle (atomic); on leg 2+ with every vehicle taken, the driver rides along as passenger |
| **Passenger** button | ride along as passenger (leg 2+, only if someone drives) |
| **Reject** button or a reject word | decline; the leg becomes `rejected` once every offered driver declined |

A button from an older offer (another batch) doesn't act on the current
offer. After a claim the driver gets the next relay leg's offer, otherwise a
return-trip (backhaul) leg in the same session, if one is open.

### 4.2 WhatsApp via Bird — `POST /webhooks/whatsapp/inbound`

Inbound Bird message events from drivers are matched to the driver's
pending WhatsApp offer (valid 1 hour). Replies `CLAIM` / `REJECT` act on
that offer; anything else gets the help text. Offers are sent as a Bird
template message (`BIRD_TEMPLATE_NAME`). WhatsApp currently handles single
offers; batch broadcasts use Telegram.

## 5. Route legs (`route_legs.status`)

One route leg per leg number of a batch; every leg
belongs to a session, and backhaul matching never crosses sessions or
tenants.

| Status | Set when |
|---|---|
| `pending` | created / reopened (retry, last driver removed) |
| `claimed_driver` | a driver claims a vehicle of the leg (Telegram or `/jobs/claim`) |
| `claimed_passenger` | a passenger joins a pending leg (`co_driver_hashes`) |
| `rejected` | every driver offered the leg declined |
| `passenger_offered` | allowed by the CHECK constraint, not set by the current code |

**Relay order:** a driver is offered (and may claim) leg N ≥ 2 only after
claiming or declining leg N − 1 of the same session.

## 6. Manifest parsing (on-premise)

1. **OCR:** images preprocessed with OpenCV (grayscale,
   `fastNlMeansDenoising`, Otsu binarization; the original image is used if
   that fails), then Tesseract with `OCR_LANGUAGES` (`eng+deu+ara`, fallback
   `OCR_FALLBACK_LANGUAGE`); region profiles via the upload's `region`.
2. **Extraction:** a local LLM (Ollama, `OLLAMA_MODEL` default `llama3`,
   timeout `OLLAMA_TIMEOUT_SECONDS` 120 s) returns JSON validated against
   the `BulkOrderManifest` schema: vehicles with
   VIN, model, plate, structured pickup/dropoff places (`place_name`,
   `place_type`, `city`, `country`), time windows, plus a `confidence` (0–1).
   Invalid output is sent back to the model with its validation errors for
   a bounded number of retries.
3. **Safeguards:** deterministic validation rules and tiered address
   fallbacks keep malformed values out of routing; low `confidence` →
   `manual_review`.
4. **Geocoding:** Nominatim (`NOMINATIM_BASE_URL`; the public instance is
   limited to 1 request/s app-wide), results cached in Redis. **Routing:**
   OSRM (`OSRM_BASE_URL`) or Mapbox (`ROUTING_PROVIDER`), straight-line
   fallback.

## 7. Realtime

No push channel: the dashboard polls `GET /batches/{batch_id}` every 20 s
and `GET /batches/{batch_id}/segments` every 10 s.

---

## Appendix: target architecture (not built)

Earlier versions of this document described these as existing; they are
not part of the code: BullMQ worker queues, gRPC / a separate VRP solver
service, Redis Redlock claiming (claims are an atomic PostgreSQL update),
Supabase Auth/Realtime and WebSocket events, Meta WhatsApp Cloud API
webhooks (the app uses Bird), and error codes such as `ERR_REDLOCK_FAILED`.
Planned work is listed in the [README roadmap](../README.md#roadmap).
