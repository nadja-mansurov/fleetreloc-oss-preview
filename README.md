# fleetreloc-oss-preview

> **Disclaimer:** This repository is an independent, personal R&D portfolio project developed strictly on my own time, using my own personal equipment, and without any relation, contribution, or connection to any current or past employers. All concepts, code structures, and architectures presented here are autonomous technical explorations and do not utilize proprietary employer resources or trade secrets.

Public engineering preview of **FleetReloc**, a specialized B2B SaaS platform designed to coordinate bulk vehicle-transport orders (10–30 vehicle fleets), manage transient driver workflows via WhatsApp, and optimize multi-stop routing for the GCC logistics market.

> **Note:** This is a public architectural skeleton and reference implementation. The proprietary business logic, core optimization engines, and production security layers remain private.

---

## Architecture Highlights

* **Monorepo Structure:** Built with a decoupled architecture featuring a FastAPI backend (`apps/api`) and a Next.js 16 / TypeScript frontend (`apps/web`).
* **Zero-PII Data Compliance:** Designed for strict privacy standards; driver phone numbers and session mapping are handled ephemerally via Redis, keeping persistent databases clear of permanent personal data.
* **Concurrency & Safety:** Leverages Redis Redlock to ensure race-condition-free, First-Come-First-Served (FCFS) driver job-claiming mechanics over WhatsApp webhooks.
* **Resilient Routing & Ingestion:** Hybrid architecture combining Google OR-Tools / OSRM fallbacks for vehicle routing problem (VRP) optimization and structured OCR text parsing supporting both enterprise cloud LLMs and self-hosted local inference runtimes (e.g., Ollama / vLLM) for zero-data-retention compliance.

---

## Core Documentation

* [`api_protocols.md`](docs/api_protocols.md) — Authoritative contracts covering WABA webhooks, routing schemas, error codes, and real-time event specs.
* [`db_schema.md`](docs/db_schema.md) — PostgreSQL / Supabase DDL definitions, state machines, and multi-tenant RLS isolation rules.

---

## Tech Stack

* **Backend:** Python 3.11+, FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, Redis (Redlock).
* **AI / Parsing:** OpenAI API / Self-hosted Local LLM Nodes (Ollama/vLLM compatible).
* **Frontend:** Next.js, React, Tailwind CSS, TypeScript.
* **Infrastructure:** Docker, Supabase (PostgreSQL), OSRM.