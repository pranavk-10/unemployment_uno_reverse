# OutreachEngine — Product Requirements Document (PRD)

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (Specification Phase)  
**Author:** Product & Architecture Team  
**Last Updated:** 2026-10-28  

---

## 1. Product Vision

**OutreachEngine** empowers professionals, job-seekers, founders, and business developers to conduct high-conversion, respectful, and deeply personalized email outreach without the risk of spam classification, reputation damage, or awkward robotic templates. 

By unifying candidate/sender profile context, contact verification, Google Gemini generative reasoning, direct Gmail API dispatch, and two-way Google Sheets synchronization, OutreachEngine turns cold networking from a stressful, error-prone manual grind into an intelligent, transparent, and auditable system.

---

## 2. Problem Statement

Modern professional networking and outbound communication suffer from acute friction points:

1. **Generic, Impersonal Outreach**: Mass-mailer tools encourage lazy copy-paste templates that recipients instantly identify as spam, leading to sub-2% reply rates and burned sender domains.
2. **Disconnected Context**: Senders struggle to weave their authentic resume highlights, portfolio links, and mutual connections into specific company contexts at scale.
3. **Operational Fatigue & Hallucinations**: Manually researching 50 contacts takes 15+ hours. Conversely, pure automated AI tools hallucinate past collaborations, fabricate facts, or send emails with unparsed template placeholders like `{{company_name}}`.
4. **Deliverability & Burnout Risks**: Outbound engines using unverified SMTP servers or blasting 500 emails per minute quickly trip Google spam filters and destroy sender domain reputation.
5. **Data Silos**: Non-technical users organize prospects in Google Sheets, but existing tools require clumsy CSV exports, lose sync with replies, and lack transparent event auditing.

---

## 3. Target Users & Personas

### Persona A: The Active Job Seeker (Elena, Senior Software Engineer)
- **Goal**: Reach out to engineering directors, founders, and hiring managers at 40 target startups.
- **Pain Point**: Needs each email to cite specific engineering blog posts or tech stacks relevant to her background, but writing 40 unique emails takes an entire weekend.
- **Requirement**: Must have an interactive approval queue to review and tweak every AI draft before it is queued.

### Persona B: The Agency Founder / Consultant (Marcus, Tech Strategy Consultant)
- **Goal**: Contact 150 VP-level decision-makers to offer specialized advisory services.
- **Pain Point**: Cannot risk getting his Google Workspace domain flagged as spam. Needs strict daily limits, working-hour dispatch, and randomized intervals.
- **Requirement**: Needs Google Sheets as his single source of truth for leads and expects real-time status write-backs.

### Persona C: The University Student / Career Transitioner (Aarav, Aspiring Product Manager)
- **Goal**: Informational interviews with alumni and senior PMs.
- **Pain Point**: Unsure how to frame his past non-technical background concisely.
- **Requirement**: Needs Gemini AI to intelligently align his resume points with the target contact's industry.

---

## 4. Goals & Non-Goals

### Goals
- **G-1**: Deliver hyper-personalized draft generation using user resume/profile context and contact metadata via Google Gemini.
- **G-2**: Implement strict pre-send verification (syntax checks, DNS MX resolution, disposable domain rejection, and placeholder validation).
- **G-3**: Provide a mandatory human-in-the-loop review workflow to guarantee draft accuracy before any automated dispatch.
- **G-4**: Dispatch emails directly through the user's authenticated Gmail account via Google OAuth 2.0 with randomized jitter (45–180 seconds) and daily quota limits.
- **G-5**: Maintain real-time two-way synchronization with Google Sheets for contact imports and status telemetry.
- **G-6**: Deliver a beautiful, high-contrast Watermelon UI dashboard displaying honest, non-fabricated metrics.

