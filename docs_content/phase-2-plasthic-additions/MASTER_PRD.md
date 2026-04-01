# Phlastic — Scope 2: Technical Specification

**Client:** Phlastic (Avi)
**Contractor:** CV Diantha (Iga Narendra)
**Ordered by:** Sellrise Limited (Ivan, Director)
**Date:** 2026-03-30
**Version:** 3.0 (Extended — includes Patient Cabinet)
**Total:** $1,000 ($500 paid upfront, $500 on acceptance)

---

## Architecture Overview

This scope extends Sellrise with a **separate patient data service** for Phlastic. Two distinct systems:

```
SYSTEM 1: SELLRISE (existing codebase, existing VPS)
├── Chatbot engine + widget (scope 1 — already working)
├── Sellrise DB: leads, sessions, scenarios, workspaces, analytics
├── CRM: lead management + new customizations (this scope)
└── Webhook: pushes lead data → Phlastic Patient Service

SYSTEM 2: PHLASTIC PATIENT SERVICE (NEW — separate VPS, AU region)
├── Patient DB: all patient PII, photos, medical data, consent records
├── REST API: CRUD patients, photos, compliance endpoints
├── File storage: patient photos (encrypted)
├── Patient Cabinet: web app for patients (auth, onboarding, dashboard)
└── Compliance: Australian Privacy Act 1988
```

**The rule:** Sellrise DB stores chatbot/lead data (scores, sessions, scenarios). Phlastic DB stores patient PII (name, email, phone, medical info, photos). Sellrise CRM calls Phlastic API to display unified view. **Zero patient PII in Sellrise DB** — only a `external_patient_id` reference.

**What we own vs. don't own:**
- ✅ **Our scope:** Patient cabinet (full frontend + backend), chatbot triggers, patient data backend, CRM customization
- ❌ **NOT our scope:** Main WordPress website — separate, we don't touch it

---

## Australian Privacy Act 1988 — Compliance

Phlastic operates in Australia. Patient data = personal information. These requirements apply to the Phlastic Patient Service:

| Principle | Requirement | How We Implement |
|-----------|------------|-----------------|
| APP 1 — Management | Transparency about data handling | Privacy notice shown before data collection |
| APP 3 — Collection | Only collect necessary data, with consent | Consent checkbox + timestamp mandatory before ANY data storage |
| APP 5 — Notification | Tell patient what's collected and why | Consent text explains: what, why, who sees it |
| APP 6 — Use/Disclosure | Use only for stated purpose | Data used only for consultation process |
| APP 8 — Cross-border | Protect data if it leaves AU | VPS in AU region |
| APP 11 — Security | Protect from misuse, loss, unauthorized access | Encryption at rest (AES-256) + transit (TLS 1.2+), access controls, audit log |
| APP 12 — Access | Individual can request their data | Data export endpoint |
| APP 13 — Correction | Individual can correct their data | Update endpoint |

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

## Data Flows

### Flow 1: New Lead from Chatbot

```
1. Visitor opens Phlastic website (WordPress)
2. Sellrise chat widget loads (already working from scope 1)
3. Visitor starts chat → Sellrise creates session + lead in Sellrise DB
4. Visitor completes qualification → answers stored in Sellrise DB
5. Visitor submits contact info + consent → Sellrise fires webhook
6. Webhook handler:
   a. POST /api/v1/patients to Phlastic service
   b. Maps chatbot answers → patient profile fields
   c. Stores consent record in Phlastic DB
   d. Returns patient_id to Sellrise
7. Sellrise stores patient_id as external_patient_id on lead record
8. (Optional) Photo upload step → photos go directly from browser to Phlastic API
```

### Flow 2: Staff Views Lead in CRM

```
1. Staff opens Sellrise CRM → lead list
2. Lead list shows data from Sellrise DB (score, stage, date)
3. Staff clicks lead → detail page
4. CRM fetches patient data from Phlastic API (using external_patient_id)
5. CRM displays unified view: Sellrise data + Phlastic patient data
6. Staff updates info → writes to Phlastic API
7. Staff changes stage → writes to BOTH Sellrise + Phlastic
```

### Flow 3: Data Export / Deletion (Compliance)

```
1. Patient requests data → staff triggers export from CRM
2. GET /api/v1/patients/:id/export → complete JSON
3. Patient requests deletion → staff triggers delete
4. POST /api/v1/patients/:id/hard-delete → permanently removes all data + photos
5. Sellrise lead record remains (anonymized — no PII, just lead_id + "deleted")
```

### Flow 4: Patient Cabinet (NEW)

```
1. Patient receives invite link from staff (via email)
2. Patient clicks link → Create Account page
3. Patient creates password → account created → auto-login
4. First login → Onboarding Wizard (7 steps)
5. Patient fills wizard → data saved to Phlastic DB via API
6. Wizard complete → redirected to Dashboard
7. Patient can view/edit profile, track services, see journey
8. All data reads/writes go through Phlastic REST API
```

---

## Module 1: Phlastic Patient Service (Separate VPS)

**Goal:** Standalone microservice with its own DB on a separate AU-compliant VPS. This is the foundation — everything else connects to it.

**Tech stack (your choice, but must support):**
- REST API (Node.js / Python / Go — whatever you prefer)
- PostgreSQL or MySQL (encrypted at rest)
- File storage for photos (local encrypted disk or S3-compatible in AU region)
- Standalone service with its own port

### Database Schema

**Table: `patients`**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | UUID | yes | auto | Primary key |
| sellrise_lead_id | string | yes | — | Reference to Sellrise lead |
| sellrise_workspace_id | string | yes | — | Workspace isolation key |
| name | string | yes | — | Patient full name |
| email | string | yes | — | Normalized: lowercase, trimmed |
| email_status | enum | yes | unknown | valid / invalid / unknown |
| phone | string | no | null | Phone number |
| date_of_birth | date | no | null | Date of birth |
| gender | string | no | null | |
| location | string | no | null | City / country |
| procedure_interest | jsonb | yes | — | Array of desired procedures, e.g. `["rhinoplasty", "facelift"]` |
| budget_range | string | no | null | Patient's budget |
| timeframe | string | no | null | When they want the procedure |
| medical_notes | text | no | null | Medical info from chat or doctor |
| qualification_score | integer | no | null | Score from Sellrise (0-100) |
| qualification_data | jsonb | no | null | Raw chatbot answers (full JSON) |
| consent_given | boolean | yes | — | MUST be `true` before storing. If `false` → reject with 400 |
| consent_text_version | string | yes | — | Version string of consent text shown to patient |
| consent_timestamp | datetime | yes | — | Exact time consent was given |
| consent_ip | string | no | null | IP address at time of consent |
| stage | enum | yes | new | Pipeline stage (see enum below) |
| owner | string | no | null | Assigned staff member |
| tags | jsonb | no | [] | Array of string tags |
| source_url | string | no | null | Page URL where lead came from |
| referrer | string | no | null | HTTP referrer |
| utm_source | string | no | null | UTM parameter |
| utm_medium | string | no | null | UTM parameter |
| utm_campaign | string | no | null | UTM parameter |
| password_hash | string | no | null | Hashed password for cabinet login |
| invite_token | string | no | null | One-time token for account creation |
| invite_sent_at | datetime | no | null | When invite email was sent |
| account_created | boolean | no | false | Whether patient has created cabinet account |
| onboarding_completed | boolean | no | false | Whether onboarding wizard is done |
| onboarding_step | integer | no | 0 | Current wizard step (0 = not started) |
| avatar_url | string | no | null | Profile photo path |
| points | integer | no | 0 | Reward points balance |
| last_login_at | datetime | no | null | Last cabinet login timestamp |
| is_deleted | boolean | no | false | Soft delete flag |
| deleted_at | datetime | no | null | When soft-deleted |
| created_at | datetime | yes | now() | |
| updated_at | datetime | yes | now() | Auto-updated on every change |

**Pipeline stages enum:**
```
new → qualified → consultation_booked → photos_received → doctor_reviewed → contract_signed → procedure_done → lost
```

**Table: `patient_events`** (audit trail for everything that happens to a patient)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id (CASCADE on delete for hard-delete) |
| event_type | enum | yes | One of: `stage_changed`, `note_added`, `photo_uploaded`, `photo_deleted`, `contract_generated`, `contract_sent`, `contract_signed`, `email_sent`, `owner_changed`, `data_exported`, `data_deleted`, `patient_updated`, `account_created`, `onboarding_completed`, `login`, `invite_sent`, `reward_earned`, `review_submitted` |
| event_data | jsonb | yes | Event-specific data. Examples below. |
| created_by | string | yes | User ID or `"system"` or `"webhook"` or `"patient"` |
| ip_address | string | no | For audit trail |
| created_at | datetime | yes | |

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

**Table: `photos`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| file_path | string | yes | Path on disk or S3 key. NOT a public URL. Never exposed to client. |
| original_filename | string | yes | Original file name from upload |
| file_size | integer | yes | Size in bytes |
| mime_type | string | yes | `image/jpeg`, `image/png`, `image/heic` |
| type | enum | yes | `face_front`, `face_side`, `face_45`, `body`, `other`, `avatar` |
| uploaded_via | enum | yes | `chatbot`, `crm`, `cabinet` |
| consent_given | boolean | yes | Photo-specific consent |
| created_at | datetime | yes | |

