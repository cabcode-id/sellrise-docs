# PRD 4: CRM Unified View & Staff Dashboard

## Overview

| Field | Value |
|-------|-------|
| **Module** | 4 — CRM Customization |
| **System** | System 1: Sellrise (existing CRM) |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 4–5 days |
| **Dependencies** | Modules 1, 2, 3 |
| **Priority** | P0 — Core staff-facing interface for patient management |

## Goal

Customize Sellrise CRM to provide a unified view of patient data — fetching live data from the Phlastic Patient Service API — while keeping all PII in Phlastic. Staff sees everything in one place without leaving Sellrise.

---

## 4.1 — Data Integration Layer

**When a lead has `external_patient_id`:**

1. CRM fetches patient data from Phlastic API: `GET /patients/{external_patient_id}`
2. Response is cached for **30 seconds** (avoid hitting API on every click)
3. CRM displays unified view: Sellrise lead data + Phlastic patient data merged

**When a lead does NOT have `external_patient_id`:**

- CRM works exactly as before — no changes, no errors, no API calls to Phlastic
- This ensures full backward compatibility with non-Phlastic workspaces

**If Phlastic API is down:**

- CRM shows Sellrise data normally
- Displays non-blocking error banner: "Patient data temporarily unavailable" (does not crash the page)

---

## 4.2 — Enhanced Lead Detail Page (7 Blocks)

| Block | Data Source | Content |
|-------|-------------|---------|
| **1. Contact Info** | Phlastic API | Name, email (with validation badge), phone, location, source URL |
| **2. Medical Interest** | Phlastic API | Procedures (as tags/chips), budget, timeframe, qualification score (visual bar) |
| **3. Photos** | Phlastic API | Gallery grid — thumbnails via `GET /patients/:id/photos/:pid`. Click for full-size modal. Photo type label on each. |
| **4. Chat Transcript** | Sellrise DB | Full chatbot conversation (existing functionality — no changes needed) |
| **5. Timeline** | Both DBs | Combined events from Sellrise + Phlastic `patient_events`, sorted by `created_at` desc. Each event: icon + type + description + timestamp. |
| **6. Notes & Tags** | Phlastic API | Add note → `POST /patients/:id/notes`. Tag input → `PUT /patients/:id` (update tags array). Display existing notes and tags. |
| **7. Stage & Owner** | Both systems | Current stage (highlighted in pipeline bar). "Move to" buttons. Owner dropdown. Changes write to BOTH Sellrise + Phlastic. |

**Qualification score visualization:**

| Score | Label | Color |
|-------|-------|-------|
| 0–30 | Cold | 🔴 Red badge/bar |
| 31–70 | Warm | 🟡 Yellow badge/bar |
| 71–100 | Hot | 🟢 Green badge/bar |

**Email validation badge:**

| Status | Badge |
|--------|-------|
| `valid` | 🟢 Green |
| `invalid` | 🔴 Red |
| `unknown` | ⚫ Gray |

---

## 4.3 — Custom Pipeline Stages (Per Workspace)

Make pipeline stages configurable per workspace. Stored in workspace settings:

```json
{
  "pipeline_stages": [
    {"id": "new",                 "label": "New",                 "color": "#9CA3AF"},
    {"id": "qualified",           "label": "Qualified",           "color": "#3B82F6"},
    {"id": "consultation_booked", "label": "Consultation Booked", "color": "#8B5CF6"},
    {"id": "photos_received",     "label": "Photos Received",     "color": "#06B6D4"},
    {"id": "doctor_reviewed",     "label": "Doctor Reviewed",     "color": "#F97316"},
    {"id": "contract_signed",     "label": "Contract Signed",     "color": "#22C55E"},
    {"id": "procedure_done",      "label": "Procedure Done",      "color": "#15803D"},
    {"id": "lost",                "label": "Lost",                "color": "#EF4444"}
  ]
}
```

If no custom stages configured → use default Sellrise stages (backward compatible).

---

## 4.4 — Pipeline Views

### Inbox View

- Shows only leads with `stage = new`
- Sorted by `created_at` desc (newest first)
- Columns: name, email, procedure, score (colored badge), `created_at`
- Click → opens lead detail page
- Count badge: "12 new leads"

### Kanban View

- One column per pipeline stage
- Cards show: name, procedure (first one), score badge, date
- Lead count in column header
- **Drag & drop between columns:**
  1. Update stage in Sellrise DB
  2. `PATCH /patients/:id/stage` on Phlastic API
  3. Both systems create `stage_changed` event
- "Lost" column on far right, visually distinct (gray background)

### List View (Table)

- All leads in table format
- Columns: name, email, phone, procedure, score, email_status, stage, owner, `created_at`
- Every column sortable (click header to toggle asc/desc)
- **Filters panel:**
  - Stage (multi-select dropdown)
  - Score range (slider: 0–100)
  - Date range (date pickers: from / to)
  - Owner (dropdown)
  - Email status (multi-select: valid / invalid / unknown)
  - Search (text: searches name, email, phone)
