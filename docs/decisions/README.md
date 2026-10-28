# Architecture Decision Records (ADRs)

**Directory Status:** **`[VERIFIED]`** (Architecture Governance)  
**Process Standard:** Michael Nygard ADR Format  
**Last Updated:** 2026-10-28  

---

## 1. ADR Governance & Lifecycle

Architecture Decision Records (ADRs) capture significant architectural decisions, along with the context, trade-offs, and operational consequences. 

Every record moves through the following lifecycle states:
- `PROPOSED`: Under review and debate.
- `ACCEPTED`: Approved by architecture leads and adopted for implementation.
- `SUPERSEDED`: Replaced by a subsequent numbered decision record.
- `DEPRECATED`: No longer applicable.

---

## 2. ADR Index

| ADR ID | Title | Date | Status | Impact Areas |
|---|---|---|---|---|
| [**ADR-0001**](#adr-0001-choice-of-fastapi-and-python-311-for-backend-api) | Choice of FastAPI & Python 3.11+ for Backend API | 2026-10-28 | **ACCEPTED** | API Tier, Async Concurrency, Type Safety |
| [**ADR-0002**](#adr-0002-choice-of-postgresql-and-sqlalchemyalembic-for-data-tier) | Choice of PostgreSQL & SQLAlchemy/Alembic for Data Tier | 2026-10-28 | **ACCEPTED** | Data Model, Migrations, Idempotency |
| [**ADR-0003**](#adr-0003-database-backed-scheduling-worker-via-skip-locked) | Database-Backed Scheduling Worker via `FOR UPDATE SKIP LOCKED` | 2026-10-28 | **ACCEPTED** | Task Queue, Concurrency, Reliability |
| [**ADR-0004**](#adr-0004-minimal-privilege-google-oauth-scopes) | Minimal-Privilege Google OAuth Scopes | 2026-10-28 | **ACCEPTED** | Security, Compliance, User Trust |
| [**ADR-0005**](#adr-0005-choice-of-google-gemini-ai-for-contextual-draft-synthesis) | Choice of Google Gemini AI for Contextual Draft Synthesis | 2026-10-28 | **ACCEPTED** | AI Modeling, Safety Prompts, Generation |
| [**ADR-0006**](#adr-0006-separation-of-frontend-vercel-and-backend-render) | Decoupled Separation of Frontend (Vercel) & Backend (Render) | 2026-10-28 | **ACCEPTED** | Infrastructure, Deployment, Edge CDN |
| [**ADR-0007**](#adr-0007-google-sheets-as-a-collaborative-sync-layer) | Google Sheets as a Collaborative Sync Layer | 2026-10-28 | **ACCEPTED** | External Integrations, Status Sync |

---

## ADR-0001: Choice of FastAPI and Python 3.11+ for Backend API

### Context
OutreachEngine requires an asynchronous backend API capable of handling fast concurrent I/O requests (Google OAuth redirects, tracking pixel hits, sheet synchronization, and database polling). The ecosystem for Google APIs (`google-genai`, `google-api-python-client`) has first-class Python support.

### Decision
We adopted **FastAPI** with **Python 3.11+** as the core backend framework.

### Alternatives Considered
- *Django / Django Ninja*: Full-featured ORM and batteries included, but heavier footprint and unnecessary overhead for our headless, decoupled SPA architecture.
- *Node.js / Express or NestJS*: Excellent async performance, but Python provides superior native client SDK support for Google Gemini AI and data analysis.

### Consequences
- **Positive**: Native Pydantic v2 data validation, automated OpenAPI 3.1 schema documentation, high async throughput with `uvicorn`, and unified Python ecosystem for AI.
- **Negative**: Requires strict discipline around async database connections (`asyncpg`) to avoid accidental blocking calls.

---

## ADR-0002: Choice of PostgreSQL and SQLAlchemy/Alembic for Data Tier

### Context
Email campaign delivery requires strict transactional consistency, durable state management, and robust row-level concurrency primitives to prevent duplicate email transmissions under worker restarts.

### Decision
We chose **PostgreSQL 15+** managed on Render, paired with **SQLAlchemy 2.0 (Async)** and **Alembic** for schema migrations.

### Alternatives Considered
- *MongoDB / Document DB*: Flexible schema for contact attributes, but lacks multi-row ACID transactions and native `SKIP LOCKED` row queueing.
- *SQLite*: Simple local file, but lacks row-level locking concurrency and crashes under multi-worker write contention.

### Consequences
- **Positive**: Native UUID primary keys, JSONB for parsed skills and event metadata, strict foreign key constraints, and declarative migration tracking.
- **Operational**: Requires running migrations (`alembic upgrade head`) as part of the production release pipeline.

---

## ADR-0003: Database-Backed Scheduling Worker via `SKIP LOCKED`

### Context
Scheduled emails must be dispatched within designated sending windows with randomized jitter delays. We evaluated whether to introduce an external message broker (Redis + Celery or RabbitMQ) vs leveraging PostgreSQL.

### Decision
We implemented a **database-backed task queue** using PostgreSQL's native `SELECT ... FOR UPDATE SKIP LOCKED` primitive inside a dedicated background worker daemon.

### Alternatives Considered
- *Celery + Redis*: Standard industry pattern, but adds operational complexity, extra cloud hosting costs ($15–$30/mo for managed Redis), and risks state desynchronization between Redis task state and database message records.
- *Cron Jobs (Render Cron)*: Lacks sub-minute scheduling resolution; cannot enforce 45–180 second randomized jitter intervals.

### Consequences
- **Positive**: Zero external broker dependencies; message state is single-source-of-truth in PostgreSQL; atomic lease locking prevents duplicate sends across worker restarts.
- **Performance**: Polling interval of 5 seconds on indexed `(status, scheduled_for)` creates negligible CPU/IO load on PostgreSQL.

---

## ADR-0004: Minimal-Privilege Google OAuth Scopes

### Context
Users are highly cautious about connecting third-party tools to their personal Gmail accounts. Requesting broad mailbox permissions causes user drop-off and triggers extensive Google Security App Verification audits.

### Decision
OutreachEngine restricts requested scopes strictly to:
- `https://www.googleapis.com/auth/gmail.send`
- `https://www.googleapis.com/auth/gmail.readonly`
- `https://www.googleapis.com/auth/userinfo.email`
- `https://www.googleapis.com/auth/spreadsheets`

Full inbox modification (`gmail.modify`) and Google Drive root access are explicitly prohibited.

### Consequences
- **Positive**: Maximizes user trust, reduces security attack surface, and expedites Google Cloud OAuth consent verification.
- **Negative**: The app cannot programmatically label or move sent emails to custom Gmail folders (which would require `gmail.modify`).

---

## ADR-0005: Choice of Google Gemini AI for Contextual Draft Synthesis

### Context
Cold outreach requires natural, context-aware draft synthesis combining the sender's career highlights and recipient metadata without robotic templates.

### Decision
We adopted **Google Gemini 2.5 Flash** via the official `google-genai` SDK as the default personalization engine.

### Alternatives Considered
- *OpenAI GPT-4o-mini*: Highly capable, but Gemini 2.5 Flash provides equivalent contextual reasoning with superior cost-efficiency, lower latency, and seamless alignment with the Google Cloud / Workspace ecosystem.
- *Static Template Fillers (Jinja2)*: Zero AI hallucination risk, but produces impersonal emails that convert poorly.

### Consequences
- **Positive**: 1M token context window allows ingesting complete multi-page resumes and extensive recipient background; structured JSON mode guarantees schema conformity.
- **Mitigation**: Post-generation regex scanning checks for unparsed placeholder tokens (`{{...}}`, `TODO`), and mandatory human-in-the-loop review prevents hallucinations from being dispatched.

---

## ADR-0006: Separation of Frontend (Vercel) and Backend (Render)

### Context
We evaluated whether to serve the frontend as server-rendered templates from FastAPI or decouple the web frontend into a separate SPA deployed on an Edge CDN.

### Decision
We established a clean architectural separation:
- **Frontend**: React 18 / Vite SPA deployed on **Vercel**.
- **Backend & Worker**: FastAPI ASGI web service and Python daemon deployed on **Render**.

### Consequences
- **Positive**: Instant edge CDN loading globally for the UI; independent scaling and zero-downtime frontend releases; frontend developers work without local Python environments.
- **Operational**: Requires explicit CORS configuration (`CORS_ORIGINS`) and cookie security (`SameSite=None; Secure` in cross-domain configurations or proxy rewriting).

---

## ADR-0007: Google Sheets as a Collaborative Sync Layer

### Context
Non-technical operators, recruiters, and consultants frequently manage prospective leads in Google Sheets. Requiring manual CSV exports and re-imports causes friction and leads to stale data.

### Decision
We implemented native bidirectional synchronization with Google Sheets API v4. OutreachEngine reads prospect rows and writes back outreach status, timestamps, and message previews directly into designated sheet columns.

### Consequences
- **Positive**: Google Sheets acts as a live, collaborative CRM window for external team members without requiring extra OutreachEngine user licenses.
- **Operational**: Google Sheets API has a quota of 300 write requests per minute per project; status writes must be batched using `batchUpdate`.