**Photo storage rules:**
- Files stored on disk in `/data/photos/{patient_id}/{photo_id}.{ext}` or S3 equivalent
- Directory must NOT be web-accessible — served only through API with auth
- File names are UUIDs, not original names (no PII in file paths)

**Table: `consent_records`** (immutable audit trail — NEVER deleted, even on hard-delete of patient)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id (NO cascade — kept forever) |
| consent_type | enum | yes | `data_processing`, `photo_sharing`, `marketing` |
| consent_text | text | yes | Exact text patient agreed to (full copy, not just reference) |
| consent_version | string | yes | Version identifier (e.g. `"phlastic_v1"`) |
| given | boolean | yes | `true` = consented, `false` = withdrawn |
| ip_address | string | no | |
| user_agent | string | no | Browser user agent |
| created_at | datetime | yes | |

**Table: `services`** (NEW — for patient services/procedures tracking)

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

**Table: `staff`** (NEW — doctors, nurses, tour operators)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| name | string | yes | Full name |
| role | enum | yes | `doctor`, `nurse`, `tour_operator`, `admin` |
| specialization | string | no | Area of expertise |
| bio | text | no | Professional bio |
| photo_url | string | no | Profile photo path |
| rating | decimal | no | Average rating (1-5) |
| review_count | integer | no | 0 | Number of reviews |
| is_active | boolean | yes | true | Whether currently active |
| created_at | datetime | yes | |
| updated_at | datetime | yes | |

**Table: `reviews`** (NEW — patient reviews)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| staff_id | UUID | no | FK → staff.id (if review is about specific staff) |
| service_id | UUID | no | FK → services.id (if review is about specific service) |
| rating | integer | yes | 1-5 stars |
| title | string | no | Review title |
| text | text | no | Review body |
| is_public | boolean | yes | false | Whether visible to other patients |
| status | enum | yes | `pending`, `approved`, `rejected` |
| created_at | datetime | yes | |

**Table: `rewards`** (NEW — reward transactions)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id |
| points | integer | yes | Points earned (positive) or spent (negative) |
| reason | string | yes | Why points were awarded/spent |
| reference_type | string | no | `review`, `referral`, `service`, `onboarding` |
| reference_id | UUID | no | ID of related entity |
| created_at | datetime | yes | |

**Table: `journey_entries`** (NEW — patient journey timeline)

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

**Table: `community_groups`** (NEW — community feature)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| name | string | yes | Group name |
| description | text | no | Group description |
| cover_image_url | string | no | Cover photo path |
| member_count | integer | no | 0 | Number of members |
| is_public | boolean | yes | true | Public or invite-only |
| created_by | UUID | no | FK → patients.id or staff.id |
| created_at | datetime | yes | |

**Table: `community_posts`** (NEW)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| group_id | UUID | yes | FK → community_groups.id |
| author_id | UUID | yes | FK → patients.id |
| content | text | yes | Post content |
| image_url | string | no | Attached image |
| like_count | integer | no | 0 | |
| comment_count | integer | no | 0 | |
| status | enum | yes | `active`, `hidden`, `reported` |
| created_at | datetime | yes | |

### API Endpoints

**Authentication:** API Key in header `X-API-Key`. One key per Sellrise workspace. Minimum 32 characters.

**Patient Auth (Cabinet):** JWT token in `Authorization: Bearer {token}` header. Issued on login/registration.

**Base URL:** `https://{domain}/api/v1`

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

CONTRACTS (Module 6)
────────────────────
POST   /patients/:id/contracts           Generate contract PDF
GET    /patients/:id/contracts/:cid      Download contract PDF
PATCH  /patients/:id/contracts/:cid      Update contract status (sent/signed)

EXPORT
──────
GET    /patients/export/csv              Bulk CSV export (with same filters as GET /patients)

HEALTH
──────
GET    /health                           Service health check (returns {"status":"ok","db":"connected"})

PATIENT CABINET AUTH (Module 7 — NEW)
─────────────────────────────────────
POST   /auth/invite                      Generate invite token + send invite email
POST   /auth/register                    Create account (invite_token + password)
POST   /auth/login                       Login (email + password → JWT)
POST   /auth/refresh                     Refresh JWT token
POST   /auth/logout                      Invalidate JWT
POST   /auth/forgot-password             Send password reset email
POST   /auth/reset-password              Reset password with token

PATIENT CABINET — PROFILE (Module 7-9 — NEW)
─────────────────────────────────────────────
GET    /cabinet/me                        Get current patient profile
PUT    /cabinet/me                        Update own profile
POST   /cabinet/me/avatar                Upload avatar photo
GET    /cabinet/onboarding               Get current onboarding state
POST   /cabinet/onboarding/:step         Submit onboarding step data
POST   /cabinet/onboarding/complete      Mark onboarding as done

PATIENT CABINET — SERVICES (Module 10 — NEW)
─────────────────────────────────────────────
GET    /cabinet/services                 List patient's services/procedures
GET    /cabinet/services/:id             Service detail

PATIENT CABINET — JOURNEY (Module 10 — NEW)
────────────────────────────────────────────
GET    /cabinet/journey                  Get patient journey timeline

PATIENT CABINET — STAFF (Module 11 — NEW)
─────────────────────────────────────────
GET    /cabinet/staff                    List staff (doctors, nurses, tour ops)
GET    /cabinet/staff/:id                Staff profile detail

PATIENT CABINET — REVIEWS (Module 11 — NEW)
────────────────────────────────────────────
GET    /cabinet/reviews                  List patient's own reviews
POST   /cabinet/reviews                  Submit a review
PUT    /cabinet/reviews/:id              Edit own review (if status=pending)

PATIENT CABINET — REWARDS (Module 11 — NEW)
────────────────────────────────────────────
GET    /cabinet/rewards                  Get rewards balance + history
GET    /cabinet/rewards/history          Paginated rewards transactions

PATIENT CABINET — COMMUNITY (Module 12 — NEW)
──────────────────────────────────────────────
GET    /cabinet/community/groups         List available groups
GET    /cabinet/community/groups/:id     Group detail + posts
POST   /cabinet/community/groups/:id/join  Join a group
POST   /cabinet/community/groups/:id/leave Leave a group
GET    /cabinet/community/posts          Feed (posts from joined groups)
POST   /cabinet/community/posts          Create a post
GET    /cabinet/community/users/:id      Public user profile
POST   /cabinet/community/posts/:id/like Like a post
POST   /cabinet/community/posts/:id/report Report a post
```

### Endpoint Details (Modules 1-6)

**POST /patients** — Create patient
```
Request body (JSON):
{
  "sellrise_lead_id": "lead_abc123",          // required
  "sellrise_workspace_id": "ws_xxx",          // required
  "name": "John Smith",                        // required
  "email": "john@example.com",                 // required
  "phone": "+61412345678",                     // optional
  "procedure_interest": ["rhinoplasty"],       // required (array)
  "budget_range": "$10,000-15,000",            // optional
  "timeframe": "3-6 months",                   // optional
  "location": "Sydney, Australia",             // optional
  "qualification_score": 85,                   // optional
  "qualification_data": { ... },               // optional (raw chatbot JSON)
  "consent_given": true,                       // required — MUST be true
  "consent_text_version": "phlastic_v1",       // required
  "consent_timestamp": "2026-03-30T14:22:00Z", // required
  "consent_ip": "203.0.113.42",                // optional
  "source_url": "https://phlastic.com/rhino",  // optional
  "referrer": "https://google.com",            // optional
  "utm_source": "google",                      // optional
  "utm_medium": "cpc",                         // optional
  "utm_campaign": "rhino-au"                   // optional
}

Response 201:
{
  "id": "patient-uuid",
  "sellrise_lead_id": "lead_abc123",
  "email_status": "unknown",
  "stage": "new",
  "created_at": "2026-03-30T14:22:01Z"
}

Errors:
- 400 if consent_given != true (body: {"error": "consent_required"})
- 400 if missing required fields (body: {"error": "validation_failed", "fields": ["name", "email"]})
- 401 if no API key
- 403 if invalid API key
```

**Upsert logic:** If a patient with same `email` + same `sellrise_workspace_id` already exists → UPDATE the existing record (merge new data, don't overwrite existing non-null fields with null). Return 200 instead of 201. Create `patient_updated` event.

**GET /patients** — List patients
```
Query parameters:
  ?page=1                              Default: 1
  ?per_page=50                         Default: 50, max: 100
  ?stage=qualified                     Filter by stage
  ?email_status=valid                  Filter by email validation
  ?procedure_interest=rhinoplasty      Filter (contains in array)
  ?created_after=2026-03-01            ISO date
  ?created_before=2026-04-01           ISO date
  ?owner=dr_smith                      Filter by owner
  ?search=john                         Searches: name, email, phone (case-insensitive)
  ?sort=created_at                     Sort field (default: created_at)
  ?order=desc                          asc or desc (default: desc)

