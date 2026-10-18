# FleetReloc OSS Preview

Disclaimer: This repository is an independent, personal R&D portfolio project developed strictly on my own time, using my own personal equipment, and without any relation, contribution, or connection to any current or past employers. All concepts, code structures, and architectures presented here are autonomous technical explorations and do not utilize proprietary employer resources or trade secrets.

Public preview of **FleetReloc**: a **messenger-first dispatch platform** for car rental networks, leasing operators and fleet managers in the GCC and Europe. It automates bulk vehicle relocation, moving cars between branches, service centers and airports across multi-leg routes. Drivers work entirely in a chat app: Telegram today, WhatsApp as the second channel.

> **About this repository:** this is a public preview. It contains documentation, a demo video, a synthetic sample manifest, and configuration templates. The application source code is maintained in a private repository.

---

## What FleetReloc does today

### From manifest to dispatched jobs
- **Multi-format intake.** Dispatchers upload transport manifests as PDF, scanned image, Excel, CSV or JSON. Several files can be uploaded at once, each assigned to its leg.
- **Asynchronous processing.** Scanned documents are processed in the background (`202 Accepted`), so large manifests never hit request timeouts.
- **Human in the loop.** Low-confidence extractions are marked for review. The dispatcher corrects them in the dashboard before any offer reaches a driver, and edits flow through to the dispatched jobs (re-geocoded and re-routed).

### Privacy-first, on-premise parsing
- **OpenCV preprocessing:** grayscale, `fastNlMeansDenoising`, Otsu binarization.
- **Multi-language Tesseract:** `eng+deu+ara`, with fallback.
- **Self-hosted LLM (Ollama, Llama 3):** turns OCR text into structured vehicle and address data. Invalid output is sent back to the model with its validation errors to correct itself.
- **Deterministic safeguards:** validation rules and tiered address fallbacks stop malformed extractions from reaching routing and dispatch.
- Document parsing runs entirely within your own infrastructure; no document leaves the server for parsing.

### Multi-leg relay dispatch
- **One batch per leg**, ordered by sequence. Leg 2+ opens to drivers only once the previous leg is covered.
- **Broadcast waves:** offers go out in waves, round-robin over the vehicles still open.
- **Interactive role choice:** drivers tap **Drive**, **Passenger / Co-driver** or **Reject** on Telegram inline buttons.
- **No double assignment:** each vehicle is claimed with a single atomic database operation, so when two drivers tap at once, exactly one gets the car.
- **Return-leg (backhaul) matching** within a dispatch session.
- **Reliable webhooks:** duplicate deliveries are detected and ignored, so a retried message never runs a claim twice.
- **Pluggable channels:** a `DriverNotifier` interface with Telegram and WhatsApp (Bird) implementations.

### Security and data protection
- **Zero-PII driver identity:** phone numbers are normalized and turned into irreversible keyed hashes (HMAC-SHA256) at the API boundary. Only the hash and a masked display form are stored. Raw numbers never reach the database, the cache or the logs.
- **Verified driver registration:** a driver links their chat by sharing their *own* Telegram contact. Nothing is inferred.
- **Automatic tenant isolation:** every database query is filtered by tenant (SQLAlchemy event listeners plus request-scoped context), and the database itself rejects links between different tenants' data.
- **Fail-closed webhooks:** Telegram and WhatsApp webhooks reject any call without the shared secret.
- **Hardened by default:**
  - Production refuses to start with placeholder or weak secrets.
  - Phone numbers and tokens are kept out of URLs and redacted from access logs.
  - Error responses never expose internals.
  - Request, upload and spreadsheet size limits, plus a zip-bomb guard.
- **Demo mode:** a configurable demo number (`DEMO_MAGIC_PHONE`) routes messages to a demo chat, for live presentations without real driver data.
- Built around data minimisation, in line with GDPR and the Saudi PDPL.

