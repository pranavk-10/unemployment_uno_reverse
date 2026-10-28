# OutreachEngine Documentation Index

**Last Updated:** 2026-10-28  
**Documentation Version:** 1.0.0-draft  
**Implementation Phase:** Pre-Implementation / Architecture Specification (Phase 0)

---

## 1. Product Overview

**OutreachEngine** is an intelligent outbound email campaign management system designed for professionals, job-seekers, and business developers who require deeply personalized, verified networking workflows. Unlike bulk spam blast tools, OutreachEngine enforces a **quality-first, privacy-compliant, human-in-the-loop** paradigm:

1. **Structured Context Modeling**: Consolidates user profiles, resumes, target roles, and value propositions into a structured vector/prompt context.
2. **Deterministic Contact Verification**: Validates professional contact records (syntax, domain DNS MX records, disposable domain filtering) before any draft generation occurs.
3. **AI Personalization via Gemini**: Ingests recipient metadata (company, title, recent publications, mutual interests) and leverages Google Gemini to draft tailored outreach notes adhering to custom constraints (tone, length, call-to-action).
4. **Human Review & Guardrails**: Provides an approval interface to review, edit, or reject AI-generated drafts. No message leaves without human sign-off unless specifically marked for auto-dispatch under strict quota limits.
5. **Direct Gmail API Dispatch**: Operates through authenticated user Gmail accounts via OAuth 2.0 (`gmail.send`, `gmail.readonly`), applying random jitter intervals (45–180 seconds) and daily caps to preserve domain reputation and avoid spam categorization.
6. **Bidirectional Google Sheets Synchronization**: Reads contact lists from Google Sheets and writes real-time delivery status, open signals, and reply detections back to the user's spreadsheet.
7. **Transparent Telemetry & Watermelon UI**: Delivers a rich, accessible dashboard featuring metrics cards, donut charts, and recent-activity feeds without misleading or fabricated metrics.

---

## 2. Documentation Status Framework

To prevent documentation drift and false assertions of operational readiness, every document and subsystem in OutreachEngine uses standardized status indicators:

| Status Tag | Definition | Current Applicability |
|---|---|---|
| **`[PROPOSED]`** | Fully designed, specified, and architectural contract defined; no code yet implemented. | Core backend, frontend components, workers, and OAuth flow implementations. |
| **`[IMPLEMENTED]`** | Code has been written and merged into the working branch; pending comprehensive automated/E2E testing. | None (Repository initialized). |
| **`[VERIFIED]`** | Code is backed by automated unit, integration, or end-to-end test suites with verified passing runs. | Documentation specifications, schemas, architectural contracts, and directory structure. |
| **`[BLOCKED]`** | Implementation or verification is halted due to external dependencies, missing third-party credentials, or pending decisions. | Production OAuth client credentials (pending Google Cloud Console approval). |
| **`[DEPRECATED]`** | Obsolete interfaces retained strictly for backward compatibility or migration reference. | None. |

---

## 3. Master Documentation Directory

All documentation is maintained with strict relative linking. Below is the directory map and intended audience for each document:

| Document | Target Audience | Primary Focus & Scope | Current Status |
|---|---|---|---|
| [**PRD.md**](PRD.md) | Product Managers, Engineers, Stakeholders | Product vision, functional requirements (`FR-xxx`), non-functional requirements (`NFR-xxx`), MVP scope, and success metrics. | **`[VERIFIED]`** (Spec) |
| [**USER_GUIDE.md**](USER_GUIDE.md) | End Users, Campaign Operators, Testers | Step-by-step instructions for connecting Gmail, Gemini, Google Sheets, uploading contacts, reviewing drafts, and monitoring metrics. | **`[PROPOSED]`** (Workflow Guide) |
| [**TECHNICAL_DOCUMENTATION.md**](TECHNICAL_DOCUMENTATION.md) | Backend & Full-Stack Engineers, DevOps | Code architecture, module responsibilities, background worker loop, concurrency model, and retry/idempotency strategies. | **`[PROPOSED]`** (Design Doc) |
| [**UI_UX_SPECIFICATION.md**](UI_UX_SPECIFICATION.md) | Frontend Engineers, Designers | Design system tokens, Watermelon UI references, typography, metrics cards, donut charts, and micro-interactions. | **`[PROPOSED]`** (Design Spec) |
| [**COMPATIBILITY.md**](COMPATIBILITY.md) | DevOps, System Architects, QA | Exact runtime versions (Python 3.11+, Node 20+), third-party library constraints, API versions, and deployment hosts. | **`[VERIFIED]`** (Matrix) |
| [**USER_STORIES.md**](USER_STORIES.md) | QA, Engineers, Scrum Leads | 14 epics of detailed user stories (`US-xxx`) with Given/When/Then acceptance criteria and edge case coverage. | **`[VERIFIED]`** (Test Matrix) |
| [**ARCHITECTURE.md**](ARCHITECTURE.md) | System Architects, Lead Engineers | System topology, trust perimeters, database queue, polling loop, event processing, and failure isolation. | **`[VERIFIED]`** (Architecture) |
| [**DATABASE_SCHEMA.md**](DATABASE_SCHEMA.md) | Database Admins, Backend Engineers | Relational models, column definitions, foreign keys, unique constraints, index definitions, and state transitions. | **`[VERIFIED]`** (Schema Spec) |
| [**API_REFERENCE.md**](API_REFERENCE.md) | API Consumers, Frontend Engineers | OpenAPI-compliant REST API contracts, schemas, headers, status codes, and synthetic payloads. | **`[PROPOSED]`** (API Contract) |
| [**SECURITY_AND_PRIVACY.md**](SECURITY_AND_PRIVACY.md) | Security Auditors, Compliance, Devs | Threat modeling, OAuth scope minimization, token encryption at rest, anti-spam safeguards, and privacy handling. | **`[VERIFIED]`** (Policy) |
| [**TESTING_STRATEGY.md**](TESTING_STRATEGY.md) | QA Engineers, Backend Developers | Unit, integration, mock adapter harness, database transactions, idempotency tests, and CI/CD pipelines. | **`[PROPOSED]`** (Test Plan) |
| [**DEPLOYMENT_GUIDE.md**](DEPLOYMENT_GUIDE.md) | DevOps, System Administrators | Production deployment on Render (FastAPI + Worker) and Vercel (UI), environment variables, and zero-downtime migrations. | **`[PROPOSED]`** (Deployment Spec) |
| [**TROUBLESHOOTING.md**](TROUBLESHOOTING.md) | Support, SREs, End Users | Symptom-to-resolution playbooks for OAuth expiration, rate limits, Sheets permission errors, and worker stalls. | **`[VERIFIED]`** (Runbook) |
| [**CHANGELOG.md**](CHANGELOG.md) | All | Chronological log of versions, architectural changes, breaking changes, and bug fixes. | **`[VERIFIED]`** (Log) |
| [**decisions/README.md**](decisions/README.md) | Architecture Board, Developers | Architecture Decision Records (ADRs) capturing rationale, rejected alternatives, and consequences. | **`[VERIFIED]`** (ADR Index) |
| [**diagrams/README.md**](diagrams/README.md) | All Technical Stakeholders | Catalog and instructions for Mermaid architecture, data flow, and message lifecycle diagrams. | **`[VERIFIED]`** (Diagrams Index) |