### Non-Goals
- **NG-1**: We do **not** support mass cold-blasting (>500 unvetted emails/day) or unauthorized scraping of email addresses.
- **NG-2**: We do **not** guarantee 100% email delivery or read tracking. (Tracking relies on standard open-pixel and reply heuristics, which may be blocked by privacy filters like Apple Mail Privacy Protection).
- **NG-3**: We do **not** replace full-scale enterprise CRM suites (like Salesforce or HubSpot).
- **NG-4**: We do **not** manage external SMTP relays or domain DNS configurations (SPF/DKIM/DMARC) directly; sending occurs strictly through the user's authorized Google account.

---

## 5. Product Scope & MVP Requirements

The MVP (Milestone 1) focuses on single-user, high-control campaigns:
- Resume & profile management (upload plain text or PDF parsed context).
- Contact ingestion via manual entry or Google Sheets connection.
- Contact verification pipeline.
- AI drafting using Google Gemini API (`gemini-2.5-flash` / `gemini-1.5-flash`).
- Draft review and inline editing queue.
- Campaign scheduler with rate limiting (max 50 emails/day in MVP, 45–180s randomized delays).
- Direct Gmail dispatch via OAuth 2.0.
- Telemetry dashboard with Watermelon UI styling.

---

## 6. Functional Requirements (FR)

### Epic 1: Sender Profile & Resume Management
- **FR-PROF-001** `[P0]`: The system shall allow users to create and update a primary sender profile including Full Name, Job Title, Portfolio Links, Target Roles, and Bio.
- **FR-PROF-002** `[P0]`: The system shall accept plain text or Markdown resumes and store parsed professional experiences, core skills, and notable achievements.
- **FR-PROF-003** `[P1]`: The system shall allow defining default outreach preferences (e.g., tone: *warm professional*, max word count: *120 words*, primary call-to-action: *15-minute introductory coffee chat*).

### Epic 2: Contact Management & Verification
- **FR-CONT-001** `[P0]`: The system shall allow manual creation, bulk editing, and deletion of contact records (First Name, Last Name, Email, Company, Job Title, LinkedIn URL, Notes).
- **FR-CONT-002** `[P0]`: The system shall import contacts directly from a specified Google Sheet URL and tab name.
- **FR-VERIF-001** `[P0]`: The system shall execute automated pre-send verification on all imported contacts:
  1. RFC 5322 regex email syntax validation.
  2. DNS MX record existence check for recipient domain.
  3. Rejection of disposable/temporary email provider domains.
  4. Flagging of role-based addresses (`admin@`, `info@`, `support@`).
- **FR-VERIF-002** `[P0]`: The system shall prevent any message from being scheduled to an unverified or invalid contact address.

### Epic 3: AI Personalization & Draft Validation
- **FR-AI-001** `[P0]`: The system shall generate tailored cold outreach email drafts using the Google Gemini API, combining sender profile context and recipient metadata.
- **FR-AI-002** `[P0]`: The system shall strictly enforce system prompts prohibiting hallucinated prior interactions, fabricated mutual acquaintances, or unsupported claims.
- **FR-AI-003** `[P0]`: The system shall validate generated drafts for remaining unpopulated template tokens (e.g., `{{company}}`, `[Name]`, `TODO`). Any draft failing token validation shall be flagged with error status `DRAFT_INVALID_PLACEHOLDER`.
- **FR-AI-004** `[P1]`: The system shall provide a "Regenerate with Prompt Adjustment" action allowing users to tweak tone, emphasis, or brevity before approval.

### Epic 4: Human-in-the-Loop Review Queue
- **FR-REV-001** `[P0]`: The system shall place all newly generated drafts into an interactive review queue with status `DRAFT_PENDING_REVIEW`.
- **FR-REV-002** `[P0]`: The user shall have the ability to inspect, edit subject line, edit email body, and click "Approve" or "Reject".
- **FR-REV-003** `[P1]`: The system shall support bulk approval for drafts that have passed all verification and placeholder checks.

### Epic 5: Campaign Scheduling & Dispatch Engine
- **FR-SCHED-001** `[P0]`: The system shall allow users to create a Campaign with configurable schedule parameters:
  - Allowed sending window (e.g., Monday–Friday, 09:00–17:00 local time).
  - Daily maximum send limit (default: 30, maximum allowed: 100).
  - Minimum delay between messages (default: 60s) and random jitter range (+15s to +120s).