### Dispatcher dashboard
- Next.js 16 / TypeScript, in **English, German and Arabic**, with full right-to-left layout.
- Manifest upload, review and editing, per-leg broadcasts, live driver and vehicle status, and route maps.

---

## How it works

1. **Upload:** the dispatcher uploads one or more manifests and assigns legs.
2. **Extract:** structured files are imported directly; scans go through OpenCV, OCR and the local LLM, then geocoding and real-road routing.
3. **Review:** uncertain extractions wait for the dispatcher's approval.
4. **Broadcast:** Leg 1 offers go out to drivers in waves over Telegram (or WhatsApp).
5. **Claim:** the first driver to tap gets the vehicle, atomically. Others can join as passengers or pick another vehicle.
6. **Relay:** once a leg is covered, the next leg unlocks, and so on along the chain.

---

## 🗺️ Routing & geolocation
- **Routing:** OSRM for real-road geometry, distances and durations. Mapbox is supported as an alternative, with a straight-line fallback if neither responds.
- **Configuration (.env):**
  ```env
  ROUTING_PROVIDER=osrm
  # Public endpoint for quick evaluation:
  OSRM_BASE_URL=https://router.project-osrm.org/route/v1/driving
  # Or a self-hosted container for isolated environments:
  # OSRM_BASE_URL=http://osrm-backend:5000/route/v1/driving
  ```
- **Data isolation:** for high-compliance deployments, self-host OSRM so fleet coordinates never leave your infrastructure. Self-hosted geocoding is on the roadmap.

---

## Roadmap

Planned, not built yet:
- **Production deployment and observability:** containerized deployment, structured logging, error tracking and metrics.
- **Fully on-premise data services:** self-hosted geocoding and routing, so no address or coordinate leaves the deployment.
- **Live driver location:** Telegram live location per leg, with pickup and delivery confirmation and ETAs on the dashboard.
- **Route optimization solver:** driver-to-leg assignment and leg chaining under time windows, proposed to the dispatcher for confirmation.
- **More LLM backends:** an OpenAI-compatible inference option (e.g. vLLM) alongside Ollama.
- **High availability:** queue-based background workers, managed databases and horizontally scaled API instances.
- **WhatsApp Business at production scale** for Saudi Arabia and the UAE.

---

## Demo preview

> The dispatcher dashboard in action: instant language switching (English / RTL Arabic), manifest upload, wave-based driver broadcasts via Telegram, and live status tracking:

https://github.com/user-attachments/assets/f9bc4038-2838-4138-82be-354e70f04485

---

## Documentation

- [api_protocols.md](docs/api_protocols.md): webhook schemas, routing specs, error codes and event specs. Includes the target architecture, which goes beyond the current build.
- [db_schema.md](docs/db_schema.md): PostgreSQL schema, the route-leg state machine, session-based parallel dispatch, and multi-tenant isolation rules.

---

## Sample ingestion document

A synthetic bilingual (English/Arabic) transport manifest (`local_order_ksa.png`), of the kind the OCR and LLM pipeline turns into vehicles (Hyundai, Toyota) and Riyadh-based addresses:

![Synthetic bilingual transport manifest](docs/samples/local_order_ksa.png)

*(All company names, Commercial Registration (CR) numbers and addresses in this sample are synthetic and were generated solely for R&D demonstration.)*

---

## Tech stack

- **Backend:** Python 3.12, FastAPI, async SQLAlchemy 2.0, Alembic, Pydantic v2, PostgreSQL (JSONB), Redis.
- **AI / parsing (on-premise):** OpenCV, Tesseract (`eng+deu+ara`), Ollama (Llama 3).
- **Messaging:** Telegram Bot API, WhatsApp via Bird.
- **Routing:** OSRM (self-hosted or public), Mapbox (optional), Nominatim geocoding.
- **Frontend:** Next.js 16, React, TypeScript, Tailwind CSS, next-intl (EN/DE/AR, RTL).
- **Infrastructure:** Docker, Docker Compose.
