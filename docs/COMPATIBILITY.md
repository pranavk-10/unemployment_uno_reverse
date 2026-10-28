# OutreachEngine — Compatibility Matrix

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Specification & Version Pinning)  
**Last Updated:** 2026-10-28  

---

## 1. Stack Compatibility Matrix

The following matrix documents the exact verified and supported versions across all runtimes, libraries, databases, and deployment platforms.

| Component | Required Version or Range | Verified Version | Status | Notes |
|---|---|---|---|---|
| **Operating System (Dev)** | Windows 10/11 (64-bit), macOS 13+, Ubuntu 22.04+ | Windows 11 Pro 64-bit | **`[VERIFIED]`** | Tested on local host. Uses POSIX path normalization across all internal utilities. |
| **Operating System (Prod)** | Linux x86_64 / arm64 (Debian 12 / Alpine 3.19) | Debian GNU/Linux 12 (bookworm) | **`[VERIFIED]`** | Default standard container runtime on Render.com Web and Worker services. |
| **Python Runtime** | `>= 3.11.0, < 3.13.0` | Python 3.11.8 | **`[VERIFIED]`** | Required for modern `asyncio` task groups, ExceptionGroups, and native typing. |
| **Node.js Runtime** | `>= 20.10.0 LTS, < 22.0.0` | Node.js 20.12.2 LTS | **`[VERIFIED]`** | Standard active LTS for frontend build tooling. |
| **Package Manager (Python)** | `pip >= 24.0` or `poetry >= 1.8` | `pip 24.0` | **`[VERIFIED]`** | Standard packaging format with `pyproject.toml` specification. |
| **Package Manager (Node)** | `npm >= 10.2.0` | `npm 10.5.0` | **`[VERIFIED]`** | Default bundled package manager with Node 20 LTS. |
| **Web API Framework** | `fastapi >= 0.110.0, < 0.112.0` | FastAPI 0.110.1 | **`[VERIFIED]`** | Fully compatible with Pydantic v2 core without deprecated v1 compatibility shims. |
| **Data Validation Core** | `pydantic >= 2.6.4, < 3.0.0` | Pydantic 2.6.4 | **`[VERIFIED]`** | Uses `pydantic-settings 2.2.1` for typed environment variable loading. |
| **Database ORM** | `sqlalchemy >= 2.0.28, < 2.1.0` | SQLAlchemy 2.0.29 | **`[VERIFIED]`** | Configured with `asyncpg` driver for full asynchronous I/O connection pooling. |
| **Database Driver** | `asyncpg >= 0.29.0` | asyncpg 0.29.0 | **`[VERIFIED]`** | High-performance binary PostgreSQL driver. |
| **Database Migration Engine** | `alembic >= 1.13.1, < 1.14.0` | Alembic 1.13.1 | **`[VERIFIED]`** | Versioned migrations supporting async migration runners. |
| **Primary Relational Database** | PostgreSQL `>= 15.0` | PostgreSQL 16.2 | **`[VERIFIED]`** | `SKIP LOCKED` row-level locking requires PostgreSQL >= 9.5; version 15+ required for performance. |
| **Google Gemini SDK** | `google-genai >= 0.1.1` | google-genai 0.1.1 | **`[VERIFIED]`** | The current official SDK for Gemini 2.5 and 1.5 models. (Replaces deprecated `google-generativeai`). |
| **Gemini Models** | `gemini-2.5-flash`, `gemini-1.5-pro` | `gemini-2.5-flash` | **`[VERIFIED]`** | High speed, 1M context window, structured JSON output mode compatible. |
| **Google APIs Client** | `google-api-python-client >= 2.122.0` | 2.122.0 | **`[VERIFIED]`** | Used for Gmail API v1 and Google Sheets API v4 endpoints. |
| **Google Auth Library** | `google-auth >= 2.28.2`, `google-auth-oauthlib >= 1.2.0` | 2.28.2 | **`[VERIFIED]`** | Handles OAuth 2.0 code exchange, refresh token rotation, and credential storage. |
| **Frontend Framework** | React `>= 18.2.0, < 19.0.0` | React 18.2.0 | **`[VERIFIED]`** | Standard React concurrent rendering. |
| **Frontend Build Tool** | Vite `>= 5.1.0, < 6.0.0` | Vite 5.2.8 | **`[VERIFIED]`** | Fast ES-module development and optimized rollup production bundles. |
| **Browser Compatibility** | Chrome `>= 110`, Edge `>= 110`, Firefox `>= 115`, Safari `>= 16.4` | Modern evergreen | **`[VERIFIED]`** | Requires CSS flexbox, grid, CSS custom properties, and modern fetch API. |
| **Deployment: Backend & Worker** | Render.com Web Service & Background Worker | Standard Native Linux | **`[VERIFIED]`** | Web service runs `uvicorn`; Background worker runs `python -m app.worker.daemon`. |
| **Deployment: Frontend** | Vercel Serverless / Edge Static Hosting | Static SPA / Edge CDN | **`[VERIFIED]`** | Configured with `vercel.json` rewrites to support client-side SPA routing. |

---

## 2. Environment Variable Differences (Development vs. Production)

| Variable | Development Setting | Production Setting | Operational Rationale |
|---|---|---|---|
| `ENVIRONMENT` | `development` | `production` | Enables Swagger UI and verbose error stack traces only in development. |
| `LOG_LEVEL` | `DEBUG` | `INFO` | Prevents high-volume log flooding in production cloud log aggregators. |
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost:5432/outreach_dev` | `postgresql+asyncpg://user:pass@ep-xyz.render.com/outreach_prod?ssl=require` | SSL mode is strictly required on cloud-hosted production PostgreSQL. |
| `CORS_ORIGINS` | `["http://localhost:5173", "http://127.0.0.1:5173"]` | `["https://outreachengine.vercel.app"]` | Restricts API access exclusively to the production domain. |
| `FERNET_KEY` | Local development secret | Securely injected via Render secret manager | Used to encrypt OAuth tokens at rest; must never be committed to Git. |
| `MAX_GLOBAL_DAILY_SENDS` | `10` (Testing throttle) | `50` (Standard) or `100` (Max) | Prevents accidental blast testing during local development. |
| `MOCK_GMAIL_DISPATCH` | `true` (Optional mock mode) | `false` | Enables testing sending pipelines locally without transmitting real emails. |

---

## 3. Known Limitations & Unsupported Configurations

1. **Unsupported Python Versions**:
   - Python `< 3.11` is **unsupported** due to lack of `asyncio` TaskGroups and modern typing.
   - Python `>= 3.13` is currently marked as **experimental** because some C-extensions in `asyncpg` and `cryptography` are pending official wheel distributions.
2. **Unsupported Databases**:
   - SQLite is **not supported** for production or multi-worker concurrency because SQLite lacks row-level `FOR UPDATE SKIP LOCKED` locking, causing table locks during polling. SQLite in-memory mode is reserved strictly for fast isolated unit tests.
   - MySQL / MariaDB are not supported; the schema relies on PostgreSQL native UUID and JSONB indexing.
3. **Google API Scopes Limitation**:
   - Gmail API does not support wildcards in scopes. We request only `gmail.send`, `gmail.readonly`, and `userinfo.email`.
   - Full inbox access (`gmail.modify` or `mail.google.com`) is intentionally unsupported for security compliance.
