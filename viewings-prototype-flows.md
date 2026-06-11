# Viewings Prototype — States, Flows & Behaviour

This document describes the interactive viewings prototype built for **Homey Agency workflows** on property case **PU-0001** (12 Duarte Close, Harrow HA1 4GW).

| File | Role |
|------|------|
| **`agent-view-v5.html`** | **Current prototype** — agent viewings CRM, client (vendor) dashboard, viewer (applicant) inbox, all flows below |
| `agent-view-v4.html` | Earlier iteration — agent-only dock, simpler status CTAs, suitability rating on completed viewings |
| `design-system/homeywebfoundation.css` | Shared design tokens and components (badges, modals, autosave status, etc.) |

Open `agent-view-v5.html` in a browser to walk through every flow.

**Prototype controls:** click the small **red button** in the header (next to the avatar) to open **Prototype controls** — simulated date/time and email settings. Click outside or press Escape to close.

---

## 1. Perspectives

Three switchable perspectives (header segmented control: **Agent** | **Client** | **Viewer**):

| Perspective | User | What they see |
|-------------|------|----------------|
| **Agent** | John Doe (Towers Wills) | Case panel, viewings list + detail dock, all modals |
| **Client** | Mr Hussain Somani (vendor/seller) | Read-only dashboard: stats, tabs (Summary / Upcoming / History / Offers) |
| **Viewer** | Applicant (e.g. Lisa Okinovo) | Simulated confirmation email inbox with confirm / reschedule / cancel actions |

**Important naming:**
- **Viewer** = applicant who books and attends the viewing
- **Client** = property seller (vendor) who receives updates
- **Agent** = estate agent managing viewings on the case

All three perspectives read from the same in-memory `viewings` and `leads` data.

---

## 2. Agent UI layout

### 2.1 Viewings page

- **Title row:** `Viewings` with **Update Vendor** (secondary) and **Create Viewing** (primary)
- **Left column:** search, time filters, advanced filter menu, scrollable viewing cards
- **Right column:** detail dock (~413px); **empty on first load** (`selectedId = null`) — ghost cards + “Nothing selected yet”
- Selecting a card opens full viewing details in the dock

### 2.2 List filters

**Search:** filters by applicant name (live as you type).

**Time toggles:** `All` | `Today` | `Upcoming` | `Past`
- **All** — grouped sections: Today, Upcoming (preview + load more), Past (preview + load more)
- **Today / Upcoming / Past** — full list for that bucket

**Filter button** — multi-level dropdown:
| Category | Options |
|----------|---------|
| Status | Any + each viewing status |
| By assignee | Any + agents from viewings data |
| By applicant name | Any + viewer names |
| By price proposed | Any / Yes / No |

Active filters appear as **chips** (column name + value); each chip can be cleared. **Clear all** and **Apply** in the filter panel.

### 2.3 Case panel

Left sidebar: vendor contact (Mr Hussain Somani), property address, price, workflow list with **Viewings** active under Agency.

---

## 3. Viewing statuses

A viewing can be in exactly one of five statuses:

| Status | Label in UI | Meaning |
|--------|-------------|---------|
| `scheduled` | Scheduled | Viewing booked; confirmation may or may not have been sent |
| `confirmed` | Confirmed | Applicant (or agent) has confirmed attendance |
| `completed` | Completed | Viewing took place; agent can record applicant assessment |
| `no_show` | No Show | Applicant did not attend; rebook flow applies |
| `cancelled` | Cancelled | Viewing will not happen |

### 3.1 State transition diagram

```mermaid
stateDiagram-v2
    [*] --> scheduled: Schedule viewing

    scheduled --> confirmed: Viewer confirms / Agent marks confirmed
    scheduled --> cancelled: Agent cancels / Viewer cancels
    scheduled --> no_show: Agent marks no show
    scheduled --> completed: Agent marks completed (menu only)

    confirmed --> completed: Agent marks complete / Auto after end time
    confirmed --> cancelled: Agent cancels / Viewer cancels
    confirmed --> no_show: Agent marks no show
    confirmed --> scheduled: Agent reschedules

    completed --> no_show: Agent marks as no show (correction)

    no_show --> scheduled: Schedule new viewing / Reschedule path
    no_show --> cancelled: (via agent actions)

    cancelled --> scheduled: Schedule new viewing (reschedule reasons)
```

