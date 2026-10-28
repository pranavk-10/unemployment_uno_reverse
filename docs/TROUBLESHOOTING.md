# OutreachEngine — Troubleshooting & Diagnostic Runbook

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Runbook Specification)  
**Audience:** SREs, Support Engineers, Developers, and Campaign Operators  
**Last Updated:** 2026-10-28  

> **Security Warning:**  
> Never paste plaintext API keys, OAuth secrets, database passwords, or personal email contents into logs, GitHub issues, or support tickets. Diagnostic logs must always be sanitized.

---

## 1. Quick Diagnostic Flowchart

```text
[ Symptom Detected ]
        │
        ├── 1. API returning 500 / startup failure? ──▶ See Section 2 (Configuration & Database)
        ├── 2. Emails not sending during window?    ──▶ See Section 3 (Worker & Scheduling)
        ├── 3. Google OAuth failing / disconnecting?──▶ See Section 4 (OAuth & Gmail API)
        ├── 4. AI generation failing or slow?      ──▶ See Section 5 (Gemini AI Errors)
        ├── 5. Sheets not importing / updating?     ──▶ See Section 6 (Google Sheets Sync)
        └── 6. Dashboard metrics incorrect or stale?──▶ See Section 7 (Frontend & Telemetry)
```

---

## 2. Configuration & Database Issues

### 2.1 Database Connection Refused (`asyncpg.exceptions.CannotConnectNowError`)
- **Symptoms**: FastAPI fails to boot with `connection to server at "localhost", port 5432 failed: Connection refused`.
- **Likely Causes**: PostgreSQL service is stopped, `DATABASE_URL` is malformed, or Render database is resuming from maintenance.
- **Diagnostic Steps**:
  1. Check if PostgreSQL container or local daemon is running:
     ```bash
     pg_isready -h localhost -p 5432
     ```
  2. Verify that `DATABASE_URL` uses the async driver prefix: `postgresql+asyncpg://...` (not `postgresql://`).
- **Remediation**:
  - Restart PostgreSQL service: `sudo systemctl restart postgresql` or start Docker container.
  - In cloud environments, check Render service logs for database host unreachable alerts.

### 2.2 Migration Version Mismatch (`alembic.util.exc.CommandError: Target database is not up to date`)
- **Symptoms**: API crashes on startup with migration check assertion error.
- **Likely Causes**: Code updated to a newer revision with pending migrations that have not been applied.
- **Diagnostic Steps**:
  ```bash
  alembic current
  alembic heads
  ```
- **Remediation**:
  - Run `alembic upgrade head` to apply all pending schema migrations.

---

## 3. Worker & Scheduling Issues

### 3.1 Background Worker Stalled / No Messages Sending
- **Symptoms**: Campaign is `ACTIVE`, approved messages exist with `scheduled_for <= NOW()`, but message status remains `SCHEDULED`.
- **Likely Causes**:
  1. Background worker process (`app.worker.daemon`) crashed or was not started.
  2. Campaign schedule window is inactive (e.g. current time is outside 09:00–17:00 or active days).
  3. Daily send cap has already been reached for today.
- **Diagnostic Steps**:
  1. Inspect worker process logs for lease heartbeat:
     ```text
     {"level": "INFO", "event": "worker_poll", "active_claims": 0, "queue_depth": 5}
     ```
  2. Check campaign status and send counts:
     ```sql
     SELECT status, daily_send_limit FROM campaigns WHERE id = '...';
     SELECT COUNT(*) FROM messages WHERE status = 'SENT' AND sent_at >= CURRENT_DATE;
     ```
- **Remediation**:
  - If worker crashed, restart via Render dashboard or terminal: `python -m app.worker.daemon`.
  - If daily cap reached, messages will automatically resume at the start of the next business day.

### 3.2 Duplicate Event or Stuck `SENDING` Status
- **Symptoms**: Message status is `SENDING` for longer than 10 minutes.
- **Likely Causes**: Worker crashed or network timed out mid-transmission before receiving Gmail's HTTP response.
- **Diagnostic Steps**:
  - Query stale leases:
    ```sql
    SELECT id, idempotency_key, worker_lease_until FROM messages WHERE status = 'SENDING' AND worker_lease_until < NOW();
    ```
