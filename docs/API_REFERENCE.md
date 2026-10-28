# OutreachEngine — REST API Reference

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (API Contract Specification)  
**Base URL:** `https://api.outreachengine.com/api/v1` (Local: `http://localhost:8000/api/v1`)  
**Specification Standard:** OpenAPI 3.1  
**Last Updated:** 2026-10-28  

---

## 1. Global Conventions & Standard Error Format

### 1.1 Authentication & Headers
- All protected endpoints require a Bearer token in the `Authorization` header:
  ```http
  Authorization: Bearer <jwt_access_token>
  ```
- All request bodies must be JSON (`Content-Type: application/json`).
- All timestamps in request and response payloads are in **ISO 8601 UTC** format (e.g., `2026-10-28T10:15:30.000Z`).

### 1.2 Standard Error Response Payload
When any endpoint returns HTTP 4xx or 5xx, the body conforms to RFC 7807 Problem Details:

```json
{
  "type": "https://errors.outreachengine.com/errors/RESOURCE_CONFLICT",
  "title": "Conflict",
  "status": 409,
  "detail": "A contact with email 'elena.developer@gmail.com' already exists for this account.",
  "instance": "/api/v1/contacts",
  "correlation_id": "req_8f7b2c91-d3a4-4e2b-b891-628a7e4b9c10"
}
```

### 1.3 Standard Pagination Parameters
List endpoints accept query parameters:
- `page` (integer, default: `1`, minimum: `1`)
- `page_size` (integer, default: `20`, maximum: `100`)

Paginated responses return:
```json
{
  "items": [],
  "total_items": 142,
  "total_pages": 8,
  "page": 1,
  "page_size": 20
}
```

---

## 2. API Endpoints Catalog

### 2.1 Authentication & Integrations (`/auth`)

#### `GET /auth/google/login`
- **Purpose**: Generates Google OAuth 2.0 authorization URL and initiates handshake.
- **Auth**: None (Public).
- **Query Parameters**:
  - `integration_type`: `GMAIL` | `GOOGLE_SHEETS` (default: `GMAIL`)
- **Response `302 Found`**: Redirects user to Google OAuth consent screen.

#### `GET /auth/google/callback`
- **Purpose**: Receives authorization code from Google, exchanges for tokens, encrypts, and stores.
- **Auth**: None (OAuth redirect).
- **Query Parameters**:
  - `code`: Authorization code from Google.
  - `state`: CSRF prevention token.
- **Response `200 OK`**:
  ```json
  {
    "status": "CONNECTED",
    "provider": "GMAIL",
    "connected_email": "user@gmail.com",
    "scopes": [
      "https://www.googleapis.com/auth/gmail.send",
      "https://www.googleapis.com/auth/gmail.readonly"
    ]
  }
  ```

---

### 2.2 Sender Profile (`/profiles`)

#### `GET /profiles/me`
- **Purpose**: Retrieves the active user's professional profile, resume context, and outreach preferences.
- **Auth**: Bearer JWT.
- **Response `200 OK`**:
  ```json
  {
    "id": "7c82c3b5-4b08-410a-9d95-8c7e9e6c401a",
    "headline": "Senior Distributed Systems Engineer",
    "bio_summary": "10 years building high-throughput payment gateways and cloud backends.",
    "parsed_skills": ["Python", "Go", "PostgreSQL", "Kubernetes", "FastAPI"],
    "key_achievements": [
      "Scaled payment microservice to 45k req/sec with 99.999% uptime.",
      "Reduced infrastructure cloud costs by 32% via connection pool tuning."
    ],
    "preferred_tone": "WARM_PROFESSIONAL",
    "target_cta": "15-minute introductory coffee chat",
    "max_word_count": 120
  }
  ```

