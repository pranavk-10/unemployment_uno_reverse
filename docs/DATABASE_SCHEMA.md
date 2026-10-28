# OutreachEngine — Database Schema & Data Model Reference

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Relational Data Model Specification)  
**Database Engine:** PostgreSQL `>= 15.0` (Target: PostgreSQL 16 on Render)  
**ORM:** SQLAlchemy 2.0 (Async) | **Migrations:** Alembic 1.13+  
**Last Updated:** 2026-10-28  

---

## 1. Schema Design Principles

1. **UUID Primary Keys**: All entities use native PostgreSQL `UUIDv4` primary keys to prevent enumeration attacks and simplify distributed replication.
2. **Deterministic Status Enums**: All lifecycle states are defined using strict SQL `VARCHAR` constraints or native enums.
3. **Database-Backed Queueing**: The `messages` table doubles as a durable, transaction-safe task queue using PostgreSQL's row-level `FOR UPDATE SKIP LOCKED` capability.
4. **Encrypted Token Isolation**: OAuth refresh and access tokens are stored in binary encrypted ciphertext columns (`BYTEA` / base64 string) using Fernet AES-256.
5. **Auditing & Traceability**: All critical lifecycle events (drafting, dispatch, open detection, replies) are immutably appended to `message_events`.

---

## 2. Entity Relationship Overview

```text
+---------------+        1:N        +-----------------------+
|     users     | ─────────────────< |  integration_tokens   |
+---------------+                   +-----------------------+
        │ 1:1
        ├──────────────────────────< +-----------------------+
        │ 1:N                        |    sender_profiles    |
        ├──────────────────────────< +-----------------------+
        │ 1:N                        |       contacts        |
        ├──────────────────────────< +-----------------------+
        │ 1:N                        |   suppression_entries |
        ▼                            +-----------------------+
+---------------+        1:N        +-----------------------+
|   campaigns   | ─────────────────< |       messages        |
+---------------+                   +-----------------------+
                                                │ 1:N
                                                ▼
                                    +-----------------------+
                                    |    message_events     |
                                    +-----------------------+
```

---

## 3. Comprehensive Entity Dictionary

### 3.1 `users`
Represents the system operator or account owner.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `email` | `VARCHAR(255)` | No | — | **Unique**. Operator email address. |
| `hashed_password` | `VARCHAR(255)` | Yes | `NULL` | Hashed password (if local auth enabled). |
| `full_name` | `VARCHAR(100)` | Yes | `NULL` | User display name. |
| `is_active` | `BOOLEAN` | No | `true` | Account active state. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Account creation timestamp. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last profile update timestamp. |

- **Indexes**: `CREATE UNIQUE INDEX idx_users_email ON users(email);`
- **Deletion Behavior**: Root parent entity; cascade deletes all associated user data upon account removal.

---

### 3.2 `integration_tokens`
Stores encrypted OAuth credentials and API configurations for external Google services.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `provider` | `VARCHAR(50)` | No | — | Enum: `GMAIL`, `GOOGLE_SHEETS`, `GEMINI`. |
| `encrypted_access_token`| `TEXT` | Yes | `NULL` | Fernet AES-256 encrypted access token. |
| `encrypted_refresh_token`| `TEXT` | Yes | `NULL` | Fernet AES-256 encrypted refresh token. |
| `token_uri` | `VARCHAR(255)`| No | `'https://oauth2.googleapis.com/token'` | Token refresh endpoint. |
| `scopes` | `JSONB` | No | `'[]'` | List of granted OAuth scopes. |
| `expires_at` | `TIMESTAMPTZ` | Yes | `NULL` | Access token expiration timestamp. |
| `status` | `VARCHAR(30)` | No | `'ACTIVE'` | Enum: `ACTIVE`, `EXPIRED`, `NEEDS_REAUTH`, `REVOKED`. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Record created. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last token refresh. |

- **Indexes**: `CREATE UNIQUE INDEX idx_user_provider ON integration_tokens(user_id, provider);`

---

### 3.3 `sender_profiles`
Maintains user resume context, career narrative, target roles, and outreach tone preferences.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `headline` | `VARCHAR(255)`| No | — | Professional headline / current title. |
| `bio_summary` | `TEXT` | No | — | Concise career narrative. |
| `resume_raw_text`| `TEXT` | Yes | `NULL` | Unstructured parsed resume text. |
| `parsed_skills` | `JSONB` | No | `'[]'` | List of core technical & leadership skills. |
| `key_achievements`| `JSONB` | No | `'[]'` | Array of structured accomplishments with metrics. |
| `preferred_tone`| `VARCHAR(50)` | No | `'WARM_PROFESSIONAL'` | Enum: `WARM_PROFESSIONAL`, `DIRECT`, `CASUAL`. |
| `target_cta` | `VARCHAR(255)`| No | `'15-minute introductory coffee chat'` | Default call-to-action. |
| `max_word_count`| `INTEGER` | No | `120` | Max word length constraint for AI drafts. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Created timestamp. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last updated timestamp. |

---

