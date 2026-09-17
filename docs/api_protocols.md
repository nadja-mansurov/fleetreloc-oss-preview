1. Executive Summary & Architecture Overview

This document defines the interface control contracts, event-driven webhooks, and REST/gRPC payloads connecting the FleetReloc infrastructure components. The architecture isolates external third-party communication (Meta WABA, Maps, AI/LLM Inference) from core transaction processing, ensuring high throughput, deterministic concurrency handling during First-Come-First-Served (FCFS) job claiming, strict resilience against downstream service degradation, and zero-data-retention compliance via ephemeral hashing and hybrid/local AI processing nodes.

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 INTEGRATION LANDSCAPE                                  │
└────────────────────────────────────────────────────────────────────────────────────────┘
 [ External Parties ]             [ API Gateway Layer ]            [ Internal Services ]
   Meta Cloud API (WABA) ──(HTTPS)──► Ingestion Gateway ──(gRPC/REST)─► VRP Solver Engine
   Local / Cloud LLM Node◄─(HTTPS)─── (FastAPI / Edge)  ──(WebSockets)► Supabase Realtime
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

```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "BulkOrderManifest",
  "type": "object",
  "properties": {
    "client_name": { "type": "string" },
    "manifest_reference": { "type": "string" },
    "vehicles": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "vin": { "type": "string", "pattern": "^[A-HJ-NPR-Z0-9]{17}$" },
          "make_model": { "type": "string" },
          "license_plate": { "type": "string" },
          "pickup_location": { "type": "string" },
          "pickup_window_start": { "type": "string", "format": "date-time" },
          "pickup_window_end": { "type": "string", "format": "date-time" },
          "dropoff_location": { "type": "string" },
          "dropoff_deadline": { "type": "string", "format": "date-time" }
        },
        "required": ["vin", "pickup_location", "dropoff_location", "dropoff_deadline"]
      }
    }
  },
  "required": ["client_name", "vehicles"]
}
```

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

# 7. Error Handling, Status Codes & Circuit Breakers

| Error Code | HTTP Status | Error Category | Description & Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| `ERR_REDLOCK_FAILED` | 409 Conflict | Concurrency | Triggered when an anonymous session attempts to claim an already locked car. Returns WhatsApp text instructing driver to pick another car. |
| `ERR_OCR_CONFIDENCE_LOW` | 422 Unprocessable | AI Extraction | Document parsing confidence <0.80. Pushes batch to "Manual Review Queue" on Dispatcher Dashboard. |
| `ERR_MAPBOX_TIMEOUT` | 504 Gateway Timeout | Routing API | Mapbox response >2000 ms. Fallback instantly triggers local OSRM instance. |
| `ERR_WABA_RATE_LIMIT` | 429 Too Many Requests | Messaging | Meta API limit reached. Outbound payload requeued in BullMQ with exponential backoff (2s, 4s, 8s...). |
