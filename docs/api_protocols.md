1. Executive Summary & Architecture Overview

This document defines the interface control contracts, event-driven webhooks, and REST/gRPC payloads connecting the FleetReloc infrastructure components. The architecture isolates external third-party communication (Meta WABA, Maps, AI/LLM Inference) from core transaction processing, ensuring high throughput, deterministic concurrency handling during First-Come-First-Served (FCFS) job claiming, strict resilience against downstream service degradation, and zero-data-retention compliance via ephemeral hashing and hybrid/local AI processing nodes.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 INTEGRATION LANDSCAPE                                  │
└────────────────────────────────────────────────────────────────────────────────────────┘
 [ External Parties ]             [ API Gateway Layer ]            [ Internal Services ]
   Meta Cloud API (WABA) ──(HTTPS)──► Ingestion Gateway ──(gRPC/REST)─► VRP Solver Engine
   Local Ollama (On-Prem)◄─(HTTP)──── (FastAPI / Edge)  ──(WebSockets)► Supabase Realtime
   Mapbox / OSRM API    ◄─(HTTPS)───         │                       (PostgreSQL DB)
                                             └───────(BullMQ)───────► Redis Cluster
                                                                     (Redlock & Queue)
```
# 2. Authentication & Security Protocols

## 2.1 Internal Microservice Authentication
* **Service-to-Service Authorization:** Internal communication between FastAPI Ingestion Gateway, BullMQ Workers, and Python VRP Solver microservices requires a Bearer JWT Token issued via Supabase Auth service role or an internal symmetric HMAC-SHA256 signature passed in header: `X-Internal-Signature`.
* **API Rate Limiting:** Enforced at the Gateway level using Redis Token Bucket algorithm:
  * *WhatsApp Inbound Webhooks:* Uncapped burst limit (up to 100 req/sec) with IP whitelisting matching Meta Cloud API IP ranges.
  * *Dispatcher Dashboard REST/GraphQL API:* 60 requests/minute per authenticated user session.

## 2.2 External Webhook Verification (WhatsApp / Twilio)
* **Requirement:** All incoming webhooks from Meta Cloud API must undergo HMAC-SHA256 signature verification before pushing payload to BullMQ queue.
* **Header:** `X-Hub-Signature-256`
* **Verification Logic:** `HMAC_SHA256(APP_SECRET, RAW_REQUEST_BODY) == HEADER_SIGNATURE`

---

# 3. WhatsApp Business API (WABA) Integration Specs

## 3.1 Inbound Webhook Payload: Interactive Button Claim (FCFS Action)
Triggered when a driver taps the interactive button `[ Claim Job ]` inside a WhatsApp broadcast message.

* **HTTP Method:** `POST /api/v1/webhooks/whatsapp/inbound`
* **Payload Structure:**
```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "109823749123847",
      "changes": [
        {
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "971501234567",
              "phone_number_id": "987654321098765"
            },
            "contacts": [
              {
                "profile": { "name": "[REDACTED_ZERO_TRUST]" },
                "wa_id": "971501234567"
              }
            ],
            "messages": [
              {
                "from": "971501234567",
                "id": "wamid.HBgLMTU1NTVN...",
                "timestamp": "1710000000",
                "type": "interactive",
                "interactive": {
                  "type": "button_reply",
                  "button_reply": {
                    "id": "btn_claim_WBA33AG080FP12345",
                    "title": "Claim Job 1"
                  }
                }
              }
            ]
          },
          "field": "messages"
        }
      ]
    }
  ]
}
```

# 4. VRP Solver Engine Microservice Specs

## 4.1 Synchronous Incremental Re-route Protocol
Used for real-time recalculation when a driver claims a vehicle via FCFS button press in WhatsApp.

* **HTTP Method:** `POST /api/v1/vrp/recalculate-driver-vector`
* **Request Payload:**
```json
{
  "driver_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "claimed_vehicle": {
    "vin": "WBA33AG080FP12345",
    "pickup_lat": 25.204800,
    "pickup_lng": 55.270800,
    "pickup_window_start": "2026-08-20T08:00:00Z",
    "pickup_window_end": "2026-08-20T10:00:00Z",
    "dropoff_lat": 24.453900,
    "dropoff_lng": 54.377300,
    "dropoff_deadline": "2026-08-20T18:00:00Z"
  },
  "driver_current_state": {
    "lat": 25.200000,
    "lng": 55.271000,
    "shift_start_time": "2026-08-20T07:30:00Z",
    "has_companion": false
  },
  "active_batch_context": {
    "batch_id": "b1a2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
    "co_drivers_available": [
      {
        "driver_hash": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
        "dropoff_lat": 24.455000,
        "dropoff_lng": 54.380000,
        "estimated_dropoff_time": "2026-08-20T14:15:00Z"
      }
    ]
  }
}
```

**Response Payload (200 OK):**
```json
{
  "status": "SUCCESS",
  "execution_time_ms": 138,
  "route_plan": {
    "driver_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "vehicle_vin": "WBA33AG080FP12345",
    "leg_1_pickup": {
      "eta": "2026-08-20T08:15:00Z",
      "distance_meters": 1200
    },
    "leg_2_dropoff": {
      "eta": "2026-08-20T14:10:00Z",
      "distance_meters": 395000
    },
    "return_logistics": {
      "mode": "CONVOY_RETURN",
      "convoy_lead_driver_hash": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08",
      "return_vehicle_vin": "WAUZZZ4G0BN123456",
      "rendezvous_location": {
        "lat": 50.112000,
        "lng": 8.685000
      },
      "estimated_departure": "2026-08-20T14:30:00Z",
      "cost_savings_eur": 42.50
    }
  }
}
```

# 5. Bulk Document Parsing Protocols (PyMuPDF & Hybrid LLM / Local Inference)

## 5.1 OCR & Structured Extraction Contract
Structured output format requested from either enterprise cloud LLM endpoints or self-hosted local inference runtimes (e.g., Ollama/vLLM via OpenAI-compatible endpoints) with strict JSON Schema enforcement during PDF/Excel manifest ingestion.

> **Implementation note (live, see `apps/api/app/schemas/bulk_order_manifest.py`):**
> the deployed Pydantic schema deviates from the plain JSON Schema below in
> two deliberate ways: (1) `pickup_location`/`dropoff_location` are
> structured `PlaceDetail` objects (`place_name`, `place_type`, `city`,
> `country`) rather than bare strings — required for airport-aware
> geocoding — with a bare string still accepted and auto-normalized for
> backward compatibility; (2) `vin` and the time-window fields are
> optional/coercible rather than hard-`required`, since the OCR/LLM
> extraction prompt cannot always recover a valid 17-character VIN or exact
> time windows from unstructured source documents, and hard-failing the
> whole manifest on one bad field would defeat the confidence-gating
> workflow in Section 7. An explicit `confidence` field (float, 0.0–1.0) is
> also present on every manifest — see `ERR_OCR_CONFIDENCE_LOW` below.

### 5.1.1 On-Premise OCR/LLM Pipeline (Strict Compliance, No Cloud)

The live pipeline (`apps/api/app/utils/pdf_parser.py`) runs entirely
on-premise -- no manifest content (vehicle data, addresses, VINs) ever
leaves the deployment perimeter:

* **OCR preprocessing (OpenCV):** every raster image (PNG/JPG/JPEG) is
  preprocessed via `app/utils/ocr_preprocessing.py` -- grayscale
  conversion, `cv2.fastNlMeansDenoising`, and Otsu binarization -- before
  Tesseract OCR, improving extraction quality on low-quality phone-camera
  scans. Falls back to the original, unprocessed image on any OpenCV
  failure rather than aborting the pipeline.
* **Multi-language OCR:** Tesseract runs the combined language set
  `Settings.OCR_LANGUAGES` (default `"eng+deu+ara"`, covering DE/EN/AR
  manifests) with automatic fallback to `Settings.OCR_FALLBACK_LANGUAGE`
  (default `"eng"`) if the combined language pack isn't installed on the
  host.
* **Local Ollama (Llama 3) timeout:** `Settings.OLLAMA_TIMEOUT_SECONDS`
  (default `120.0`) gives large manifests (10+ vehicles) enough headroom
  to finish LLM generation without the request aborting mid-response.
  `Settings.OLLAMA_BASE_URL`/`OLLAMA_MODEL` point at the local/on-prem
  Ollama instance -- never a cloud endpoint.
* **Deterministic hub-matching fallback (defensive post-processing):**
  after JSON parsing and Pydantic structural validation succeed, every
  extracted pickup/dropoff address is re-validated against a known-hub
  directory (`app/utils/hub_resolver.py::KNOWN_HUBS` -- e.g. "Berlin Hub",
  "Rostock Port Hub", "Ketzin Hub") using fuzzy string matching. This
  corrects LLM typos/transliteration drift (e.g. "Rostok Hub" ->
  "Rostock Port Hub") and substitutes a clearly-labeled placeholder
  (never `None`/null) when the LLM returns a null or unresolved address,
  so malformed LLM output is corrected deterministically in Python rather
  than forcing a request back to the LLM or routing every imperfect
  extraction to `manual_review`.

```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "BulkOrderManifest",
  "type": "object",
  "properties": {
    "client_name": { "type": "string" },
    "manifest_reference": { "type": "string" },
    "confidence": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
    "vehicles": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "vin": { "type": "string", "pattern": "^[A-HJ-NPR-Z0-9]{17}$" },
          "make_model": { "type": "string" },
          "license_plate": { "type": "string" },
          "pickup_location": {
            "type": "object",
            "properties": {
              "place_name": { "type": "string" },
              "place_type": { "type": "string" },
              "city": { "type": "string" },
              "country": { "type": "string" }
            }
          },
          "pickup_window_start": { "type": "string", "format": "date-time" },
          "pickup_window_end": { "type": "string", "format": "date-time" },
          "dropoff_location": {
            "type": "object",
            "properties": {
              "place_name": { "type": "string" },
              "place_type": { "type": "string" },
              "city": { "type": "string" },
              "country": { "type": "string" }
            }
          },
          "dropoff_deadline": { "type": "string", "format": "date-time" }
        },
        "required": ["vin", "pickup_location", "dropoff_location", "dropoff_deadline"]
      }
    }
  },
  "required": ["client_name", "vehicles"]
}
```

## 5.2 Multi-File Manifest Uploads & Parallel Optimization Sessions

A dispatcher may upload several manifest files concurrently (e.g. via the
dispatcher dashboard's multi-file drop zone). Each file is parsed and
ingested as its own independent `bulk_batches` row, but many such batches
can be grouped under a single `optimization_sessions` row via
`bulk_batches.session_id`, letting a dispatcher track and operate on
several files feeding one logical routing/chaining stream together —
while each file's own parse outcome (`status`, `successful_rows`,
`failed_rows`) stays independently auditable. See `docs/db_schema.md`
("Multi-file manifest uploads") for the underlying schema.

### 5.2.1 Asynchronous Upload Contract (202 Accepted)

`POST /api/v1/batches/upload` (non-JSON file types: PDF/PNG/JPG/JPEG)
returns **`202 Accepted`** immediately with a placeholder `BulkBatch`
(`status="processing"`) rather than waiting for OCR/Ollama/geocoding/
routing to finish synchronously -- large, multi-page, multi-language
manifests can take well over typical browser/reverse-proxy gateway
timeouts (30-60s) to process. The heavy pipeline is scheduled via FastAPI
`BackgroundTasks` and runs after the response is sent.

Clients MUST poll `GET /api/v1/batches/{batch_id}` until `status`
transitions out of `"processing"`:

| `status` | Meaning |
| :--- | :--- |
| `processing` | Upload accepted; OCR/LLM/routing pipeline running in the background. |
| `completed` | Parsing succeeded; `payload.vehicles` is populated and ready for dispatch. |
| `manual_review` | Parsed successfully but LLM `confidence` < 0.80 -- see `ERR_OCR_CONFIDENCE_LOW`. |
| `failed` | The background pipeline raised an unrecoverable error (e.g. malformed LLM JSON that survived all defensive layers); inspect server logs for `tenant_id`-tagged details. |

JSON-file uploads (`.json`) are handled synchronously (no OCR/LLM work
required) and also return `202 Accepted` for API consistency, but the
final `BulkBatch` is already fully populated in that same response.

* `GET /api/v1/sessions/` — list all `OptimizationSession` streams for the
  caller's tenant (strictly tenant-scoped via the automatic ORM filter).
* `GET /api/v1/sessions/{session_id}/batches` — list every `bulk_batches`
  row (one per uploaded file) belonging to a given session, 404 if the
  session does not belong to the caller's tenant.

A dispatcher can run **2+ independent, parallel** `OptimizationSession`
streams simultaneously (e.g. a Ketzyn→Berlin stream alongside a
Frankfurt→Hamburg stream) without any cross-contamination: every
`route_legs` row carries a mandatory `session_id`, and backhaul-chaining
offers (Section 5.3) are only ever matched within the same
`(tenant_id, session_id)` pair.

## 5.3 Route Leg State Machine & Round-Trip Backhaul Chaining

Multi-leg route segments are tracked in the `route_legs` table (see
`docs/db_schema.md`), constrained by the `ck_route_legs_status` CHECK
constraint to exactly five states:

```
pending ──▶ claimed_driver ──▶ passenger_offered ──▶ claimed_passenger
   ▲               │
   │               ▼
   └────────── rejected
