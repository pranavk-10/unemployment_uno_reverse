# OutreachEngine — UI/UX Design Specification

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (Design Specification)  
**Design System Aesthetic:** Watermelon UI-Inspired Modernist Palette  
**Last Updated:** 2026-10-28  

---

## 1. Product Design Principles

1. **Truthful Telemetry**: The UI must never imply or state that an email was delivered, opened, or read when the underlying evidence does not substantiate that conclusion. All tracking data must be clearly labeled (e.g., *Estimated Opens*).
2. **High-Contrast Clarity**: Employ high-contrast typography, crisp boundaries, and generous whitespace to ensure fatigue-free operation during long review sessions.
3. **Continuous Operator Agency**: The human operator must always retain final say. Destructive or automated actions (e.g., dispatching emails, deleting contacts) must be reversible or require explicit confirmation.
4. **State Transparency**: Every asynchronous operation (AI generation, sheet syncing, email dispatch) must visually communicate its progress with micro-states (Idle, Processing, Succeeded, Failed with Retry).

---

## 2. Visual Direction & Watermelon UI References

The visual language draws inspiration from the **Watermelon UI** design language: a harmonious blend of crisp, deep obsidian/slate dark-mode surfaces paired with punchy watermelon-red (`#FF3B5C` / `#E11D48`), fresh rind green (`#10B981` / `#059669`), crisp white text, and translucent glassmorphism overlays.

*Note on Provenance: The design system is inspired by the modern high-contrast aesthetic characterized as "Watermelon UI" in executive product briefings. All component primitives specified below are implemented natively using modern CSS variables without unverified external dependencies.*

---

## 3. Design Tokens: Colors, Typography, & Spacing

### 3.1 Color Palette & Semantic Status Tokens

```css
:root {
  /* Surface & Background */
  --bg-primary: #0F172A;          /* Deep Slate / Midnight canvas */
  --bg-secondary: #1E293B;        /* Elevated Card background */
  --bg-tertiary: #334155;         /* Hover / Active interactive surface */
  --border-subtle: #334155;       /* Card & Divider border */
  --border-focus: #FF3B5C;        /* Watermelon Coral focus ring */

  /* Text & Foreground */
  --text-primary: #F8FAFC;        /* High-contrast crisp white */
  --text-secondary: #94A3B8;      /* Muted secondary labels */
  --text-tertiary: #64748B;       /* Timestamps & placeholders */

  /* Watermelon Brand Accents */
  --accent-watermelon-primary: #FF3B5C; /* Primary CTA button & Active highlight */
  --accent-watermelon-hover:   #E11D48; /* Button hover state */
  --accent-rind-green:         #10B981; /* Rind Green for positive telemetry */
  --accent-glow: rgba(255, 59, 92, 0.25);

  /* Semantic Status Tokens */
  --status-verified:   #10B981;   /* Green: DNS MX confirmed, RFC valid */
  --status-warning:    #F59E0B;   /* Amber: Role-based email or soft warning */
  --status-error:      #EF4444;   /* Red: Invalid syntax, missing MX, bounced */
  --status-scheduled:  #3B82F6;   /* Blue: Queued for timed dispatch */
  --status-sending:    #8B5CF6;   /* Purple: Worker lock acquired / in flight */
  --status-suppressed: #6B7280;   /* Gray: Opted out / blocked */
}
```