Response 200:
{
  "data": [ ... patient objects ... ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 142,
    "total_pages": 3
  }
}

Notes:
- is_deleted=true patients are EXCLUDED by default
- Results filtered by workspace (determined from API key)
```

**GET /patients/:id** — Single patient with full details
```
Response 200:
{
  "patient": { ... all patient fields ... },
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

Note: photos array contains metadata only. To download actual photo file,
use GET /patients/:id/photos/:pid
```

**PATCH /patients/:id/stage** — Change stage
```
Request body:
{"stage": "qualified"}

Logic:
1. Validate new stage is a valid enum value
2. Update patient.stage
3. Create patient_event: type=stage_changed, data={"from":"old","to":"new"}
4. Update patient.updated_at

Response 200:
{"id": "...", "stage": "qualified", "previous_stage": "new"}
```

**POST /patients/:id/notes** — Add note
```
Request body:
{"note": "Patient prefers morning appointments"}

Logic:
1. Create patient_event: type=note_added, data={"note": "..."}
2. Update patient.updated_at

Response 201:
{"event_id": "event-uuid", "created_at": "..."}
```

**POST /patients/:id/photos** — Upload photo
```
Content-Type: multipart/form-data

Fields:
  file          — the image file (required)
  type          — face_front|face_side|face_45|body|other|avatar (required)
  uploaded_via  — chatbot|crm|cabinet (required)
  consent_given — true (required)

Validation:
  - Max file size: 10MB
  - Accepted MIME types: image/jpeg, image/png, image/heic
  - consent_given must be true

Response 201:
{
  "id": "photo-uuid",
  "type": "face_front",
  "file_size": 2048000,
  "created_at": "2026-03-30T15:00:00Z"
}

Creates events:
  - photo_uploaded in patient_events
  - consent record in consent_records (type: photo_sharing)
```

**GET /patients/:id/photos/:pid** — Download photo
```
Response: binary file with correct Content-Type header
No public URLs — every request must have valid API key or patient JWT
```

**DELETE /patients/:id** — Soft delete
```
Logic:
1. Set is_deleted=true, deleted_at=now()
2. Anonymize PII: name="[deleted]", email="deleted_{id}@deleted", phone=null, etc.
3. Keep consent_records intact
4. Create event: data_deleted

Response 204 No Content
```

**POST /patients/:id/hard-delete** — Permanent deletion (compliance)
```
Logic:
1. Delete all photos from disk/S3
2. Delete all records from: photos, patient_events, contracts
3. Delete patient record
4. DO NOT delete consent_records (legal requirement — keep forever)
5. Create one final consent_record: type=data_processing, given=false (withdrawal)

Response 204 No Content
```

**GET /patients/:id/export** — Data export (compliance)
```
Response 200:
{
  "patient": { ... all fields ... },
  "photos": [ ... metadata + base64 encoded files ... ],
  "events": [ ... all events ... ],
  "consent_records": [ ... all consent records ... ],
  "contracts": [ ... all contracts metadata ... ],
  "exported_at": "2026-03-30T16:00:00Z"
}
```

**GET /patients/export/csv** — Bulk CSV export
```
Same filters as GET /patients (query params)
Response: CSV file download
Columns: created_at, name, email, phone, procedure_interest, budget_range, timeframe, qualification_score, email_status, stage, owner
```

### Error Response Format (consistent across all endpoints)
```json
{
  "error": "error_code",
  "message": "Human-readable description",
  "details": { ... optional additional info ... }
}

Error codes:
  consent_required    — 400 — consent_given was false or missing
  validation_failed   — 400 — required fields missing or invalid
  unauthorized        — 401 — no API key / no JWT provided
  forbidden           — 403 — invalid API key / invalid JWT
  not_found           — 404 — patient/photo/contract not found
  conflict            — 409 — upsert scenario (informational)
  file_too_large      — 413 — photo exceeds 10MB
  unsupported_type    — 415 — invalid file MIME type
  internal_error      — 500 — unexpected server error
```

### CORS Configuration
```
Access-Control-Allow-Origin: https://app.sellrise.ai, https://cabinet.phlastic.com.au (Sellrise + Cabinet domains — NOT *)
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-API-Key, Authorization
```

### Module 1 Acceptance Tests

- [ ] Service runs on separate VPS (not on Sellrise server)
- [ ] VPS is in AU region (verify with `curl ifconfig.me` or hosting dashboard)
- [ ] Database encrypted at rest
- [ ] All API traffic over HTTPS (HTTP redirects to HTTPS)
- [ ] POST /patients with `consent_given=false` → 400 error, patient NOT created
- [ ] POST /patients with `consent_given=true` + all required fields → 201, patient created
- [ ] POST /patients with same email + same workspace → 200, existing record updated (upsert)
- [ ] POST /patients with same email + DIFFERENT workspace → 201, new record (workspace isolation)
- [ ] GET /patients returns paginated list, default sort by created_at desc
- [ ] GET /patients?stage=qualified filters correctly
- [ ] GET /patients?search=john finds by name, email, or phone
- [ ] GET /patients/:id returns patient + photos metadata + events
- [ ] PUT /patients/:id updates fields, creates patient_updated event
- [ ] PATCH /patients/:id/stage changes stage, creates stage_changed event
- [ ] POST /patients/:id/notes creates note_added event
- [ ] POST /patients/:id/photos uploads file, returns 201 with photo metadata
- [ ] POST /patients/:id/photos with file > 10MB → 413 error
- [ ] POST /patients/:id/photos with invalid MIME type → 415 error
- [ ] GET /patients/:id/photos/:pid returns binary file with correct Content-Type
- [ ] GET /patients/:id/photos/:pid WITHOUT API key → 401 (no public URLs)
- [ ] DELETE /patients/:id soft-deletes (is_deleted=true, PII anonymized)
- [ ] POST /patients/:id/hard-delete removes all data + files permanently
- [ ] POST /patients/:id/hard-delete does NOT delete consent_records
- [ ] GET /patients/:id/export returns complete JSON with all data
- [ ] GET /audit-log?patient_id=xxx returns access events
- [ ] consent_records table has entry for every consent action
- [ ] Request without X-API-Key → 401
- [ ] Request with wrong X-API-Key → 403
- [ ] CORS allows only Sellrise + Cabinet domains
- [ ] GET /health returns `{"status":"ok","db":"connected"}`
- [ ] New patient fields (password_hash, invite_token, account_created, onboarding_completed, etc.) created correctly

**Estimate:** 3-4 days
**Dependencies:** None — start immediately (after VPS access)

---

## Module 2: Sellrise → Phlastic Webhook Pipeline

**Goal:** When a chatbot conversation completes in Sellrise, automatically push lead data to Phlastic Patient Service.

**Where:** Sellrise codebase (existing repo)

### What to Build

**1. New workspace setting: Patient Service configuration**

Add to workspace settings (Sellrise admin panel or config):
```json
{
  "patient_service": {
    "enabled": true,
    "base_url": "https://patients.phlastic.com.au/api/v1",
    "api_key": "pk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "profile_mapping": {
      "step_procedure_interest": "procedure_interest",
      "step_budget": "budget_range",
      "step_timeline": "timeframe",
      "step_location": "location"
    }
  }
}
```

`profile_mapping` maps Sellrise chatbot step IDs to Phlastic patient fields. This makes the integration configurable — different chatbot scenarios can map different steps.

**2. New field on Sellrise `leads` table**

```sql
ALTER TABLE leads ADD COLUMN external_patient_id VARCHAR(255) NULL;
```

This is the ONLY new column in Sellrise DB. No patient PII.

**3. Webhook logic (fires on `lead_submitted` event)**

```
1. lead_submitted event fires in Sellrise
2. Check: does this workspace have patient_service.enabled = true?
3. If no → stop (normal Sellrise behavior, no webhook)
4. If yes → build payload:
   a. Take lead data (name, email, phone from contact form)
   b. Map chatbot answers to patient fields using profile_mapping config
   c. Include UTM, source_url, referrer from session
   d. Include consent data from chatbot
5. POST payload to {base_url}/patients
6. On success (201/200):
   a. Save returned patient_id as external_patient_id on lead record
   b. Log: success, timestamp, patient_id
7. On failure (4xx/5xx):
   a. Retry up to 3 times with exponential backoff: 1s, 5s, 30s
   b. Log each attempt: URL, status code, response body, timestamp
   c. After 3 failures: log final failure, mark webhook as failed
   d. Lead is still saved in Sellrise (webhook failure does NOT block lead creation)
