# OutreachEngine — User Stories & Acceptance Criteria

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Specification & Test Traceability)  
**Last Updated:** 2026-10-28  

This document specifies the end-to-end user stories and testable acceptance criteria across all 14 core epics of OutreachEngine. All acceptance criteria follow the standard **Given / When / Then** BDD syntax.

---

## Epic 1: Contact Management

### US-CONT-01: Manual Contact Creation
- **Epic**: Contact Management
- **User Story**: As an outreach operator, I want to manually create and edit contact profiles with specific company and role details, so that I can add high-priority individual connections without importing a spreadsheet.
- **Rationale**: Not all outreach targets come from bulk lead lists; ad-hoc individual networking requires direct entry.
- **Priority**: P0
- **Preconditions**: User is authenticated and on the Contacts page.
- **Acceptance Criteria**:
  - **Given** valid contact details (First Name, Last Name, Email, Company, Job Title),  
    **When** the user clicks "Save Contact",  
    **Then** the contact is saved in PostgreSQL with a unique UUID and default status `UNVERIFIED`.
  - **Given** an invalid email syntax (e.g., `user@@example..com`),  
    **When** the user clicks "Save Contact",  
    **Then** the API returns HTTP 422 with a clear validation error, and no record is created.
- **Failure & Edge Cases**: Duplicate email address for the same user account rejected with HTTP 409 Conflict.
- **Related PRD ID**: `FR-CONT-001`
- **Related Test ID**: `TEST-CONT-01`

---

## Epic 2: Contact Verification

### US-VERIF-01: Automated DNS MX and Disposable Domain Verification
- **Epic**: Contact Verification
- **User Story**: As an outreach operator, I want the system to automatically verify the recipient's email syntax, MX record validity, and check for disposable domain providers, so that I avoid hard bounces that destroy my sender reputation.
- **Rationale**: Bounces trigger Google spam filters; pre-send verification protects domain health.
- **Priority**: P0
- **Preconditions**: Contact exists with status `UNVERIFIED`.
- **Acceptance Criteria**:
  - **Given** a contact with email `elena@validcompany.com` where the domain has active DNS MX records,  
    **When** the verification job executes,  
    **Then** the contact verification status updates to `VERIFIED` with MX timestamp recorded.
  - **Given** a contact with a disposable domain (e.g., `test@tempmail.com`),  
    **When** the verification job executes,  
    **Then** the contact status updates to `INVALID` with reason `DISPOSABLE_DOMAIN_REJECTED`.
  - **Given** a domain that has no DNS MX or A records,  
    **When** the verification job executes,  
    **Then** the contact status updates to `INVALID` with reason `NO_MX_RECORDS`.
- **Failure & Edge Cases**: DNS resolver timeout handles gracefully by retrying once before marking `VERIFICATION_TIMEOUT`.
- **Related PRD ID**: `FR-VERIF-001`
- **Related Test ID**: `TEST-VERIF-01`

---

## Epic 3: AI Personalization

### US-AI-01: Context-Grounded Draft Generation via Gemini
- **Epic**: AI Personalization
- **User Story**: As a job-seeker, I want Gemini AI to synthesize my resume highlights with the target contact's role and company context to draft a tailored outreach message, so that my emails are compelling and relevant.
- **Rationale**: Manual personalization takes hours; generic templates convert poorly.
- **Priority**: P0
- **Preconditions**: User has an active Resume Profile; target contact is `VERIFIED`.
- **Acceptance Criteria**:
  - **Given** a verified contact and user resume context,  
    **When** the user clicks "Generate AI Draft",  
    **Then** the Gemini API returns a structured JSON payload with `subject` and `body_markdown`, and a message record is created with status `DRAFT_PENDING_REVIEW`.
  - **Given** the system prompt strictness rules,  
    **When** draft generation occurs,  
    **Then** the AI draft never fabricates mutual acquaintances or unmentioned past roles.
- **Failure & Edge Cases**: Gemini API rate limit or outage returns an informative error to the user without dropping contact state.
- **Related PRD ID**: `FR-AI-001`
- **Related Test ID**: `TEST-AI-01`