### 3.2 Lead status (separate from viewing status)

Leads (`leads[]`) have `status: 'active' | 'lost'`.

Marking a lead as **lost** does not delete the viewing record; it sets `viewing.leadLost = true` and stores `lostReason`. The viewing remains in its current status (`no_show` or `cancelled`).

---

## 4. Data model (per viewing)

Each viewing is created via `base()` with these fields:

| Field | Purpose |
|-------|---------|
| `id`, `viewer`, `date`, `time`, `duration` | Identity and schedule (default duration 30 min) |
| `status` | One of five statuses above |
| `type`, `agent`, `accompaniedBy` | e.g. Agent accompanied, Owner accompanied, assignee name |
| `confirmationSent` | Confirmation email sent to applicant |
| `rescheduleRequested` | Viewer requested reschedule (agent sees **Reschedule** as primary CTA) |
| `reminderEnabled` | Data field (reminder UI not shown in v5 dock) |
| `notesInternal` | Agent notes — **team only**, never shared with vendor |
| `summary` | Feedback & notes — **only field shared with vendor** in update emails |
| `interest` | Very interested / Interested / Neutral / Not interested (assessment; not in vendor email) |
| `buyingSituation` | First-time buyer, chain position, cash buyer, etc. (assessment; not in vendor email) |
| `priceProposed`, `proposedPrice`, `offerPrice` | Offer tracking (assessment / client dashboard; not in vendor email) |
| `vendorNote` | Optional note on client dashboard (separate from vendor update email body) |
| `cancelReason` | Why viewing was cancelled |
| `rebook`, `rebookEmailSentAt` | No-show rebook attempt counter (max 3) |
| `leadLost`, `lostReason` | Lead marked lost with reason |
| `activity[]` | Chronological activity log entries |

**Agency agents** (for assignee): John Doe, Sarah Malik, Michael Torres. Logged-in agent: **John Doe**.

---

## 5. Seed data (demo viewings)

Default **Simulated now**: 29 May 2026, 11:00.

| Viewer | Date | Status | Notes |
|--------|------|--------|-------|
| Lisa Okinovo | 29 May | `scheduled` | No confirmation sent — full happy path |
| Mark Jensen | 29 May | `confirmed` | Confirmation sent; viewer confirmed |
| Sofia Romano | 29 May | `completed` | Assessment filled; feedback summary present |
| Derik Alrhtia | 29 May | `no_show` | Rebook attempt 1 of 3; rebook email sent |
| James Parker | 29 May | `cancelled` | “Viewer no longer interested” — Mark as Lost CTA |
| Emma Walsh | 30 May | `scheduled` | Future; confirmation sent |
| Nina Patel | 30 May | `confirmed` | Future |
| Oliver Chen | 1 Jun | `scheduled` | Future |
| Priya Sharma | 2 Jun | `scheduled` | Future |
| Tom Bradley | 27 May | `completed` | Past; feedback summary present |

**Last vendor update** meta: button tooltip shows “sent 3 days ago” until agent sends a new update.

---

## 6. Agent flows — by viewing status

### 6.1 Scheduled

**Dock shows:** Hero (avatar + name), date/time, status, accompanied by, assigned agent, **agent notes** (team only), activity log.

**Footer:**
| Condition | UI |
|-----------|-----|
| Reschedule requested | Primary: **Reschedule** |
| Auto confirmation **off** | Primary: **Send Confirmation Email** / **Resend**; copy below button: *“The confirmation email is the main touchpoint…”* |
| Auto confirmation **on** | Note only: confirmation was sent automatically (no manual button, no touchpoint copy) |

**Auto confirmation** requires **both** prototype toggles: Auto reminders **and** Auto confirmation. When enabled, confirmation sends when viewing is scheduled, when dock opens for a scheduled viewing without confirmation, and when settings change.

**Three-dot menu — Update status:** Mark as Confirmed, Mark as No Show, Cancel Viewing  
**Three-dot menu — Edit viewing:** Reschedule, Edit Applicant, Delete Viewing  
*Hidden:* Mark as Completed, Schedule New Viewing

---

### 6.2 Confirmed