```

**4. Payload Sellrise sends to Phlastic:**

```json
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
  "qualification_data": {
    "answers": [
      {"step_id": "step_procedure", "question": "What procedure are you interested in?", "answer": "Rhinoplasty"},
      {"step_id": "step_budget", "question": "What is your budget range?", "answer": "$10,000-15,000"},
      {"step_id": "step_timeline", "question": "When would you like the procedure?", "answer": "3-6 months"}
    ]
  },
  "consent_given": true,
  "consent_text_version": "phlastic_v1",
  "consent_timestamp": "2026-03-30T14:22:00Z",
  "consent_ip": "203.0.113.42",
  "source_url": "https://phlastic.com/rhinoplasty",
  "referrer": "https://google.com",
  "utm_source": "google",
  "utm_medium": "cpc",
  "utm_campaign": "rhino-au"
}
```

**5. Webhook log (visible in Sellrise admin)**

Store each webhook attempt:
```json
{
  "lead_id": "lead_abc123",
  "workspace_id": "ws_xxx",
  "url": "https://patients.phlastic.com.au/api/v1/patients",
  "method": "POST",
  "status_code": 201,
  "response_time_ms": 340,
  "success": true,
  "attempt": 1,
  "patient_id_returned": "patient-uuid",
  "created_at": "2026-03-30T14:22:01Z"
}
```

### Module 2 Acceptance Tests

- [ ] Completed chatbot session with consent=true → patient auto-created in Phlastic DB
- [ ] Completed chatbot session with consent=false → webhook NOT fired (no data sent)
- [ ] sellrise_lead_id correctly links the two records
- [ ] Sellrise lead record has external_patient_id populated after successful webhook
- [ ] profile_mapping config correctly maps chatbot step answers to patient fields
- [ ] Repeat submission with same email → upsert in Phlastic (no duplicate patient)
- [ ] Webhook failure → retries 3 times with 1s/5s/30s backoff
- [ ] After 3 failures → logged as failed, lead still saved in Sellrise
- [ ] Webhook call log visible in Sellrise admin (timestamp, status, response time)
- [ ] Workspace WITHOUT patient_service config → no webhook fired, normal behavior
- [ ] Webhook payload includes all UTM/source/referrer data from session

**Estimate:** 2 days
**Dependencies:** Module 1 (Phlastic service must be running and accepting requests)

---

## Module 3: Photo Upload in Chatbot

**Goal:** Add photo upload capability to Sellrise chatbot. Photos go directly from the user's browser to Phlastic Patient Service — Sellrise backend never touches the photo files.

**Where:** Sellrise codebase (scenario engine + widget)

### What to Build

**1. New scenario step type: `photo_upload`**

```json
{
  "id": "step_photos",
  "type": "photo_upload",
  "config": {
    "prompt": "Please upload photos for your consultation. Our doctor will review them before your appointment.",
    "required": false,
    "max_files": 5,
    "max_size_mb": 10,
    "accepted_types": ["image/jpeg", "image/png", "image/heic"],
    "consent_text": "I consent to sharing these photos with the Phlastic medical team for consultation purposes. Photos will be stored securely and used only for medical assessment.",
    "photo_types": ["face_front", "face_side", "face_45", "body", "other"],
    "upload_endpoint": "configured from patient_service.base_url"
  }
}
```

This step type should be available in the scenario builder just like text/buttons/form steps.

**2. Widget UI for photo upload step**

When chatbot reaches this step, widget displays:
```
┌─────────────────────────────────────────┐
│ Please upload photos for your           │
│ consultation. Our doctor will review    │
│ them before your appointment.           │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌────────┐  │
│  │ + Add   │  │ + Add   │  │ + Add  │  │
│  │ Photo   │  │ Photo   │  │ Photo  │  │
│  └─────────┘  └─────────┘  └────────┘  │
│                                         │
│  Photo type: [face_front ▾]             │
│                                         │
│  ☐ I consent to sharing these photos    │
│    with the Phlastic medical team...    │
│                                         │
│  [Upload & Continue]  (disabled until   │
│                        consent checked) │
└─────────────────────────────────────────┘
```

- File picker + camera option on mobile
- Photo type selector (dropdown per photo)
- Consent checkbox — MUST be checked before upload button activates
- Upload progress bar per photo
- Success/failure indicators per photo
- "Skip" option if step is not required

**3. Upload flow (browser → Phlastic directly)**

```
1. User selects photo(s) + checks consent + picks photo type
2. Widget reads patient_service config from workspace settings
3. Widget gets external_patient_id from lead record
   (lead must be submitted before photos — photo step must come AFTER contact form)
4. For each photo:
   a. Widget sends POST /api/v1/patients/{patient_id}/photos directly to Phlastic
   b. Headers: X-API-Key from patient_service config
   c. Body: multipart/form-data (file + type + uploaded_via=chatbot + consent_given=true)
   d. Show progress bar during upload
   e. On success → show green checkmark
   f. On failure → show retry button
5. After all photos uploaded (or skipped) → proceed to next chatbot step
6. Sellrise logs event: photo_uploaded (photo_id references only, NO photo data)
```

**Important constraint:** The photo_upload step MUST come AFTER the lead submission step in the scenario. The patient must exist in Phlastic DB before photos can be uploaded (we need the patient_id). If the scenario builder places photo_upload before lead submission, show a warning.

**4. Fallback if patient doesn't exist yet**

If for some reason the lead hasn't been submitted yet when the user reaches the photo step:
- Queue photos in browser memory (as blobs)
- After lead submission + webhook completes + patient_id received → upload queued photos
- This is a fallback, not the primary flow

### Module 3 Acceptance Tests

- [ ] `photo_upload` step type available in scenario builder
- [ ] Widget shows file picker + consent checkbox when reaching this step
- [ ] Consent checkbox must be checked before upload button is active
- [ ] Upload goes directly to Phlastic API (verify in browser Network tab: request goes to Phlastic domain, not Sellrise)
- [ ] Upload request includes X-API-Key header
- [ ] Max 5 files enforced (widget prevents adding more)
- [ ] Max 10MB per file enforced (widget shows error for larger files)
- [ ] Only JPEG, PNG, HEIC accepted (widget shows error for other types)
- [ ] Mobile browser shows camera option in file picker
- [ ] Upload progress bar shown per photo
- [ ] Successful upload shows confirmation in chat
- [ ] Failed upload shows retry button
- [ ] "Skip" button works if step is not required
- [ ] Consent record created in Phlastic consent_records (type: photo_sharing)
- [ ] Sellrise logs photo_uploaded event (with photo_id, without photo data)
- [ ] Scenario builder warns if photo_upload is placed before lead submission step

**Estimate:** 2-3 days
**Dependencies:** Module 1 (Phlastic API) + Module 2 (webhook, for external_patient_id)

---

## Module 4: CRM Customization (Sellrise Codebase)

**Goal:** Customize Sellrise CRM to display Phlastic patient data from the Phlastic API. Unified view — staff sees everything in one place.

**Where:** Sellrise codebase (existing CRM)

### What to Build

**4.1 — Data integration layer**

When a lead has `external_patient_id`:
- CRM fetches patient data from Phlastic API: `GET /patients/{external_patient_id}`
- Caches response for 30 seconds (to avoid hitting API on every click)
- Displays unified view merging Sellrise lead data + Phlastic patient data

When a lead does NOT have `external_patient_id`:
- CRM works exactly as before — no changes, no errors, no API calls

**4.2 — Enhanced Lead Detail Page (7 blocks)**

| Block | Source | Content |
|-------|--------|---------|
| **1. Contact Info** | Phlastic API | Name, email (with validation badge: green=valid, red=invalid, gray=unknown), phone, location, source URL |
| **2. Medical Interest** | Phlastic API | Procedures (as tags/chips), budget, timeframe, qualification score (visual bar) |
| **3. Photos** | Phlastic API | Gallery grid. Thumbnails (loaded via GET /patients/:id/photos/:pid). Click for full-size in modal. Photo type label on each. |
| **4. Chat Transcript** | Sellrise DB | Full chatbot conversation (existing functionality, no changes needed) |
| **5. Timeline** | Both DBs | Combined events from Sellrise lead events + Phlastic patient_events, sorted by created_at desc. Each event: icon + type + description + timestamp. |
| **6. Notes & Tags** | Phlastic API | Add note button -> POST /patients/:id/notes. Tag input -> PUT /patients/:id (update tags array). Display existing notes and tags. |
| **7. Stage & Owner** | Both | Current stage (highlighted in pipeline bar). "Move to" buttons for next/previous stage. Owner dropdown (workspace users). Changes write to BOTH Sellrise + Phlastic. |

**Qualification score visualization:**
```
0-30  = Cold  — Red badge/bar    — Label: "Cold"
31-70 = Warm  — Yellow badge/bar — Label: "Warm"
71-100 = Hot  — Green badge/bar  — Label: "Hot"
```

**4.3 — Custom Pipeline Stages (per workspace)**

Make pipeline stages configurable per workspace. Stored in workspace settings:

```json
{
  "pipeline_stages": [
    {"id": "new", "label": "New", "color": "#9CA3AF"},
    {"id": "qualified", "label": "Qualified", "color": "#3B82F6"},
    {"id": "consultation_booked", "label": "Consultation Booked", "color": "#8B5CF6"},
    {"id": "photos_received", "label": "Photos Received", "color": "#06B6D4"},
    {"id": "doctor_reviewed", "label": "Doctor Reviewed", "color": "#F97316"},
    {"id": "contract_signed", "label": "Contract Signed", "color": "#22C55E"},
    {"id": "procedure_done", "label": "Procedure Done", "color": "#15803D"},
    {"id": "lost", "label": "Lost", "color": "#EF4444"}
  ]
}
```

If no custom stages configured → use default Sellrise stages (backward compatible).

**4.4 — Pipeline Views**

**Inbox View:**
- Shows only leads with stage = `new`
- Sorted by created_at desc (newest first)
- Columns: name, email, procedure, score (colored badge), created_at
- Click → opens lead detail page
- Count badge: "12 new leads"

**Kanban View:**
- One column per pipeline stage
- Cards show: name, procedure (first one), score badge, date
- Lead count in column header
- Drag & drop between columns:
  1. Update stage in Sellrise DB
  2. PATCH /patients/:id/stage on Phlastic API
  3. Both systems create stage_changed event
- "Lost" column on far right, visually distinct (gray background)

**List View (Table):**
- All leads in table format
- Columns: name, email, phone, procedure, score, email_status, stage, owner, created_at
- Every column sortable (click header to toggle asc/desc)
- Filters panel:
  - Stage (multi-select dropdown)
  - Score range (slider: 0-100)
  - Date range (date pickers: from/to)
  - Owner (dropdown)
  - Email status (multi-select: valid/invalid/unknown)
  - Search (text: searches name, email, phone)
- Pagination: 50 per page, with page numbers

**4.5 — Bulk Actions**

Select multiple leads (checkboxes) → action bar appears:
- **Change Stage:** dropdown to select new stage → applies to all selected
- **Assign Owner:** dropdown to select owner → applies to all selected
- **Export CSV:** downloads CSV with columns: created_at, name, email, phone, procedure_interest, budget_range, timeframe, qualification_score, email_status, stage, owner

For stage changes and owner changes: update BOTH Sellrise and Phlastic (for leads with external_patient_id).

**4.6 — Stage Change Sync Logic**

Any stage change in CRM must update both systems:
```
1. User changes stage in CRM (drag-drop, button, or bulk action)
2. Update lead.stage in Sellrise DB
3. If lead has external_patient_id:
   a. PATCH /patients/:id/stage on Phlastic API
   b. If Phlastic API fails: show error toast, rollback Sellrise stage