- **FR-SCHED-002** `[P0]`: The system shall transition approved drafts to `SCHEDULED` status and compute an execution timestamp based on the campaign's active queue and rate rules.
- **FR-SCHED-003** `[P0]`: The system shall support pausing, resuming, and emergency cancelling of active campaigns. Cancelling an active campaign shall immediately revoke all `SCHEDULED` messages.

### Epic 6: Gmail OAuth Integration & Dispatch
- **FR-GMAIL-001** `[P0]`: The system shall authenticate the user via Google OAuth 2.0 with minimal required scopes:
  - `https://www.googleapis.com/auth/gmail.send`
  - `https://www.googleapis.com/auth/gmail.readonly`
  - `https://www.googleapis.com/auth/userinfo.email`
- **FR-GMAIL-002** `[P0]`: The system shall send authorized emails directly through the Gmail API `users.messages.send` endpoint formatted as multipart MIME.
- **FR-GMAIL-003** `[P0]`: The system shall record the external Gmail Message ID and Thread ID upon successful dispatch.
- **FR-GMAIL-004** `[P0]`: The system shall handle Gmail API rate limits (HTTP 429 / 403 quota exceeded) with exponential backoff and automatically suspend campaign dispatch if daily quota is reached.

### Epic 7: Google Sheets Bidirectional Synchronization
- **FR-SHT-001** `[P0]`: The system shall authenticate Google Sheets API access using user OAuth credentials.
- **FR-SHT-002** `[P0]`: The system shall read contact rows from a specified tab and map required columns (First Name, Last Name, Email, Company, Notes).
- **FR-SHT-003** `[P0]`: The system shall write status updates back to designated status columns in the sheet (e.g., `Outreach Status`, `Last Updated`, `Draft Body Preview`) upon campaign progress.

### Epic 8: Tracking, Suppression, & Privacy
- **FR-TRK-001** `[P1]`: The system shall inject an optional transparent 1x1 tracking pixel into HTML emails to record open events.
- **FR-TRK-002** `[P1]`: The UI shall clearly label open metrics as *estimated* and include explicit disclaimers regarding Apple Mail Privacy Protection (MPP) limitations.
- **FR-TRK-003** `[P0]`: The system shall periodically poll Gmail thread history for replies associated with sent outreach threads and update message status to `REPLIED`.
- **FR-SUP-001** `[P0]`: The system shall maintain a global suppression list (opted-out emails and domains). No email shall ever be generated, scheduled, or sent to a suppressed address.
- **FR-SUP-002** `[P0]`: Senders shall be able to manually add contacts or entire domains (`*@domain.com`) to the suppression list.

### Epic 9: Dashboard & Analytics
- **FR-DASH-001** `[P0]`: The dashboard shall display primary telemetry cards: Total Contacts, Drafts Pending Review, Messages Scheduled, Messages Sent, Estimated Opens, and Replies.
- **FR-DASH-002** `[P0]`: The dashboard shall feature a Watermelon-themed Donut Chart depicting message breakdown across states.
- **FR-DASH-003** `[P0]`: The dashboard shall display a real-time "Recent Messages" card with status badges, recipient name, timestamp, and one-click detail drawer.

---

## 7. Non-Functional Requirements (NFR)

- **NFR-SEC-001 (Security)**: All Google OAuth access and refresh tokens must be encrypted at rest using AES-256 (Fernet) before persistence in PostgreSQL. Plaintext tokens must never appear in application logs.
- **NFR-SEC-002 (Least Privilege)**: Application must request only necessary Google OAuth scopes; full Gmail inbox management (`gmail.modify` or mail root) is strictly prohibited.
- **NFR-PERF-001 (Performance)**: Background polling worker loop must complete its dispatch evaluation within 5 seconds under a load of 1,000 scheduled messages.
- **NFR-REL-001 (Reliability & Idempotency)**: All dispatch actions must be guarded by database-backed idempotency locks (`SELECT ... FOR UPDATE SKIP LOCKED`) to ensure no message can ever be sent twice if workers restart or scale horizontally.
- **NFR-USAB-001 (Accessibility)**: Web UI must meet WCAG 2.1 AA standards for color contrast, keyboard navigation, and ARIA labels.
- **NFR-OBS-001 (Observability)**: Every state transition and API call must emit structured JSON logs containing `user_id`, `campaign_id`, `message_id`, and `correlation_id`.