### 3.2 Typography Conventions
- **Primary Font Family**: `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
- **Monospace Font Family**: `'JetBrains Mono', 'Fira Code', monospace` (used for IDs, email headers, and timestamps)
- **Scale**:
  - `Display / Hero Title`: `32px` (2rem), `font-weight: 700`, `letter-spacing: -0.025em`
  - `Section Heading (H1)`: `24px` (1.5rem), `font-weight: 600`
  - `Card Header (H2)`: `18px` (1.125rem), `font-weight: 600`
  - `Body Text`: `14px` (0.875rem), `line-height: 1.5`, `font-weight: 400`
  - `Caption & Badges`: `12px` (0.75rem), `font-weight: 500`, uppercase tracking

### 3.3 Spacing & Grid System
- 8-point spatial grid: `4px`, `8px`, `12px`, `16px`, `24px`, `32px`, `48px`, `64px`.
- Card corner radius: `12px` (`border-radius: 0.75rem`).
- Modals & Drawers: `16px` (`border-radius: 1.0rem`).

---

## 4. Navigation & Information Architecture

The application layout uses a fixed left sidebar on desktop with fluid responsive collapse on tablet/mobile:

```text
+------------------+-------------------------------------------------------------+
| OutreachEngine 🍉| [Search Contacts or Campaigns...]       (User: Elena D. ▾)  |
+------------------+-------------------------------------------------------------+
| 📊 Dashboard     | [ HERO SUMMARY: "Active Job Search Campaign (Q4)" ]         |
| 👥 Contacts      |                                                             |
| ✍️ Review Queue  | +---------------------------------------------------------+ |
| 🚀 Campaigns     | | [Total Contacts] [Pending Review] [Scheduled] [Sent]     | |
| 🛡️ Suppression   | +---------------------------------------------------------+ |
| ⚙️ Settings      |                                                             |
|                  | [ Donut Chart: Campaign Mix ]   [ Recent Message Telemetry ]|
+------------------+-------------------------------------------------------------+
```

---

## 5. Screen & Component Specifications

### 5.1 Hero Section
- **Location**: Top of the Dashboard view.
- **Content**:
  - Active Campaign Name & Status Badge (`[🟢 ACTIVE]`, `[🟡 PAUSED]`, `[⚪ COMPLETED]`).
  - Next Scheduled Dispatch Window countdown (e.g., *"Next batch in 14 minutes (10:15 AM)"*).
  - Primary Action Buttons:
    - **Pause Campaign** (Toggles campaign state immediately).
    - **Sync Google Sheets** (Triggers immediate contact fetch).
    - **Open Review Queue** (Displays badge count of unapproved drafts).

### 5.2 Metrics Cards (Telemetry Band)
Displays 5 primary telemetry cards across a responsive grid:

| Card Title | Calculation / Source Query | Visual Presentation | Click Interaction |
|---|---|---|---|
| **Total Contacts** | `COUNT(*) FROM contacts WHERE user_id = :uid` | Slate card with neutral white number. | Navigates to Contacts view. |
| **Pending Review** | `COUNT(*) FROM messages WHERE status = 'DRAFT_PENDING_REVIEW'` | Highlighted with subtle amber border. | Opens Review Queue drawer. |
| **Scheduled** | `COUNT(*) FROM messages WHERE status = 'SCHEDULED'` | Blue accent badge with clock icon. | Filters campaign by scheduled. |
| **Dispatched (Sent)** | `COUNT(*) FROM messages WHERE status = 'SENT'` | Rind Green badge with check icon. | Shows sent message log. |
| **Estimated Opens** | `COUNT(DISTINCT message_id) FROM message_events WHERE event_type = 'OPEN'` | Watermelon Coral badge with info disclaimer tooltip. | Displays open timestamp list. |
| **Replies** | `COUNT(*) FROM messages WHERE status = 'REPLIED'` | Vibrant violet badge with reply icon. | Shows replied contact threads. |

### 5.3 Donut Chart: Campaign Composition
- **Purpose**: Displays the proportional breakdown of messages across lifecycle stages.
- **Segments**:
  1. `DRAFT` (Slate / Gray)
  2. `SCHEDULED` (Electric Blue)
  3. `SENT` (Rind Green)
  4. `OPENED` (Watermelon Coral)
  5. `REPLIED` (Violet)
  6. `BOUNCED/FAILED` (Crimson Red)
- **Center Label**: Shows total message count and overall conversion rate:
  $$\text{Response Rate} = \frac{\text{Replies}}{\text{Messages Sent}} \times 100\%$$
- **Hover Behavior**: Tooltip displaying exact count and percentage for the hovered slice.

### 5.4 Recent-Message Card Specification
- **List Display**: Displays the 10 most recent message events in reverse chronological order.
- **Row Elements**:
  - Recipient Name and Company (e.g., *Sarah Jenkins — Stripe*).
  - Subject Line snippet (truncated with ellipsis after 45 characters).
  - Status Badge with pulse animation if `SENDING`.
  - Timestamp (relative formatting: *"12m ago"*, *"2h ago"*).
- **Interactive Drawer**: Clicking a row slides open a right-side detail drawer containing:
  - Full email body.
  - Verification audit log (DNS MX timestamp, disposable check pass).
  - Gmail API Message ID and external thread link.
  - Raw event timeline (Drafted ➔ Approved ➔ Dispatched ➔ Open Pixel Loaded ➔ Reply Detected).

### 5.5 Contact Management Screen
- **Table Columns**: Checkbox, Name, Email, Company, Title, Verification Badge, Source (Sheet / Manual), Actions.
- **Filtering**: By Verification Status (`Verified`, `Warning`, `Invalid`, `Suppressed`), Company, or Tag.
- **Bulk Actions**:
  - *Verify Selected*: Re-runs DNS/MX check.
  - *Generate AI Drafts*: Opens modal to initiate Gemini drafting for selected rows.
  - *Add to Suppression List*: Immediately suppresses selected contacts.

### 5.6 Review Queue Interface (Human-in-the-Loop)
- **Split-Pane Layout**:
  - **Left Pane (Context)**: Recipient details, scraped company notes, user's resume highlights used as context.
  - **Right Pane (Editor)**:
    - Subject input field.
    - Markdown/Rich Text body editor.
    - Token validator banner (turns red if any `{{token}}` or placeholder is detected).
  - **Action Footer**:
    - `[Reject]` (Destructive outline)
    - `[Regenerate with AI]` (Opens prompt modifier input)
    - `[Approve & Schedule]` (Primary Watermelon Coral button)

### 5.7 Integration Settings
- Clean toggle cards for **Gmail OAuth**, **Google Gemini API**, and **Google Sheets Sync**.
- Each card indicates current connection status, verified email/project ID, last sync time, and an explicit **Disconnect** button with warning dialog.

---

## 6. Micro-Interactions & Animation Guidelines

- **Transitions**: Smooth 150ms–200ms ease-out transitions on button hover, card elevation, and modal backdrops (`cubic-bezier(0.16, 1, 0.3, 1)`).
- **Pulse Indicators**: Active sending processes display a subtle pulsing dot in the status badge (`animation: pulse 2s infinite`).
- **Skeleton Loaders**: Shimmer loaders for metric cards and tables during data fetching to eliminate layout shifts (CLS < 0.05).

---

## 7. Accessibility (a11y) Requirements

- **Contrast Ratios**: All text vs background must meet a minimum 4.5:1 contrast ratio (WCAG AA). Large text and headings must meet 3:1.
- **Focus Rings**: All interactive inputs, buttons, and links must have a visible `2px solid var(--border-focus)` focus outline when navigated via keyboard.
- **Screen Reader Support**: All charts and status badges must include descriptive `aria-label` attributes (e.g., `aria-label="Message sent on October 28 at 10:14 AM"`).