4. Both systems create stage_changed event in their respective event tables
```

**4.7 — "Send Invite" Button (NEW)**

On lead detail page, add "Send Cabinet Invite" button:
```
1. Staff clicks "Send Cabinet Invite"
2. CRM calls POST /auth/invite with patient_id
3. Phlastic generates invite_token, saves to patient record
4. Phlastic sends email to patient with invite link: https://cabinet.phlastic.com.au/register?token={invite_token}
5. CRM shows confirmation: "Invite sent to john@example.com"
6. Creates event: invite_sent
```

Button visible when:
- Patient has external_patient_id
- Patient's account_created = false
- Patient's email_status != "invalid"

### Module 4 Acceptance Tests

- [ ] Lead detail page shows all 7 blocks when external_patient_id exists
- [ ] Lead detail page works normally when NO external_patient_id (regular lead, no errors)
- [ ] Contact info shows email validation badge (green/red/gray)
- [ ] Photos display as thumbnails grid, clickable for full-size modal
- [ ] Photo images load from Phlastic API (not from Sellrise storage)
- [ ] Score shows colored badge: cold (red 0-30), warm (yellow 31-70), hot (green 71-100)
- [ ] Timeline shows merged events from both Sellrise and Phlastic, sorted chronologically
- [ ] Notes can be added (saves to Phlastic API, appears in timeline)
- [ ] Tags can be added and removed (saves to Phlastic API)
- [ ] Stage can be changed via buttons on detail page
- [ ] Owner can be assigned from dropdown on detail page
- [ ] Custom pipeline stages display correctly for Phlastic workspace
- [ ] Default Sellrise stages still work for workspaces without custom config
- [ ] Inbox view shows only stage=new leads, sorted newest first
- [ ] Kanban view shows correct columns matching pipeline stages config
- [ ] Kanban drag & drop changes stage in BOTH Sellrise and Phlastic
- [ ] List view shows all columns, all sortable
- [ ] List view filters work: stage, score range, date range, owner, email_status, search
- [ ] List view pagination works (50 per page)
- [ ] Bulk stage change: works for selected leads, updates both systems
- [ ] Bulk owner assignment: works for selected leads
- [ ] Bulk CSV export: downloads file with correct data
- [ ] Workspace isolation: Phlastic patient data not visible in other workspaces
- [ ] API failure: if Phlastic API is down, CRM shows Sellrise data + error message (doesn't crash)
- [ ] "Send Cabinet Invite" button visible when patient has no account
- [ ] "Send Cabinet Invite" sends email with correct link
- [ ] After invite sent, event logged in timeline

**Estimate:** 4-5 days
**Dependencies:** Modules 1, 2, 3

---

## Module 5: Email Validation

**Goal:** Validate patient emails automatically on creation.

**Where:** Phlastic Patient Service (since email is PII stored there)

### What to Build

**Validation trigger:**
```
1. Patient created (POST /patients) → email_status = "unknown"
2. Response sent immediately (201) — validation does NOT block creation
3. Background async job starts:
   a. Format check (regex): valid email format?
   b. MX record check (DNS lookup): does the domain have mail servers?
   c. (Optional) API check via external service if configured
4. After checks complete → update email_status:
   - "valid" — format OK + MX exists
   - "invalid" — bad format OR no MX record
   - "unknown" — couldn't check (timeout, DNS failure)
```

**Also triggers on email update:** If patient email is changed via PUT /patients/:id, re-run validation.

**External validation service (optional, Phase 2):**
- Support configurable API: ZeroBounce, NeverBounce, or AbstractAPI
- Only if free tier / API key is configured
- Config:
```json
{
  "email_validation": {
    "external_service": "zerobounce",
    "api_key": "zb_xxx",
    "enabled": false
  }
}
```
- For now: implement format + MX only. External service is a bonus.

### Module 5 Acceptance Tests

- [ ] `valid@gmail.com` → email_status = `valid`
- [ ] `test@nonexistent-domain-xyz123abc.com` → email_status = `invalid`
- [ ] `not-an-email` → email_status = `invalid`
- [ ] `user@` → email_status = `invalid`
- [ ] Validation runs async — patient creation returns 201 before validation completes
- [ ] email_status readable via GET /patients/:id
- [ ] email_status filterable via GET /patients?email_status=valid
- [ ] Updating patient email triggers re-validation
- [ ] DNS timeout handled gracefully (returns "unknown", not error)

**Estimate:** 1 day
**Dependencies:** Module 1

---

## Module 6: Contract Automation

**Goal:** Generate patient contracts as PDF from CRM with one click.

### BLOCKED — Waiting from Daria/Avi:
1. Contract template (Word or PDF) with placeholder fields
2. Which data fields to insert
3. Signature format: e-sign (DocuSign/similar) or PDF download + wet signature
4. At which pipeline stage is the contract generated
5. Who initiates (doctor? admin? automatic?)

### What to Build (after receiving template)

**Phlastic Patient Service — new endpoints + table:**

```
POST   /api/v1/patients/:id/contracts       Generate contract PDF
GET    /api/v1/patients/:id/contracts/:cid   Download contract PDF
PATCH  /api/v1/patients/:id/contracts/:cid   Update status (sent/signed)
```

**New table: `contracts`**

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | PK |
| patient_id | UUID | FK → patients |
| template_version | string | Which template version was used |
| generated_pdf_path | string | File path to generated PDF |
| variables_used | jsonb | Snapshot of all variables inserted |
| status | enum | `generated` → `sent` → `viewed` → `signed` |
| sent_at | datetime | When emailed to patient |
| signed_at | datetime | When marked as signed |
| created_by | string | Who generated it |
| created_at | datetime | |

**Template variables (preliminary — confirm with Daria):**
```
{{patient_name}}        — from patient record
{{patient_email}}       — from patient record
{{patient_phone}}       — from patient record
{{procedure_name}}      — from procedure_interest
{{procedure_price}}     — TBD (needs pricing data)
{{procedure_date}}      — TBD (needs scheduling)
{{clinic_name}}         — "Phlastic"
{{clinic_address}}      — from config
{{doctor_name}}         — from config or assigned doctor
{{today_date}}          — generation date
```

**Sellrise CRM changes:**
- "Generate Contract" button on lead detail page (visible only when stage >= doctor_reviewed)
- Button calls Phlastic API → generates PDF → shows download link
- Contract status shown in timeline (generated → sent → signed)
- "Send to Patient" button → emails PDF to patient email
- "Mark as Signed" button → updates contract status + moves patient stage to `contract_signed`

### Module 6 Acceptance Tests

- [ ] "Generate Contract" button visible on patient card (when stage >= doctor_reviewed)
- [ ] Clicking generates PDF with patient data correctly inserted
- [ ] PDF downloadable from CRM
- [ ] "Send to Patient" sends PDF to patient email
- [ ] Event `contract_generated` created in patient_events
- [ ] Event `contract_sent` created when emailed
- [ ] "Mark as Signed" → contract status=signed + patient stage=contract_signed + event
- [ ] Contract PDF stored in Phlastic service (NOT in Sellrise)
- [ ] Multiple contracts possible per patient (e.g., revised contract)

**Estimate:** 2-3 days (after receiving template from Daria)
**Dependencies:** Modules 1, 4 + template from Daria

---

## Module 7: Patient Cabinet — Auth (NEW)

**Goal:** Patient-facing authentication system. Patients receive invite from staff, create account, log in to their personal cabinet.

**Where:** Phlastic Patient Service (new web app served from same VPS)

### What to Build

**Tech stack:**
- Frontend: React / Next.js / any SPA framework (your choice)
- Served from: `https://cabinet.phlastic.com.au` (or subdomain of main Phlastic service)
- API calls: same Phlastic REST API (new `/auth/*` and `/cabinet/*` endpoints)
- Auth: JWT tokens (access + refresh)

