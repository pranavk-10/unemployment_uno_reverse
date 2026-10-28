# OutreachEngine

> **Intelligent, Verified Outbound Networking & Email Outreach Platform**  
> Powered by FastAPI, PostgreSQL, Gemini AI, Gmail API, and Google Sheets.

---

## Overview

**OutreachEngine** is a developer-grade, AI-assisted cold outreach and professional networking platform designed to run highly personalized, verified email campaigns with strict rate limiting, robust anti-spam compliance, and transparent tracking.

Whether conducting job-search outreach, professional networking, or partnership development, OutreachEngine provides:
- **Profile & Resume Context Engine**: Ingests your professional background, target roles, and value proposition.
- **AI-Powered Draft Personalization**: Uses Google Gemini to generate contextual, tailored outreach drafts adhering to sender-defined tone and boundaries.
- **Two-Stage Human-in-the-Loop Review**: Ensures every AI draft is validated, reviewed, and approved before scheduling.
- **Gmail API Direct Integration**: Sends messages directly through authenticated user Gmail accounts via OAuth 2.0 with minimal scopes, jittered intervals, and daily quota safeguards.
- **Bidirectional Google Sheets Sync**: Imports contacts directly from Google Sheets and writes real-time outreach status, timestamps, and metrics back to external spreadsheets.
- **Privacy-First Tracking & Suppression**: Non-invasive tracking with transparent open/reply heuristics and strict global suppression list enforcement.
- **Watermelon-Inspired UI**: A modern, high-contrast, accessible user interface with real-time campaign health telemetry, metrics cards, donut charts, and granular message inspection.

---

## Project Status

| Area | Current Status | Description |
|---|---|---|
| **Documentation** | **VERIFIED** | Comprehensive architecture, PRD, API reference, schemas, and UI/UX specs completed. |
| **Backend Core** | **PROPOSED** | FastAPI application structure, SQLAlchemy models, and Alembic migrations. |
| **Background Worker** | **PROPOSED** | Database-backed polling and scheduling worker for message dispatch. |
| **Integrations** | **PROPOSED** | OAuth2 flows for Gmail, Gemini API client, Google Sheets API v4. |
| **Frontend UI** | **PROPOSED** | Modern dashboard with campaign builder, contact manager, and telemetry. |

*Note: In accordance with our documentation-first engineering standard, all functional capabilities are explicitly tagged as `[PROPOSED]`, `[IMPLEMENTED]`, or `[VERIFIED]`. No feature is documented as operational until backed by passing automated and end-to-end tests.*

---

## Documentation Index

The complete, authoritative product and engineering documentation is maintained in the [`docs/`](file:///c:/Users/hp/Desktop/Projects/email/docs/README.md) directory:

- [**Documentation Index (`docs/README.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/README.md) — Main entry point and recommended reading paths.
- [**Product Requirements Document (`docs/PRD.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/PRD.md) — Vision, user personas, functional requirements (FR-xxx), and acceptance criteria.
- [**User Guide (`docs/USER_GUIDE.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/USER_GUIDE.md) — Step-by-step end-user guide for connecting integrations and running campaigns.
- [**Technical Documentation (`docs/TECHNICAL_DOCUMENTATION.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/TECHNICAL_DOCUMENTATION.md) — Deep-dive developer manual, execution flows, and module design.
- [**UI/UX Specification (`docs/UI_UX_SPECIFICATION.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/UI_UX_SPECIFICATION.md) — Component inventory, Watermelon UI styling tokens, and interaction flows.
- [**Compatibility Matrix (`docs/COMPATIBILITY.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/COMPATIBILITY.md) — Verified and supported versions across OS, runtime, and third-party APIs.
- [**User Stories & Test Cases (`docs/USER_STORIES.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/USER_STORIES.md) — Traceable epics, user stories (US-xxx), and Given/When/Then acceptance criteria.
- [**System Architecture (`docs/ARCHITECTURE.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/ARCHITECTURE.md) — Component topology, service boundaries, and security perimeters.
- [**Database Schema (`docs/DATABASE_SCHEMA.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/DATABASE_SCHEMA.md) — Relational data model, indexes, state machines, and idempotency keys.
- [**API Reference (`docs/API_REFERENCE.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/API_REFERENCE.md) — Complete REST API contract with synthetic request and response payloads.
- [**Security & Privacy (`docs/SECURITY_AND_PRIVACY.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/SECURITY_AND_PRIVACY.md) — Threat model, OAuth scope restrictions, token encryption, and compliance controls.
- [**Testing Strategy (`docs/TESTING_STRATEGY.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/TESTING_STRATEGY.md) — Unit, integration, idempotency, and mock adapter test suites.
- [**Deployment Guide (`docs/DEPLOYMENT_GUIDE.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/DEPLOYMENT_GUIDE.md) — Step-by-step instructions for Render (backend/worker) and Vercel (frontend).
- [**Troubleshooting Guide (`docs/TROUBLESHOOTING.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/TROUBLESHOOTING.md) — Diagnostic playbooks, error catalogs, and recovery steps.
- [**Architecture Decision Records (`docs/decisions/`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/decisions/README.md) — Historical logs of architectural trade-offs and decisions.
- [**Mermaid Architecture Diagrams (`docs/diagrams/`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/diagrams/README.md) — Editable system architecture, data flow, and message lifecycle diagrams.
- [**Changelog (`docs/CHANGELOG.md`)**](file:///c:/Users/hp/Desktop/Projects/email/docs/CHANGELOG.md) — Version history and milestone tracking.

---

## Quick Architecture Summary

```text
[ Browser / Frontend (Vercel) ]
               │  HTTPS / REST
               ▼
[ FastAPI Application (Render Web Service) ] ──▶ [ PostgreSQL Database (Render) ]
               │                                            ▲
               │ Enqueues / Updates                         │ Polls / Updates state
               ▼                                            │
[ Background Dispatch Worker (Render Worker) ] ─────────────┘
       ├── Google Gemini API (Personalization Engine)
       ├── Gmail API (OAuth2 Message Dispatch & Thread Polling)
       └── Google Sheets API v4 (Bidirectional Contact/Status Sync)
```

---

## License

Internal proprietary project. All rights reserved.