**Dock:** Same info rows as scheduled + agent notes.

**Footer:**
| Condition | UI |
|-----------|-----|
| Reschedule requested | **Reschedule** |
| Default | Hint: *“Marks as complete automatically after the scheduled viewing ends…”* |

**Auto-complete:** When simulated time ≥ viewing end (start + duration) → `completed`, activity logged, toast shown.

**Three-dot menu — Update status:** Mark as Completed, Mark as No Show, Cancel Viewing  
*Hidden:* Mark as Confirmed, Schedule New Viewing

---

### 6.3 Completed

**Dock layout:**
- Compact hero: avatar, name, Completed badge, schedule
- **Agent notes** (team only)
- **Applicant Assessment** with **autosave**:
  - Feedback & Notes (`summary`) — caption: *visible to team and vendor*
  - Interest Level, Buying Situation, Price Proposed + Proposed Price (£)
  - Autosave: Saving… / Saved / error if price toggle on but amount empty
- Activity log (+ View full activity if >2 entries)

**Footer:** **Mark as no show** only (correction for auto-complete or mistaken completion).

**Three-dot menu:** Status actions hidden; Reschedule + Edit Applicant only; Delete hidden.

**Vendor update:** See §7.8 — not limited to completed-only; includes today/past viewings of any status.

---

### 6.4 No Show

**Dock shows:** Rebook attempt X of 3; warning at max attempts.

**Footer:**

| Condition | Primary | Secondary |
|-----------|---------|-----------|
| `rebook < 3` | Send Rebook Email | Schedule New Viewing |
| Rebook email already sent for attempt | Send Rebook Email (disabled) + footnote | Schedule New Viewing |
| `rebook ≥ 3`, not lost | Mark as Lost | Schedule New Viewing |
| `leadLost` | — | — |

**Three-dot menu:** Status actions hidden; Reschedule hidden; Schedule New Viewing, Edit Applicant available.

---

### 6.5 Cancelled

**Dock shows:** Cancellation reason; lost reason if `leadLost`.

**Footer** branches on `cancelReason`:

| Cancel reason | Footer CTA |
|---------------|------------|
| Viewer wants to reschedule / Scheduling conflict | Schedule New Viewing |
| Viewer found another property / Viewer no longer interested | Mark as Lost (if not lost) |
| Other reasons | No footer CTA |

**Three-dot menu:** Status actions and Reschedule hidden; Schedule New Viewing and Edit Applicant available.

---

## 7. Agent modals

### 7.1 Create Viewing (`modal-new-viewing`)

- **Applicant** (required): searchable picker — **+ New Lead** (purple) then lead names; or type to filter / create new lead inline
- Date, time, duration (30 / 45 / 60 min)
- **Accompanied by:** `Owner` | `Agent` (default **Agent**)
  - **Agent** → **Assigned to** dropdown (agency agents; default **John Doe**)
  - **Owner** → Assigned to hidden; viewing saved as Owner accompanied
- **Agent notes** — team only
- **Book Viewing** → `scheduled`; confirmation auto-sent only if both email settings on; toast reflects manual vs auto; selects new viewing in list
- 30-minute conflict warning; second click acknowledges and proceeds

### 7.2 New Lead (`modal-new-lead`)

- Required: first name, last name, email, phone, lead source
- Optional additional fields (expandable)
- Returns to Create Viewing with new lead selected

### 7.3 Reschedule (`modal-reschedule`)

- Date, time, duration, accompanied by, agent, agent notes
- On confirm → `status: scheduled`, `rescheduleRequested` cleared

### 7.4 Cancel Viewing — agent (`modal-cancel`)

Required reason dropdown → `cancelled`, reason stored, activity logged.

### 7.5 Mark as Lost (`modal-mark-lost`)

Required lost-reason dropdown (+ Other free text) → lead `lost`, `viewing.leadLost`, activity logged.

**Entry points:** No show (rebook ≥ 3), cancelled (found another property / no longer interested).

### 7.6 Delete Viewing (`modal-delete`)

Permanent removal; lead contact unaffected.

### 7.7 Update Applicant (`modal-update-applicant`)

Edit lead name, email, phone linked to viewing.

### 7.8 Send Vendor Update (`modal-vendor-update`)

