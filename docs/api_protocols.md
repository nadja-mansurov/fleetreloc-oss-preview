# FleetReloc API & Integration Protocols

> **Synthetic data:** all example values in this document (companies, addresses, vehicles, VINs, plates, phone numbers, hashes and IDs) are synthetic, generated for R&D demonstration only. See the synthetic data notice in the [README](../README.md).

The contracts between the dispatcher dashboard, the API, drivers' messengers
and the on-premise parsing pipeline — **as built**. Last checked against the
code: 2026-11-28. Base path: `/api/v1`. Planned work, and what is
implemented but still waiting for an end-to-end check: see
[Planned](../README.md#planned) in the README.

```text
  Dispatcher dashboard (Next.js) ──Bearer (OIDC)──►  FastAPI  ──► PostgreSQL (tenant-filtered ORM, job queue, outbox, Telegram updates)
        ▲   login: Keycloak (OIDC, PKCE)              │  ▲      ──► Redis (pending offers, relay chain)
        └──────────── polls every 10–20 s ────────────┘  │      ──► Nominatim (geocoding), OSRM / Mapbox (routing)
  Drivers: Telegram bot ──webhook (secret header)────────┘
                                                      Manifest worker ──► RapidOCR + Ollama (on-prem)
                                                      Outbox relay    ──► Telegram Bot API (outbound messages)
```

Three processes share one code base and one database:

- **API** — the HTTP endpoints below.
- **Manifest worker** — reads scanned manifests (OCR + LLM) from a job queue
  in PostgreSQL (procrastinate), and runs the maintenance jobs (stalled-job
  recovery, file cleanup, batch retention).
- **Outbox relay** — sends the driver messages the API and worker queued
  (section 4.2).

WhatsApp has no process of its own: the API writes offer texts, and the
dispatcher sends them from their own WhatsApp (section 4.3).

---

## 1. Authentication

| Caller | Credentials | Resolved tenant |
|---|---|---|
| Dashboard user | `Authorization: Bearer <access token>` from the OIDC provider (Keycloak, authorization code + PKCE) | the token's tenant claim; `sub` = dispatcher id |
| Internal service | shared internal secret plus a tenant header (constant-time compare) | the tenant header |
| Telegram | Telegram's webhook secret-token header | from the offered task |

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
- **Status codes used:** 400 unreadable upload (bad file, not UTF-8, no
  vehicles, invalid leg number) · 401 no/invalid credentials · 402 message
  quota used up · 403 unknown tenant / bad webhook token · 404 not found
  *for this tenant* · 409 state conflict (see each endpoint) · 411 body
  without Content-Length · 413 request/file too large · 422 invalid input ·
  429 too many requests (`Retry-After` header) · 503 OIDC provider
  unreachable.
- **Limits** (settings): request body `MAX_REQUEST_BYTES` 100 MB (checked from
  Content-Length before parsing); per file `MAX_UPLOAD_BYTES` 40 MB; files per
  upload `MAX_UPLOAD_FILES` 20; rows/vehicles per file `MAX_SPREADSHEET_ROWS`
  5000; unpacked `.xlsx` `MAX_SPREADSHEET_UNCOMPRESSED_BYTES` 100 MB plus a
  zip-bomb ratio check; phones per broadcast `MAX_BROADCAST_DRIVERS` 50.
- **Scanned manifests** (OCR + LLM are CPU-heavy): pages per PDF
  `MAX_PDF_PAGES` 30 and pixels per image `MAX_IMAGE_PIXELS` 40 M (422, read
  from the file header at upload and checked again before OCR); scanned
  files waiting to be read per tenant, including the upload being made,
  `MAX_QUEUED_SCANS_PER_TENANT` 20 (429; a multi-leg order fits in one
  upload); upload requests per tenant per hour `UPLOADS_PER_TENANT_PER_HOUR`
  30 (429). In the worker: files read at once `MAX_PARALLEL_PARSE_JOBS` 2
  (later ones wait), in separate parser processes, so a file that crashes
  or hangs its parser fails alone; each file is limited to
  `PARSE_FILE_TIMEOUT_SECONDS` 600 s. A file whose reading never finishes
  (worker killed) is retried and fails after `PARSE_MAX_ATTEMPTS` 3, while
  the other files of the batch go on. File formats and sizes are checked
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
| `POST /batches/upload` | upload manifests (below) → `202` + `BatchUploadRead` |
| `GET /batches/` | the tenant's batches |
| `GET /batches/{batch_id}` | batch with `payload` (vehicles, waypoints), `drivers` (hash, masked phone or name, status), `paths`, `routing_provider`, `stats` |
| `DELETE /batches/{batch_id}` | delete the batch and everything that refers to it (below) → `204`; 409 while its files are still being read |
| `PUT /batches/{batch_id}/payload` | manifest review editor, "Save & approve" (below) |
| `POST /batches/{batch_id}/broadcast` | offer the leg's unclaimed vehicles to drivers (below) |
| `POST /batches/{batch_id}/next-wave` | same, for the next wave of drivers |
| `POST /batches/{batch_id}/drivers/{driver_hash}/resend-offer` | queue again an offer that couldn't be delivered (status `undelivered`), for the same vehicle; counted against the quota again (402); 404 without an undelivered offer, 409 if the vehicle was taken meanwhile, the driver can't be offered it now, or it was already resent |
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
are geocoded and routed, the batch is `ready`. PDFs and images are stored
and queued for the manifest worker in the same transaction as the batch;
the request returns at once with status `processing`, and OCR → LLM →
geocoding → routing runs in the worker. Clients poll
`GET /batches/{batch_id}` (the dashboard every 20 s).

**Safe retries:** an optional `Idempotency-Key` header (kept
`IDEMPOTENCY_KEY_TTL_HOURS` 24 h) makes a repeated upload return the first
response instead of creating a second batch. Same key while the first
request is still running: 409 with `Retry-After`; same key for a different
upload: 422; a failed request releases its key. Without a key, the response
`warnings` list files whose content was already uploaded to another batch of
the tenant in the last `DUPLICATE_UPLOAD_WINDOW_DAYS` 30 days
(`{"code": "duplicate_file", "file_name", "duplicate_of_batch_id",
"uploaded_at"}`); the upload still goes through.

**Batch status:**

| Status | Meaning | Offers / claims |
|---|---|---|
| `processing` | the worker is reading the scans | blocked (409) |
| `manual_review` | the extraction needs a person: low LLM confidence (< 0.80) or a failed check (`payload.review_reasons`, see section 6); needs "Save & approve" | blocked (409) |
| `failed` | pipeline failed | blocked (409) |
| `ready` | parsed, can be dispatched | allowed |
| `broadcasting` | offers sent, not every vehicle claimed | allowed |
| `completed` | every vehicle claimed | — |

**Payload editor** (`PUT /batches/{batch_id}/payload`, body
`{"vehicles": [...]}`, same count and order as the batch, 422 otherwise):
only `vin`, `model`, `plate`, `origin`, `originTitle`, `destination`,
`destinationTitle` can change. Edits reach the vehicle tasks (changed
addresses are re-geocoded and re-routed). 409 while `processing` or once
offers were sent. A `manual_review` batch becomes `ready`.

**Broadcast / next wave** — body `{"driver_phones": ["+49…", …]}` (1–50,
normalized, duplicates dropped):
- 409 if the batch isn't dispatchable or no vehicle on the leg is unclaimed.
- Drivers are spread over the unclaimed vehicles round-robin; each offer is
  a Telegram message with `[✅ Claim Driving]`, `[👥 Claim as Passenger]`
  (leg 2+), `[❌ Reject]`, valid for 1 hour.
- Each offer is counted against the plan's message quota in the same
  transaction that queues it, so a crash can't count an offer that was never
  queued; it is refunded if it is never delivered.
- Response: `offered`, `pending`, `unreachable` (not registered with the
  bot), `ineligible` (hasn't cleared the previous leg), `quota_exceeded`,
  `vehicles`; 402 if the quota stopped every offer. The same statuses appear
  per driver in `GET /batches/{batch_id}` → `drivers`, plus `undelivered`
  once the relay gives up on an offer (section 4.2) and `removed`.

**Deleting a batch** (`DELETE /batches/{batch_id}`) removes its vehicle
tasks, driver answers, route legs, queued messages, upload keys and file
hashes, registry vehicles nothing else uses, and the session if this was its
last batch. Offers still out are withdrawn: a driver who answers one is told
it is no longer active. The worker applies the same deletion hourly to
batches inactive for longer than the tenant's retention period
(`BATCH_RETENTION_DAYS` 90 by default, set per tenant by the operator).

### Vehicles, drivers, offers, claims

`{vehicle_id}` below is the id of a vehicle **task** (`batch_vehicles.id`),
not of the registry vehicle.

| Method & path | Does |
|---|---|
| `GET /vehicles/?limit=&offset=` | vehicle tasks, newest first; `limit` 1–500 (default 100) |
| `GET /vehicles/{vehicle_id}` | one vehicle task |
| `POST /vehicles/` | create a task; with `batch_id` the batch must be the tenant's (404) and accept vehicles (409); it goes through the upload ingestion (geocoded, routed, added to the batch) |
| `GET /drivers`, `POST /drivers`, `PATCH /drivers/{driver_id}`, `DELETE /drivers/{driver_id}` | the dispatcher's driver list: body `{"display_name", "language": "de"\|"ar"\|"en"}`; no phone numbers (a name with 7+ digits is rejected, 422); names unique per tenant (409); delete replaces the name in batch driver lists with "Removed driver" |
| `POST /orders/{vehicle_id}/send-offer` | body `{"driver_phone": "…"}`: offer one vehicle on Telegram; 402 quota, 404 unknown vehicle, 409 if taken, the batch needs review, or the driver hasn't registered with the bot |
| `POST /jobs/claim` | body `{"batch_id", "driver_id", "task_id"?, "mode"?}`: the dispatcher records a driver's answer from WhatsApp for a driver from the list. `mode` `drive` (default): claims that car, or the first unclaimed one of the batch's leg; `passenger`: rides along on the leg (leg 2+, someone must drive a car of it; recorded in `co_driver_hashes`, no car taken); `reject`: declines the leg (409 for a driver holding a car there; a passenger seat is given up). 404 unknown driver or car, 409 if the batch isn't dispatchable, the driver hasn't cleared the previous leg, the car / every car is taken, or the answer can't be recorded (reason in `detail`). The batch's `drivers` list shows the driver's name and `claimed` / `passenger` / `rejected` |
| `POST /batches/{batch_id}/whatsapp-offers` | body `{"driver_ids": [...], "group_language"?: "de"\|"ar"\|"en"}`: writes the leg's WhatsApp offers (section 4.3): per driver a text in their language for one open car (round-robin) and a `wa.me` share link, optionally one group text listing every open car (`cars` gives their numbers). The drivers show as `pending` (`ineligible` without a text if they haven't cleared the previous leg; `claimed` / `passenger` answers are kept). Sends nothing, uses no quota. 404 unknown batch or driver, 409 batch not dispatchable or no open car, 422 neither drivers nor group |
| `POST /vrp/batches/{batch_id}/optimize-fleet` | real-road routes for the batch's vehicles |
| `POST /vrp/recalculate-driver-vector` | route metrics for a driver and co-drivers on a batch |

**Claiming is atomic:** one conditional `UPDATE … WHERE status = 'unassigned'`
per vehicle, whichever way the claim arrives (Telegram or `/jobs/claim`).
When two drivers claim the same vehicle, exactly one wins; the same driver
claiming twice is a no-op. No Redis lock is used: the database row is the
single source of truth, so there is no lock to expire or get out of sync
with the data.

**Batch updates are serialized:** every change to a batch's `payload`,
`claimed_count` or `status` (claims, offers, edits, driver removal) first
locks the batch row (`SELECT … FOR UPDATE`) and commits soon after, so two
concurrent requests can't overwrite each other's changes.

Vehicle task statuses: `unassigned`, `locked` (unused), `assigned`,
`in_transit`, `delivered`, `cancelled` (database CHECK constraint).

## 4. Driver messaging

### 4.1 Telegram — `POST /telegram/webhook`

Rejects a missing or wrong secret-token header with 403. With a valid
secret, each update is **processed at least once until its decision is
saved, and decided at most once**. Every `update_id` gets a row in
`telegram_updates` before processing:

| Situation | Answer |
|---|---|
| processed: its decision (claim, decline, passenger) saved, or nothing to save (`/start`, a reply) | 200 |
| the same update already `done` or `failed` | 200, nothing processed again |
| the same update being processed right now (a PostgreSQL advisory lock is held) | 503 with `Retry-After: 5`, so Telegram repeats it later |
| a failure before anything was saved | 500, so Telegram repeats it; safe, because nothing was saved |
| a failure after the decision was saved | 200 and an error log; a repeat would find the offer gone |
| failed `TELEGRAM_UPDATE_MAX_ATTEMPTS` (5) times | recorded `failed` and answered 200, so Telegram stops repeating it |

A crashed request releases its lock with its connection, so Telegram's next
repeat is processed without waiting for a timeout. Rows hold no chat id or
text and are deleted after 24 hours (Telegram keeps undelivered updates no
longer). Tested by simulating the state a crash leaves; a real crash of the
API during a tap has not been checked yet.

| Update | Effect |
|---|---|
| `/start` | asks the driver to share their phone number (contact button) |
| contact message | registers the chat — only the sender's **own** contact (`contact.user_id == from.id`); only the `driver_hash` is stored |
| **✅ Claim Driving** or text such as "CLAIM" | claim the offered vehicle (atomic); on leg 2+ with every vehicle taken, the driver rides along as passenger |
| **👥 Claim as Passenger** | ride along as passenger (leg 2+, only if someone drives) |
| **❌ Reject** or a reject word | decline; the leg becomes `rejected` once every offered driver declined |

A button from an older offer (another batch), an expired offer or one whose
batch was deleted doesn't act; the driver is told the offer is no longer
active. After a claim the driver gets the next relay leg's offer, otherwise
a return-trip (backhaul) leg in the same session, if one is open.

### 4.2 Outbound messages (outbox)

Implemented and tested; the end-to-end check with a real bot is still open.

Every message to a driver (offer, claim confirmation, passenger added,
declined, removed by the dispatcher) is written to `outbox_messages` in the
same transaction as the change it reports, so it exists if and only if that
change was committed. The relay service sends it afterwards:

- Messages to one driver go out in the order they were queued.
- Transient failures (network, Telegram 429/5xx) are retried with backoff,
  up to `OUTBOX_MAX_ATTEMPTS` 8.
- An offer not delivered within its 1-hour validity fails instead of
  arriving late.
- A failed offer is refunded and the driver shows as `undelivered`; the
  dispatcher can resend it (section 3).
- Rows store the `driver_hash`, never a chat id; the relay looks the chat up
  when it sends. Sent rows are deleted after 24 h, failed ones after 14 days.

### 4.3 WhatsApp — share links, sent by the dispatcher

Implemented and tested; the end-to-end check in WhatsApp is still open (see
[Planned](../README.md#planned)).

No WhatsApp API and no webhook: `POST /batches/{batch_id}/whatsapp-offers`
returns the texts, and the dashboard shows each with **Copy** and **Send via
WhatsApp** (`https://wa.me/?text=<encodeURIComponent(text)>`, no phone
number, opened in a new tab with `rel="noopener noreferrer"`). The
dispatcher picks the chat in their own WhatsApp and records each answer with
`POST /jobs/claim` (`drive` / `passenger` / `reject`, section 3).

- **Texts:** route, pickup window, delivery deadline, car (make/model,
  plate), the leg, and the reply codes `1` drive (group: the car's number),
  `P` passenger (leg 2+ only), `0` no. Built only from those fields: no phone
  number, token, tenant, batch, task or driver ID, and no VIN. Times are
  printed as stored (the manifest's local wall-clock time), like the
  Telegram offer.
- **Arabic:** Latin values (addresses, plates, times) are wrapped in Unicode
  directional isolates (U+2068 … U+2069); directional control characters in
  customer data are removed. A language or phrase without a translation
  falls back to English and logs a warning. The German and Arabic texts are
  drafts waiting for a native speaker's review.

## 5. Route legs (`route_legs.status`)

One route leg per leg number of a batch; every leg
belongs to a session, and backhaul matching never crosses sessions or
tenants.

| Status | Set when |
|---|---|
| `pending` | created / reopened (retry, last driver removed) |
| `claimed_driver` | a driver claims a vehicle of the leg (Telegram or `/jobs/claim`) |
| `claimed_passenger` | a passenger joins a pending leg (`co_driver_hashes`; Telegram or `/jobs/claim` with `mode: passenger`) |
| `rejected` | every driver offered the leg declined |
| `passenger_offered` | allowed by the CHECK constraint, not set by the current code |

**Relay order:** a driver is offered (and may claim) leg N ≥ 2 only after
claiming or declining leg N − 1 of the same session.

## 6. Manifest parsing (on-premise, in the manifest worker)

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

Each file's progress is kept in `manifest_file_results` until the batch is
final: after a worker restart, files already read are not read again, and
the file that was being read is retried.

## 7. Realtime

No push channel: the dashboard polls `GET /batches/{batch_id}` every 20 s
and `GET /batches/{batch_id}/segments` every 10 s.

---

## Appendix: target architecture (not built)

Earlier versions of this document described these as existing; they are
not part of the code: BullMQ worker queues (the job queue is procrastinate,
in PostgreSQL), gRPC / a separate VRP solver service, Redis Redlock
claiming (claims are an atomic PostgreSQL update), Supabase Auth/Realtime
and WebSocket events, a WhatsApp API integration (Meta Cloud API or Bird;
WhatsApp offers go through the dispatcher's own WhatsApp, section 4.3),
a general API rate limit, and error codes such as `ERR_REDLOCK_FAILED`.
Planned work is listed under [Planned](../README.md#planned) in the README.
