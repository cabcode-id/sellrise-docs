# PRD 1: Phlastic Patient Service (Core Engine)

## Overview

| Field | Value |
|-------|-------|
| **Module** | 1 — Patient Service (Core Engine) |
| **System** | System 2: Phlastic Patient Service (NEW — separate VPS, AU region) |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 3–4 days |
| **Dependencies** | VPS access from Ivan |
| **Priority** | P0 — Foundation for all other modules |

## Goal

Build a standalone, AU-compliant microservice to manage all patient PII and medical data, completely isolated from the Sellrise CRM database. This service is the foundation of the entire Phlastic system — every other module connects to it.

## Architecture Context

```
SYSTEM 2: PHLASTIC PATIENT SERVICE (NEW — separate VPS, AU region)
├── Patient DB: all patient PII, photos, medical data, consent records
├── REST API: CRUD patients, photos, compliance endpoints
├── File storage: patient photos (encrypted)
├── Patient Cabinet: web app for patients (auth, onboarding, dashboard)
└── Compliance: Australian Privacy Act 1988
```

**Core rule:** Zero patient PII in Sellrise DB. Sellrise stores only a `external_patient_id` reference. All PII lives exclusively in Phlastic Patient Service.

---

## Infrastructure Requirements

| Item | Requirement |
|------|-------------|
| Cloud provider | DigitalOcean / AWS / Vultr (AU region required) |
| Region | **Sydney (AU)** — preferred. Singapore only as fallback. |
| Minimum specs | 2 vCPU, 4 GB RAM, 50 GB SSD |
| OS | Ubuntu 22.04 LTS |
| Domain (API) | `patients.phlastic.com.au` |
| Domain (Cabinet) | `cabinet.phlastic.com.au` |
| SSL | Let's Encrypt — auto-renew via certbot |
| Backup | Daily automated DB backup, 30-day retention |
| Firewall | Port 443 (HTTPS) open; port 80 redirects to HTTPS |
| SSH | Key-based auth only — no password login |
| Email service | Resend / SendGrid / AWS SES (for invites and notifications) |

---

## Tech Stack