**Split modal** — left: selection; right: email preview.

**Header:** “Last message sent …” (today / N days ago / date / never).

**Left panel:**
- Intro copy: only **feedback summaries** shared; agent notes stay internal
- **Personalise message** (optional textarea) — included in sent email above viewing blocks
- **Select all** + count; checkbox per eligible viewing with status badge, schedule, snippet

**Eligibility:** viewings with `date ≤ simulated today` (today + past only; **not** future). **All statuses** included (completed, no show, cancelled, scheduled today, etc.).

**Right panel — preview only:**
- Homey email component (To, Subject, greeting, personal message if any)
- **Placeholder badge** (e.g. “6 viewings selected”) — *feedback summaries will appear here for the vendor*
- Does **not** render full viewing content in preview

**Send update:**
- Builds full vendor email with per-viewing blocks
- **Only `summary` (feedback)** shared for completed viewings with feedback
- No shows / cancelled / pending: status line only (*no feedback to share*); **no** agent notes, interest, buying situation, or internal fields
- Email uses Homey template (agency footer, View feedback CTA, Powered by Homey)
- Client **Summary** tab shows sent email HTML; Update Vendor button tooltip → “sent today”

---

## 8. Viewer (applicant) flows

**Access:** Viewer perspective + persona dropdown.

**Inbox visibility:** Active invitation when `confirmationSent === true` and `status` is `scheduled` or `confirmed`. Otherwise empty state.

### 8.1 Confirmation email actions

| Action | Behaviour |
|--------|-----------|
| **Confirm attendance** | When `scheduled` → `confirmed`; clears reschedule flag |
| **Request reschedule** | Sets `rescheduleRequested`; agent primary → **Reschedule** |
| **Cancel viewing** | Opens modal |

### 8.2 Viewer cancel modal

**Required:** Note for your agent → `cancelled`, `cancelReason` = note, activity logged.

---

## 9. Client (vendor) dashboard flows

**Property:** 12 Duarte Close — read-only viewings for Mr Hussain Somani.

### 9.1 Tabs

| Tab | Content |
|-----|---------|
| **Summary** | Last **vendor update email** (after agent sends) or placeholder; recent activity cards |
| **Upcoming** | `scheduled` + `confirmed`; search |
| **History** | `completed`, `cancelled`, `no_show`; search + date-from filter |
| **Offers** | Viewings with `offerPrice`; express preference (localStorage) |

### 9.2 Status pills (client-facing)

Confirmed / Scheduled · Viewing took place / Feedback received · Did not attend (+ Rebook X/3) · Cancelled

### 9.3 Data surfaced to client

On expandable viewing cards: interest, buying situation, viewing type, date/time, offer price when relevant, **feedback summary** when completed, optional agent note (`vendorNote`).

**Not shared with vendor via update email:** agent notes (`notesInternal`), interest/buying situation/price fields (those appear on client dashboard from assessment data, but vendor **email** only carries feedback summaries).

---

## 10. Three-dot menu visibility matrix

| Menu item | Scheduled | Confirmed | Completed | No Show | Cancelled |
|-----------|:---------:|:---------:|:---------:|:-------:|:---------:|
| Mark as Confirmed | ✓ | — | — | — | — |
| Mark as Completed | — | ✓ | — | — | — |
| Mark as No Show | ✓ | ✓ | — | — | — |
| Cancel Viewing | ✓ | ✓ | — | — | — |
| Reschedule | ✓ | ✓ | ✓ | — | — |
| Schedule New Viewing | — | — | — | ✓ | ✓ |
| Edit Applicant | ✓ | ✓ | ✓ | ✓ | ✓ |
| Delete Viewing | ✓ | ✓ | — | — | ✓ |

---

## 11. Footer CTA matrix (agent dock)

| Status | Condition | Primary CTA | Secondary / notes |
|--------|-----------|-------------|-------------------|
| Scheduled | Reschedule requested | Reschedule | — |
| Scheduled | Auto confirmation off | Send / Resend Confirmation Email | Touchpoint copy below button |
| Scheduled | Auto confirmation on | — | Auto-sent note only |
| Confirmed | Default | — | Auto-complete hint |
| Confirmed | Reschedule requested | Reschedule | — |
| Completed | — | Mark as no show | — |
| No Show | rebook < 3 | Send Rebook Email | Schedule New Viewing |
| No Show | rebook ≥ 3, not lost | Mark as Lost | Schedule New Viewing |
| Cancelled | Reschedule reasons | Schedule New Viewing | — |
| Cancelled | Lost reasons, not lost | Mark as Lost | — |