### 3.4 `contacts`
Stores prospective recipients, company metadata, and verification audit results.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `first_name` | `VARCHAR(100)`| No | — | Recipient first name. |
| `last_name` | `VARCHAR(100)`| Yes | `NULL` | Recipient last name. |
| `email` | `VARCHAR(255)`| No | — | Recipient email address. |
| `company_name` | `VARCHAR(150)`| Yes | `NULL` | Recipient employer / organization. |
| `job_title` | `VARCHAR(150)`| Yes | `NULL` | Recipient professional title. |
| `linkedin_url` | `VARCHAR(255)`| Yes | `NULL` | Public LinkedIn profile URL. |
| `custom_notes` | `TEXT` | Yes | `NULL` | Specific hooks, shared articles, or background. |
| `source_type` | `VARCHAR(50)` | No | `'MANUAL'` | Enum: `MANUAL`, `GOOGLE_SHEETS`, `CSV_IMPORT`. |
| `external_row_id`| `VARCHAR(50)` | Yes | `NULL` | Row index in Google Sheet (e.g., `'Sheet1!A14'`). |
| `verification_status`| `VARCHAR(30)` | No | `'UNVERIFIED'` | Enum: `UNVERIFIED`, `VERIFIED`, `WARNING_ROLE`, `INVALID`, `SUPPRESSED`. |
| `verification_details`| `JSONB` | Yes | `NULL` | Audit info: `{mx_host: "aspmx.l.google.com", disposable: false}`. |
| `verified_at` | `TIMESTAMPTZ` | Yes | `NULL` | Timestamp of DNS/syntax verification. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Record created. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last updated. |

- **Indexes**:
  - `CREATE UNIQUE INDEX idx_user_contact_email ON contacts(user_id, email);`
  - `CREATE INDEX idx_contacts_verification ON contacts(verification_status);`

---

### 3.5 `campaigns`
Defines an outreach campaign with rate limiting rules, dispatch windows, and state.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `title` | `VARCHAR(150)`| No | — | Campaign name (e.g., "Q4 Engineering Outreach"). |
| `status` | `VARCHAR(30)` | No | `'DRAFT'` | Enum: `DRAFT`, `ACTIVE`, `PAUSED`, `COMPLETED`, `CANCELLED`. |
| `daily_send_limit`| `INTEGER` | No | `30` | Maximum messages allowed per 24-hour cycle. |
| `base_delay_seconds`| `INTEGER`| No | `60` | Minimum delay between messages. |
| `min_jitter_seconds`| `INTEGER`| No | `15` | Minimum random offset added to base delay. |
| `max_jitter_seconds`| `INTEGER`| No | `90` | Maximum random offset added to base delay. |
| `active_days` | `JSONB` | No | `'[1,2,3,4,5]'`| Array of active days (1=Monday ... 7=Sunday). |
| `send_window_start`| `TIME` | No | `'09:00:00'` | Start of allowed sending window (local time). |
| `send_window_end` | `TIME` | No | `'17:00:00'` | End of allowed sending window (local time). |
| `timezone` | `VARCHAR(50)` | No | `'UTC'` | User local timezone string (e.g., `'America/New_York'`). |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Created timestamp. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last updated. |

- **Indexes**: `CREATE INDEX idx_campaigns_status ON campaigns(status);`

---

### 3.6 `messages`
The authoritative task and lifecycle entity. Every email drafted, scheduled, or sent resides here.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `campaign_id` | `UUID` | No | — | **Foreign Key** `campaigns(id)` ON DELETE CASCADE. |
| `contact_id` | `UUID` | No | — | **Foreign Key** `contacts(id)` ON DELETE RESTRICT. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `status` | `VARCHAR(35)` | No | `'DRAFT_PENDING_REVIEW'` | Enum: See Allowed Status Values below. |
| `subject` | `VARCHAR(255)`| No | — | Email subject line. |
| `body_markdown` | `TEXT` | No | — | Email body in Markdown/text. |
| `body_html` | `TEXT` | Yes | `NULL` | Compiled HTML body with tracking pixel injected. |
| `scheduled_for` | `TIMESTAMPTZ` | Yes | `NULL` | Calculated execution timestamp. |
| `sent_at` | `TIMESTAMPTZ` | Yes | `NULL` | Confirmed Gmail API dispatch timestamp. |
| `worker_lease_until`| `TIMESTAMPTZ`| Yes| `NULL` | Lease lock expiration for background worker. |
| `retry_count` | `INTEGER` | No | `0` | Number of attempted send retries. |
| `idempotency_key`| `VARCHAR(64)` | No | — | **Unique**. SHA-256 hash preventing duplicate sends. |
| `gmail_message_id`| `VARCHAR(100)`| Yes | `NULL` | External Gmail Message ID returned by Google API. |
| `gmail_thread_id` | `VARCHAR(100)`| Yes | `NULL` | External Gmail Thread ID for reply tracking. |
| `tracking_id` | `VARCHAR(64)` | No | `gen_random_uuid()::text` | Unique token for open pixel URL matching. |
| `error_details` | `TEXT` | Yes | `NULL` | Diagnostic error message if failed. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Created timestamp. |
| `updated_at` | `TIMESTAMPTZ` | No | `NOW()` | Last updated timestamp. |