**7.1 — Invite Flow**

```
1. Staff clicks "Send Cabinet Invite" in Sellrise CRM (Module 4.7)
2. CRM calls POST /auth/invite { patient_id }
3. Phlastic service:
   a. Generates unique invite_token (UUID, single-use, expires in 7 days)
   b. Saves invite_token + invite_sent_at on patient record
   c. Sends email to patient:
      Subject: "Welcome to Phlastic — Set Up Your Account"
      Body: Link to https://cabinet.phlastic.com.au/register?token={invite_token}
   d. Creates event: invite_sent
4. Returns 200 { "invite_sent": true, "expires_at": "..." }
```

**7.2 — Create Account Page**

**URL:** `/register?token={invite_token}`

**Layout:** Split-screen (matches Figma)
- Left side: Full-bleed photo (Phlastic branding image)
- Right side: Registration form (562px wide)

**Content:**
```
Welcome to the Aesthetic

We are glad to welcome you to our platform.
Please create your password to access your personal cabinet.

[Password]          — min 8 chars, show/hide toggle
[Confirm Password]  — must match

[Create Account]    — button, disabled until valid

Already have an account? [Sign In]
```

**Logic:**
```
1. Page loads → validate invite_token via GET /auth/validate-token?token=xxx
2. If invalid/expired → show error: "This invite link has expired. Please contact Phlastic to get a new one."
3. If valid → show form, pre-fill patient name from API response
4. On submit:
   a. POST /auth/register { invite_token, password }
   b. Backend: hash password, save to patient.password_hash, set account_created=true
   c. Backend: invalidate invite_token (single-use)
   d. Backend: create event: account_created
   e. Backend: return JWT tokens (access + refresh)
   f. Frontend: auto-login, redirect to onboarding wizard
5. Password validation:
   - Minimum 8 characters
   - At least 1 letter + 1 number
   - Client-side validation + server-side validation
```

**7.3 — Sign In Page**

**URL:** `/login`

**Layout:** Split-screen (matches Figma)
- Left side: Photo
- Right side: Login form

**Content:**
```
Welcome back

Please enter your details

[Email]
[Password]          — show/hide toggle

[Sign In]

[Forgot password?]
```

**Logic:**
```
1. POST /auth/login { email, password }
2. Success → JWT tokens returned, redirect to dashboard
3. If onboarding_completed=false → redirect to onboarding wizard instead of dashboard
4. Wrong credentials → "Invalid email or password" (don't reveal which one is wrong)
5. Account not created (no password) → "Account not found. Check your email for an invite link."
6. Rate limiting: max 5 attempts per email per 15 minutes
```

**7.4 — Forgot Password**

**URL:** `/forgot-password`

```
1. User enters email
2. POST /auth/forgot-password { email }
3. If email exists + account_created=true → send password reset email with token (1 hour expiry)
4. Always show "If this email exists, we sent a reset link" (don't reveal if email exists)
5. Reset link: https://cabinet.phlastic.com.au/reset-password?token={reset_token}
6. Reset page: new password + confirm → POST /auth/reset-password { token, password }
```

**7.5 — JWT Token Management**

```
- Access token: 15 minute expiry
- Refresh token: 30 day expiry, stored in httpOnly cookie
- On 401 response → try refresh → if refresh fails → redirect to login
- POST /auth/logout → invalidate refresh token
```

### Module 7 Acceptance Tests