#### `PUT /profiles/me`
- **Purpose**: Updates sender profile and resume narrative.
- **Auth**: Bearer JWT.
- **Request Body**:
  ```json
  {
    "headline": "Staff Distributed Systems Engineer",
    "bio_summary": "10+ years engineering resilient transactional architectures.",
    "parsed_skills": ["Python", "Go", "PostgreSQL", "FastAPI", "AsyncIO"],
    "key_achievements": [
      "Scaled payment microservice to 45k req/sec.",
      "Reduced AWS infrastructure costs by 32%."
    ],
    "preferred_tone": "WARM_PROFESSIONAL",
    "target_cta": "15-minute introductory coffee chat",
    "max_word_count": 120
  }
  ```
- **Response `200 OK`**: Returns updated profile object.

---

### 2.3 Contact Management (`/contacts`)

#### `GET /contacts`
- **Purpose**: Lists contacts with pagination and verification filters.
- **Auth**: Bearer JWT.
- **Query Parameters**:
  - `verification_status`: `VERIFIED` | `UNVERIFIED` | `INVALID` | `SUPPRESSED`
  - `page`: 1
  - `page_size`: 20
- **Response `200 OK`**:
  ```json
  {
    "items": [
      {
        "id": "e9b9d628-87a4-4a2b-93f5-081cfb591fa7",
        "first_name": "Sarah",
        "last_name": "Jenkins",
        "email": "sjenkins@stripe.com",
        "company_name": "Stripe",
        "job_title": "Director of Infrastructure",
        "verification_status": "VERIFIED",
        "verified_at": "2026-10-28T08:30:00Z"
      }
    ],
    "total_items": 1,
    "total_pages": 1,
    "page": 1,
    "page_size": 20
  }
  ```

#### `POST /contacts`
- **Purpose**: Creates an individual contact and immediately triggers async verification.
- **Auth**: Bearer JWT.
- **Request Body**:
  ```json
  {
    "first_name": "David",
    "last_name": "Wu",
    "email": "dwu@earlystage.io",
    "company_name": "EarlyStage AI",
    "job_title": "Founding Partner",
    "linkedin_url": "https://linkedin.com/in/davidwu",
    "custom_notes": "Read his recent Substack piece on autonomous agents."
  }
  ```
- **Response `201 Created`**:
  ```json
  {
    "id": "4a7f0e9b-6b4d-48d7-862d-0a8c2f1f0b4a",
    "first_name": "David",
    "last_name": "Wu",
    "email": "dwu@earlystage.io",
    "verification_status": "VERIFIED",
    "created_at": "2026-10-28T10:14:00Z"
  }
  ```

#### `POST /contacts/sync-sheets`
- **Purpose**: Reads contacts from a configured Google Sheet and imports newly added rows.
- **Auth**: Bearer JWT.
- **Request Body**:
  ```json
  {
    "spreadsheet_url": "https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgvE2upms/edit",
    "tab_name": "Prospects"
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "imported_count": 34,
    "skipped_duplicates": 2,
    "verified_count": 32,
    "invalid_count": 2
  }
  ```

---

### 2.4 Campaigns (`/campaigns`)

#### `POST /campaigns`
- **Purpose**: Creates a new campaign with rate limiting and schedule parameters.
- **Auth**: Bearer JWT.
- **Request Body**:
  ```json
  {
    "title": "Q4 Engineering Networking",
    "daily_send_limit": 35,
    "base_delay_seconds": 60,
    "min_jitter_seconds": 15,
    "max_jitter_seconds": 90,
    "active_days": [1, 2, 3, 4, 5],
    "send_window_start": "09:00:00",
    "send_window_end": "17:00:00",
    "timezone": "America/New_York"
  }
  ```
- **Response `201 Created`**: Returns created campaign object with status `DRAFT`.

#### `POST /campaigns/{id}/activate`
- **Purpose**: Activates a campaign. Background worker begins dispatching scheduled messages.
- **Auth**: Bearer JWT.
- **Response `200 OK`**: `{"id": "...", "status": "ACTIVE"}`

