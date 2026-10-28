# OutreachEngine — Testing Strategy & Quality Assurance Plan

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (Test Strategy Specification)  
**Target Coverage:** `>= 85%` line coverage on backend core; `>= 80%` on services  
**Last Updated:** 2026-10-28  

---

## 1. Testing Pyramid & Philosophy

The OutreachEngine testing strategy balances high-speed, isolated unit tests with realistic integration and mock adapter testing to guarantee reliability without incurring cloud costs or sending real emails during test runs.

```text
               / \
              /   \      E2E Web UI & API Tests (Playwright / Cypress)
             /     \     [Critical user journeys: login, review, schedule]
            /───────\
           /         \   Integration Tests with Testcontainers (PostgreSQL)
          /           \  [Database migrations, SKIP LOCKED concurrency, repos]
         /─────────────\
        /               \ Mock Adapter Tests (MockGmail, MockGemini, MockSheets)
       /                 \[Third-party API rate limits, backoff, MIME builds]
      /───────────────────\
     /                     \ Isolated Unit Tests (Pytest, Vitest)
    /                       \[Email RFC regex, DNS MX mock, token checks]
   /─────────────────────────\
```

---

## 2. Mock Adapters & Dependency Inversion

All external service clients are abstracted behind abstract base classes (ABCs) to allow seamless substitution with mock test harnesses:

### 2.1 Mock Adapters Summary
- **`MockGeminiClient`**: Simulates Gemini API text generation, returns deterministic JSON fixtures, and simulates rate limits (HTTP 429) or token count overflows on demand.
- **`MockGmailClient`**: Records outgoing MIME payloads in memory, verifies header structure (`X-OutreachEngine-Message-ID`), simulates success responses with fake `gmail_message_id`, and tests network timeout recovery.
- **`MockSheetsClient`**: Simulates Google Sheets API v4 ranges in memory without requiring Google service account keys.
- **`MockDNSResolver`**: Injects custom DNS MX responses (valid MX, no MX, timeout, disposable domain) to test verification logic deterministically.

---

## 3. Database Testing Strategy

- **PostgreSQL Testcontainer**: Integration tests run against a real, ephemeral PostgreSQL 16 container managed via `testcontainers-python` or an isolated local test database (`outreach_test`).
- **Transactional Rollbacks**: Each test runs within a nested transaction (`savepoint`) rolled back after test completion to ensure zero state leakage between tests.
- **Concurrency & Idempotency Testing**: Dedicated integration tests spawn multiple concurrent async worker tasks competing for the same scheduled messages to verify that `FOR UPDATE SKIP LOCKED` guarantees exactly-once dispatch.

---

## 4. Test Suites & Execution Commands

### 4.1 Backend (Python / Pytest)
```bash
# Run all unit and service tests
pytest tests/unit/ -v

# Run integration tests against test PostgreSQL database
pytest tests/integration/ -v

# Run concurrency and worker idempotency tests
pytest tests/integration/test_worker_concurrency.py -v

# Generate terminal coverage report
pytest --cov=app --cov-report=term-missing tests/

# Generate HTML coverage report
pytest --cov=app --cov-report=html tests/
```

### 4.2 Frontend (Node.js / Vitest / Playwright)
```bash
# Run frontend component unit tests
npm run test

# Run accessibility (axe-core) checks
npm run test:a11y

# Run end-to-end browser tests
npm run test:e2e
```

---

## 5. Critical Failure & Edge Case Scenarios

| Test ID | Scenario | Expected Verified Behavior |
|---|---|---|
| `TEST-IDEMP-01` | Worker crashes mid-transmission while status is `SENDING`. | On restart, worker queries Gmail thread history using RFC message ID; marks `SENT` if found, preventing duplicate send. |
| `TEST-RATE-01` | Gmail API returns HTTP 429 `User Rate Limit Exceeded`. | Dispatcher catches 429, logs structured warning, sets campaign backoff timer to 15m, and preserves queue. |
| `TEST-REVOK-01`| User revokes Google OAuth token from Google Account. | Dispatcher catches HTTP 401, flags integration as `NEEDS_REAUTH`, pauses active campaign, alerts user. |
| `TEST-TOKEN-01`| AI draft contains unresolved placeholder `{{company_name}}`. | Validator flags message as `DRAFT_INVALID_PLACEHOLDER`, blocks approval until resolved. |
| `TEST-SUPP-01` | Email on suppression list imported from Google Sheet. | Verification pipeline detects suppressed email, immediately sets status `SUPPRESSED`, blocks sending. |
| `TEST-CONCUR-01`| Two worker instances poll simultaneously for scheduled messages. | PostgreSQL `SKIP LOCKED` ensures each message is claimed by exactly one worker instance with zero race conditions. |

---

## 6. Acceptance Criteria Traceability Matrix

| Epic | User Story | Requirement ID | Test Identifier | Automated Status |
|---|---|---|---|---|
| Contact Management | `US-CONT-01` | `FR-CONT-001` | `TEST-CONT-01` | `[PROPOSED]` |
| Contact Verification | `US-VERIF-01`| `FR-VERIF-001` | `TEST-VERIF-01` | `[PROPOSED]` |
| AI Personalization | `US-AI-01`   | `FR-AI-001`    | `TEST-AI-01`    | `[PROPOSED]` |
| Draft Validation | `US-VAL-01`  | `FR-AI-003`    | `TEST-REV-01`   | `[PROPOSED]` |
| Campaign Scheduling | `US-SCHED-01`| `FR-SCHED-001` | `TEST-SCHED-01` | `[PROPOSED]` |
| Gmail Dispatch | `US-GMAIL-01`| `FR-GMAIL-001` | `TEST-SEND-01`  | `[PROPOSED]` |
| Suppression Enforcement | `US-SUP-01`  | `FR-SUP-001`   | `TEST-SUP-01`   | `[PROPOSED]` |
| Lifecycle Tracking | `US-TRK-01`  | `FR-TRK-001`   | `TEST-TRK-01`   | `[PROPOSED]` |
| Reply Detection | `US-REP-01`  | `FR-TRK-003`   | `TEST-REP-01`   | `[PROPOSED]` |
| Sheets Sync | `US-SHT-01`  | `FR-SHT-003`   | `TEST-SHT-01`   | `[PROPOSED]` |
| Telemetry Dashboard | `US-DASH-01` | `FR-DASH-001`  | `TEST-DASH-01`  | `[PROPOSED]` |
| Authentication | `US-AUTH-01` | `FR-GMAIL-001` | `TEST-AUTH-01`  | `[PROPOSED]` |
| Error Recovery | `US-ERR-01`  | `FR-GMAIL-004` | `TEST-ERR-01`   | `[PROPOSED]` |
| Operational Health | `US-OPS-01`  | `NFR-OBS-001`  | `TEST-OPS-01`   | `[PROPOSED]` |

---

## 7. Continuous Integration (CI) Pipeline Recommendations

In GitHub Actions or GitLab CI, pull requests must satisfy:
1. **Linter & Formatter**: `ruff check app/` and `black --check app/`.
2. **Type Checking**: `mypy app/` (strict mode enabled).
3. **Unit & Mock Tests**: `pytest tests/unit/` (100% passing required).
4. **Integration Tests**: Ephemeral PostgreSQL test container run with `alembic upgrade head`.
5. **Coverage Threshold**: PRs causing global coverage to drop below 80% are automatically blocked.