```

* **`pending`** — default state; eligible for FCFS claiming.
* **`claimed_driver`** — set when a driver claims the outbound leg via the
  Telegram webhook (`apps/api/app/api/v1/endpoints/telegram.py`), strictly
  scoped to `(tenant_id, batch_id)`. This automatically triggers a lookup
  for a matching round-trip backhaul leg (destination → origin) within the
  **same** `(tenant_id, session_id)` — never across tenants or sessions.
* **`rejected`** — set when a driver explicitly declines the leg.
* **`passenger_offered`** — set automatically on the matching backhaul leg
  once the outbound leg is claimed, entering the round-trip chaining offer
  pool for a return passenger/driver.
* **`claimed_passenger`** — terminal state once the backhaul leg itself is
  claimed.

### Dispatcher Manual Controls

* `POST /api/v1/batches/{batch_id}/segments/{leg_id}/retry` — resets a
  `rejected`/`claimed_driver` leg back to `pending` and clears
  `driver_phone`, so it re-enters the FCFS/backhaul-offer pool. Strictly
  verified against `(tenant_id, batch_id, leg_id)` before mutation — 404
  if the batch or leg does not belong to the caller's tenant.
* `DELETE /api/v1/batches/{batch_id}/segments/{leg_id}/driver` —
  unassigns the currently claimed `driver_phone` (driver_hash) from a
  segment and resets it to `pending`, under the same strict tenant/batch/
  leg verification.
* `GET /api/v1/batches/{batch_id}/segments` — lists all `route_legs` for a
  batch, tenant-scoped, used by the dispatcher dashboard's per-phone,
  per-leg status table across every file/batch in the active session.

# 6. Realtime WebSockets & Event Specifications (Supabase)

## 6.1 Dispatcher Command Center WebSocket Events
* **Channel:** `realtime:public:vehicles:batch_id=eq.{BATCH_ID}`
* **Event Payload:** `VEHICLE_CLAIMED` (Broadcasted to the Desktop Web Dashboard when a vehicle transitions from unassigned to assigned).

```json
{
  "event": "UPDATE",
  "schema": "public",
  "table": "vehicles",
  "commit_timestamp": "2026-08-20T10:14:22.102Z",
  "old": {
    "id": "a9b8c7d6-e5f4-3a2b-1c0d-9e8f7a6b5c4d",
    "status": "unassigned",
    "assigned_driver_hash": null
  },
  "new": {
    "id": "a9b8c7d6-e5f4-3a2b-1c0d-9e8f7a6b5c4d",
    "status": "assigned",
    "assigned_driver_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "locked_until": null
  }
}
```

## 6.2 Route Leg Status Polling (Dispatcher Dashboard)
The dispatcher dashboard's `SegmentControlPanel` component polls
`GET /api/v1/batches/{batch_id}/segments` (tenant-scoped) every 5 seconds
per batch tracked in the active `OptimizationSession` to render live
per-phone, per-leg status across every uploaded file/batch in that
session — this currently supplements, rather than replaces, event-driven
WebSocket pushes for `route_legs` (no dedicated `ROUTE_LEG_UPDATED`
Supabase Realtime channel exists yet; see Section 5.3 for the full
`route_legs.status` state machine).

# 7. Error Handling, Status Codes & Circuit Breakers

| Error Code | HTTP Status | Error Category | Description & Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| `ERR_REDLOCK_FAILED` | 409 Conflict | Concurrency | Triggered when an anonymous session attempts to claim an already locked car. Returns WhatsApp text instructing driver to pick another car. |
| `ERR_OCR_CONFIDENCE_LOW` | 422 Unprocessable | AI Extraction | Document parsing confidence <0.80. Pushes batch to "Manual Review Queue" on Dispatcher Dashboard. |
| `ERR_MAPBOX_TIMEOUT` | 504 Gateway Timeout | Routing API | Mapbox response >2000 ms. Fallback instantly triggers OSRM. **Live default:** `ROUTING_PROVIDER=osrm` makes OSRM the zero-config primary provider; Mapbox is only primary if explicitly configured. A straight-line approximation (`route_provider="fallback"`) is the absolute last resort if both real providers fail. |
| `ERR_WABA_RATE_LIMIT` | 429 Too Many Requests | Messaging | Meta API limit reached. Outbound payload requeued in BullMQ with exponential backoff (2s, 4s, 8s...). |