#### `POST /campaigns/{id}/pause`
- **Purpose**: Pauses a campaign immediately. In-flight messages finish; queued messages wait.
- **Auth**: Bearer JWT.
- **Response `200 OK`**: `{"id": "...", "status": "PAUSED"}`

---

### 2.5 Message Generation & Review Queue (`/messages`)

#### `POST /campaigns/{id}/generate-drafts`
- **Purpose**: Ingests selected verified contacts and invokes Gemini AI to generate drafts.
- **Auth**: Bearer JWT.
- **Request Body**:
  ```json
  {
    "contact_ids": ["4a7f0e9b-6b4d-48d7-862d-0a8c2f1f0b4a"]
  }
  ```
- **Response `202 Accepted`**:
  ```json
  {
    "batch_id": "b7c2a1e9-4e5a-4b92-b432-8e7c10b29a1f",
    "generated_count": 1,
    "status": "COMPLETED"
  }
  ```

#### `GET /messages/review-queue`
- **Purpose**: Lists messages requiring human review (`DRAFT_PENDING_REVIEW`).
- **Auth**: Bearer JWT.
- **Response `200 OK`**:
  ```json
  {
    "items": [
      {
        "id": "5f1a8c9e-2d3b-4c5e-8f0a-1a2b3c4d5e6f",
        "contact": {
          "first_name": "Sarah",
          "last_name": "Jenkins",
          "company_name": "Stripe"
        },
        "subject": "Quick question on Stripe's distributed caching patterns",
        "body_markdown": "Hi Sarah,\n\nI really enjoyed reading your recent piece on Stripe's ledger consistency models. As an engineer who recently scaled our payment gateway to 45k req/sec, I was fascinated by your approach to idempotency keys.\n\nWould you be open to a brief 15-minute introductory coffee chat next Tuesday or Thursday?\n\nBest regards,\nElena",
        "status": "DRAFT_PENDING_REVIEW",
        "has_invalid_placeholders": false
      }
    ],
    "total_items": 1
  }
  ```

#### `POST /messages/{id}/approve`
- **Purpose**: Approves a reviewed draft and queues it for scheduling.
- **Auth**: Bearer JWT.
- **Request Body**: Optional edited subject/body.
  ```json
  {
    "subject": "Quick question on Stripe's distributed caching patterns",
    "body_markdown": "Hi Sarah,\n\nI really enjoyed reading your recent piece on Stripe's ledger consistency models..."
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "id": "5f1a8c9e-2d3b-4c5e-8f0a-1a2b3c4d5e6f",
    "status": "SCHEDULED",
    "scheduled_for": "2026-10-28T14:12:45Z"
  }
  ```

---

### 2.6 Telemetry & Public Tracking (`/telemetry`)

#### `GET /telemetry/dashboard`
- **Purpose**: Computes and returns aggregated metrics cards, donut slices, and recent activity.
- **Auth**: Bearer JWT.
- **Response `200 OK`**:
  ```json
  {
    "metrics": {
      "total_contacts": 142,
      "pending_review": 12,
      "scheduled": 24,
      "sent": 86,
      "estimated_opens": 52,
      "replies": 18
    },
    "donut_chart": {
      "draft": 12,
      "scheduled": 24,
      "sent": 68,
      "opened": 34,
      "replied": 18,
      "failed": 0
    },
    "recent_messages": [
      {
        "id": "5f1a8c9e-2d3b-4c5e-8f0a-1a2b3c4d5e6f",
        "recipient_name": "Sarah Jenkins",
        "company": "Stripe",
        "status": "SENT",
        "timestamp": "2026-10-28T10:15:00Z"
      }
    ]
  }
  ```

#### `GET /telemetry/pixel/{tracking_id}.png`
- **Purpose**: Serves 1x1 transparent GIF tracking pixel; records open event asynchronously.
- **Auth**: None (Public image request from mail clients).
- **Response `200 OK`**: Binary GIF (`image/gif`), Cache-Control: `no-cache, no-store, must-revalidate`.
