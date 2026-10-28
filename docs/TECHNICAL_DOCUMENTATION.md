# OutreachEngine — Technical Documentation & Engineering Manual

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (Architecture Specification)  
**Author:** Principal Software Architecture & Engineering Team  
**Last Updated:** 2026-10-28  

---

## 1. Architectural Philosophy & Technology Stack

OutreachEngine is engineered around three core technical tenets:
1. **Deterministic State & Idempotency**: Network operations (especially email dispatch) are inherently fallible. Every transmission must be strictly idempotent and guarded by database-backed lease locks to eliminate duplicate sends.
2. **Decoupled Asynchronous Processing**: Time-consuming actions (AI synthesis, external API calls, reply polling) must never block the REST API request-response loop.
3. **Auditability & Observability**: Every transition in the message lifecycle must be persisted with structured metadata, timestamps, and error traces.

### 1.1 Technology Stack & Version Constraints

| Layer | Technology | Version | Architectural Rationale |
|---|---|---|---|
| **Language Runtime** | Python | `>= 3.11` | High-performance async support (`asyncio`, ExceptionGroups), native typing enhancements, and broad ecosystem support for Google client libraries. |
| **API Framework** | FastAPI | `>= 0.110.0` | High-throughput asynchronous routing, automatic OpenAPI 3.1 schema generation, native Pydantic v2 data validation, and dependency injection. |
| **Validation Engine** | Pydantic | `>= 2.6.0` | Rust-backed validation core, strict schema enforcement, fast JSON serialization. |
| **ORM & DB Toolkit** | SQLAlchemy | `>= 2.0.28` | Full async ORM support via `asyncpg`, robust transaction management, and explicit typing. |
| **Migration Engine** | Alembic | `>= 1.13.1` | Declarative, version-controlled database schema migrations with automated upgrade/downgrade scripts. |
| **Primary Database** | PostgreSQL | `>= 15.0` | Row-level locking with `FOR UPDATE SKIP LOCKED`, JSONB querying, transactional integrity, and native UUID generation. |
| **AI SDK** | `google-genai` | `>= 0.1.1` | Unified modern SDK for Google Gemini models (`gemini-2.5-flash`, `gemini-1.5-pro`), structured outputs, and token counting. |
| **Google APIs** | `google-api-python-client` / `google-auth` | `>= 2.28.0` | Official Google client libraries for Gmail API v1 and Google Sheets API v4. |
| **Frontend Framework**| React / Vite / TypeScript | Node `>= 20.x`, React `>= 18.2` | Clean component-driven architecture, fast HMR dev server, strict type checking, and Watermelon UI aesthetic tokens. |
| **Production Hosting**| Render (Backend/Worker) & Vercel (Frontend) | Standard Cloud | Managed PostgreSQL, decoupled web/worker service separation on Render, edge CDN distribution on Vercel. |

---

## 2. System Architecture & Component Boundaries

The OutreachEngine topology consists of four distinct execution boundaries:

```text
               +-------------------------------------------------------+
               |                  Browser Client                       |
               |       React 18 / Vite / Vanilla CSS (Watermelon UI)   |
               +-------------------------------------------------------+
                                           │
                                           │ HTTPS (Bearer JWT / Session Cookie)
                                           ▼
+--------------------------------------------------------------------------------------+
|                           FastAPI Application (Render Web)                           |
|                                                                                      |
|  [ Auth Router ]      [ Campaigns Router ]    [ Contacts Router ]   [ Telemetry Router]
|         │                      │                      │                     │        |
|         ▼                      ▼                      ▼                     ▼        |
|  [ OAuth2 Service ]   [ Campaign Service ]   [ Ingestion Service ]  [ Analytics Svc ]|
+--------------------------------------------------------------------------------------+
                                           │
                                           ▼
               +-------------------------------------------------------+
               |             PostgreSQL 15+ (Render Managed)           |
               |  - users, integration_tokens, profiles, contacts      |
               |  - campaigns, messages, message_events, suppression   |
               +-------------------------------------------------------+
                                           ▲
                                           │ Database-Backed Lease Locks
                                           │ (SELECT ... FOR UPDATE SKIP LOCKED)
+--------------------------------------------------------------------------------------+
|                     Background Dispatch Worker (Render Worker)                       |
|                                                                                      |
|  [ Polling Daemon ] ──▶ [ Scheduler Loop ] ──▶ [ Jitter & Quota Evaluator ]          |
|                                                     │                                |
|         ┌───────────────────────────────────────────┴────────────────────────────┐   |
|         ▼                                           ▼                            ▼   |
|  [ Gemini AI Client ]                     [ Gmail API Dispatcher ]        [ Sheets Sync ]
|  (Draft Synthesis)                        (MIME Build & Send)             (Status Write)
+--------------------------------------------------------------------------------------+
```

### 2.1 Backend Architecture & Directory Structure