- **Remediation**:
  - The worker's built-in lease recovery routine will automatically query Gmail API for the unique idempotency key / subject. If found, it marks `SENT`; if not found, it resets the lease to `SCHEDULED` with a 60-second backoff.

---

## 4. Google OAuth & Gmail API Failures

### 4.1 Gmail API HTTP 401 Unauthorized (`invalid_grant`)
- **Symptoms**: Message send fails with `google.auth.exceptions.RefreshError: ('invalid_grant: Token has been expired or revoked.')`.
- **Likely Causes**:
  - User changed their Google password.
  - User explicitly revoked third-party access in Google Account permissions.
  - OAuth app was in "Testing" mode and the 7-day token expiration window lapsed.
- **Remediation**:
  - The system automatically transitions integration status to `NEEDS_REAUTH`.
  - Operator must navigate to **Settings > Integrations** and click **Reconnect Gmail Account**.

### 4.2 Gmail API HTTP 429 / 403 `User Rate Limit Exceeded`
- **Symptoms**: Worker logs `HttpError 403: "Daily user sending limit exceeded"`.
- **Likely Causes**: Free Gmail accounts are capped at 500 emails/24 hours; Workspace accounts at 2,000. Senders who blast rapidly hit rolling burst limits.
- **Remediation**:
  - The dispatcher automatically initiates a 15-minute cool-down and pauses the campaign.
  - Operator should verify that `MAX_GLOBAL_DAILY_SENDS` is set to a safe value (recommended: `< 50` for standard accounts).

---

## 5. Gemini AI Generation Issues

### 5.1 Gemini API HTTP 429 Quota Exceeded
- **Symptoms**: Draft generation batch returns `ResourceExhausted: 429 Quota exceeded for quota metric 'Queries'`.
- **Likely Causes**: Free tier Google AI Studio keys have a rate limit of 15 Requests Per Minute (RPM).
- **Remediation**:
  - The draft generator automatically retries with exponential backoff.
  - For large contact imports (> 20 contacts), batch generation is throttled to 10 requests per minute.

### 5.2 Draft Flagged as `DRAFT_INVALID_PLACEHOLDER`
- **Symptoms**: Draft is created but cannot be approved; badge shows red warning.
- **Likely Causes**: Gemini returned an unparsed template token (e.g. `{{company}}` or `[Your Name]`).
- **Remediation**:
  - Open the draft in the Review Queue.
  - Manually fill in the missing details in the editor, or click **Regenerate with AI** with explicit instructions.

---

## 6. Google Sheets Synchronization Failures

### 6.1 `HttpError 404: "Requested entity was not found"` or `HttpError 403`
- **Symptoms**: Contact sync fails with error banner in UI.
- **Likely Causes**:
  - The spreadsheet URL is incorrect or points to a private sheet that the connected Google account cannot access.
  - Tab name specified in settings does not match the actual spreadsheet tab (case-sensitive).
- **Remediation**:
  - Open Google Sheets, verify sharing permissions include the connected Gmail address with Editor access.
  - Verify that tab name in OutreachEngine settings matches the tab in the sheet (e.g. `Prospects` vs `Sheet1`).

---

## 7. Frontend & Telemetry Issues

### 7.1 CORS Failure (`Access-Control-Allow-Origin Missing`)
- **Symptoms**: Browser console shows `CORS error: No 'Access-Control-Allow-Origin' header is present`.
- **Likely Causes**: Frontend origin (e.g. `http://localhost:5173` or Vercel production domain) is not listed in `CORS_ORIGINS` environment variable on the backend.
- **Remediation**:
  - Add the exact origin (including protocol and port) to the `CORS_ORIGINS` JSON array in backend environment variables and restart the web service.

### 7.2 Stale Dashboard Metrics
- **Symptoms**: Telemetry cards do not update after sending messages.
- **Likely Causes**: Browser cache or network latency.
- **Diagnostic Steps**:
  - Click the **Refresh** button on the Dashboard.
  - Check browser DevTools Network tab for `GET /telemetry/dashboard` response payload.
- **Remediation**:
  - Telemetry endpoint implements `Cache-Control: no-cache`. Ensure no intermediate reverse proxy is caching API responses.