---

## 8. Requirements Traceability Matrix (Sample)

| Requirement ID | Priority | Description | Target Epic | Acceptance Test Mapping | Status |
|---|---|---|---|---|---|
| **FR-PROF-001** | P0 | Sender Profile creation & storage | Epic 1 | `TEST-PROF-01` | `[PROPOSED]` |
| **FR-CONT-001** | P0 | Contact CRUD & metadata | Epic 2 | `TEST-CONT-01` | `[PROPOSED]` |
| **FR-VERIF-001** | P0 | Syntax, MX, & disposable domain check | Epic 2 | `TEST-VERIF-01` | `[PROPOSED]` |
| **FR-AI-001** | P0 | Gemini draft generation | Epic 3 | `TEST-AI-01` | `[PROPOSED]` |
| **FR-AI-003** | P0 | Unresolved placeholder token check | Epic 3 | `TEST-AI-02` | `[PROPOSED]` |
| **FR-REV-001** | P0 | Review queue interface & state | Epic 4 | `TEST-REV-01` | `[PROPOSED]` |
| **FR-SCHED-001**| P0 | Campaign rate rules & jitter config | Epic 5 | `TEST-SCHED-01` | `[PROPOSED]` |
| **FR-GMAIL-001**| P0 | Minimal Google OAuth2 authorization | Epic 6 | `TEST-AUTH-01` | `[PROPOSED]` |
| **FR-GMAIL-002**| P0 | Multipart MIME dispatch via Gmail API | Epic 6 | `TEST-SEND-01` | `[PROPOSED]` |
| **FR-SHT-003**  | P0 | Status write-back to Google Sheets | Epic 7 | `TEST-SHT-01` | `[PROPOSED]` |
| **FR-SUP-001**  | P0 | Suppression list block rule | Epic 8 | `TEST-SUP-01` | `[PROPOSED]` |
| **FR-DASH-001** | P0 | Telemetry metrics card computation | Epic 9 | `TEST-DASH-01` | `[PROPOSED]` |

---

## 9. Constraints & Dependencies

1. **Google API Quotas**: Free Gmail accounts have an enforced limit of 500 emails/day; Google Workspace accounts allow up to 2,000/day. The platform must cap daily sends conservative to these limits.
2. **Google OAuth App Verification**: Using restricted or sensitive scopes requires Google Cloud App Verification. In development and internal testing, the app operates in "Testing" mode limited to registered test users.
3. **Database Dependency**: Requires PostgreSQL 15+ for `SKIP LOCKED` query concurrency primitives.

---

## 10. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation Strategy |
|---|---|---|---|
| Sender email marked as spam | High | Medium | Enforce randomized delay (45–180s), daily send caps (<50/day), mandatory draft review, and MX/disposable domain pre-validation. |
| AI hallucinating facts or fake mutual friends | High | Medium | Strict system prompts with few-shot examples; automated placeholder detection; human-in-the-loop review before sending. |
| Worker crash during send causing duplicate emails | Critical | Low | Two-phase transactional lease: set status to `SENDING` with lock; record Gmail message ID upon return before marking `SENT`. |
| Google OAuth token revocation mid-campaign | High | Low | Catch 401 unauthorized errors, immediately pause campaign, mark integration as `NEEDS_REAUTH`, and alert user. |

---

## 11. Explicit Out-of-Scope Functionality

- Automated CAPTCHA solving or email scraping from websites.
- High-volume cold B2B mass blasts (>500 emails/day).
- Direct manipulation of DNS SPF/DKIM records.
- Multi-tenant enterprise SSO (SAML/Okta) in MVP.
