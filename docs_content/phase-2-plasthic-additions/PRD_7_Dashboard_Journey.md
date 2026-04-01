# PRD 7: Patient Dashboard & Journey Tracking

## Overview

| Field | Value |
|-------|-------|
| **Modules** | 9 (Patient Dashboard) + 10 (Services & Journey) |
| **System** | System 2: Phlastic Cabinet Frontend |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 5–7 days (Modules 9 + 10 combined) |
| **Dependencies** | Modules 7 (auth) + 8 (onboarding) |
| **Priority** | P1 — Core patient experience after onboarding |

---

## Module 9: Patient Dashboard

### Goal

Main patient view after login. Shows a personalized overview of the patient's status, recent activity, and quick actions — all loaded dynamically from the Phlastic API.

---

### 9.1 — Layout

```
┌──────────────────────────────────────────────────────────────┐
│  [Logo]                            [Avatar] John  [Chat icon]│
├──────────┬───────────────────────────────────────────────────┤
│          │                                                   │
│ Sidebar  │  Main Content Area (1025px)                       │
│ (300px)  │                                                   │
│ #F0F6FF  │  ┌─────────────────────────────────────────────┐  │
│          │  │  Welcome back, John!                         │  │
│ Dashboard│  │                                              │  │
│ Personal │  │  My Activity          100 points             │  │
│ Services │  │                                              │  │
│ Journey  │  ├─────────────────────────────────────────────┤  │
│ Rewards  │  │  Section 1: Status Overview                  │  │
│ Reviews  │  │  (pipeline stage + next appointment)         │  │
│ Community│  ├─────────────────────────────────────────────┤  │
│          │  │  Section 2: Recent Activity                  │  │
│          │  │  (last 5 events timeline)                    │  │
│          │  ├─────────────────────────────────────────────┤  │
│          │  │  Section 3: Quick Actions                    │  │
│          │  │  (Upload Photos / Chat / Leave Review)       │  │
│          │  └─────────────────────────────────────────────┘  │
└──────────┴───────────────────────────────────────────────────┘
```

---

### 9.2 — Header

| Element | Behavior |
|---------|----------|
| Logo | Left-aligned, links to `/dashboard` |
| Patient avatar + name | Right-aligned, click opens dropdown: Profile / Settings / Logout |
| Chat icon | Right-aligned, opens Sellrise chatbot widget as overlay |

---

### 9.3 — Sidebar Navigation

| # | Item | Route | Icon |
|---|------|-------|------|
| 1 | Dashboard | `/dashboard` | Home / Grid |
| 2 | Personal Information | `/profile` | User |
| 3 | My Services | `/services` | Medical / Clipboard |
| 4 | Journey | `/journey` | Timeline / Map |
| 5 | Phlastic Rewards | `/rewards` | Star / Gift |
| 6 | My Reviews | `/reviews` | Star / Message |
| 7 | Community | `/community` | Users / Group |

- Active item highlighted with accent color
- Sidebar collapses to icon-only on mobile (hamburger toggle)

---

### 9.4 — Dashboard Content

**Section 1: Status Overview**

| Element | Source | Detail |
|---------|--------|--------|
| Welcome message | `patients.name` | "Welcome back, {name}!" |
| Pipeline stage bar | `patients.stage` | Visual bar: all stages shown, current one highlighted |
| Points balance | `patients.points` | "{X} points" |
| Next appointment | `journey_entries` | Nearest upcoming entry, or "No upcoming appointments" |

**Section 2: Recent Activity**

- Last 5 events from `patient_events` (timeline format)
- Each event: icon + description + relative time ("2 hours ago", "Yesterday")
- "View all" link → navigates to `/journey`

**Section 3: Quick Actions**

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 📷 Upload    │  │ 💬 Chat with │  │ ⭐ Leave a   │
│    Photos    │  │    Us        │  │    Review    │
└──────────────┘  └──────────────┘  └──────────────┘
```

| Action | Behavior |
|--------|----------|
| Upload Photos | Opens photo upload dialog (same flow as onboarding Step 5) |
| Chat with Us | Opens Sellrise chatbot widget overlay |
| Leave a Review | Navigates to `/reviews` |

---

### 9.5 — Mobile Responsive

- Sidebar collapses to bottom tab bar or hamburger menu on mobile
- Content stacks vertically
- Touch-friendly tap targets (minimum 44 px height)
- Header remains visible

---

### API Calls (Dashboard)

| Data | Endpoint |
|------|----------|
| Patient profile + points + stage | `GET /cabinet/me` |
| Recent events (last 5) | `GET /cabinet/journey` (limited to 5) |
| Next appointment | Derived from journey entries |

---

### Module 9 Acceptance Tests

- [ ] Dashboard loads after login (or after onboarding completion)
- [ ] Welcome message shows patient's first name
- [ ] Current stage displayed correctly in visual pipeline bar
- [ ] Points balance shows correct number from `patients.points`
- [ ] Recent activity shows last 5 events with correct description and relative timestamps
- [ ] "View all" link navigates to `/journey`
- [ ] Quick action "Upload Photos" opens upload dialog
- [ ] Quick action "Chat with Us" opens Sellrise chatbot widget
- [ ] Quick action "Leave a Review" navigates to `/reviews`
- [ ] Sidebar navigation works — all 7 routes accessible
- [ ] Active sidebar item highlighted with accent color
- [ ] Header shows patient avatar + name
- [ ] Profile dropdown: Profile / Settings / Logout all functional
- [ ] Chatbot icon opens Sellrise widget as overlay (not new page)
- [ ] Mobile: sidebar collapses, content stacks vertically
- [ ] Layout matches Figma (colors, spacing, structure)

---

## Module 10: Services & Journey

### Goal

Patients can view their assigned medical procedures/services and track their full journey timeline — all powered by the Phlastic API.

---

### 10.1 — My Services Page (`/services`)

Lists all of the patient's services/procedures from the `services` table.

**Layout:**

```
My Services

