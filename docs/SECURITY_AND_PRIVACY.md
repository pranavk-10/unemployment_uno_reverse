# OutreachEngine — Security & Privacy Architecture

**Document Version:** 1.0.0-draft  
**Status:** **`[VERIFIED]`** (Policy & Security Specification)  
**Security Classification:** High (Credential & PII Processing)  
**Last Updated:** 2026-10-28  

---

## 1. Threat Model & Trust Boundaries

OutreachEngine processes sensitive user data, including professional resumes, third-party contact lists, and Google OAuth tokens. The threat model is structured around STRIDE principles across three primary trust boundaries:

```text
[ Browser / Public Internet ] 
              │ Boundary 1: TLS 1.3 / CORS / CSRF Token / Rate Limits
              ▼
[ FastAPI Application Core ]
              │ Boundary 2: Encryption at Rest (Fernet AES-256) / Database Pool
              ▼
[ PostgreSQL Database ]
              ▲
              │ Boundary 3: Strict Least-Privilege OAuth Scopes & Mutual TLS
              ▼
[ External Google Cloud APIs (Gmail, Sheets, Gemini) ]
```

### 1.1 Threat Assessment (STRIDE)

| Threat Category | Potential Attack Vector | Applied Mitigation |
|---|---|---|
| **Spoofing** | Attacker impersonates legitimate user or triggers sends. | Stateless JWT with HMAC-SHA256 (`HS256`) stored in `HttpOnly, Secure, SameSite=Strict` cookies. |
| **Tampering** | In-transit modification of email drafts or recipient addresses. | HTTPS/TLS 1.3 enforced on all endpoints; database transactions with immutable audit logs. |
| **Repudiation** | Operator denies sending a specific email. | Every send event is immutably recorded in `message_events` with Gmail Message ID and timestamp. |
| **Information Disclosure** | Leakage of OAuth refresh tokens from database dumps or log traces. | Refresh tokens are encrypted at rest with AES-256 (Fernet). Plaintext tokens are stripped from log formatters. |
| **Denial of Service** | Flooding tracking pixel or draft generation endpoints. | Cloudflare / reverse-proxy rate limiting; background worker bounded task queueing. |
| **Elevation of Privilege** | Tenant accessing another tenant's contacts or OAuth credentials. | All queries explicitly enforce `WHERE user_id = :current_user_id` at the repository/ORM layer. |

---

## 2. Google OAuth 2.0 Scopes & Credential Security

### 2.1 Principle of Least Privilege
OutreachEngine strictly requests the minimum necessary Google OAuth scopes required for operation:

| Requested Scope | Sensitivity Level | Operational Justification |
|---|---|---|
| `https://www.googleapis.com/auth/gmail.send` | Sensitive | Required to dispatch authorized outreach messages via `users.messages.send`. |
| `https://www.googleapis.com/auth/gmail.readonly` | Sensitive | Required strictly to poll thread IDs for incoming replies to outbound campaigns. |
| `https://www.googleapis.com/auth/userinfo.email` | Non-Sensitive | Required to identify and display the authenticated sender email in the UI. |
| `https://www.googleapis.com/auth/spreadsheets` | Sensitive | Required to read prospect rows and write back status updates in user-selected spreadsheets. |

**Explicitly Excluded Scopes**:
- `https://mail.google.com/` (Full mailbox access) is **prohibited**.
- `https://www.googleapis.com/auth/gmail.modify` (Modifying labels, trash, or deleting mail) is **prohibited**.
- `https://www.googleapis.com/auth/drive` (Full Google Drive access) is **prohibited**.

### 2.2 Token Encryption & Storage
- **Algorithm**: Fernet symmetric encryption built on AES-128-CBC with PKCS7 padding and HMAC-SHA256 for message authentication.
- **Key Storage**: The encryption key (`FERNET_KEY`) is provided strictly via an environment variable at runtime. It is never stored in source code or committed to repositories.
- **Revocation**:
  - When a user disconnects their integration, OutreachEngine calls `https://oauth2.googleapis.com/revoke?token={token}` to invalidate Google credentials immediately and deletes the database record.

---

## 3. Contact Privacy, Provenance, & Anti-Spam Compliance

### 3.1 Provenance & Data Minimization
- OutreachEngine only stores data necessary to facilitate outreach: Name, Email, Company, Job Title, and user-provided notes.
- Users are solely responsible for ensuring that their imported contacts have been obtained lawfully and ethically.
- Contact records are retained only as long as the campaign or user account is active.

### 3.2 Global Suppression List Enforcement
- Anti-spam regulations (CAN-SPAM in the US, GDPR in the EU, CASL in Canada) mandate honoring opt-out requests promptly.
- OutreachEngine maintains a database-level `suppression_entries` table.
- **Enforcement Rule**: Before draft generation, before scheduling, and immediately prior to Gmail API transmission, the system checks whether the recipient email or domain exists in the suppression list. If present, the send is aborted and recorded as `SUPPRESSED`.

### 3.3 Open-Event Tracking Privacy Realities & Limitations
- Open tracking relies on requesting an invisible 1x1 GIF tracking pixel (`GET /telemetry/pixel/{id}.png`).
- **Apple Mail Privacy Protection (MPP)**: Apple Mail pre-fetches remote images via proxy servers, causing false-positive open events.
- **Image Blockers**: Privacy extensions and enterprise email gateways frequently strip or block remote images, resulting in false-negative open counts.
- **Policy**: The UI and documentation explicitly label open telemetry as **directional estimates** rather than definitive evidence of human readership.

---

## 4. Personal Data Retention & Right to Erasure

In compliance with modern privacy standards (GDPR Article 17 "Right to Erasure" and CCPA):
1. **User Erasure Request**: When a user selects "Delete All My Data" in Settings, an atomic cascade delete removes all `sender_profiles`, `contacts`, `campaigns`, `messages`, and `integration_tokens`.
2. **Audit Retention**: High-level pseudonymized audit counts may be retained for operational billing and anti-abuse verification, devoid of personally identifiable information (PII).

---

## 5. Security Incident Response & Audit Logging

- **Logging Rules**:
  - Application logs strictly format structured JSON.
  - Custom log filters mask headers containing `Authorization: Bearer` and parameters named `access_token`, `refresh_token`, `api_key`, or `password`.
- **Incident Playbook**:
  - In the event of suspected credential compromise, the administrator immediately revokes the Google Cloud OAuth Client Secret in Google Cloud Console, which terminates all active sessions globally.
  - The database is placed into read-only mode while audit logs are analyzed to determine blast radius.

---

## 6. Compliance Disclaimer

*Notice: OutreachEngine provides technical mechanisms (suppression lists, volume limits, jittered delays, and human review queues) designed to support privacy and anti-spam best practices. However, technical safeguards do not guarantee legal compliance. Operators are legally responsible for complying with the CAN-SPAM Act, GDPR, CASL, and Google API Services User Data Policy within their respective jurisdictions.*
