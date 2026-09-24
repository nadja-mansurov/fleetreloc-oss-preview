# FleetReloc OSS Preview

Disclaimer: This repository is an independent, personal R&D portfolio project developed strictly on my own time, using my own personal equipment, and without any relation, contribution, or connection to any current or past employers. All concepts, code structures, and architectures presented here are autonomous technical explorations and do not utilize proprietary employer resources or trade secrets.

Public architectural preview and core boilerplate of FleetReloc — a specialized technical framework designed for coordinating bulk vehicle-transport orders (10–30 vehicle fleets), managing transient driver workflows via webhooks, and optimizing multi-stop routing tailored for the GCC logistics market.

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

### Dispatch, Routing & Concurrency
- Iterative Bulk Batch & Broadcast Waves: Relocation orders structured into BulkBatch and BroadcastWave models, distributing notifications iteratively via hashed arrays.
- Concurrency & Safety: Leverages Redis Redlock to ensure race-condition-free, First-Come-First-Served (FCFS) driver job-claiming mechanics over webhooks.
- Resilient Routing & Ingestion: Hybrid architecture combining routing problem (VRP) optimization with structured text parsing supporting both enterprise cloud LLMs and self-hosted local inference runtimes (e.g., Ollama / vLLM) optimized for multilingual (Arabic/English) document parsing.
- Pluggable Notification Architecture: Decoupled multi-channel driver notification engine built using the Strategy and Factory patterns (`DriverNotifier` interface). Seamlessly switches between Telegram Bot API (for local R&D and rapid MVP testing) and WhatsApp Business API (for GCC production environments) without altering core booking or dispatch logic.

---

## How It Works: Zero-Trust Ingestion & Dispatch Flow

FleetReloc demonstrates how to solve a core logistical challenge: coordinating multi-vehicle transport orders while maintaining absolute compliance with regional data privacy laws (GDPR/PDPL) and zero-trust security standards.

1. Manifest Ingestion & Parsing: Dispatchers upload transport orders (manifests, spreadsheets, or images via OCR) containing vehicle details and driver contact identifiers.
2. Zero-Trust Normalization & Hashing (`identity.py`):
   - *No PII Persistence:* Raw contact strings are intercepted at the API boundary, normalized, and mapped into a salted, irreversible driver_hash (`HMAC-SHA256`).
   - *Database Safety:* PostgreSQL stores only the driver_hash and relational statuses. Raw phone numbers or handles are never written to persistent storage or application logs.
3. Iterative Broadcast Waves: Orders are grouped into a BulkBatch (e.g., 10–30 vehicles) and distributed in structured waves targeting hashed arrays iteratively.
4. Race-Condition-Free Claiming (FCFS): When a driver responds via webhook, Redis Redlock coordinates atomic slot-claiming, ensuring fairness without race conditions.

---

## Core Documentation

- api_protocols.md — Authoritative contracts covering webhook schemas, routing specs, error codes, and real-time event specs.
- db_schema.md — PostgreSQL DDL definitions, state machines, and multi-tenant RLS isolation rules.

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
- Infrastructure: Docker, PostgreSQL (with JSONB and array-based broadcast wave tracking), OSRM.
