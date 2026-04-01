# PRD 8: Staff Profiles, Services & Reviews

## Overview

| Field | Value |
|-------|-------|
| **Module** | 11 — Staff Profiles, Reviews & Rewards |
| **System** | System 2: Phlastic Cabinet Frontend + Patient Service |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 3–4 days |
| **Dependencies** | Modules 7 (auth), 9 (dashboard), 10 (services) |
| **Priority** | P2 — Engagement and trust features |

## Goal

Enable patients to view medical staff profiles, track their own procedures/services, submit reviews, and manage reward points — building trust and engagement within the Phlastic platform.

---

## 11.1 — Staff Profiles

### Staff Detail Page (`/staff/:id`)

Accessible from: service detail page, dashboard links, or direct URL.

**Layout:**

```
┌──────────────────────────────────────┐
│  [Photo]                              │
│  Dr. Sarah Smith                      │
│  Plastic Surgeon                      │
│  ★★★★★ (4.8) — 24 reviews           │
│                                       │
│  Bio:                                 │
│  Dr. Smith specializes in facial      │
│  procedures with 15 years of          │
│  experience in rhinoplasty and        │
│  facelift techniques.                 │
│                                       │
│  Reviews:                             │
│  ┌────────────────────────────────┐   │
│  │ John D. ★★★★★                 │   │
│  │ "Amazing results, very..."     │   │
│  │ March 2026                     │   │
│  └────────────────────────────────┘   │
└──────────────────────────────────────┘
```

**Staff profile shows:**
- Profile photo
- Full name + role (e.g., "Plastic Surgeon")
- Average star rating + review count
- Professional bio
- Public reviews (only `status=approved` + `is_public=true`)

**API:** `GET /cabinet/staff/:id`

### Staff List

Not in sidebar navigation — accessible via links from service detail pages and dashboard.

**API:** `GET /cabinet/staff` — returns active staff (`is_active=true`)

### Staff DB Schema (`staff` table)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| name | string | yes | Full name |
| role | enum | yes | `doctor`, `nurse`, `tour_operator`, `admin` |
| specialization | string | no | Area of expertise |
| bio | text | no | Professional bio |
| photo_url | string | no | Profile photo path |
| rating | decimal | no | Average rating (1.0–5.0) — auto-calculated from reviews |
| review_count | integer | no | Count of approved public reviews |
| is_active | boolean | yes | Soft visibility toggle for staff listing |
| created_at | datetime | yes | |
| updated_at | datetime | yes | |

---

## 11.2 — My Reviews (`/reviews`)

### Patient Review Listing

Shows all reviews the patient has submitted, with current status:

| Status | Meaning | Badge |
|--------|---------|-------|
| `pending` | Awaiting staff approval | 🟡 Yellow |
| `approved` | Approved — visible if `is_public=true` | 🟢 Green |
| `rejected` | Not approved | 🔴 Red |

Patients can **edit** their own review while `status=pending`. Once approved or rejected, editing is locked.

---

### Write Review Form

```
Leave a Review

Who is this review for?
[Select staff member ▾]  — optional

Which service?
[Select service ▾]  — optional

Rating
★★★★★  (clickable stars, 1–5)

Title (optional)
[Text input]

Your review
[Textarea]

☐ Make this review public (visible to other patients)

[Submit Review]
```

**Logic:**
```
1. POST /cabinet/reviews {
     staff_id: "uuid" (optional),
     service_id: "uuid" (optional),
     rating: 5,
     title: "...",
     text: "...",
     is_public: true
   }
2. New review gets status=pending (awaiting moderation)
3. Backend creates event: review_submitted { review_id: "...", rating: 5 }
4. Backend awards 25 reward points for submitting
5. Staff must approve before review is visible on staff profile
```

### Reviews DB Schema (`reviews` table)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| patient_id | UUID | yes | — | FK → patients.id |
| staff_id | UUID | no | null | FK → staff.id |
| service_id | UUID | no | null | FK → services.id |
| rating | integer | yes | — | 1–5 stars |
| title | string | no | null | Review title |
| text | text | no | null | Review body |
| is_public | boolean | yes | `false` | Visible to other patients |
| status | enum | yes | `pending` | `pending` / `approved` / `rejected` |
| created_at | datetime | yes | | |

**Staff `rating` field** is auto-calculated as the average of all `approved` reviews for that staff member.

---

### Review Moderation (Admin/Staff Side)

- Staff approves/rejects via Sellrise CRM or Phlastic admin panel
- When `status → approved`: `staff.rating` and `staff.review_count` are recalculated
- When `is_public=true` + `status=approved`: review shows on staff profile

**API endpoints for staff/admin:**
- `GET /patients/:id` → includes reviews list
- `PATCH /reviews/:id/status` (admin endpoint, not in cabinet)

---

## 11.3 — Service Tracking (Staff Side)

Staff manage patient services via CRM / admin (not in patient cabinet).

**Service CRUD endpoints (API key auth):**
```
GET    /patients/:id/services         List patient's services
POST   /patients/:id/services         Add a service
PUT    /patients/:id/services/:sid    Update service details
PATCH  /patients/:id/services/:sid/status  Change service status
```

**Status transitions:**

```
planned → scheduled → in_progress → completed
                  ↘                 ↗
                    cancelled
```

Each status change creates a `patient_event` for the audit trail.

**Services DB Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| name | string | yes | Service/procedure name |
| description | text | no | Details |
| status | enum | yes | `planned` / `scheduled` / `in_progress` / `completed` / `cancelled` |
| scheduled_date | datetime | no | When scheduled |
| completed_date | datetime | no | When completed |
| doctor_id | UUID | no | FK → staff.id |
| price | decimal | no | Cost |
| notes | text | no | Internal staff notes |
| created_at | datetime | yes | |
| updated_at | datetime | yes | |

---

## Acceptance Tests

### Staff Profiles
- [ ] Staff profile page (`/staff/:id`) shows photo, name, role, specialization, bio, rating, review count
- [ ] Only approved + public reviews shown on staff profile
- [ ] Staff rating calculated correctly from approved reviews
- [ ] Staff profiles accessible from service detail page and dashboard links
- [ ] Staff list shows only `is_active=true` staff

### Reviews
- [ ] Write review form shows staff + service dropdowns, star rating, text fields
- [ ] `POST /cabinet/reviews` saves review with `status=pending`
- [ ] Patient can see their own reviews with status badges (pending / approved / rejected)
- [ ] Patient can edit their review while `status=pending`
- [ ] Patient cannot edit review once approved or rejected
- [ ] 25 reward points awarded automatically on review submission
- [ ] `review_submitted` event created in `patient_events`
- [ ] Approved + public review appears on staff profile page
- [ ] `staff.rating` recalculated correctly when review is approved

### Service Tracking
- [ ] Service status updates (`planned → scheduled → completed`) create patient events
- [ ] Service detail accessible from patient cabinet (`/services/:id`)
- [ ] Staff can update service status and notes via CRM/admin API
- [ ] Patient can view but NOT edit service records

### General
- [ ] All pages (`/reviews`, `/staff/:id`) are mobile responsive
- [ ] Navigation between Reviews, Staff Profile, and Dashboard works correctly