---

## Epic 4: Draft Validation & Human Review

### US-VAL-01: Placeholder Token Detection & Human Approval Queue
- **Epic**: Draft Validation
- **User Story**: As an outreach operator, I want all AI drafts scanned for unparsed placeholder tokens and placed into an interactive review queue, so that I can inspect, edit, and approve them before any sending occurs.
- **Rationale**: Unparsed tokens (e.g., `Hello {{FirstName}}`) ruin credibility; human-in-the-loop oversight is mandatory.
- **Priority**: P0
- **Preconditions**: Message exists in `DRAFT_PENDING_REVIEW`.
- **Acceptance Criteria**:
  - **Given** an AI draft containing raw placeholders like `{{company}}` or `[Insert achievement]`,  
    **When** token validation runs,  
    **Then** the message status is flagged as `DRAFT_INVALID_PLACEHOLDER` with offending tokens highlighted in the UI.
  - **Given** a valid draft in `DRAFT_PENDING_REVIEW`,  
    **When** the user edits the body and clicks "Approve",  
    **Then** the message transitions to `APPROVED` and becomes eligible for scheduling.
- **Failure & Edge Cases**: User attempts to approve a draft with invalid placeholders; system prevents approval until tokens are resolved.
- **Related PRD ID**: `FR-AI-003`, `FR-REV-001`
- **Related Test ID**: `TEST-REV-01`

---

## Epic 5: Campaign Scheduling

### US-SCHED-01: Schedule Windows, Daily Caps, & Jitter Allocation
- **Epic**: Campaign Scheduling
- **User Story**: As an outreach operator, I want to define active sending days, hours, daily volume caps, and randomized delay jitter, so that my dispatch patterns mirror authentic human behavior.
- **Rationale**: Burst sending flags accounts as spam bots.
- **Priority**: P0
- **Preconditions**: Campaign exists with active schedule configuration.
- **Acceptance Criteria**:
  - **Given** an approved message and a campaign schedule of Mon–Fri 09:00–17:00 with 60s base delay + 30s jitter,  
    **When** the message is scheduled,  
    **Then** `scheduled_for` is calculated within the active window with a randomized jitter offset.
  - **Given** a daily limit of 30 emails and 30 emails have already been sent today,  
    **When** scheduling additional messages,  
    **Then** they are queued for the start of the next business day's window.
- **Failure & Edge Cases**: Campaign paused mid-day; all remaining scheduled messages for that day remain untouched in `SCHEDULED` status until resumed.
- **Related PRD ID**: `FR-SCHED-001`
- **Related Test ID**: `TEST-SCHED-01`

---

## Epic 6: Gmail Sending & API Dispatch

### US-GMAIL-01: Direct Dispatch via Gmail API with Idempotency
- **Epic**: Gmail Sending
- **User Story**: As an outreach operator, I want scheduled messages sent directly through my connected Gmail account using the Gmail API, so that emails originate from my legitimate Google mailbox.
- **Rationale**: Direct Gmail dispatch guarantees SPF/DKIM alignment with the user's Google domain.
- **Priority**: P0
- **Preconditions**: Gmail OAuth token is valid; message is `SCHEDULED` and `scheduled_for <= NOW()`.
- **Acceptance Criteria**:
  - **Given** a message ready for dispatch,  
    **When** the background worker claims the row using `FOR UPDATE SKIP LOCKED`,  
    **Then** the status transitions to `SENDING`, transmits via `users.messages.send`, records the `gmail_message_id` and `gmail_thread_id`, and transitions to `SENT`.
  - **Given** an expired Google OAuth access token,  
    **When** dispatch is attempted,  
    **Then** the system automatically uses the encrypted refresh token to obtain a new access token and retries dispatch.
- **Negative Scenario (Worker Crash during Send)**: Worker dies while message is `SENDING`. Upon reboot, worker detects lease expiration, queries Gmail API for the message subject/RFC ID to verify whether it was sent, preventing duplicate transmission.
- **Related PRD ID**: `FR-GMAIL-001`, `FR-GMAIL-002`
- **Related Test ID**: `TEST-SEND-01`

---