---

## 12. Cross-cutting behaviour

### 12.1 Prototype controls (red header button)

Popover contains:

| Control | Effect |
|---------|--------|
| **Simulated now** (date + time) | Schedule labels (Today / Tomorrow / Yesterday); time filters; client dates; auto-complete confirmed viewings past end time |
| **Auto reminders** | Part of email settings (with auto confirmation) |
| **Auto confirmation** | When both toggles on, sends confirmation on schedule / dock open / settings change |

### 12.2 Activity log

Newest-first entries for scheduling, confirmations, viewer actions, status changes, rebook emails, vendor updates, mark as lost.

### 12.3 Autosave (completed assessment)

Debounced 500ms on: agent notes, feedback, interest, buying situation, proposed price. Price toggle requires amount when on.

### 12.4 Interest & buying situation options

**Interest:** Very interested, Interested, Neutral, Not interested

**Buying situation:** First-time buyer, Has property to sell (no offer yet), Has property under offer, Cash buyer, Investor / Buy-to-let

---

## 13. End-to-end test scenarios

### Happy path
1. **Agent** → Lisa → Send Confirmation Email (or enable auto confirmation in prototype controls)
2. **Viewer** → Lisa → Confirm attendance
3. **Agent** → Mark complete *or* advance simulated time past 13:00
4. **Agent** → Fill feedback & notes on Sofia/Tom (autosaves)
5. **Agent** → **Update Vendor** → select today/past viewings → Send update
6. **Client** → Summary shows vendor email; check Upcoming / Offers

### Filter & list
1. Search “Sofia” · toggle **Today** · open **Filter** → Status: Completed · apply chip
2. Load with empty dock → select viewing → detail appears

### Reschedule path
1. **Viewer** → Request reschedule
2. **Agent** → **Reschedule** primary CTA

### No-show → lost
1. Mark viewing no show → Send Rebook Email ×3 → **Mark as Lost**

### Vendor update (mixed statuses)
1. **Update Vendor** → includes Derik (no show, no feedback) + Sofia (feedback) + Lisa (scheduled today)
2. Preview shows placeholder badge; sent email includes status lines + feedback where present only

### Owner-accompanied viewing
1. **Create Viewing** → Accompanied by **Owner** → Assigned to hidden → book

---

## 14. Differences from v4

| Area | v4 | v5 (current) |
|------|----|----|
| Perspectives | Agent only | Agent + Client + Viewer |
| List layout | Single list | Search + time filters + advanced Filter + chips |
| Detail dock | Always shows selection | Empty state until viewing selected |
| Page actions | Create only | **Update Vendor** + **Create Viewing** |
| Completed assessment | Suitability 1–5 | Interest + buying situation + price proposed + autosave |
| Confirmed → completed | Manual only | Manual + auto after scheduled end |
| Confirmation email | Always manual send on book | Manual or auto (prototype settings) |
| Viewer inbox | — | Confirm / reschedule / cancel |
| Mark as lost | Toast only | Required reason modal |
| Vendor update | — | Split modal, Homey email, today/past viewings, feedback-only sharing |
| Accompanied by | Agent / owner / unaccompanied list | Owner \| Agent + Assigned to |
| Applicant picker | Basic dropdown | + New Lead + name list |
| Prototype settings | Inline header bars | Compact red button → popover |
| Reminder toggle in dock | Visible | Removed from UI (data field remains) |

---

## 15. Out of scope / not in prototype

- Module locked until contract signed
- Full new-lead CRM beyond basics
- Email actually delivered (simulated via toasts and perspective switches)
- Multi-property / multi-case navigation
- Persisting viewings across reload (in-memory only; client offer preference in localStorage)
- Personalise message button on email preview (personalise is left panel textarea only)

---

*Last updated to reflect `agent-view-v5.html` — viewings prototype with split vendor update, advanced filters, prototype controls popover, and confirmation email settings.*
