# FleetReloc OSS Preview

Disclaimer: This repository is an independent, personal R&D portfolio project developed strictly on my own time, using my own personal equipment, and without any relation, contribution, or connection to any current or past employers. All concepts, code structures, and architectures presented here are autonomous technical explorations and do not utilize proprietary employer resources or trade secrets.

Public architectural preview and core boilerplate of **FleetReloc** — a specialized **Messenger-first SaaS framework designed for GCC** car rental networks, leasing operators, and fleet managers to automate intra-city vehicle rotation, recovery/flatbed dispatching, and transient driver workflows (running seamlessly on Telegram for R&D/MVP and architected for WhatsApp Business API).

> Commercial Notice: This open-source repository contains only the security core, ingestion boundary skeletons, and UI boilerplate. The proprietary optimization heuristics, fine-tuned parsing models, production worker clusters, and enterprise billing/tenant layers are proprietary and maintained in a private core repository.

---

## Architectural Highlights

### Core System & Frontend
- Monorepo Structure: Decoupled architecture featuring a FastAPI backend (`apps/api`) and a Next.js 16 / TypeScript frontend (`apps/web`).
- Bilingual & RTL-Ready Workflows: Native support for mixed Arabic/English B2B communications, including Right-to-Left (RTL) payload parsing and multilingual NLP extractors for webhook events.

### Security, Identity & Data Sovereignty
- Zero-Trust Identity & Hashing Core: Strict data ingestion pipeline mapping raw contacts into irreversible HMAC-SHA256 driver_hash identifiers at the API boundary for full GDPR/PDPL compliance, ensuring no raw PII ever hits persistent storage.
- Ephemeral Redis Storage: Driver phone numbers and session mapping are handled ephemerally via Redis with strict TTL, keeping persistent databases clear of permanent personal data.
- Demo Magic Number Routing: Isolated configuration override (`DEMO_MAGIC_PHONE`) for safe live presentations and automated end-to-end testing.
- Enterprise Multi-Tenancy & Data Isolation: Automated database-level query filtering and context-bound tenant separation (via SQLAlchemy event listeners and request-scoped context variables) to ensure strict data sovereignty and zero cross-tenant leaks for enterprise clients.

### Dispatch, Routing & Concurrency
- Parallel Optimization Sessions & Multi-File Ingestion: Supports independent, parallel batch sessions per tenant, enabling dispatchers to upload multiple manifest files concurrently and group them into isolated routing streams without cross-contamination.
- Iterative Bulk Batch & Broadcast Waves: Relocation orders structured into BulkBatch and BroadcastWave models, distributing notifications iteratively via hashed arrays.
- Concurrency & Safety: Leverages Redis Redlock to ensure race-condition-free, First-Come-First-Served (FCFS) driver job-claiming mechanics over webhooks.
- Resilient Routing & Ingestion: Hybrid architecture combining routing problem (VRP) optimization with structured text parsing supporting both enterprise cloud LLMs and self-hosted local inference runtimes (e.g., Ollama / vLLM) optimized for multilingual (Arabic/English) document parsing.
- Messenger-First Workflow & Pluggable Notifications: Built around a messenger-first user experience where field drivers receive tasks and interact via chat without installing heavy apps. The decoupled engine (DriverNotifier interface) currently powers fast MVP iteration and live demos via Telegram Bot API, while being architecturally primed to scale instantly to WhatsApp Business API for production deployments across Saudi Arabia and the UAE.
---

## How It Works: Zero-Trust Ingestion & Dispatch Flow

FleetReloc demonstrates how to solve a core logistical challenge: coordinating multi-vehicle transport orders while maintaining absolute compliance with regional data privacy laws (GDPR/PDPL) and zero-trust security standards.

1. The GCC Fleet Challenge: Managing intra-city fleet rotations (moving cars between rental branches, service centers, and airports) and dispatching local recovery/flatbed units currently relies on manual chaos, phone calls, and fragmented WhatsApp chats, leading to high vehicle downtime and human errors.
2. Multi-File Manifest Ingestion: Dispatchers can upload multiple transport orders and manifest files concurrently, routing them into dedicated, isolated optimization sessions (e.g., separating distinct regional hubs or parallel shift schedules).
3. Zero-Trust Normalization & Hashing (`identity.py`):
   - *No PII Persistence:* Raw contact strings are intercepted at the API boundary, normalized, and mapped into a salted, irreversible driver_hash (`HMAC-SHA256`).
   - *Database Safety:* PostgreSQL stores only the driver_hash and relational statuses. Raw phone numbers or handles are never written to persistent storage or application logs.
4. Iterative Broadcast Waves & Route Legs: Orders are broken down into granular route segments (RouteLeg) across independent session batches, distributing notifications iteratively across hashed arrays while supporting real-time dispatcher overrides.
5. Race-Condition-Free Claiming (FCFS): When a driver responds via webhook, Redis Redlock coordinates atomic slot-claiming, ensuring fairness without race conditions.

---

## 🗺️ Routing & Geolocation
FleetReloc features dynamic route visualization and distance calculation. 
- **Default Provider:** By default, the application connects to the open-source OSRM routing engine to fetch real-road geometries, distances, and ETAs.
- **Configuration (.env):** Easily switchable via environment variables depending on the deployment environment:
  ```env
  ROUTING_PROVIDER=osrm
  # Public endpoint used for quick out-of-the-box evaluations:
  OSRM_BASE_URL=[https://router.project-osrm.org/route/v1/driving](https://router.project-osrm.org/route/v1/driving)
  # Or local Docker container for isolated environments:
  # OSRM_BASE_URL=http://osrm-backend:5000/route/v1/driving
  ```
- **Security & Data Isolation:** For high-compliance environments (such as GDPR or GCC PDPL guidelines), deploying a self-hosted containerized OSRM instance ensures that fleet coordinates and telemetry data never leave the local infrastructure.

---

## Demo Preview

> Watch how the FleetReloc dispatcher dashboard works in action—featuring instant language switching (English / RTL Arabic), transport manifest uploading, iterative driver broadcasts via Telegram, and real-time status tracking:

https://github.com/user-attachments/assets/c6e52b01-9377-46be-96ed-e82c47fd0cca



---

## Core Documentation

- api_protocols.md — Authoritative contracts covering webhook schemas, routing specs, error codes, and real-time event specs.
- db_schema.md — PostgreSQL DDL definitions, route segment state machines (RouteLeg), session-based parallel stream controls, and multi-tenant isolation rules.

---

## Sample Ingestion Document

Here is a synthetic example of a bilingual (English/Arabic) transport manifest (`local_order_ksa.png`) processed by the OCR and LLM pipeline to automatically extract batch payloads, vehicles (Hyundai, Toyota), and Riyadh-based coordinates:

*(All company names, Commercial Registration (CR) numbers, and addresses shown in the sample above are entirely synthetic and generated solely for R&D demonstration purposes).*

---

## Tech Stack

- Backend: Python 3.11+, FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, Redis (Redlock).
- Security & Identity: HMAC-SHA256 zero-trust normalization, ephemeral demo-mode routing fixtures, strict PII-free persistence guards.
- AI / Parsing: OpenAI API / Self-hosted Local LLM Nodes (Ollama / vLLM compatible) with multilingual Arabic/English extraction pipelines.
- Frontend: Next.js, React, Tailwind CSS, TypeScript (RTL/BiDi layout support).
- Infrastructure: Docker, PostgreSQL (with JSONB and array-based broadcast wave tracking), OSRM (Self-hosted / Public routing engine).