## Epic 7: Suppression and Opt-Outs

### US-SUP-01: Global Suppression List Enforcement
- **Epic**: Suppression and Opt-outs
- **User Story**: As a compliance-conscious sender, I want the system to strictly block drafting or sending to any email or domain on my suppression list, so that I respect opt-outs and avoid complaints.
- **Rationale**: Continuing to email opted-out contacts violates CAN-SPAM / GDPR regulations.
- **Priority**: P0
- **Preconditions**: Email `lead@blockme.com` exists in the suppression list table.
- **Acceptance Criteria**:
  - **Given** an email address on the suppression list,  
    **When** a user attempts to import, generate a draft, or schedule an email to this address,  
    **Then** the system immediately marks the status as `SUPPRESSED` and blocks sending.
  - **Given** a domain-level wildcard suppression rule `*@competitor.com`,  
    **When** any contact with `@competitor.com` is processed,  
    **Then** all actions are blocked and recorded in the audit log.
- **Related PRD ID**: `FR-SUP-001`
- **Related Test ID**: `TEST-SUP-01`

---

## Epic 8: Message Lifecycle Tracking

### US-TRK-01: Multi-Stage Status Tracking & Event Auditing
- **Epic**: Message Lifecycle Tracking
- **User Story**: As an outreach operator, I want to view the complete event history of every outreach message, so that I have a clear audit trail from initial drafting to final reply.
- **Rationale**: Operational visibility is essential for understanding campaign health.
- **Priority**: P0
- **Preconditions**: Message has been created.
- **Acceptance Criteria**:
  - **Given** a message progressing through the pipeline,  
    **When** any state transition occurs,  
    **Then** an entry is inserted into `message_events` containing the previous status, new status, timestamp, and optional metadata payload.
  - **Given** an open tracking pixel request,  
    **When** the endpoint `/api/v1/telemetry/pixel/{tracking_id}.png` is requested,  
    **Then** an `OPEN` event is recorded, and status updates to `OPENED` (if not already `REPLIED`).
- **Related PRD ID**: `FR-TRK-001`
- **Related Test ID**: `TEST-TRK-01`

---

## Epic 9: Reply Association

### US-REP-01: Inbound Reply Detection & Thread Association
- **Epic**: Reply Association
- **User Story**: As an outreach operator, I want the system to detect incoming replies to my outreach threads in Gmail, so that the message status automatically marks as `REPLIED` and stops follow-ups.
- **Rationale**: Senders must know who responded without constantly context-switching to their inbox.
- **Priority**: P0
- **Preconditions**: Outbound message was sent with recorded `gmail_thread_id`.
- **Acceptance Criteria**:
  - **Given** a sent message with an active Gmail thread ID,  
    **When** the periodic reply detector queries Gmail thread history and finds an incoming message from the contact,  
    **Then** the message status updates to `REPLIED`, a `REPLY_RECEIVED` event is logged, and campaign metrics are updated.
- **Related PRD ID**: `FR-TRK-003`
- **Related Test ID**: `TEST-REP-01`

---

## Epic 10: Google Sheets Synchronization

### US-SHT-01: Bidirectional Contact Fetch and Status Write-Back
- **Epic**: Google Sheets Synchronization
- **User Story**: As an outreach operator, I want my Google Spreadsheet to automatically receive status updates as emails are drafted and sent, so that my external team tracker stays 100% synchronized.
- **Rationale**: Non-technical stakeholders collaborate via Google Sheets; real-time sync eliminates manual status updates.
- **Priority**: P0
- **Preconditions**: Google Sheet is mapped with designated status columns.
- **Acceptance Criteria**:
  - **Given** a contact imported from Google Sheet row 14,  
    **When** the message status transitions to `SENT` or `REPLIED`,  
    **Then** the background worker issues a batch update to row 14 updating the status, timestamp, and preview text.
- **Negative Scenario (Google Sheets Outage/Permission Revoked)**: If Google Sheets returns HTTP 403 or 503, the error is logged to `integration_logs`, but the primary email send is NOT aborted.
- **Related PRD ID**: `FR-SHT-003`
- **Related Test ID**: `TEST-SHT-01`