- [ ] Invite email sent with correct link when staff clicks "Send Cabinet Invite"
- [ ] Register page loads with valid token, shows patient name
- [ ] Register page shows error with invalid/expired token
- [ ] Password validation: min 8 chars, 1 letter + 1 number
- [ ] Password mismatch shows error
- [ ] Successful registration → auto-login → redirect to onboarding
- [ ] Invite token is single-use (second use shows error)
- [ ] Sign in with correct credentials → redirect to dashboard (or onboarding if not completed)
- [ ] Sign in with wrong credentials → generic error message
- [ ] Rate limiting: 6th attempt within 15 min → blocked
- [ ] Forgot password sends reset email (doesn't reveal if email exists)
- [ ] Password reset works with valid token
- [ ] Password reset token expires after 1 hour
- [ ] JWT access token expires after 15 min, refresh works
- [ ] Logout invalidates refresh token
- [ ] All auth pages responsive (mobile + desktop)
- [ ] Split-screen layout matches Figma design

**Estimate:** 3-4 days
**Dependencies:** Module 1 (patient DB with new auth fields)

---

## Module 8: Patient Cabinet — Onboarding Wizard (NEW)

**Goal:** Step-by-step data collection wizard shown on first login. Collects patient info that wasn't captured in chatbot.

**Where:** Phlastic Cabinet frontend + API

### What to Build

**Trigger:** After registration or first login when `onboarding_completed = false`

**Layout:** Modal overlay (1144px wide on desktop, full-screen on mobile)
- Left sidebar: Step numbers with progress indicator (completed = green check, current = blue, future = gray)
- Right content: Step-specific form
- Top right: X button to close (saves progress, can resume later)

**Steps:**

**Step 1 — Welcome**
```
Welcome to Phlastic!

We'd love to learn more about you to provide the best possible experience.
This will only take a few minutes.

[Let's Start]
```
No data collected. Just introduction.

**Step 2 — Personal Details**
```
Nice to meet you, {name}!

What is your full name?
[First Name]     [Last Name]

Date of Birth
[DD / MM / YYYY]

Gender
[Select ▾]  — Male / Female / Other / Prefer not to say

Phone
[+61 ...]
```

**Save to:** `patients.name`, `patients.date_of_birth`, `patients.gender`, `patients.phone`

**Step 3 — Location & Contact**
```
Where are you located?

Country
[Select ▾]    — Searchable dropdown

City
[Text input]

Preferred contact method
[Email / Phone / WhatsApp]    — Radio buttons
```

**Save to:** `patients.location` (composed from country + city)

**Step 4 — Medical Interest**
```
What procedures are you interested in?

[Checkbox grid:]
☐ Rhinoplasty          ☐ Facelift
☐ Breast Augmentation  ☐ Liposuction
☐ Tummy Tuck           ☐ Eyelid Surgery
☐ Hair Transplant       ☐ Botox / Fillers
☐ Other: [text input]

What is your approximate budget?
[Select ▾]  — Under $5k / $5-10k / $10-20k / $20-50k / $50k+

When would you like to have the procedure?
[Select ▾]  — ASAP / 1-3 months / 3-6 months / 6-12 months / Just exploring

Any previous surgeries or medical conditions we should know about?
[Textarea — optional]
```

**Save to:** `patients.procedure_interest`, `patients.budget_range`, `patients.timeframe`, `patients.medical_notes`

**Step 5 — Photo Upload**
```
Upload photos for your consultation

Our doctor will review these before your appointment.
Photos help us give you the most accurate assessment.

[Drag & drop or click to upload]

Recommended photos:
• Front face        → [Upload slot]
• Side profile      → [Upload slot]
• 45° angle         → [Upload slot]
• Area of interest  → [Upload slot]

☐ I consent to sharing these photos with the Phlastic medical team
  for consultation purposes only.

[Upload]    [Skip for now]
```

**Save to:** POST /patients/:id/photos for each file

**Step 6 — Preferences**
```
Almost done! A few more preferences:

How did you hear about Phlastic?
[Select ▾]  — Google / Social Media / Friend/Family / Doctor referral / Other

Would you like to receive updates about new services and offers?
[Yes / No]   — Creates marketing consent record

Preferred appointment times
☐ Morning (8am-12pm)
☐ Afternoon (12pm-5pm)
☐ Evening (5pm-8pm)
☐ Weekends
```

**Save to:** Additional fields on patient record (or qualification_data JSON)

**Step 7 — Confirmation**
```
Thank you, {name}! 🎉

Your profile is set up. Here's what happens next:

1. Our team will review your information
2. A specialist will contact you to schedule a consultation
3. You can track everything in your dashboard

[Go to Dashboard]
```

**Save:** Set `onboarding_completed = true`, `onboarding_step = 7`. Create event: `onboarding_completed`. Award 50 reward points.

### Progress Saving

- Each step saves data immediately on "Next" (not just at the end)
- POST /cabinet/onboarding/{step_number} with step data
- If user closes wizard mid-way → `onboarding_step` records progress
- Next login → wizard resumes from last incomplete step
- Data from chatbot pre-fills overlapping fields (procedure_interest, budget, etc.)

### Module 8 Acceptance Tests

- [ ] Wizard shows automatically after first login
- [ ] Wizard shows on subsequent logins if onboarding_completed=false
- [ ] Step progress indicator shows correct state (completed/current/future)
- [ ] Each step saves data to patient record via API
- [ ] Closing wizard mid-way saves progress (resume on next login)
- [ ] Pre-fill from chatbot data works (no duplicate entry for existing fields)
- [ ] Photo upload in step 5 works (same flow as chatbot upload, but uploaded_via=cabinet)
- [ ] Photo consent required before upload
- [ ] "Skip for now" works on photo step
- [ ] Step 7 sets onboarding_completed=true
- [ ] 50 reward points awarded on completion
- [ ] After completion, redirect to dashboard (wizard doesn't show again)
- [ ] Mobile responsive (full-screen wizard on mobile)
- [ ] Sidebar step navigation matches Figma (4 numbered steps with progress)
- [ ] Marketing consent creates proper consent_record

**Estimate:** 3-4 days
**Dependencies:** Modules 1, 7

---

## Module 9: Patient Cabinet — Dashboard (NEW)

**Goal:** Main patient view after login. Shows overview of patient's status, services, and quick actions.

**Where:** Phlastic Cabinet frontend

### What to Build

**Layout (matches Figma):**
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
│ Rewards  │  │                                              │  │
│ Reviews  │  │  [Content Section 1]                         │  │
│ Community│  │  Upcoming appointments / Current stage       │  │
│          │  │                                              │  │
│          │  ├─────────────────────────────────────────────┤  │
│          │  │                                              │  │
│          │  │  [Content Section 2]                         │  │
│          │  │  Recent activity / Timeline                  │  │
│          │  │                                              │  │
│          │  ├─────────────────────────────────────────────┤  │
│          │  │                                              │  │
│          │  │  [Content Section 3]                         │  │
│          │  │  Quick actions / Helpful links               │  │
│          │  │                                              │  │
│          │  └─────────────────────────────────────────────┘  │
└──────────┴───────────────────────────────────────────────────┘
```

**9.1 — Header**
- Logo (left)
- Patient avatar + name (right) — click opens profile dropdown (Profile, Settings, Logout)
- Chatbot icon (right) — opens Sellrise chat widget overlay

**9.2 — Sidebar (6 menu items)**

| # | Item | Route | Icon |
|---|------|-------|------|
| 1 | Dashboard | `/dashboard` | Home/Grid |
| 2 | Personal Information | `/profile` | User |
| 3 | My Services | `/services` | Medical/Clipboard |
| 4 | Journey | `/journey` | Timeline/Map |
| 5 | Phlastic Rewards | `/rewards` | Star/Gift |
| 6 | My Reviews | `/reviews` | Star/Message |
| 7 | Community | `/community` | Users/Group |

Active item highlighted with accent color. Sidebar collapses to icons on mobile (hamburger menu).

**9.3 — Dashboard Content (3 sections, vertical, gap 32px)**

**Section 1: Status Overview**
- Welcome message: "Welcome back, {name}!"
- Current pipeline stage (visual pipeline bar showing all stages, current one highlighted)
- Points balance: "{X} points"
- Next appointment date (if scheduled) or "No upcoming appointments"

**Section 2: Recent Activity**
- Last 5 events from patient_events (timeline format)
- Each event: icon + description + relative time ("2 hours ago")
- "View all" link → Journey page

**Section 3: Quick Actions**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 📷 Upload    │  │ 💬 Chat with │  │ ⭐ Leave a   │
│    Photos    │  │    Us        │  │    Review    │
└──────────────┘  └──────────────┘  └──────────────┘
```
- Upload Photos → opens photo upload dialog
- Chat with Us → opens Sellrise chatbot widget
- Leave a Review → navigates to Reviews page

**9.4 — Mobile Responsive**
- Sidebar becomes bottom tab bar or hamburger menu
- Content stacks vertically
- Touch-friendly tap targets (min 44px)

### Module 9 Acceptance Tests

- [ ] Dashboard loads after login (or after onboarding completion)
- [ ] Welcome message shows patient name
- [ ] Current stage displayed correctly in pipeline bar
- [ ] Points balance shows correct number
- [ ] Recent activity shows last 5 events with correct formatting
- [ ] Quick action buttons work (photos, chat, review)
- [ ] Sidebar navigation works — all 7 routes accessible
- [ ] Active sidebar item highlighted
- [ ] Header shows avatar + name
- [ ] Profile dropdown works (Profile, Settings, Logout)
- [ ] Chatbot icon opens Sellrise widget overlay
- [ ] Mobile responsive: sidebar collapses, content stacks
- [ ] Layout matches Figma (colors, spacing, structure)

**Estimate:** 3-4 days
**Dependencies:** Modules 7, 8

---

## Module 10: Patient Cabinet — Services & Journey (NEW)

**Goal:** Patient can view their procedures/services and track their journey timeline.

**Where:** Phlastic Cabinet frontend

### What to Build

**10.1 — My Services Page (`/services`)**

List of patient's services/procedures from `services` table.

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

**Service Detail Page (`/services/:id`):**
- Full service info: name, description, status, dates, doctor, price
- Doctor card (mini profile with photo + name + specialization)
- Related photos (if any)
- Notes from doctor
- Related journey entries

**Empty state:** "No services yet. Your team will add services after your consultation."

**10.2 — Journey Page (`/journey`)**

Visual timeline of patient's journey from `journey_entries` table.

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

- Vertical timeline with dots (green = completed, blue = upcoming, gray = planned)
- Each entry: date, title, description, status badge
- Staff name if applicable
- Chronological order (ascending — oldest first)

**Empty state:** "Your journey hasn't started yet. It will appear here after your first consultation."

### Module 10 Acceptance Tests

- [ ] Services page lists all patient's services
- [ ] Service detail shows full info + doctor card
- [ ] Empty state shown when no services
- [ ] Journey page shows timeline in chronological order
- [ ] Timeline entries color-coded by status (completed/upcoming/planned)
- [ ] Journey entries linked to staff and services where applicable
- [ ] Both pages mobile responsive
- [ ] Data loads from API correctly (GET /cabinet/services, GET /cabinet/journey)

**Estimate:** 2-3 days
**Dependencies:** Modules 7, 9

---

## Module 11: Patient Cabinet — Staff Profiles, Reviews & Rewards (NEW)

**Goal:** Patients can view staff profiles, leave reviews, and track their reward points.

**Where:** Phlastic Cabinet frontend

### What to Build

**11.1 — Staff Profiles**

**Staff list page (`/staff`)** — not in sidebar, accessible from service detail or dashboard links.

**Staff detail page (`/staff/:id`):**
```
┌──────────────────────────────────────┐
│  [Photo]                              │
│  Dr. Sarah Smith                      │
│  Plastic Surgeon                      │
│  ★★★★★ (4.8) — 24 reviews           │
│                                       │
│  Bio:                                 │
│  Dr. Smith specializes in facial      │
│  procedures with 15 years of...       │
│                                       │
│  Reviews:                             │
│  ┌────────────────────────────────┐   │
│  │ John D. ★★★★★                 │   │
│  │ "Amazing results, very..."     │   │
│  │ March 2026                     │   │
│  └────────────────────────────────┘   │
└──────────────────────────────────────┘
```

**11.2 — My Reviews (`/reviews`)**

List of patient's own reviews + ability to write new ones.

**Write Review Form:**
```
Leave a Review

Who is this review for?
[Select staff member ▾]  — optional

Which service?
[Select service ▾]  — optional

Rating
★★★★★  (clickable stars)

Title (optional)
[Text input]

Your review
[Textarea]

☐ Make this review public (visible to other patients)

[Submit Review]
```

**Logic:**
- POST /cabinet/reviews
- New reviews get status=`pending` (staff approves before public display)
- Patient can edit own review while status=pending
- Reviews show in patient's list with status badge

**11.3 — Phlastic Rewards (`/rewards`)**

**Layout:**
```
Phlastic Rewards

┌─────────────────────────────────┐
│     🏆 Your Balance             │
│         150 points              │
└─────────────────────────────────┘

How to earn points:
• Complete onboarding — 50 points
• Leave a review — 25 points
• Refer a friend — 100 points
• Complete a procedure — 200 points

History:
┌─────────────────────────────────────────┐
│ +50   Completed onboarding    March 30  │
│ +25   Left a review           March 31  │
│ +100  Referred Sarah Johnson  April 2   │
│ -75   Redeemed: Free consult  April 5   │
└─────────────────────────────────────────┘
```

**Logic:**
- GET /cabinet/rewards → current balance
- GET /cabinet/rewards/history → paginated list of transactions
- Points awarded automatically by backend on events (onboarding, review, referral, procedure)
- Redemption logic: TBD (implement point tracking now, redemption later)

### Module 11 Acceptance Tests

- [ ] Staff profile page shows photo, name, specialization, bio, rating, reviews
- [ ] Staff profiles accessible from service detail and dashboard
- [ ] Review form allows selecting staff + service + rating + text
- [ ] New reviews saved with status=pending
- [ ] Patient can see own reviews with status badges
- [ ] Patient can edit pending reviews
- [ ] Rewards page shows correct point balance
- [ ] Rewards history shows all transactions (earned + spent)
- [ ] Points awarded automatically on: onboarding completion, review submission
- [ ] All pages mobile responsive

**Estimate:** 3-4 days
**Dependencies:** Modules 7, 9, 10

---

## Module 12: Patient Cabinet — Community (NEW)

**Goal:** Social feature where patients can join groups, share experiences, and interact.

**Where:** Phlastic Cabinet frontend

### What to Build

**12.1 — Community Groups (`/community`)**

**Groups list:**
```
Community

Join groups to connect with other patients

┌───────────────────────┐  ┌───────────────────────┐
│ [Cover image]          │  │ [Cover image]          │
│ Rhinoplasty Support   │  │ Recovery Tips          │
│ 45 members            │  │ 128 members            │
│ [Join]                │  │ [Joined ✓]             │
└───────────────────────┘  └───────────────────────┘
```

**Create Group (2-step modal):**
- Step 1: Group name + description + cover image + public/private toggle
- Step 2: Invite members (optional) → Create

**12.2 — Group Detail + Posts (`/community/groups/:id`)**

```
┌─────────────────────────────────────────┐
│ [Cover image]                            │
│ Rhinoplasty Support — 45 members         │
│ [Leave Group]                            │
├─────────────────────────────────────────┤
│ [Write something...]            [Post]  │
├─────────────────────────────────────────┤
│ Sarah J. — 2 hours ago                  │
│ "Just had my consultation and I'm so    │
│  excited! Dr. Smith was amazing..."      │
│ ❤️ 12    💬 3                            │
├─────────────────────────────────────────┤
│ Mike T. — Yesterday                      │
│ "Day 5 post-op. Swelling is going down" │
│ [Photo]                                  │
│ ❤️ 24    💬 8                            │
└─────────────────────────────────────────┘
```

**Features:**
- Create post (text + optional image)
- Like posts
- Report posts (sends to admin for review)
- Chronological feed (newest first)
- Paginated (load more on scroll)

**12.3 — Public User Profiles (`/community/users/:id`)**

Visible to other community members:
```
Sarah J.
Member since March 2026
Groups: Rhinoplasty Support, Recovery Tips
Posts: 12
```

- Shows only: first name + last initial, join date, groups, post count
- NO medical info, NO contact info, NO photos (privacy)
- Patient can control what's visible in their settings

**12.4 — Moderation**

- Reported posts go to a moderation queue (visible in Sellrise CRM / admin)
- Admin can: approve (dismiss report), hide post, ban user from group
- This is admin-side functionality — basic implementation is fine

### Module 12 Acceptance Tests

- [ ] Groups list shows available groups with member count
- [ ] Patient can join/leave groups
- [ ] Create group flow works (2 steps)
- [ ] Group detail shows posts feed
- [ ] Patient can create text + image posts
- [ ] Like button works (toggle on/off)
- [ ] Report button works (sends to moderation)
- [ ] Public user profiles show limited info only (no PII, no medical)
- [ ] Posts paginated (infinite scroll or "Load more")
- [ ] Community pages mobile responsive
- [ ] Post author shown as "First name + Last initial" (privacy)

**Estimate:** 4-5 days
**Dependencies:** Modules 7, 9

---

## VPS Setup (Module 1 prerequisite)

Ivan provides VPS access. Iga sets up the Phlastic Patient Service on it.

| Item | Requirement |
|------|-------------|
| Cloud provider | DigitalOcean / AWS / Vultr (must have AU region) |
| Region | **Sydney (AU)** preferred. Singapore acceptable if no AU option. |
| Minimum specs | 2 vCPU, 4GB RAM, 50GB SSD |
| OS | Ubuntu 22.04 LTS |
| Domain | `patients.phlastic.com.au` (API) + `cabinet.phlastic.com.au` (Patient Cabinet) |
| SSL | Let's Encrypt (auto-renew via certbot) |
| Backup | Daily automated DB backup, 30-day retention |
| Firewall | Port 443 (HTTPS): open for API (Sellrise IP) + Cabinet (public for patients) |
| SSH | Key-based auth only. No password login. |
| Email service | For invites, password resets, notifications. Resend / SendGrid / AWS SES. |

**Ivan's action:** Create VPS + give Iga SSH access. OR give Iga DigitalOcean API token to create it himself.

---

## Execution Plan

```
WEEK 1 — Backend Foundation
──────
Day 1-2:   VPS setup + Module 1 (Patient Service: DB + API + all endpoints + new tables)
Day 2-3:   Module 5 (Email Validation) — can parallelize with Module 1 API work
Day 3-4:   Module 2 (Sellrise → Phlastic webhook pipeline)
Day 4-5:   Module 3 (Photo upload in chatbot widget)

WEEK 2 — CRM + Auth
──────
Day 6-10:  Module 4 (CRM customization — largest backend module)
Day 8-10:  Module 7 (Patient Cabinet Auth — can start in parallel with Module 4)

WEEK 3 — Patient Cabinet
──────
Day 11-12: Module 8 (Onboarding Wizard)
Day 12-14: Module 9 (Dashboard)
Day 14-15: Module 10 (Services & Journey)

WEEK 4 — Social + Contracts
──────
Day 16-18: Module 11 (Staff Profiles, Reviews, Rewards)
Day 18-22: Module 12 (Community)
Day 20-22: Module 6 (Contract automation — IF template received from Daria)

BUFFER
──────
Day 23-25: Integration testing, bug fixes, edge cases, polish
```

**Total estimate: 20-25 working days (4-5 weeks)**

### Start immediately (zero blockers):
- VPS setup
- Module 1: Patient Service (DB + API + new tables for cabinet)
- Module 5: Email Validation

### Needed from Ivan before Week 1:
- VPS access credentials OR DigitalOcean API token
- Domain setup (patients.phlastic.com.au + cabinet.phlastic.com.au)
- Email service credentials (Resend / SendGrid / SES)

### Needed from Daria/Avi (before Module 6):
1. Contract template (Word/PDF with placeholders)
2. Detailed photo flow (what photos, when in journey, labels)
3. Confirm pipeline stages are correct

### Needed from Daria/Avi (before Module 12):
1. What community groups to pre-create
2. Moderation rules / guidelines
3. Branding assets (logo, colors, cover images)

---

## Payment

| Milestone | Amount | Status |
|-----------|--------|--------|
| Upfront (scope 2 start) | $500 | **PAID** |
| Final (all modules accepted) | $500 | Due on acceptance |

## Acceptance Process

Same as MVP contract (scope 1):
1. Iga declares module or full scope delivery
2. Ivan tests within 48 hours using acceptance tests listed above
3. Issues reported → Iga fixes within reasonable time
4. Final acceptance → final $500 payment released

## Communication

| Channel | What |
|---------|------|
| Git (NEW repo) | Phlastic Patient Service code (backend + cabinet frontend) |
| Git (existing repo) | Sellrise CRM changes |
| Telegram (daily) | Progress update: what done today, what planned tomorrow, any blockers |
| Telegram (immediate) | Blockers, questions, decisions needed |

---

## Module Summary

| # | Module | Where | Estimate | Dependencies |
|---|--------|-------|----------|-------------|
| 1 | Patient Service (DB + API) | New VPS | 3-4 days | VPS access |
| 2 | Webhook Pipeline | Sellrise | 2 days | Module 1 |
| 3 | Photo Upload (Chatbot) | Sellrise | 2-3 days | Modules 1, 2 |
| 4 | CRM Customization | Sellrise | 4-5 days | Modules 1, 2, 3 |
| 5 | Email Validation | New VPS | 1 day | Module 1 |
| 6 | Contract Automation | Both | 2-3 days | Modules 1, 4 + Daria |
| 7 | **Cabinet Auth** | New VPS | 3-4 days | Module 1 |
| 8 | **Onboarding Wizard** | New VPS | 3-4 days | Modules 1, 7 |
| 9 | **Dashboard** | New VPS | 3-4 days | Modules 7, 8 |
| 10 | **Services & Journey** | New VPS | 2-3 days | Modules 7, 9 |
| 11 | **Staff, Reviews, Rewards** | New VPS | 3-4 days | Modules 7, 9, 10 |
| 12 | **Community** | New VPS | 4-5 days | Modules 7, 9 |
| | **TOTAL** | | **~33-42 days** | |

---

*Phlastic Scope 2 — Technical Specification v3.0 (Extended — includes Patient Cabinet)*
*Date: 2026-03-30*
*Architecture: Separate VPS for patient data + cabinet, Australian Privacy Act 1988 compliance*
*Modules 1-6: Backend + CRM (original scope 2)*
*Modules 7-12: Patient Cabinet Frontend (extended scope)*