| Component | Requirement |
|-----------|-------------|
| API framework | Node.js / Python / Go (developer's choice) |
| Database | PostgreSQL or MySQL — **encrypted at rest** |
| File storage | Local encrypted disk (`/data/photos/`) or S3-compatible in AU region |
| Auth (staff/API) | API Key (`X-API-Key` header) — min 32 chars, one key per Sellrise workspace |
| Auth (patient) | JWT tokens (access 15 min + refresh 30 days, httpOnly cookie) |

---

## Database Schema

### Table: `patients`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | auto | Primary key |
| sellrise_lead_id | string | yes | — | Reference to Sellrise lead |
| sellrise_workspace_id | string | yes | — | Workspace isolation key |
| name | string | yes | — | Patient full name |
| email | string | yes | — | Normalized: lowercase, trimmed |
| email_status | enum | yes | `unknown` | `valid` / `invalid` / `unknown` |
| phone | string | no | null | Phone number |
| date_of_birth | date | no | null | Date of birth |
| gender | string | no | null | |
| location | string | no | null | City / country |
| procedure_interest | jsonb | yes | — | Array of desired procedures, e.g. `["rhinoplasty", "facelift"]` |
| budget_range | string | no | null | Patient's budget |
| timeframe | string | no | null | When they want the procedure |
| medical_notes | text | no | null | Medical info from chat or doctor |
| qualification_score | integer | no | null | Score from Sellrise (0–100) |
| qualification_data | jsonb | no | null | Raw chatbot answers (full JSON) |
| consent_given | boolean | yes | — | MUST be `true` before storing. If `false` → reject with 400 |
| consent_text_version | string | yes | — | Version string of consent text shown to patient |
| consent_timestamp | datetime | yes | — | Exact time consent was given |
| consent_ip | string | no | null | IP address at time of consent |
| stage | enum | yes | `new` | Pipeline stage (see enum below) |
| owner | string | no | null | Assigned staff member |
| tags | jsonb | no | `[]` | Array of string tags |
| source_url | string | no | null | Page URL where lead came from |
| referrer | string | no | null | HTTP referrer |
| utm_source | string | no | null | UTM parameter |
| utm_medium | string | no | null | UTM parameter |
| utm_campaign | string | no | null | UTM parameter |
| password_hash | string | no | null | Hashed password for cabinet login |
| invite_token | string | no | null | One-time token for account creation |
| invite_sent_at | datetime | no | null | When invite email was sent |
| account_created | boolean | no | `false` | Whether patient has created cabinet account |
| onboarding_completed | boolean | no | `false` | Whether onboarding wizard is done |
| onboarding_step | integer | no | `0` | Current wizard step (0 = not started) |
| avatar_url | string | no | null | Profile photo path |
| points | integer | no | `0` | Reward points balance |
| last_login_at | datetime | no | null | Last cabinet login timestamp |
| is_deleted | boolean | no | `false` | Soft delete flag |
| deleted_at | datetime | no | null | When soft-deleted |
| created_at | datetime | yes | `now()` | |
| updated_at | datetime | yes | `now()` | Auto-updated on every change |

**Pipeline stages enum:**
```
new → qualified → consultation_booked → photos_received → doctor_reviewed → contract_signed → procedure_done → lost
```

---

### Table: `patient_events`

Audit trail for everything that happens to a patient.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id (CASCADE on delete for hard-delete) |
| event_type | enum | yes | See full enum below |
| event_data | jsonb | yes | Event-specific data payload |
| created_by | string | yes | User ID, `"system"`, `"webhook"`, or `"patient"` |
| ip_address | string | no | For audit trail |
| created_at | datetime | yes | |

**`event_type` enum:**
```
stage_changed | note_added | photo_uploaded | photo_deleted |
contract_generated | contract_sent | contract_signed |
email_sent | owner_changed | data_exported | data_deleted |
patient_updated | account_created | onboarding_completed |
login | invite_sent | reward_earned | review_submitted
```

**Event data examples:**
```json
// stage_changed
{"from": "new", "to": "qualified"}

// note_added
{"note": "Patient prefers morning appointments"}

// photo_uploaded
{"photo_id": "uuid", "type": "face_front", "uploaded_via": "chatbot"}

// owner_changed
{"from": null, "to": "dr_smith"}

// patient_updated
{"fields_changed": ["phone", "location"], "changed_by": "admin_user_1"}

// account_created
{"method": "invite_link"}

// onboarding_completed
{"steps_completed": 7, "time_spent_seconds": 340}

// reward_earned
{"points": 50, "reason": "onboarding_completed", "new_balance": 50}

// review_submitted
{"review_id": "uuid", "rating": 5}
```

---

### Table: `photos`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| file_path | string | yes | Disk path or S3 key — **never exposed to client** |
| original_filename | string | yes | Original file name from upload |
| file_size | integer | yes | Size in bytes |
| mime_type | string | yes | `image/jpeg`, `image/png`, `image/heic` |
| type | enum | yes | `face_front`, `face_side`, `face_45`, `body`, `other`, `avatar` |
| uploaded_via | enum | yes | `chatbot`, `crm`, `cabinet` |
| consent_given | boolean | yes | Photo-specific consent |
| created_at | datetime | yes | |

**Photo storage rules:**
- Files stored at `/data/photos/{patient_id}/{photo_id}.{ext}` or equivalent S3 path
- Directory **must NOT be web-accessible** — served only via authenticated API
- File names are UUIDs (no PII in file paths)

---

### Table: `consent_records`

Immutable legal audit trail — **NEVER deleted**, even on patient hard-delete.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id — **NO cascade delete** |
| consent_type | enum | yes | `data_processing`, `photo_sharing`, `marketing` |
| consent_text | text | yes | Full text patient agreed to (not just a reference) |
| consent_version | string | yes | Version identifier (e.g. `"phlastic_v1"`) |
| given | boolean | yes | `true` = consented, `false` = withdrawn |
| ip_address | string | no | |
| user_agent | string | no | Browser user agent |
| created_at | datetime | yes | |

---

### Table: `services`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| name | string | yes | Service/procedure name |
| description | text | no | Details |
| status | enum | yes | `planned`, `scheduled`, `in_progress`, `completed`, `cancelled` |
| scheduled_date | datetime | no | When scheduled |
| completed_date | datetime | no | When completed |
| doctor_id | UUID | no | FK → staff.id |
| price | decimal | no | Cost |
| notes | text | no | Internal notes |
| created_at | datetime | yes | |
| updated_at | datetime | yes | |

---

### Table: `staff`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| name | string | yes | — | Full name |
| role | enum | yes | — | `doctor`, `nurse`, `tour_operator`, `admin` |
| specialization | string | no | null | Area of expertise |
| bio | text | no | null | Professional bio |
| photo_url | string | no | null | Profile photo path |
| rating | decimal | no | null | Average rating (1–5) |
| review_count | integer | no | `0` | Number of reviews |
| is_active | boolean | yes | `true` | Whether currently active |
| created_at | datetime | yes | | |
| updated_at | datetime | yes | | |

---

### Table: `reviews`

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
| status | enum | yes | `pending` | `pending`, `approved`, `rejected` |
| created_at | datetime | yes | | |

---

### Table: `rewards`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| points | integer | yes | Points earned (positive) or spent (negative) |
| reason | string | yes | Why points were awarded/spent |
| reference_type | string | no | `review`, `referral`, `service`, `onboarding` |
| reference_id | UUID | no | ID of related entity |
| created_at | datetime | yes | |

---

### Table: `journey_entries`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| title | string | yes | Entry title |
| description | text | no | Details |
| entry_type | enum | yes | `milestone`, `appointment`, `procedure`, `follow_up`, `note` |
| date | datetime | yes | When this happened/will happen |
| status | enum | yes | `upcoming`, `completed`, `cancelled` |
| staff_id | UUID | no | FK → staff.id |
| service_id | UUID | no | FK → services.id |
| created_at | datetime | yes | |

---

### Table: `community_groups`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| name | string | yes | — | Group name |
| description | text | no | null | Group description |
| cover_image_url | string | no | null | Cover photo path |
| member_count | integer | no | `0` | Number of members |
| is_public | boolean | yes | `true` | Public or invite-only |
| created_by | UUID | no | null | FK → patients.id or staff.id |
| created_at | datetime | yes | | |

---

### Table: `community_posts`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | — | Primary key |
| group_id | UUID | yes | — | FK → community_groups.id |
| author_id | UUID | yes | — | FK → patients.id |
| content | text | yes | — | Post content |
| image_url | string | no | null | Attached image |
| like_count | integer | no | `0` | |
| comment_count | integer | no | `0` | |
| status | enum | yes | `active` | `active`, `hidden`, `reported` |
| created_at | datetime | yes | | |

---

## API Specification

### Authentication

| Context | Method |
|---------|--------|
| Staff / Sellrise CRM | `X-API-Key` header — min 32 chars, one key per workspace |
| Patient Cabinet | `Authorization: Bearer {jwt}` — access token (15 min expiry) |

**Base URL:** `https://patients.phlastic.com.au/api/v1`

### CORS Configuration

```
Access-Control-Allow-Origin: https://app.sellrise.ai, https://cabinet.phlastic.com.au
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-API-Key, Authorization
```

> **Not `*`** — only the two specific domains above.

---

### Full Endpoint List

```
PATIENTS
────────
GET    /patients                         List patients (paginated, filtered)
GET    /patients/:id                     Single patient with events + photos metadata
POST   /patients                         Create patient (requires consent_given=true)
PUT    /patients/:id                     Update patient fields
PATCH  /patients/:id/stage               Change pipeline stage
POST   /patients/:id/notes               Add a note

PHOTOS
──────
POST   /patients/:id/photos              Upload photo (multipart/form-data)
GET    /patients/:id/photos/:pid         Download photo binary (authenticated)
DELETE /patients/:id/photos/:pid         Delete a photo

COMPLIANCE
──────────
GET    /patients/:id/export              Export ALL patient data as JSON
POST   /patients/:id/hard-delete         Permanently delete all data + photos
DELETE /patients/:id                     Soft delete (set is_deleted=true, anonymize PII)
GET    /audit-log?patient_id=:id         Access log for specific patient

CONTRACTS
────────────────────
POST   /patients/:id/contracts           Generate contract PDF
GET    /patients/:id/contracts/:cid      Download contract PDF
PATCH  /patients/:id/contracts/:cid      Update contract status (sent/signed)

EXPORT
──────
GET    /patients/export/csv              Bulk CSV export (same filters as GET /patients)

HEALTH
──────
GET    /health                           Service health check

PATIENT CABINET AUTH
─────────────────────────────────────
POST   /auth/invite                      Generate invite token + send invite email
POST   /auth/register                    Create account (invite_token + password)
POST   /auth/login                       Login (email + password → JWT)
POST   /auth/refresh                     Refresh JWT token
POST   /auth/logout                      Invalidate JWT
POST   /auth/forgot-password             Send password reset email
POST   /auth/reset-password              Reset password with token
GET    /auth/validate-token              Validate invite or reset token

PATIENT CABINET — PROFILE
─────────────────────────────────────────────
GET    /cabinet/me                        Get current patient profile
PUT    /cabinet/me                        Update own profile
POST   /cabinet/me/avatar                Upload avatar photo
GET    /cabinet/onboarding               Get current onboarding state
POST   /cabinet/onboarding/:step         Submit onboarding step data
POST   /cabinet/onboarding/complete      Mark onboarding as done

PATIENT CABINET — SERVICES
─────────────────────────────────────────────
GET    /cabinet/services                 List patient's services/procedures
GET    /cabinet/services/:id             Service detail

PATIENT CABINET — JOURNEY
────────────────────────────────────────────
GET    /cabinet/journey                  Get patient journey timeline

PATIENT CABINET — STAFF
─────────────────────────────────────────
GET    /cabinet/staff                    List staff (doctors, nurses, tour ops)
GET    /cabinet/staff/:id                Staff profile detail

PATIENT CABINET — REVIEWS
────────────────────────────────────────────
GET    /cabinet/reviews                  List patient's own reviews
POST   /cabinet/reviews                  Submit a review
PUT    /cabinet/reviews/:id              Edit own review (if status=pending)

PATIENT CABINET — REWARDS
────────────────────────────────────────────
GET    /cabinet/rewards                  Get rewards balance + history
GET    /cabinet/rewards/history          Paginated rewards transactions

PATIENT CABINET — COMMUNITY
──────────────────────────────────────────────
GET    /cabinet/community/groups         List available groups
GET    /cabinet/community/groups/:id     Group detail + posts
POST   /cabinet/community/groups/:id/join   Join a group
POST   /cabinet/community/groups/:id/leave  Leave a group
GET    /cabinet/community/posts          Feed (posts from joined groups)
POST   /cabinet/community/posts          Create a post
GET    /cabinet/community/users/:id      Public user profile
POST   /cabinet/community/posts/:id/like    Like a post
POST   /cabinet/community/posts/:id/report  Report a post
```

---

### Endpoint Details

#### `POST /patients` — Create patient

```json
// Request body
{
  "sellrise_lead_id": "lead_abc123",
  "sellrise_workspace_id": "ws_xxx",
  "name": "John Smith",
  "email": "john@example.com",
  "phone": "+61412345678",
  "procedure_interest": ["rhinoplasty"],
  "budget_range": "$10,000-15,000",
  "timeframe": "3-6 months",
  "location": "Sydney, Australia",
  "qualification_score": 85,
  "qualification_data": { "...": "raw chatbot answers" },
  "consent_given": true,
  "consent_text_version": "phlastic_v1",
  "consent_timestamp": "2026-03-30T14:22:00Z",
  "consent_ip": "203.0.113.42",
  "source_url": "https://phlastic.com/rhino",
  "referrer": "https://google.com",
  "utm_source": "google",
  "utm_medium": "cpc",
  "utm_campaign": "rhino-au"
}

// Response 201
{
  "id": "patient-uuid",
  "sellrise_lead_id": "lead_abc123",
  "email_status": "unknown",
  "stage": "new",
  "created_at": "2026-03-30T14:22:01Z"
}
```

**Upsert logic:** If a patient with same `email` + same `sellrise_workspace_id` already exists → UPDATE existing record (merge new data, don't overwrite non-null fields with null). Return `200` instead of `201`. Create `patient_updated` event.

**Error cases:**
- `400` if `consent_given != true` → `{"error": "consent_required"}`
- `400` if missing required fields → `{"error": "validation_failed", "fields": ["name", "email"]}`
- `401` if no API key
- `403` if invalid API key

#### `GET /patients` — List patients

```
Query parameters:
  ?page=1                        Default: 1
  ?per_page=50                   Default: 50, max: 100
  ?stage=qualified               Filter by stage
  ?email_status=valid            Filter by email validation status
  ?procedure_interest=rhinoplasty Filter (contains in array)
  ?created_after=2026-03-01      ISO date
  ?created_before=2026-04-01     ISO date
  ?owner=dr_smith                Filter by owner
  ?search=john                   Searches: name, email, phone (case-insensitive)
  ?sort=created_at               Sort field (default: created_at)
  ?order=desc                    asc or desc (default: desc)
```

- `is_deleted=true` patients excluded by default
- Results filtered by workspace (determined from API key)

#### `GET /patients/:id` — Single patient (full detail)

```json
{
  "patient": { "...all patient fields..." },
  "photos": [
    {
      "id": "photo-uuid",
      "type": "face_front",
      "original_filename": "photo1.jpg",
      "file_size": 2048000,
      "mime_type": "image/jpeg",
      "uploaded_via": "chatbot",
      "created_at": "2026-03-30T15:00:00Z"
    }
  ],
  "events": [
    {
      "id": "event-uuid",
      "event_type": "stage_changed",
      "event_data": {"from": "new", "to": "qualified"},
      "created_by": "system",
      "created_at": "2026-03-30T14:23:00Z"
    }
  ]
}
```

Note: `photos` contains metadata only. Use `GET /patients/:id/photos/:pid` to download the binary.

#### `PATCH /patients/:id/stage` — Change stage

```json
// Request
{"stage": "qualified"}

// Logic
// 1. Validate new stage is valid enum value
// 2. Update patient.stage
// 3. Create patient_event: type=stage_changed, data={"from":"old","to":"new"}
// 4. Update patient.updated_at

// Response 200
{"id": "...", "stage": "qualified", "previous_stage": "new"}
```

#### `POST /patients/:id/notes` — Add note

```json
// Request
{"note": "Patient prefers morning appointments"}

// Logic
// 1. Create patient_event: type=note_added, data={"note": "..."}
// 2. Update patient.updated_at

// Response 201
{"event_id": "event-uuid", "created_at": "..."}
```

#### `POST /patients/:id/photos` — Upload photo

```
Content-Type: multipart/form-data

Fields:
  file          — the image file (required)
  type          — face_front|face_side|face_45|body|other|avatar (required)
  uploaded_via  — chatbot|crm|cabinet (required)
  consent_given — true (required)

Validation:
  - Max file size: 10 MB
  - Accepted MIME types: image/jpeg, image/png, image/heic
  - consent_given must be true
```

```json
// Response 201
{
  "id": "photo-uuid",
  "type": "face_front",
  "file_size": 2048000,
  "created_at": "2026-03-30T15:00:00Z"
}
```

Side effects:
- Creates `photo_uploaded` event in `patient_events`
- Creates consent record in `consent_records` (type: `photo_sharing`)

#### `GET /patients/:id/photos/:pid` — Download photo

- Response: binary file with correct `Content-Type` header
- Requires valid `X-API-Key` or patient JWT — **no public URLs**

#### `DELETE /patients/:id` — Soft delete

```
Logic:
1. Set is_deleted=true, deleted_at=now()
2. Anonymize PII: name="[deleted]", email="deleted_{id}@deleted", phone=null, etc.
3. Keep consent_records intact
4. Create event: data_deleted

Response: 204 No Content
```

#### `POST /patients/:id/hard-delete` — Permanent deletion

```
Logic:
1. Delete all photos from disk/S3
2. Delete all records from: photos, patient_events, contracts
3. Delete patient record
4. DO NOT delete consent_records (legal requirement — kept forever)
5. Create one final consent_record: type=data_processing, given=false (withdrawal)

Response: 204 No Content
```

#### `GET /patients/:id/export` — Full data export

```json
{
  "patient": { "...all fields..." },
  "photos": [ "...metadata + base64 encoded files..." ],
  "events": [ "...all events..." ],
  "consent_records": [ "...all consent records..." ],
  "contracts": [ "...all contracts metadata..." ],
  "exported_at": "2026-03-30T16:00:00Z"
}
```

#### `GET /health` — Health check

```json
{"status": "ok", "db": "connected"}
```

---

### Error Response Format

```json
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": { "...optional additional info..." }
}
```

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `consent_required` | 400 | `consent_given` was false or missing |
| `validation_failed` | 400 | Required fields missing or invalid |
| `unauthorized` | 401 | No API key / no JWT |
| `forbidden` | 403 | Invalid API key / invalid JWT |
| `not_found` | 404 | Patient/photo/contract not found |
| `conflict` | 409 | Upsert scenario (informational) |
| `file_too_large` | 413 | Photo exceeds 10 MB |
| `unsupported_type` | 415 | Invalid file MIME type |
| `internal_error` | 500 | Unexpected server error |

---

## Australian Privacy Act 1988 — Compliance Requirements

| Principle | Requirement | Implementation |
|-----------|-------------|----------------|
| APP 1 — Management | Transparency about data handling | Privacy notice shown before data collection |
| APP 3 — Collection | Only collect necessary data, with consent | Consent checkbox + timestamp mandatory before ANY data storage |
| APP 5 — Notification | Tell patient what's collected and why | Consent text explains: what, why, who sees it |
| APP 6 — Use/Disclosure | Use only for stated purpose | Data used only for consultation process |
| APP 8 — Cross-border | Protect data if it leaves AU | VPS in AU region |
| APP 11 — Security | Protect from misuse, loss, unauthorized access | AES-256 at rest + TLS 1.2+ in transit + audit log |
| APP 12 — Access | Individual can request their data | `/patients/:id/export` endpoint |
| APP 13 — Correction | Individual can correct their data | `PUT /patients/:id` endpoint |

**Technical compliance checklist:**

- [ ] VPS in Australian region (DigitalOcean SYD / AWS ap-southeast-2)
- [ ] Database encrypted at rest (AES-256)
- [ ] All API traffic over TLS 1.2+
- [ ] Consent collected before ANY patient data stored (timestamp + IP + consent text version)
- [ ] Access log: every read/write logged (who, when, what)
- [ ] Data deletion endpoint: permanently delete patient + all associated data
- [ ] Data export endpoint: export all data for a single patient (JSON)
- [ ] No patient PII in Sellrise DB
- [ ] Photos stored encrypted, accessible only via authenticated API
- [ ] API keys rotatable, minimum 32 characters

---

## Acceptance Tests

- [ ] Service runs on separate VPS (not on Sellrise server)
- [ ] VPS is in AU region (verify with `curl ifconfig.me` or hosting dashboard)
- [ ] Database encrypted at rest
- [ ] All API traffic over HTTPS (HTTP redirects to HTTPS)
- [ ] `POST /patients` with `consent_given=false` → 400 error, patient NOT created
- [ ] `POST /patients` with `consent_given=true` + all required fields → 201, patient created
- [ ] `POST /patients` with same email + same workspace → 200, existing record updated (upsert)
- [ ] `POST /patients` with same email + DIFFERENT workspace → 201, new record (workspace isolation)
- [ ] `GET /patients` returns paginated list, default sort by `created_at` desc
- [ ] `GET /patients?stage=qualified` filters correctly
- [ ] `GET /patients?search=john` finds by name, email, or phone
- [ ] `GET /patients/:id` returns patient + photos metadata + events
- [ ] `PUT /patients/:id` updates fields, creates `patient_updated` event
- [ ] `PATCH /patients/:id/stage` changes stage, creates `stage_changed` event
- [ ] `POST /patients/:id/notes` creates `note_added` event
- [ ] `POST /patients/:id/photos` uploads file, returns 201 with photo metadata
- [ ] `POST /patients/:id/photos` with file > 10 MB → 413 error
- [ ] `POST /patients/:id/photos` with invalid MIME type → 415 error
- [ ] `GET /patients/:id/photos/:pid` returns binary file with correct `Content-Type`
- [ ] `GET /patients/:id/photos/:pid` WITHOUT API key → 401 (no public URLs)
- [ ] `DELETE /patients/:id` soft-deletes (`is_deleted=true`, PII anonymized)
- [ ] `POST /patients/:id/hard-delete` removes all data + files permanently
- [ ] `POST /patients/:id/hard-delete` does NOT delete `consent_records`
- [ ] `GET /patients/:id/export` returns complete JSON with all data
- [ ] `GET /audit-log?patient_id=xxx` returns access events
- [ ] `consent_records` table has entry for every consent action
- [ ] Request without `X-API-Key` → 401
- [ ] Request with wrong `X-API-Key` → 403
- [ ] CORS allows only Sellrise + Cabinet domains (not `*`)
- [ ] `GET /health` returns `{"status":"ok","db":"connected"}`
- [ ] All new patient fields (`password_hash`, `invite_token`, `account_created`, `onboarding_completed`, etc.) created correctly in DB
