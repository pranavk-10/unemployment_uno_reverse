# OutreachEngine — Changelog

All notable changes to the OutreachEngine project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.1.0-draft] - 2026-10-28

### Added
- **Documentation Suite (Phase 0 Architecture Specification)**:
  - Root `README.md` providing project overview, status indicators, and documentation links.
  - `docs/README.md` master documentation index with reading paths and status framework.
  - `docs/PRD.md` Product Requirements Document defining functional (`FR-xxx`) and non-functional (`NFR-xxx`) requirements, user personas, and success metrics.
  - `docs/USER_GUIDE.md` end-user walkthrough for Gmail OAuth, Gemini AI setup, Sheets mapping, review queue, and telemetry.
  - `docs/TECHNICAL_DOCUMENTATION.md` detailed engineering manual outlining FastAPI, background worker daemon, idempotency, and retry strategies.
  - `docs/UI_UX_SPECIFICATION.md` design specification for Watermelon UI palette, typography, metrics cards, donut charts, and micro-interactions.
  - `docs/COMPATIBILITY.md` verified compatibility matrix across Python 3.11+, Node 20+, PostgreSQL 16, FastAPI, SQLAlchemy, and Google client libraries.
  - `docs/USER_STORIES.md` testable user stories across 14 epics with Given/When/Then acceptance criteria and negative failure scenarios.
  - `docs/ARCHITECTURE.md` system topology, C4 context diagrams, sequence data flow, and message lifecycle state machines.
  - `docs/DATABASE_SCHEMA.md` complete relational data model covering tables (`users`, `integration_tokens`, `sender_profiles`, `contacts`, `campaigns`, `messages`, `message_events`, `suppression_entries`), column types, foreign keys, indexes, and migration procedures.
  - `docs/API_REFERENCE.md` OpenAPI 3.1 REST API reference with synthetic payloads for authentication, profiles, contacts, campaigns, review queue, and telemetry.
  - `docs/SECURITY_AND_PRIVACY.md` threat model (STRIDE), OAuth scope restrictions, AES-256 Fernet token encryption at rest, and anti-spam suppression enforcement.
  - `docs/TESTING_STRATEGY.md` testing pyramid, mock adapters (`MockGmailClient`, `MockGeminiClient`, `MockSheetsClient`), database testcontainers, and CI guidelines.
  - `docs/DEPLOYMENT_GUIDE.md` infrastructure blueprint (`render.yaml`) for Render web service, worker, PostgreSQL, and Vercel frontend.
  - `docs/TROUBLESHOOTING.md` diagnostic runbook for database connection, worker polling, OAuth revocation, Gemini quota, and CORS errors.
  - `docs/decisions/README.md` Architecture Decision Records index and initial ADRs (ADR-0001 through ADR-0007).
  - `docs/diagrams/` editable Mermaid source files (`system-architecture.mmd`, `data-flow.mmd`, `message-lifecycle.mmd`, `campaign-execution.mmd`).

### Changed
- Initialized documentation repository structure.

### Security
- Defined mandatory encryption at rest for OAuth tokens using Fernet AES-256.
- Enforced strict least-privilege OAuth scopes (`gmail.send`, `gmail.readonly`, `userinfo.email`).
- Prohibited full inbox modification scopes (`gmail.modify`) and Google Drive root access.

### Known Limitations
- Core application code is in `[PROPOSED]` state pending implementation phase.
- Live email dispatch is disabled by default via `MOCK_GMAIL_DISPATCH` safety flag until verification tests pass.