---

## Epic 11: Dashboard Analytics

### US-DASH-01: Watermelon UI Telemetry Cards & Donut Chart
- **Epic**: Dashboard Analytics
- **User Story**: As an outreach operator, I want to see an overview of total contacts, pending drafts, scheduled sends, and verified replies on a high-contrast dashboard, so that I can quickly assess campaign momentum.
- **Rationale**: Operators need immediate, accurate campaign health insights.
- **Priority**: P0
- **Preconditions**: User is logged in and campaigns exist.
- **Acceptance Criteria**:
  - **Given** active campaign data in the database,  
    **When** the user opens the Dashboard,  
    **Then** telemetry cards display the exact database counts, the Donut chart renders proportional slices, and open rates are clearly labeled as *Estimated*.
- **Related PRD ID**: `FR-DASH-001`, `FR-DASH-002`
- **Related Test ID**: `TEST-DASH-01`

---

## Epic 12: Authentication & Integration Settings

### US-AUTH-01: Google OAuth Handshake with Minimal Scopes
- **Epic**: Authentication & Integration Settings
- **User Story**: As an outreach operator, I want to connect my Gmail and Google Sheets accounts using standard OAuth 2.0 with minimal required scopes, so that I never share my primary password.
- **Rationale**: OAuth 2.0 ensures secure, revokable token-based authorization.
- **Priority**: P0
- **Preconditions**: Google Cloud OAuth client credentials configured in backend.
- **Acceptance Criteria**:
  - **Given** an unauthenticated integration,  
    **When** user clicks "Connect Gmail",  
    **Then** user is redirected to Google's consent screen requesting only `gmail.send`, `gmail.readonly`, and `userinfo.email`.
  - **Given** successful consent,  
    **When** Google redirects back with an authorization code,  
    **Then** tokens are exchanged, encrypted with AES-256 Fernet, and stored securely.
- **Related PRD ID**: `FR-GMAIL-001`
- **Related Test ID**: `TEST-AUTH-01`

---

## Epic 13: Error Handling & Recovery

### US-ERR-01: Graceful Degradation on Third-Party Rate Limits & Failures
- **Epic**: Error Handling & Recovery
- **User Story**: As an engineer, I want the system to handle external API failures (Gmail, Gemini, Sheets) with exponential backoff and circuit-breaker pausing, so that temporary external glitches do not corrupt database state.
- **Rationale**: Cloud APIs fail frequently; the platform must be resilient.
- **Priority**: P0
- **Preconditions**: A network request to Google API fails with HTTP 429 or 503.
- **Acceptance Criteria**:
  - **Given** an HTTP 429 rate limit from Gmail API,  
    **When** detected by the worker,  
    **Then** the worker triggers a 15-minute cool-down, pauses the campaign, and records an alert in `audit_logs`.
  - **Given** an HTTP 401 Unauthorized (revoked token),  
    **When** detected,  
    **Then** the integration status is set to `NEEDS_REAUTH` and an email notification/UI banner is raised.
- **Related PRD ID**: `FR-GMAIL-004`
- **Related Test ID**: `TEST-ERR-01`

---

## Epic 14: Deployment & Operational Monitoring

### US-OPS-01: Health Checks & Structured JSON Observability
- **Epic**: Deployment & Operational Monitoring
- **User Story**: As an SRE, I want `/health` and `/health/ready` endpoints along with structured JSON logging, so that Render/Vercel and monitoring tools can verify uptime and diagnose anomalies.
- **Rationale**: Automated orchestration requires reliable liveness and readiness probes.
- **Priority**: P0
- **Preconditions**: Application is booted.
- **Acceptance Criteria**:
  - **Given** healthy database and worker connections,  
    **When** `GET /health/ready` is called,  
    **Then** HTTP 200 is returned with `{"status": "ready", "database": "connected"}`.
  - **Given** database connection is down,  
    **When** `GET /health/ready` is called,  
    **Then** HTTP 503 is returned with descriptive error metadata.
- **Related PRD ID**: `NFR-OBS-001`
- **Related Test ID**: `TEST-OPS-01`