```text
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI application entry point, middleware, lifecycle
│   ├── core/
│   │   ├── config.py            # Pydantic BaseSettings, environment variable loading
│   │   ├── security.py          # Token encryption (Fernet AES-256), hashing, JWT logic
│   │   ├── database.py          # Async SQLAlchemy engine, session maker, base model
│   │   └── logging.py           # Structured JSON logger configuration
│   ├── models/                  # SQLAlchemy ORM models
│   │   ├── user.py
│   │   ├── credential.py
│   │   ├── profile.py
│   │   ├── contact.py
│   │   ├── campaign.py
│   │   ├── message.py
│   │   └── suppression.py
│   ├── schemas/                 # Pydantic DTOs for request/response serialization
│   │   ├── auth.py
│   │   ├── campaign.py
│   │   ├── contact.py
│   │   ├── message.py
│   │   └── telemetry.py
│   ├── api/                     # REST API versioned endpoints
│   │   ├── v1/
│   │   │   ├── api.py           # Main APIRouter aggregation
│   │   │   ├── endpoints/
│   │   │   │   ├── auth.py      # Google OAuth handshake, token refresh
│   │   │   │   ├── profiles.py  # User profile & resume context CRUD
│   │   │   │   ├── contacts.py  # Contact import, CSV/Sheets sync, verification
│   │   │   │   ├── campaigns.py # Campaign configuration & lifecycle controls
│   │   │   │   ├── messages.py  # Review queue, draft approval, editing
│   │   │   │   └── telemetry.py # Metrics, donut chart data, recent feeds
│   ├── services/                # Business logic & domain abstractions
│   │   ├── verification.py      # RFC syntax, DNS MX validation, disposable domain filter
│   │   ├── ai_generator.py      # Gemini API prompt building, token checking, generation
│   │   ├── gmail_sender.py      # MIME encoding, exponential backoff, users.messages.send
│   │   ├── sheets_service.py    # Google Sheets reading and row update batching
│   │   ├── scheduler.py         # Slot calculation, randomized jitter allocation
│   │   └── reply_detector.py    # Thread history polling and reply state matching
│   └── worker/                  # Background worker daemon entry point
│       ├── daemon.py            # Async polling worker event loop
│       └── tasks.py             # Transactional unit of work handlers
├── alembic/                     # Database migration scripts
│   ├── env.py
│   └── versions/
├── tests/                       # Automated test suites
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   └── mocks/
├── pyproject.toml
└── requirements.txt
```

---

## 3. Execution Flows & Module Responsibilities

### 3.1 Python Entry Point & Lifecycle (`backend/app/main.py`)
The application uses FastAPI's asynchronous `lifespan` context manager:
1. **Startup**:
   - Initialize structured logging.
   - Verify PostgreSQL connectivity and validate database schema revision via Alembic head check.
   - Initialize HTTP client connection pools (`httpx.AsyncClient`).
   - Validate encryption keys (`SECRET_KEY`, `FERNET_KEY`).
2. **Shutdown**:
   - Gracefully drain active HTTP connections.
   - Dispose of SQLAlchemy async engine pools.

### 3.2 Background Worker Design (`backend/app/worker/daemon.py`)
Why a database-backed worker instead of Celery/Redis?
- **Decision Rationale**: Running Redis + Celery adds infrastructure overhead and an additional point of failure. PostgreSQL 15+'s `FOR UPDATE SKIP LOCKED` allows the worker to atomically poll, claim, and lock scheduled message rows directly in PostgreSQL with zero external broker dependencies.
- **Worker Execution Loop**:
  1. **Poll**: Every 5 seconds, query `messages` where `status = 'SCHEDULED'` and `scheduled_for <= NOW()` and `campaign.status = 'ACTIVE'`.
  2. **Acquire Lease**: Execute `SELECT id FROM messages WHERE ... FOR UPDATE SKIP LOCKED LIMIT 1`.
  3. **Transition to `SENDING`**: Update message status to `SENDING` within an atomic transaction.
  4. **Evaluate Daily Caps & Jitter**: Check total messages sent today by this sender. If within limit, dispatch via Gmail API.
  5. **Commit Outcome**: If successful, record `gmail_message_id`, `gmail_thread_id`, set status to `SENT`, and write back to Google Sheets. If transient error, apply backoff and reset to `SCHEDULED`. If fatal (e.g., account suspended), mark `FAILED`.

---

## 4. Third-Party Integrations Deep Dive

### 4.1 Google Gemini API (`services/ai_generator.py`)
- **Model Target**: Default `gemini-2.5-flash` via the official `google-genai` SDK.
- **Prompt Structure**:
  - **System Instructions**: Imparts the persona, tone, safety constraints, and absolute prohibition against fabricating personal relations or unmentioned achievements.
  - **Context Ingestion**: Structured JSON containing user's key career achievements, target contact's name, company, title, and public notes.
  - **Output Schema**: Structured JSON response containing `subject` (string) and `body_markdown` (string).
- **Post-Generation Sanitization**:
  - Scans generated draft using regex for dangling tokens (`{{...}}`, `[...]`, `TODO`, `REPLACE`). If found, marks draft as `DRAFT_INVALID_PLACEHOLDER` for human intervention.