#### Allowed Message Status Values:
1. `DRAFT_PENDING_REVIEW` — Generated by Gemini, waiting for user approval.
2. `DRAFT_INVALID_PLACEHOLDER` — Unparsed placeholder tokens detected; blocked from approval.
3. `REJECTED` — Discarded by operator during review.
4. `APPROVED` — Approved by operator; eligible for scheduler.
5. `SCHEDULED` — Assigned a future dispatch time slot.
6. `SENDING` — Claimed by worker lease; API call in flight.
7. `SENT` — Confirmed sent by Gmail API (HTTP 200 OK).
8. `OPENED` — Tracking pixel request received.
9. `REPLIED` — Inbound reply message detected on Gmail thread.
10. `BOUNCED` — Non-delivery report received.
11. `FAILED` — Fatal dispatch error (exhausted retries).
12. `CANCELLED` — Revoked prior to transmission.
13. `SUPPRESSED` — Contact was added to suppression list.

- **Indexes**:
  - `CREATE UNIQUE INDEX idx_messages_idempotency ON messages(idempotency_key);`
  - `CREATE INDEX idx_messages_queue ON messages(status, scheduled_for) WHERE status = 'SCHEDULED';`
  - `CREATE INDEX idx_messages_tracking ON messages(tracking_id);`
  - `CREATE INDEX idx_messages_gmail_thread ON messages(gmail_thread_id) WHERE gmail_thread_id IS NOT NULL;`

---

### 3.7 `message_events`
Append-only immutable audit trail for telemetry and state transitions.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `message_id` | `UUID` | No | — | **Foreign Key** `messages(id)` ON DELETE CASCADE. |
| `event_type` | `VARCHAR(50)` | No | — | Enum: `DRAFTED`, `EDITED`, `APPROVED`, `SCHEDULED`, `DISPATCHED`, `OPEN`, `REPLY_DETECTED`, `BOUNCE`, `FAILED`. |
| `previous_status`| `VARCHAR(35)`| Yes | `NULL` | Status before transition. |
| `new_status` | `VARCHAR(35)` | Yes | `NULL` | Status after transition. |
| `metadata` | `JSONB` | No | `'{}'` | Contextual payload (IP, user agent, Gmail response snippet). |
| `occurred_at` | `TIMESTAMPTZ` | No | `NOW()` | Event timestamp. |

- **Indexes**:
  - `CREATE INDEX idx_events_message_occurred ON message_events(message_id, occurred_at);`
  - `CREATE INDEX idx_events_type ON message_events(event_type);`

---

### 3.8 `suppression_entries`
Global blocklist preventing any contact or domain from being targeted.

| Column Name | Type | Nullable | Default | Constraints & Description |
|---|---|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | **Primary Key**. |
| `user_id` | `UUID` | No | — | **Foreign Key** `users(id)` ON DELETE CASCADE. |
| `email` | `VARCHAR(255)`| Yes | `NULL` | Specific email address blocked. |
| `domain` | `VARCHAR(150)`| Yes | `NULL` | Entire domain blocked (e.g. `'competitor.com'`). |
| `reason` | `VARCHAR(100)`| No | `'UNSUBSCRIBED'` | Enum: `UNSUBSCRIBED`, `BOUNCE_CRITICAL`, `MANUAL_BLOCK`. |
| `created_at` | `TIMESTAMPTZ` | No | `NOW()` | Timestamp added. |

- **Indexes**:
  - `CREATE UNIQUE INDEX idx_user_suppressed_email ON suppression_entries(user_id, email) WHERE email IS NOT NULL;`
  - `CREATE UNIQUE INDEX idx_user_suppressed_domain ON suppression_entries(user_id, domain) WHERE domain IS NOT NULL;`

---

## 4. Authoritative vs. Derived Fields

To eliminate state ambiguity:
- **Authoritative Fields**:
  - `messages.status`: The single source of truth for message lifecycle.
  - `messages.idempotency_key`: Guarantees exactly-once send evaluation.
  - `contacts.verification_status`: Authoritative outcome of DNS/MX checks.
- **Derived / Synchronized Fields**:
  - Telemetry card counts (e.g. *Total Sent*, *Response Rate*): Computed dynamically via aggregation queries over `messages` and `message_events`.
  - External Google Sheet status columns: Synchronized downstream replicas; if a conflict occurs, the PostgreSQL database is authoritative.

---

## 5. Migration Procedure (Alembic)

All schema changes must proceed via versioned Alembic migration scripts:
```bash
# Generate a new migration script following model changes
alembic revision --autogenerate -m "create_initial_outreach_tables"

# Upgrade database to latest revision
alembic upgrade head

# Downgrade one revision in case of rollback
alembic downgrade -1
```
Every migration script must include a fully reversible `upgrade()` and `downgrade()` function.
