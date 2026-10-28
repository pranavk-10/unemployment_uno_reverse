# OutreachEngine — System Architecture

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Architecture Specification)  
**Author:** Principal Systems Architecture Team  
**Last Updated:** 2026-10-28  

---

## 1. System Context & High-Level Architecture

OutreachEngine is a modern, decoupled asynchronous web application. It connects human operators to external AI models and communication APIs via a resilient, transactional backend.

```mermaid
C4Context
    title System Context Diagram - OutreachEngine

    Person(user, "Outreach Operator / Job Seeker", "Conducts personalized networking campaigns and reviews AI drafts.")
    System(outreach_engine, "OutreachEngine Platform", "Manages profiles, contact verification, AI drafting, and safe Gmail dispatch.")
    
    System_Ext(gemini, "Google Gemini AI", "Generates context-aware, hyper-personalized draft copy.")
    System_Ext(gmail, "Google Gmail API", "Dispatches MIME emails and polls for thread replies.")
    System_Ext(sheets, "Google Sheets API", "Reads raw prospect lists and writes real-time status updates.")
    System_Ext(dns, "DNS MX Nameservers", "Resolves mail exchange records for contact verification.")

    Rel(user, outreach_engine, "Interacts via Web UI (Vercel SPA)", "HTTPS")
    Rel(outreach_engine, gemini, "Synthesizes email copy", "HTTPS / JSON")
    Rel(outreach_engine, gmail, "Dispatches messages & polls threads", "OAuth2 / REST")
    Rel(outreach_engine, sheets, "Synchronizes contacts & status", "OAuth2 / REST")
    Rel(outreach_engine, dns, "Checks MX records", "DNS UDP/53")
```

---

## 2. Component Topology & Service Boundaries

The platform is split into four primary tiers:

1. **Client Tier (Frontend)**:
   - Built with React 18, Vite, and modern CSS conforming to the Watermelon UI design system.
   - Deployed as a static single-page application (SPA) on Vercel with global edge CDN caching.
   - Communicates with the backend exclusively via HTTPS JSON REST APIs.
2. **API Tier (FastAPI Web Service)**:
   - Stateless asynchronous ASGI web service running on Render.
   - Handles authentication, CRUD operations for profiles and contacts, draft reviews, campaign orchestration, and tracking pixel ingestion.
   - Uses Pydantic v2 for request validation and response serialization.
3. **Data Tier (PostgreSQL Database)**:
   - Managed PostgreSQL 15+ hosted on Render.
   - Acts as the single source of truth for all relational entities, audit logs, and the **transactional dispatch queue**.
4. **Execution Tier (Background Worker)**:
   - Dedicated headless Python daemon running on Render.
   - Continuously evaluates scheduled campaigns, acquires database-backed lease locks (`SELECT ... FOR UPDATE SKIP LOCKED`), enforces daily caps, applies randomized jitter, and executes third-party API transmissions.

---

## 3. Detailed Data Flow Architecture

The end-to-end data flow from prospect ingestion to message delivery and reply detection is illustrated below:

```mermaid
sequenceDiagram
    autonumber
    actor User as Operator
    participant UI as Watermelon UI (SPA)
    participant API as FastAPI Web Service
    participant DB as PostgreSQL 16
    participant Worker as Background Worker
    participant Gemini as Google Gemini API
    participant Gmail as Gmail API v1
    participant Sheets as Google Sheets

    User->>UI: Import Prospects from Sheet URL
    UI->>API: POST /api/v1/contacts/sync-sheets
    API->>Sheets: Fetch Rows (A2:G50)
    Sheets-->>API: Contact Rows
    API->>API: Run Verification (RFC Syntax & DNS MX Check)
    API->>DB: Batch Insert Contacts (Status: VERIFIED)
    API-->>UI: Return Imported & Verified Contacts

    User->>UI: Click "Generate AI Drafts"
    UI->>API: POST /api/v1/campaigns/{id}/generate-drafts
    API->>Gemini: Prompt with Resume + Contact Context
    Gemini-->>API: Structured JSON (Subject, Body)
    API->>API: Run Placeholder & Safety Token Check
    API->>DB: Insert Message (Status: DRAFT_PENDING_REVIEW)
    API-->>UI: Drafts Ready for Review

    User->>UI: Review & Edit Draft, Click "Approve"
    UI->>API: POST /api/v1/messages/{id}/approve
    API->>DB: Update Message (Status: SCHEDULED, scheduled_for = T + Jitter)
    API-->>UI: Message Scheduled Confirmation

    loop Every 5 Seconds (Worker Polling)
        Worker->>DB: SELECT * FROM messages WHERE scheduled_for <= NOW() FOR UPDATE SKIP LOCKED
        DB-->>Worker: Claim Message Row (Status: SENDING)
        Worker->>Gmail: users.messages.send (MIME with Tracking Headers)
        Gmail-->>Worker: HTTP 200 OK (gmail_message_id, thread_id)
        Worker->>DB: UPDATE messages SET status = 'SENT', gmail_id = ...
        Worker->>Sheets: Writeback Row (Status: "SENT", Timestamp)
    end

    opt Remote Tracking Event
        Gmail->>API: GET /api/v1/telemetry/pixel/{tracking_id}.png
        API->>DB: Record OPEN Event in message_events
    end

    loop Periodic Reply Scan (Every 5 Minutes)
        Worker->>Gmail: users.threads.get (thread_id)
        Gmail-->>Worker: Thread Messages
        Worker->>DB: If contact replied: UPDATE messages SET status = 'REPLIED'
    end
```