┌─────────────────────────────────────────────────────┐
│ Rhinoplasty                                          │
│ Status: Scheduled  |  Date: April 15, 2026          │
│ Doctor: Dr. Smith                                    │
│ Price: $12,000                                       │
│                                          [View Details]│
├─────────────────────────────────────────────────────┤
│ Botox — Forehead                                     │
│ Status: Completed  |  Date: March 10, 2026          │
│ Doctor: Dr. Johnson                                  │
│                                          [View Details]│
└─────────────────────────────────────────────────────┘
```

**Status colors:**

| Status | Display |
|--------|---------|
| `planned` | ⚫ Gray |
| `scheduled` | 🔵 Blue |
| `in_progress` | 🟠 Orange |
| `completed` | 🟢 Green |
| `cancelled` | 🔴 Red |

**Empty state:** "No services yet. Your team will add services after your consultation."

---

### Service Detail Page (`/services/:id`)

Full service detail:
- Service name, description, status, scheduled date, completed date
- Price
- Doctor card (mini profile: photo + name + specialization)
- Related photos (if any linked to this service)
- Notes from doctor
- Related journey entries

**API:** `GET /cabinet/services/:id`

---

### 10.2 — Journey Page (`/journey`)

Visual timeline of the patient's full journey from `journey_entries` table.

**Layout:**

```
My Journey

──── 2026 ────────────────────────────────────

  ● March 28    First Consultation          ✅ Completed
                Met with Dr. Smith to
                discuss rhinoplasty options

  ● March 30    Photos Submitted            ✅ Completed
                Uploaded consultation photos

  ● April 5     Doctor Review               🔵 Upcoming
                Dr. Smith will review your
                case and prepare recommendations

  ● April 15    Procedure Day               ○ Planned
                Rhinoplasty — Main OR

  ● April 22    Follow-up                   ○ Planned
                Post-op check-up
```

**Status indicators:**

| Status | Visual |
|--------|--------|
| `completed` | ✅ Green dot |
| `upcoming` | 🔵 Blue dot |
| `cancelled` | ❌ Red dot |
| `planned` | ○ Gray dot (hollow) |

- Vertical timeline with dots
- Each entry: date, title, description, status badge, staff name (if applicable)
- Sorted chronologically ascending (oldest first)
- Year grouping headers

**Empty state:** "Your journey hasn't started yet. It will appear here after your first consultation."

**API:** `GET /cabinet/journey`

---

### API Calls (Services & Journey)

| Data | Endpoint |
|------|----------|
| Services list | `GET /cabinet/services` |
| Service detail | `GET /cabinet/services/:id` |
| Journey timeline | `GET /cabinet/journey` |

---

### Module 10 Acceptance Tests

**Services:**
- [ ] Services page lists all of the patient's services/procedures
- [ ] Service status shown with correct color coding
- [ ] Service detail page shows full info (name, description, dates, price, doctor)
- [ ] Doctor card on service detail shows photo, name, specialization
- [ ] Related photos shown on service detail (if linked)
- [ ] Empty state shown when patient has no services yet

**Journey:**
- [ ] Journey page shows timeline entries in chronological order (oldest first)
- [ ] Timeline entries are correctly color-coded by status
- [ ] Year grouping headers shown correctly
- [ ] Each entry shows: date, title, description, status badge
- [ ] Staff name shown on relevant entries
- [ ] Journey entries linked to services where applicable
- [ ] Empty state shown when no journey entries yet

**General:**
- [ ] Both pages are mobile responsive
- [ ] Data loads from `GET /cabinet/services` and `GET /cabinet/journey` correctly
- [ ] Navigation between Services, Service Detail, and Journey works
- [ ] Back navigation works correctly from service detail
