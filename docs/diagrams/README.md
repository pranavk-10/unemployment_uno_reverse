# OutreachEngine — Architecture Diagrams Catalog

**Directory Status:** **`[VERIFIED]`** (Editable Mermaid Diagram Sources)  
**Syntax Standard:** Mermaid.js v10+  
**Last Updated:** 2026-10-28  

---

## 1. Overview

This directory contains the authoritative, editable Mermaid source files (`.mmd`) for all system architecture, execution data flow, state machines, and concurrency routines in OutreachEngine.

These source files can be rendered using:
- Visual Studio Code / IDE Markdown and Mermaid preview extensions.
- [Mermaid Live Editor](https://mermaid.live/).
- Automated documentation generators (e.g. GitHub native rendering, VitePress, or MkDocs).

---

## 2. Diagram Inventory

| File | Diagram Type | System Area Represented |
|---|---|---|
| [**system-architecture.mmd**](system-architecture.mmd) | C4 Context & Container | End-to-end service boundaries, SPA client, FastAPI API, PostgreSQL, Worker, and external Google APIs. |
| [**data-flow.mmd**](data-flow.mmd) | Sequence Diagram | Complete sequence from Google Sheets prospect import to Gemini draft generation, human review, Gmail send, and telemetry recording. |
| [**message-lifecycle.mmd**](message-lifecycle.mmd) | State Machine (v2) | Complete finite state machine transitions for messages (`DRAFT_PENDING_REVIEW` through `REPLIED` / `FAILED`). |
| [**campaign-execution.mmd**](campaign-execution.mmd) | Flowchart (TD) | Background worker polling loop, database lease locking (`SKIP LOCKED`), daily cap enforcement, jitter delay, and failure handling. |

---

## 3. Maintenance Guidelines

Whenever system boundaries, background worker polling logic, external API integrations, or state transitions change in code:
1. Update the corresponding `.mmd` file in this directory.
2. Verify Mermaid syntax in [Mermaid Live Editor](https://mermaid.live/) or IDE preview.
3. Synchronize the embedded Mermaid blocks in [docs/ARCHITECTURE.md](../ARCHITECTURE.md).
4. Record the update in [docs/CHANGELOG.md](../CHANGELOG.md).
