# FleetReloc API & Integration Protocols

> **Synthetic data:** all example values in this document (companies, addresses, vehicles, VINs, plates, phone numbers, hashes and IDs) are synthetic, generated for R&D demonstration only. See the synthetic data notice in the [README](../README.md).

The contracts between the dispatcher dashboard, the API, drivers' messengers
and the on-premise parsing pipeline — **as built**. Last checked against the
code: 2026-11-11. Base path: `/api/v1`. Planned work: see
[Planned](../README.md#planned) in the README.

```text
  Dispatcher dashboard (Next.js) ──Bearer (OIDC)──►  FastAPI  ──► PostgreSQL (tenant-filtered ORM)
        ▲   login: Keycloak (OIDC, PKCE)              │  ▲      ──► Redis (offers, relay chain, dedupe)
        └──────────── polls every 10–20 s ────────────┘  │      ──► RapidOCR + Ollama (on-prem)
  Drivers: Telegram bot ──webhook (secret header)────────┤      ──► Nominatim (geocoding), OSRM / Mapbox (routing)
           WhatsApp via Bird (demo) ──webhook (secret)───┘      ──► Telegram Bot API, Bird API (outbound messages)
```

---

## 1. Authentication

| Caller | Credentials | Resolved tenant |
|---|---|---|
| Dashboard user | `Authorization: Bearer <access token>` from the OIDC provider (Keycloak, authorization code + PKCE) | the token's tenant claim; `sub` = dispatcher id |
| Internal service | shared internal secret plus a tenant header (constant-time compare) | the tenant header |
| Telegram | Telegram's webhook secret-token header | from the offered task |
| Bird (WhatsApp, demo) | shared webhook secret | from the offered task |

- **OIDC tokens** are verified against the provider's JWKS (RS256 only),
  with issuer, audience and expiry checks; keys are cached and refetched on
  rotation.
- **Every tenant endpoint requires credentials** — there is no default
  tenant. The tenant must exist in `companies` (403 otherwise). Every ORM
  query is then filtered by that tenant automatically.
- **Webhooks fail closed:** with the secret unset, every call is rejected.
- **Production** refuses to start with placeholder/short secrets or without a
  login method.
- **Rate limits** apply to uploads and scans (section 2); there is no general
  API rate limit yet.

## 2. Conventions

- **Errors:** `{"detail": "<message>"}`. Unhandled errors return
  `500 {"detail": "Internal server error", "request_id": "<id>"}` — the
  traceback is logged under that id, never returned. Validation errors: 422
  (FastAPI format).
- **Status codes used:** 401 no/invalid credentials · 402 message quota used
  up · 403 unknown tenant / bad webhook token · 404 not found *for this
  tenant* · 409 state conflict (see each endpoint) · 411 body without
  Content-Length · 413 request/file too large · 422 invalid input ·
  429 too many requests (`Retry-After` header) · 503 OIDC provider
  unreachable.
- **Limits** (settings): request body `MAX_REQUEST_BYTES` 100 MB (checked from
  Content-Length before parsing); per file `MAX_UPLOAD_BYTES` 40 MB; files per
  upload `MAX_UPLOAD_FILES` 20; rows/vehicles per file `MAX_SPREADSHEET_ROWS`
  5000; unpacked `.xlsx` `MAX_SPREADSHEET_UNCOMPRESSED_BYTES` 100 MB plus a
  zip-bomb ratio check; phones per broadcast `MAX_BROADCAST_DRIVERS` 50.
  Scanned manifests (OCR + LLM are CPU-heavy): pages per PDF `MAX_PDF_PAGES`
  30 and pixels per image `MAX_IMAGE_PIXELS` 40 M (422, read from the file
  header at upload and checked again before OCR); scanned files waiting to
  be read per tenant, including the upload being made,
  `MAX_QUEUED_SCANS_PER_TENANT` 20 (429; a multi-leg order fits in one
  upload); manifests read at once per API process `MAX_PARALLEL_PARSE_JOBS`
  2 (later ones wait); upload requests per tenant per hour
  `UPLOADS_PER_TENANT_PER_HOUR` 30 (429). File formats and sizes are checked
  before any database work.
- **Zero-PII:** phone numbers only enter request *bodies* (never URLs) and
  are reduced to `driver_hash` (keyed HMAC-SHA256) plus a masked display
  form; access logs redact secret and phone values. The dispatcher's driver
  list (section 3) holds names or aliases only, no phone numbers.

## 3. Endpoints (dashboard / internal)

All require credentials (section 1).

### Sessions and batches

| Method & path | Does |
|---|---|
| `GET /sessions/` | the tenant's dispatch sessions (`OptimizationSession`) |
| `GET /sessions/{session_id}/batches` | the session's batches — one per leg; 404 if not the tenant's |
| `POST /batches/upload` | upload manifests (below) → `202` + `BulkBatchRead` |
| `GET /batches/` | the tenant's batches |
| `GET /batches/{batch_id}` | batch with `payload` (vehicles, waypoints), `drivers` (hash, masked phone or name, status), `paths`, `routing_provider`, `stats` |
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
| `region` | optional OCR language profile: `eu` (Latin) or `gcc` (Arabic) |

Structured files (JSON, Excel, CSV) are imported in the request: vehicles
are geocoded and routed, the batch is `ready`. PDFs and images return at once
with status `processing`; OCR → LLM → geocoding → routing runs in the
background. Clients poll `GET /batches/{batch_id}` (the dashboard every 20 s).

**Batch status:**

| Status | Meaning | Offers / claims |
|---|---|---|
| `processing` | OCR/LLM pipeline running | blocked (409) |
| `manual_review` | the extraction needs a person: low LLM confidence (< 0.80) or a failed check (`payload.review_reasons`, see section 6); needs "Save & approve" | blocked (409) |
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
- Each offer is counted against the plan's message quota before sending and
  refunded if not delivered.
- Response: `offered`, `pending`, `unreachable` (not registered with the
  bot / not delivered), `ineligible` (hasn't cleared the previous leg),
  `quota_exceeded`, `vehicles`; 402 if the quota stopped every offer. The same
  statuses appear per driver in `GET /batches/{batch_id}` → `drivers`.

### Vehicles, drivers, offers, claims

| Method & path | Does |
|---|---|
| `GET /vehicles/?limit=&offset=` | vehicle tasks, newest first; `limit` 1–500 (default 100) |
| `GET /vehicles/{vehicle_id}` | one vehicle task |
| `POST /vehicles/` | create a task; with `batch_id` the batch must be the tenant's (404) and accept vehicles (409); it goes through the upload ingestion (geocoded, routed, added to the batch) |
| `GET /drivers`, `POST /drivers`, `PATCH /drivers/{driver_id}`, `DELETE /drivers/{driver_id}` | the dispatcher's driver list: body `{"display_name", "language": "de"\|"ar"\|"en"}`; no phone numbers (a name with 7+ digits is rejected, 422); names unique per tenant (409); delete replaces the name in batch driver lists with "Removed driver" |
| `POST /orders/{vehicle_id}/send-offer` | body `{"driver_phone": "…"}`: offer one vehicle through the configured channel (Telegram, or Bird as a demo); 402 quota, 409 if taken or the batch needs review |
| `POST /jobs/claim` | body `{"batch_id", "driver_id", "task_id"?}`: the dispatcher records a claim for a driver from the list — that car, or the first unclaimed one of the batch's leg; 404 unknown driver or car, 409 if the batch isn't dispatchable, the driver hasn't cleared the previous leg, or the car / every car is taken. The batch's `drivers` list shows the driver's name |
| `POST /vrp/batches/{batch_id}/optimize-fleet` | real-road routes for the batch's vehicles |
| `POST /vrp/recalculate-driver-vector` | route metrics for a driver and co-drivers on a batch |

**Claiming is atomic:** one conditional `UPDATE … WHERE status = 'unassigned'`
per vehicle, whichever way the claim arrives (Telegram, WhatsApp demo,
`/jobs/claim`). When two drivers claim the same vehicle, exactly one wins;
the same driver claiming twice is a no-op. No Redis lock is used: the
database row is the single source of truth, so there is no lock to expire
or get out of sync with the data.
Vehicle task statuses: `unassigned`, `locked` (unused), `assigned`,
`in_transit`, `delivered`, `cancelled` (database CHECK constraint).

## 4. Driver messaging

### 4.1 Telegram — `POST /telegram/webhook`

Always answers 200 (a 5xx would make Telegram redeliver); each `update_id` is
processed once.

| Update | Effect |
|---|---|
| `/start` | asks the driver to share their phone number (contact button) |
| contact message | registers the chat — only the sender's **own** contact (`contact.user_id == from.id`); only the `driver_hash` is stored |
| **✅ Claim Driving** or text such as "CLAIM" | claim the offered vehicle (atomic); on leg 2+ with every vehicle taken, the driver rides along as passenger |
| **👥 Claim as Passenger** | ride along as passenger (leg 2+, only if someone drives) |
| **❌ Reject** or a reject word | decline; the leg becomes `rejected` once every offered driver declined |

A button from an older offer (another batch) doesn't act on the current
offer. After a claim the driver gets the next relay leg's offer, otherwise a
return-trip (backhaul) leg in the same session, if one is open.

### 4.2 WhatsApp via Bird — `POST /webhooks/whatsapp/inbound`

**Demo only, never run against a real Bird account.** The planned WhatsApp
path doesn't use an API: the dispatcher sends offers from their own
WhatsApp and records claims with `POST /jobs/claim` (section 3).

Inbound Bird message events from drivers are matched to the driver's
pending WhatsApp offer (valid 1 hour). Replies `CLAIM` / `REJECT` act on
that offer; anything else gets the help text. Offers are sent as a Bird
template message. Batch broadcasts always use Telegram.

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

1. **OCR:** PDFs with a text layer skip OCR. Images and scanned pages are
   read by RapidOCR (PaddleOCR PP-OCRv5 models on the CPU): the Latin model
   for `region=eu`, the Arabic model for `region=gcc`, with Arabic lines
   joined right to left. The models are part of the Docker image; nothing is
   downloaded at runtime. **Fallback**, when RapidOCR fails or finds no
   text: OpenCV preprocessing (grayscale, small scans upscaled, denoising,
   Otsu binarization), then Tesseract (`eng+deu` or `eng+ara` by region).
2. **Extraction:** a local LLM (Ollama, `OLLAMA_MODEL` default `llama3`,
   timeout `OLLAMA_TIMEOUT_SECONDS` 120 s; temperature 0, fixed seed and an
   8192-token context, so the same document gives the same output) returns
   JSON validated against the `BulkOrderManifest` schema: vehicles with
   VIN, model, plate, structured pickup/dropoff places (`place_name`,
   `place_type`, `city`, `country`), time windows, plus a `confidence` (0–1).
   A pickup window or deadline printed once for the whole order is copied to
   every vehicle without its own. Invalid output is sent back to the model
   with its validation errors, up to 3 attempts in total.
3. **Safeguards:** deterministic validation rules and tiered address
   fallbacks keep malformed values out of routing. VINs: I/O/Q are read as
   1/0/0, and a VIN split by a space after `VIN:` is joined. A missing VIN
   or plate is filled only from the same car's row of the text, otherwise it
   stays `UNKNOWN`.
   **Review doesn't depend on the LLM's own confidence** (it is
   self-reported, and a document can carry text aimed at the model): the
   batch goes to `manual_review` when `confidence < 0.80` **or** a check
   fails — a VIN, plate, pickup or dropoff not found in the document's text,
   a missing value, fewer vehicles than VINs in the document, or
   instruction-like text ("ignore previous instructions"). Each reason is
   stored in `payload.review_reasons` (`code`, `vehicle`, `leg`, `value`).
   The prompt marks the document as untrusted data.
4. **Geocoding:** Nominatim (`NOMINATIM_BASE_URL`; the public instance is
   limited to 1 request/s app-wide), results cached in Redis. **Routing:**
   OSRM (`OSRM_BASE_URL`) or Mapbox (`ROUTING_PROVIDER`), straight-line
   fallback. `ON_PREMISE_GEO=true` requires self-hosted geocoding and
   routing; the API refuses to start otherwise.

## 7. Realtime

No push channel: the dashboard polls `GET /batches/{batch_id}` every 20 s
and `GET /batches/{batch_id}/segments` every 10 s.

---

## Appendix: target architecture (not built)

Earlier versions of this document described these as existing; they are
not part of the code: BullMQ worker queues, gRPC / a separate VRP solver
service, Redis Redlock claiming (claims are an atomic PostgreSQL update),
Supabase Auth/Realtime and WebSocket events, Meta WhatsApp Cloud API
webhooks (the app uses Bird, as a demo), a general API rate limit, and
error codes such as `ERR_REDLOCK_FAILED`. Planned work is listed under
[Planned](../README.md#planned) in the README.
