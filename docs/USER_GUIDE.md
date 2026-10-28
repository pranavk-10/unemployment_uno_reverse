# OutreachEngine — End-User Guide

**Document Status:** **`[PROPOSED]`** (Pre-Implementation Draft)  
**Target Audience:** Job Seekers, Consultants, Networkers, and Campaign Operators  
**Last Updated:** 2026-10-28  

> **Notice Regarding Feature Availability:**  
> This guide outlines the standard operating procedures for OutreachEngine. Because the platform is currently under active development, all workflows documented below are marked **`[PROPOSED]`**. No workflow should be assumed operational in a live environment until verified in the [Changelog](CHANGELOG.md).

---

## 1. Product Introduction

Welcome to **OutreachEngine**! OutreachEngine is an AI-powered outbound outreach companion designed to help you build genuine professional relationships, land dream job interviews, and connect with business partners.

Unlike mass blast tools that send hundreds of generic, spammy templates, OutreachEngine focuses on **high-touch, verified, deeply contextual conversations**. It takes your authentic resume and career story, examines your recipient's company and background, and uses Google Gemini to write tailored introductory emails that feel 100% human. 

Every single message passes through your **Review Queue**, meaning you maintain complete editorial control before anything is delivered through your personal Gmail account.

---

## 2. First-Time Setup & Prerequisites