### 4.2 Gmail API Integration (`services/gmail_sender.py`)
- **OAuth Scopes**:
  - `https://www.googleapis.com/auth/gmail.send`
  - `https://www.googleapis.com/auth/gmail.readonly`
  - `https://www.googleapis.com/auth/userinfo.email`
- **MIME Building**:
  - Constructs `multipart/alternative` MIME messages (text/plain and text/html).
  - Injects custom tracking headers: `X-OutreachEngine-Message-ID` and an optional 1x1 transparent tracking pixel.
- **Quota Handling**:
  - Free Gmail: Max 500 recipients/24h.
  - Workspace: Max 2,000 recipients/24h.
  - OutreachEngine defaults to a conservative maximum of 50 sends/day to preserve sender domain reputation.

### 4.3 Google Sheets Integration (`services/sheets_service.py`)
- **API**: Google Sheets API v4 via `google-api-python-client`.
- **Batch Processing**:
  - Reads contact ranges in chunks (e.g., `A2:G500`).
  - Implements atomic batch update requests (`batchUpdate`) when updating status columns to conserve Google API write quotas (max 300 write requests per minute per project).

---

## 5. Idempotency, Concurrency, & Retry Strategy

### 5.1 Idempotency Guarantee
To prevent duplicate emails under any crash or network partition scenario:
1. Each message has an `idempotency_key` derived from `hash(campaign_id, contact_id, version)`.
2. When the background worker transmits a message to Gmail, it maintains the open transaction. If the network times out before receiving Gmail's response:
   - Worker queries Gmail API thread history using the unique message subject and RFC Message-ID to verify if the message actually dispatched.
   - If found, it marks the local record as `SENT` with the recovered Gmail ID.
   - If not found, it is retried with exponential backoff.

### 5.2 Retry Policy
- **Transient Network Errors (HTTP 500, 502, 503, 504)**: Exponential backoff with full jitter:
  $$\text{Delay} = \min(300, 2^{\text{attempt}} \times 5) + \text{Uniform}(0, 5)$$
- **Rate Limit Errors (HTTP 429 / 403 `quotaExceeded`)**: Immediately sleep for 15 minutes and pause campaign dispatch.
- **Permanent Errors (HTTP 400 Bad Request, 401 Unauthorized)**: Mark message as `FAILED`, pause campaign, and flag integration for user re-authentication.

---

## 6. Configuration & Environment Variables

All settings are strongly typed via Pydantic (`BaseSettings`) and loaded from environment variables:

```bash
# Core Environment
ENVIRONMENT=development                       # development | staging | production
LOG_LEVEL=INFO                                # DEBUG | INFO | WARNING | ERROR
SECRET_KEY=generate_with_openssl_rand_hex_32 # Used for internal JWT session tokens
FERNET_KEY=generate_with_cryptography_fernet  # AES-256 base64 key for encrypting OAuth tokens

# Database Configuration
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/outreachengine_dev
DATABASE_POOL_SIZE=10
DATABASE_MAX_OVERFLOW=20

# Google OAuth & APIs
GOOGLE_CLIENT_ID=your-google-oauth-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-google-oauth-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/api/v1/auth/google/callback

# Gemini AI Configuration
GEMINI_API_KEY=AIzaSy...your-gemini-api-key
GEMINI_DEFAULT_MODEL=gemini-2.5-flash

# Worker & Safety Guardrails
WORKER_POLL_INTERVAL_SECONDS=5
MAX_GLOBAL_DAILY_SENDS=50
DEFAULT_BASE_DELAY_SECONDS=60
DEFAULT_MIN_JITTER_SECONDS=15
DEFAULT_MAX_JITTER_SECONDS=90

# Frontend Configuration (for CORS)
CORS_ORIGINS=["http://localhost:5173", "https://outreachengine.vercel.app"]
```

---

## 7. Local Development & Testing Instructions

### 7.1 Local Setup
```bash
# 1. Clone repository and navigate to workspace
cd c:\Users\hp\Desktop\Projects\email

# 2. Set up Python virtual environment
python -m venv .venv
.venv\Scripts\activate   # Windows
# source .venv/bin/activate # Linux/Mac

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
cp .env.example .env
# Fill in your database and API keys

# 5. Run Database Migrations
alembic upgrade head

# 6. Start the API Server
uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload

# 7. Start the Background Dispatch Worker (in separate terminal)
python -m app.worker.daemon
```

### 7.2 Running Tests
```bash
# Run complete test suite
pytest -v

# Run with coverage report
pytest --cov=app --cov-report=term-missing tests/

# Run integration tests only
pytest tests/integration/ -m "not slow"
```

---

## 8. Extension Points & Known Limitations

1. **Multi-User Multi-Tenancy**: The current schema supports user isolation via `user_id` foreign keys, but multi-user organization sharing (teams, workspaces) is slated for Milestone 2.
2. **Apple Mail Privacy Protection (MPP)**: Remote pixel tracking is inherently imperfect. Technical metrics must always be labeled as *estimates*.
3. **Gmail Send-as Aliases**: OutreachEngine currently dispatches from the authenticated primary Google address; alias support (`sendAs` settings) is an extension point.