- Pagination: 50 per page, with page numbers

---

## 4.5 — Bulk Actions

Select multiple leads (checkboxes) → action bar appears:

| Action | Behavior |
|--------|----------|
| **Change Stage** | Dropdown to select new stage → applies to all selected leads |
| **Assign Owner** | Dropdown to select owner → applies to all selected leads |
| **Export CSV** | Downloads CSV with columns: `created_at`, name, email, phone, `procedure_interest`, `budget_range`, timeframe, `qualification_score`, `email_status`, stage, owner |

For stage changes and owner changes: update BOTH Sellrise and Phlastic (for leads with `external_patient_id`).

---

## 4.6 — Stage Change Sync Logic

Any stage change in CRM (drag-drop, button, or bulk action) must update both systems:

```
1. User changes stage in CRM
2. Update lead.stage in Sellrise DB
3. If lead has external_patient_id:
   a. PATCH /patients/:id/stage on Phlastic API
   b. If Phlastic API fails → show error toast, rollback Sellrise stage to previous value
4. Both systems create stage_changed event in their respective event tables
```

---

## 4.7 — "Send Cabinet Invite" Button

On lead detail page, add **Send Cabinet Invite** button.

**Visibility conditions** (all must be true):
- Patient has `external_patient_id`
- Patient's `account_created = false`
- Patient's `email_status != "invalid"`

**Flow:**
```
1. Staff clicks "Send Cabinet Invite"
2. CRM calls POST /auth/invite with { patient_id }
3. Phlastic:
   - Generates unique invite_token (UUID, single-use, expires in 7 days)
   - Saves invite_token + invite_sent_at on patient record
   - Sends email to patient:
       Subject: "Welcome to Phlastic — Set Up Your Account"
       Body: Link to https://cabinet.phlastic.com.au/register?token={invite_token}
   - Creates event: invite_sent
4. CRM shows confirmation: "Invite sent to john@example.com"
5. Event logged in timeline
```

---

## 4.8 — "Generate Contract" Button

On lead detail page (visible only when `stage >= doctor_reviewed`):

```
1. Staff clicks "Generate Contract"
2. CRM calls POST /patients/:id/contracts
3. Phlastic generates PDF with patient data
4. CRM shows download link
5. "Send to Patient" button → emails PDF to patient email
6. "Mark as Signed" button → contract status=signed + patient stage=contract_signed
```

Contract status shown in timeline: `generated → sent → signed`

---

## Acceptance Tests

- [ ] Lead detail page shows all 7 blocks when `external_patient_id` exists
- [ ] Lead detail page works normally when NO `external_patient_id` (regular lead, no errors)
- [ ] Contact info shows email validation badge (green valid / red invalid / gray unknown)
- [ ] Photos display as thumbnails grid, clickable for full-size modal
- [ ] Photo images load from Phlastic API (not from Sellrise storage)
- [ ] Score shows colored badge: cold (red 0–30), warm (yellow 31–70), hot (green 71–100)
- [ ] Timeline shows merged events from both Sellrise and Phlastic, sorted chronologically
- [ ] Notes can be added (saves to Phlastic API, appears in timeline immediately)
- [ ] Tags can be added and removed (saves to Phlastic API)
- [ ] Stage can be changed via buttons on detail page
- [ ] Owner can be assigned from dropdown on detail page
- [ ] Custom pipeline stages display correctly for Phlastic workspace
- [ ] Default Sellrise stages still work for workspaces without custom config
- [ ] Inbox view shows only `stage=new` leads, sorted newest first
- [ ] Kanban view shows correct columns matching pipeline stages config
- [ ] Kanban drag & drop changes stage in BOTH Sellrise and Phlastic
- [ ] List view shows all columns, all sortable
- [ ] List view filters work: stage, score range, date range, owner, `email_status`, search
- [ ] List view pagination works (50 per page)
- [ ] Bulk stage change: works for selected leads, updates both systems
- [ ] Bulk owner assignment: works for selected leads
- [ ] Bulk CSV export: downloads file with correct columns and data
- [ ] Workspace isolation: Phlastic patient data not visible in other workspaces
- [ ] API failure: if Phlastic API is down, CRM shows Sellrise data + error banner (doesn't crash)
- [ ] "Send Cabinet Invite" button visible only when: `external_patient_id` exists + `account_created=false` + `email_status != invalid`
- [ ] "Send Cabinet Invite" sends email with correct invite link
- [ ] After invite sent, event `invite_sent` logged in timeline
- [ ] "Generate Contract" button visible only when `stage >= doctor_reviewed`
- [ ] Stage change rollback works if Phlastic API call fails