---

## 4. Architecture Diagrams Index

Editable Mermaid source files are located in [`docs/diagrams/`](diagrams/README.md):
- [**System Architecture Diagram**](diagrams/system-architecture.mmd) — Service boundaries, frontend client, API router, worker, database, and external Google APIs.
- [**Data Flow Diagram**](diagrams/data-flow.mmd) — Step-by-step data movement from Sheet ingestion to Gemini inference, human approval, Gmail dispatch, and telemetry reporting.
- [**Message Lifecycle State Machine**](diagrams/message-lifecycle.mmd) — Finite state machine of an outreach message (`DRAFT` → `VALIDATED` → `APPROVED` → `SCHEDULED` → `SENDING` → `SENT` → `DELIVERED_ESTIMATED` → `OPENED` / `REPLIED` / `BOUNCED` / `FAILED`).
- [**Campaign Execution & Worker Polling**](diagrams/campaign-execution.mmd) — Background worker transactional lease locking, jitter evaluation, rate quota enforcement, and failure recovery.

---

## 5. Recommended Reading Order

To understand OutreachEngine systematically, we recommend reading documents in the following sequence based on your role:

### For Product Managers and Decision Makers
1. [**PRD.md**](PRD.md) — Comprehensive product requirements, user personas, and success metrics.
2. [**USER_STORIES.md**](USER_STORIES.md) — Acceptance criteria across all 14 epics.
3. [**USER_GUIDE.md**](USER_GUIDE.md) — Planned end-user experience and operational workflows.

### For Software Engineers & Contributors
1. [**PRD.md**](PRD.md) — High-level functional goals and constraints.
2. [**ARCHITECTURE.md**](ARCHITECTURE.md) — Top-level system design and security boundaries.
3. [**TECHNICAL_DOCUMENTATION.md**](TECHNICAL_DOCUMENTATION.md) — Runtime specifics, execution flows, and module breakdown.
4. [**DATABASE_SCHEMA.md**](DATABASE_SCHEMA.md) — Database tables, foreign keys, indexes, and state machines.
5. [**API_REFERENCE.md**](API_REFERENCE.md) — RESTful endpoints, request/response structures.
6. [**decisions/README.md**](decisions/README.md) — Historical ADRs for design choices.
7. [**TESTING_STRATEGY.md**](TESTING_STRATEGY.md) — Testing conventions and mock harness usage.

### For Frontend Developers & Designers
1. [**UI_UX_SPECIFICATION.md**](UI_UX_SPECIFICATION.md) — Design tokens, component inventory, and Watermelon UI guidelines.
2. [**API_REFERENCE.md**](API_REFERENCE.md) — Client-side data fetching contracts and error payloads.
3. [**USER_GUIDE.md**](USER_GUIDE.md) — Anticipated end-user journeys.

### For DevOps & SREs
1. [**COMPATIBILITY.md**](COMPATIBILITY.md) — System requirements and runtime versions.
2. [**DEPLOYMENT_GUIDE.md**](DEPLOYMENT_GUIDE.md) — Deployment instructions for Render and Vercel.
3. [**SECURITY_AND_PRIVACY.md**](SECURITY_AND_PRIVACY.md) — Token encryption, scopes, and data retention.
4. [**TROUBLESHOOTING.md**](TROUBLESHOOTING.md) — Failure diagnostics and runbooks.

---

## 6. Documentation Maintenance Workflow

Whenever architecture, code interfaces, or dependencies change:
1. Update the corresponding technical document in `docs/`.
2. Synchronize schemas in `docs/DATABASE_SCHEMA.md` and endpoints in `docs/API_REFERENCE.md`.
3. If user workflows change, update `docs/USER_GUIDE.md` and `docs/USER_STORIES.md`.
4. If dependencies or runtime environments change, update `docs/COMPATIBILITY.md`.
5. If architectural decisions are made or revised, record an ADR in `docs/decisions/`.
6. Log all modifications with date and rationale in `docs/CHANGELOG.md`.
7. Verify that all relative links resolve cleanly.