---

## 4. Message Lifecycle State Machine

A message strictly adheres to the following deterministic finite state transitions:

```mermaid
stateDiagram-v2
    [*] --> DRAFT_PENDING_REVIEW: AI Synthesized
    
    DRAFT_PENDING_REVIEW --> DRAFT_INVALID_PLACEHOLDER: Failed Token Check
    DRAFT_INVALID_PLACEHOLDER --> DRAFT_PENDING_REVIEW: User Fixes Tokens

    DRAFT_PENDING_REVIEW --> REJECTED: User Discards
    REJECTED --> [*]

    DRAFT_PENDING_REVIEW --> APPROVED: User Approves
    APPROVED --> SCHEDULED: Scheduled into Active Window
    
    SCHEDULED --> CANCELLED: Campaign Paused / Cancelled
    CANCELLED --> SCHEDULED: Campaign Resumed

    SCHEDULED --> SENDING: Worker Claims Lease Lock
    
    SENDING --> SENT: Gmail API Confirms 200 OK
    SENDING --> FAILED: Fatal API Error (e.g., Auth Revoked)
    SENDING --> SCHEDULED: Transient Failure (Retry with Backoff)

    SENT --> OPENED_ESTIMATED: Pixel Requested
    SENT --> REPLIED: Inbound Thread Reply Detected
    OPENED_ESTIMATED --> REPLIED: Inbound Thread Reply Detected
    
    SENT --> BOUNCED: Delivery Status Notification (NDR)
    
    REPLIED --> [*]
    BOUNCED --> [*]
    FAILED --> [*]
```

---

## 5. Campaign Scheduling & Execution Engine

```mermaid
flowchart TD
    Start([Worker Polling Tick - 5s]) --> QueryDB{Query SCHEDULED messages<br/>where scheduled_for <= NOW<br/>and campaign ACTIVE}
    
    QueryDB -- No messages --> Sleep[Sleep 5 Seconds] --> Start
    QueryDB -- Found Message --> Lock[Acquire DB Lock:<br/>SELECT FOR UPDATE SKIP LOCKED]
    
    Lock --> CheckCap{Check Daily Cap:<br/>Sent Today < Max Limit?}
    CheckCap -- Exceeded --> Reschedule[Reschedule to Next Business Day 09:00] --> CommitSkip[Commit DB & Release Lock] --> Start
    
    CheckCap -- Under Limit --> CheckSupp{Is Contact Suppressed?}
    CheckSupp -- Yes --> MarkSupp[Mark Message SUPPRESSED] --> CommitSkip
    
    CheckSupp -- No --> MarkSending[Update Status to SENDING]
    MarkSending --> Dispatch[Call Gmail API users.messages.send]
    
    Dispatch --> Result{API Response}
    Result -- 200 OK --> MarkSent[Record gmail_message_id<br/>Set Status to SENT]
    MarkSent --> SheetsWrite[Write Status to Google Sheet]
    SheetsWrite --> ReleaseLock[Commit Transaction] --> Start

    Result -- 429 Rate Limit --> PauseCamp[Pause Campaign<br/>Backoff 15m] --> ReleaseLock
    Result -- 401 Auth Revoked --> MarkAuthFail[Flag Integration NEEDS_REAUTH<br/>Mark Message FAILED] --> ReleaseLock
    Result -- 5xx Network Timeout --> CheckSentBeforeRetry{Query Gmail Thread<br/>Did message actually send?}
    CheckSentBeforeRetry -- Yes --> MarkSent
    CheckSentBeforeRetry -- No --> Requeue[Increment Retry Count<br/>Set Backoff Delay] --> ReleaseLock
```

---

## 6. Security Boundaries & Token Isolation

1. **OAuth Credentials**:
   - Google client secrets and user refresh tokens are stored in PostgreSQL using **AES-256 (Fernet) symmetric encryption**.
   - The encryption key is injected via environment variables at runtime (`FERNET_KEY`) and is never accessible to the frontend client.
2. **Session Authentication**:
   - Web API endpoints are protected using short-lived JWT access tokens stored in `HttpOnly`, `Secure`, `SameSite=Strict` cookies.
3. **Least-Privilege Scopes**:
   - OutreachEngine restricts its Google API scopes exclusively to `gmail.send`, `gmail.readonly`, and `spreadsheets`. It never requests full inbox modification (`gmail.modify`) or Google Drive root access.

---

## 7. High Availability & Failure Recovery

- **Stateless Web Services**: The FastAPI service holds zero in-memory state; any web instance can process any API call or tracking pixel request.
- **Worker Crash Resilience**: If the background worker terminates abruptly while a message is in `SENDING` status, the lease timestamp expires after 5 minutes. The subsequent worker run will inspect the stale lease, verify delivery state with the Gmail API, and either mark the message `SENT` or re-queue it safely without duplicate sends.