Before launching your first campaign, ensure you have:
1. A standard **Google Account** (free `@gmail.com` or Google Workspace account).
2. A **Google Gemini API Key** (obtainable free from [Google AI Studio](https://aistudio.google.com/)).
3. A **Google Spreadsheet** containing your list of prospective contacts.
4. Your current **Resume** or a summary of your professional background (plain text, markdown, or PDF).

---

## 3. Connecting Integrations `[PROPOSED]`

OutreachEngine connects directly to official Google APIs so you never have to expose your master Google password or manage custom SMTP servers.

### 3.1 Connecting Gmail via OAuth 2.0
1. Navigate to the **Settings** tab in the sidebar navigation.
2. Under the **Integrations** section, locate the **Gmail Integration** card.
3. Click **Connect Gmail Account**.
4. A Google OAuth consent popup will appear. Select your desired Google account.
5. Review the requested permissions:
   - *Send email on your behalf* (`gmail.send`)
   - *View email messages and settings to detect replies* (`gmail.readonly`)
   - *View your email address* (`userinfo.email`)
6. Click **Allow**.
7. Once redirected back to OutreachEngine, the badge will update to **`Connected`** displaying your email address.

```text
+-----------------------------------------------------------+
| [ Gmail Integration ]                   [ Connected (✓) ] |
| Account: elena.developer@gmail.com                        |
| Scopes: Send Email, Read Replies                          |
| [ Test Connection ]                    [ Disconnect ]    |
+-----------------------------------------------------------+
```

### 3.2 Configuring Google Gemini AI
1. In the **Settings** view, locate the **Gemini AI Configuration** card.
2. Enter your Gemini API key in the input field.
3. Select your preferred model from the dropdown:
   - **Gemini 2.5 Flash** *(Recommended: fastest, highly contextual, cost-effective)*
   - **Gemini 1.5 Pro** *(Best for complex, multi-page resume synthesis)*
4. Click **Validate Key**. OutreachEngine will issue a low-latency test ping to verify your quota.
5. Click **Save Settings**.

### 3.3 Connecting Google Sheets
1. Under **Settings > Google Sheets**, click **Connect Google Sheets**.
2. Grant read/write access to your spreadsheets (`spreadsheets` scope).
3. Paste the URL of your outreach spreadsheet.
4. Select the primary tab name (e.g., `Sheet1` or `Prospects`).
5. Map your sheet's column headers to OutreachEngine's standard contact fields:
   - `First Name` ➔ Column A
   - `Last Name` ➔ Column B
   - `Email Address` ➔ Column C
   - `Company` ➔ Column D
   - `Job Title` ➔ Column E
   - `Outreach Status` (Writeback target) ➔ Column H
6. Click **Save Mapping**.

---

## 4. Preparing Your Resume & Persona Profile `[PROPOSED]`

To generate relevant emails, the AI needs to understand who you are and what value you bring.

1. Go to the **Profile** tab.
2. Enter your **Headline** and **Current Role** (e.g., *"Senior Distributed Systems Engineer specializing in Go & Kubernetes"*).
3. Paste your **Resume Text** or upload your resume file.
4. Define your **Key Talking Points**:
   - Highlight 2–3 major achievements with quantifiable metrics.
   - Example: *"Scaled payment microservice to 45,000 req/sec with 99.999% uptime."*
5. Set your **Outreach Voice & Tone**:
   - Tone options: *Warm Professional (Default)*, *Direct & Concise*, or *Conversational Academic*.
   - Target Call-to-Action (CTA): e.g., *"15-minute virtual coffee chat"*, *"Advice on team culture"*, or *"Exploring synergy"*.
6. Click **Save Profile**.

---

## 5. Importing & Verifying Contacts `[PROPOSED]`

OutreachEngine prevents deliverability issues before any email is ever drafted.

### 5.1 Importing Contacts
- **From Google Sheets**: Click **Contacts > Sync from Sheet**. OutreachEngine will pull newly added rows from your mapped spreadsheet.
- **Manual Entry**: Click **+ Add Contact** to add an individual recipient with their name, email, company, and custom notes.

### 5.2 Understanding Contact Verification Badges
Every imported contact automatically undergoes multi-stage verification:
- 🟢 **VERIFIED**: Email syntax is valid, the company domain has active DNS MX mail exchange records, and it is not a temporary/throwaway inbox.
- 🟡 **WARNING (Role Account)**: The email is an impersonal role address (e.g., `info@company.com`, `careers@company.com`). Sending is allowed but requires explicit confirmation.
- 🔴 **INVALID (Syntax / No MX)**: The email address is malformed or the recipient domain cannot receive email.
- ⛔ **SUPPRESSED**: The email or domain is on your opt-out suppression list.

```text
+-------------------------------------------------------------------------------+
| Name           | Email                 | Company      | Verification Status   |
+-------------------------------------------------------------------------------+
| Sarah Jenkins  | sjenkins@stripe.com   | Stripe       | [ 🟢 VERIFIED ]       |
| David Wu       | dwu@earlystage.io     | EarlyStage   | [ 🟢 VERIFIED ]       |
| Support Team   | support@startup.com   | Startup Inc  | [ 🟡 ROLE ACCOUNT ]   |
| Alex Miller    | amiller@invalid-xyz   | Unknown      | [ 🔴 NO MX RECORDS ]  |
+-------------------------------------------------------------------------------+
```

---

## 6. Generating & Reviewing AI Drafts `[PROPOSED]`

### 6.1 Triggering Draft Generation
1. In the **Contacts** view, select the verified contacts you wish to reach.
2. Click **Generate AI Drafts**.
3. Select your campaign theme (e.g., *"Q4 Engineering Networking"* or *"Fall Informational Interviews"*).
4. Watch the progress bar as Gemini analyzes your resume against each recipient's profile and synthesizes customized subject lines and body copy.

### 6.2 The Review Queue (Human-in-the-Loop)
1. Navigate to the **Review Queue** tab.
2. Each draft displays side-by-side with the recipient's information and your selected resume highlights.
3. Review the subject and body:
   - You can edit any sentence directly in the text editor.
   - If an email contains an unresolved placeholder (e.g., `{{company}}`), it will be highlighted in red with an alert.
4. Available Actions:
   - **Approve Draft**: Moves the email to `SCHEDULED` status.
   - **Regenerate Draft**: Prompts Gemini to rewrite the email with custom instructions (e.g., *"Make it shorter"*, *"Focus on distributed caching"*).
   - **Reject / Discard**: Deletes the draft and skips outreach for this contact.

---

## 7. Configuring Sending Rules & Schedules `[PROPOSED]`

To maintain high deliverability and avoid Google's spam filters, OutreachEngine does not blast all approved emails at once.

1. Go to **Campaigns > Settings**.
2. Configure your **Dispatch Window**:
   - **Active Days**: e.g., Monday through Friday.
   - **Active Hours**: e.g., 09:00 AM to 05:00 PM (recipient's local timezone).
3. Set **Pacing & Limits**:
   - **Daily Limit**: Recommended 25–40 emails per day (maximum platform cap: 100/day).
   - **Base Delay**: 60 seconds.
   - **Randomized Jitter**: +15 to +90 seconds (ensures dispatch intervals look natural and erratic, e.g., 73s, 112s, 89s).
4. Save the campaign schedule.

---

## 8. Starting, Pausing, & Cancelling Campaigns `[PROPOSED]`

- **Start Campaign**: Click **Activate Campaign**. The background worker will begin picking up `SCHEDULED` emails in chronological order during your allowed window.
- **Pause Campaign**: Click **Pause Campaign**. Any email currently in transmission will finish, but no new emails will be picked up until resumed.
- **Cancel Campaign**: Click **Emergency Stop / Cancel**. All pending and scheduled messages will be marked `CANCELLED` immediately.

---

## 9. Understanding Message Statuses `[PROPOSED]`

Every message follows a clear, verifiable lifecycle:

| Status | Meaning | Can It Be Edited? |
|---|---|---|
| `DRAFT_PENDING_REVIEW` | Generated by Gemini; waiting for your approval. | Yes (Full editing). |
| `APPROVED` | Approved by user; waiting to be scheduled. | Yes (Un-approve). |
| `SCHEDULED` | Assigned an exact dispatch time slot. | Yes (Cancel or reschedule). |
| `SENDING` | Worker has acquired a transactional lock and is transmitting via Gmail API. | No (In flight). |
| `SENT` | Successfully accepted by Google Gmail API servers. Gmail Message ID recorded. | No (Delivered to provider). |
| `OPENED (ESTIMATED)` | Tracking pixel was requested by recipient email client. | No. |
| `REPLIED` | A response thread was detected in your Gmail inbox. | No. |
| `BOUNCED` | A delivery failure notice (NDR) was received. | No. |
| `FAILED` | Internal error (e.g., Gmail API quota exceeded). | Yes (Retryable). |
| `CANCELLED` | User cancelled campaign before send time. | Yes (Re-schedule). |

---

## 10. Reading Dashboard Metrics & Understanding Tracking `[PROPOSED]`

The **Telemetry Dashboard** gives you an honest, unvarnished look at campaign performance:

- **Total Contacts**: Number of contacts imported and verified.
- **Pending Review**: Drafts waiting for your sign-off.
- **Scheduled**: Approved emails queued for upcoming dispatch windows.
- **Messages Sent**: Actual count of emails dispatched via Gmail API.
- **Estimated Opens**: Number of emails where the remote client loaded the tracking pixel.
  > **Note on Open Tracking:** Open rates are estimates. Privacy protections (such as Apple Mail Privacy Protection) may pre-fetch tracking pixels automatically, or some email clients may block remote images entirely. OutreachEngine treats open metrics as directional signals rather than definitive proof of human readership.
- **Replies Received**: Number of outbound threads that received an authentic incoming reply in your Gmail inbox.

---

## 11. Managing Opt-Outs & Suppression Lists `[PROPOSED]`

OutreachEngine strictly respects contact privacy and anti-spam regulations:

- If a contact replies with an unsubscribe request, click **Add to Suppression List** on their message card.
- You can also add entire company domains (e.g., `@example.com`) to the suppression list under **Settings > Privacy & Suppression**.
- Once suppressed, OutreachEngine's database constraints permanently prevent any new drafts or campaigns from sending messages to that target.

---

## 12. Disconnecting Integrations & Deleting Data `[PROPOSED]`

- **Disconnecting Gmail**: Go to **Settings > Integrations** and click **Disconnect Gmail**. OutreachEngine immediately deletes your stored OAuth refresh and access tokens from the database.
- **Deleting Campaign & Contact Data**: Under **Settings > Data Management**, you can click **Delete All Contacts & Drafts** to perform a complete GDPR/CCPA-compliant wipe of your personal outreach records.

---

## 13. Troubleshooting Common User Errors

| Problem | Likely Reason | Recommended Solution |
|---|---|---|
| Gmail disconnects or shows `Token Expired` | You changed your Google account password or revoked third-party access. | Click **Reconnect Gmail** in Settings to perform a fresh OAuth handshake. |
| Draft generation fails with `Gemini Quota Error` | Your Gemini API key exceeded free tier queries-per-minute (QPM). | Wait 60 seconds and retry, or verify your billing tier in Google AI Studio. |
| Emails are not sending during scheduled hours | Campaign is paused, or daily send limit has been reached. | Check the Campaign status bar; verify whether today's send count reached your daily maximum. |
| Google Sheets status column is not updating | Column mapping changed or Google Sheet tab name was renamed. | Re-open **Settings > Google Sheets** and re-verify tab name and column mappings. |
