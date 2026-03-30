# PRD – Chatbot Photo Collection for Pre-Consultation

## Metadata

- Feature ID: `CHAT-PHOTO-01`
- Priority: `High`
- Release Focus: `Phase 2 – Pre-Consultation Enhancement`
- Category: `Chatbot / Widget`
- Owner: `Product + Backend + Frontend`

---

## Objective

Enable the chatbot to proactively request and securely collect photos from users prior to an online consultation. Photos help specialists assess the user's needs more accurately, improve surgeon matching, and allow consultants to prepare a more precise plan before the session.

---

## User Story

**As a visitor** who has expressed clear intent to book a consultation, I want the chatbot to ask me to upload relevant photos so that my assigned consultant can review them in advance and provide better, more personalized recommendations.

**As a CRM operator / consultant**, I want to access photos uploaded by a lead directly within the lead detail view so that I can prepare for the consultation without needing a separate communication channel.

---

## Trigger Conditions

The photo collection prompt **must only appear** when **both** of the following are true:

1. **Basic info has been collected** — the conversation has already captured:
   - Name
   - Age
   - Procedure of interest
   - Budget range
   - Preferred timeline

2. **Clear consultation intent is confirmed** — the user has:
   - Explicitly agreed to proceed with a consultation, **or**
   - Clicked a booking CTA / handoff step

> The photo request must **never** appear during casual browsing or before intent is confirmed. It should feel like a natural, helpful next step — not a data grab.

---

## Conversation Flow

### Step 1 – Photo Request Message

After intent is confirmed, the chatbot displays:

```
To provide you with the most accurate recommendations, our specialists 
may need to review a few photos.

We'll ask for:
  • A front-facing photo
  • A side profile photo
  • A close-up of the area you'd like to address

This helps us match you with the right surgeon and give you a more 
precise plan.

Your photos are completely private and will only be shared with your 
assigned consultant.

Would you be comfortable uploading them now, or would you prefer to 
do this later?
```

**Buttons:**
- `📎 Upload photos`
- `Later – skip for now`

---

### Step 2a – User chooses "Upload photos"

The chatbot opens a file upload input within the widget and displays:

```
Great! Please upload your photos below.

We need 3 photos:
  1. Front-facing photo (face looking straight ahead)
  2. Side profile photo (left or right side)
  3. Close-up of the area you'd like to address

Accepted formats: JPG, PNG, HEIC, JPEG
Maximum: 3 photos, 10 MB each

Tip: Clear, well-lit photos taken in natural light give consultants 
the best view.
```

On successful upload:
```
Thank you! Your photos have been securely saved.

Your consultant will review them before your session.
```

The conversation then continues to the booking / confirmation step.

---

### Step 2b – User chooses "Later – skip for now"

The chatbot respects the choice without friction:

```
No problem! You can always share them later by replying to your 
consultation confirmation email.

Let's continue with booking your session.
```

The conversation continues to the booking / confirmation step. The system logs that the user declined and provides a follow-up reminder link in the confirmation email.

---

### Step 2c – Upload fails or is incomplete

If the upload encounters an error:

```
It looks like something went wrong with the upload. 

Please check your file format (JPG, PNG, HEIC, JPEG) and size (max 10 MB each), 
then try again — or skip for now and share them later.
```

**Buttons:**
- `Try again`
- `Skip for now`

---

## Functional Requirements

### Chatbot / Scenario Engine

1. A new step type `question_upload` (or `file_upload`) must be added to the step processor to support photo collection as a first-class step within a scenario.
2. The step type must support configurable options:
   - `accepted_types`: list of MIME types (e.g., `image/jpeg`, `image/png`, `image/heic`)
   - `max_files`: integer (default: 3)
   - `max_size_mb`: integer per file (default: 10)
   - `required`: boolean (default: `false` — upload is always optional)
   - `slots`: ordered list of named photo slots (e.g., `front`, `side`, `close_up`) used to label each upload in the CRM
3. The photo prompt must only be injected into the flow when the trigger conditions (see above) are satisfied. This can be implemented as a `conditional_branch` step that checks for `intent_confirmed = true` and `basic_info_complete = true` before routing to the upload step.
4. If `required` is `false` and the user skips, the conversation must proceed without blocking.
5. Uploaded file references (secure URL or storage key) must be stored in the conversation context under a reserved variable, e.g., `uploaded_photos`.

### Backend

6. A new endpoint must be created for secure file upload from the widget:
   - `POST /v1/widget/upload`
   - Authenticated via widget session token (existing `session_id` mechanism)
   - Accepts `multipart/form-data`
   - Returns a list of secure file references (opaque URLs or storage keys, never raw paths)
7. Files must be stored securely on the VPS in an access-controlled directory with restricted permissions. The original filename is sanitized and stored in the `filename` column of the `lead_attachments` table; the server maintains an internal `storage_key` reference to the file's actual location on disk. Direct file access via path traversal or URL enumeration must be prevented through authentication checks. (put this in chatbot's slot)
8. Each uploaded file must be linked to the `lead_id` and `session_id` in the database (`lead_attachments` table).
9. File access must require authenticated operator credentials (with JWT token if accessed in sellrise app, or API key in the request via API) — files must **not** be publicly accessible by URL alone.
10. Uploaded files must be virus/malware scanned before being made available to operators (use a scanning service such as ClamAV or a cloud equivalent) (SKIPPED for 30/03/2026).
11. File retention policy must be defined and enforced (e.g., auto-delete after 90 days unless explicitly retained).
12. Event logging must record `photo_upload_initiated`, `photo_upload_completed`, and `photo_upload_skipped`.

### CRM / Lead Detail View

13. Uploaded photos must be visible in the lead detail view under a new **"Attachments"** section.
14. Only authenticated operators with access to the lead's workspace may view or download photos.
- Each attachment entry must display: thumbnail, slot label (Front / Side / Close-up), file name, upload timestamp, and a download button.
16. CRM operators must be able to delete an attachment (with confirmation prompt).

---

## API Contract

### Upload Endpoint

- **Method:** `POST`
- **Path:** `/v1/widget/upload`
- **Auth:** Widget session token in header (`X-Widget-Session: <session_id>`)
- **Content-Type:** `multipart/form-data`

**Request fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | string | Yes | Active widget session ID |
| `lead_id` | string | Yes | Associated lead UUID |
| `files` | file[] | Yes | One or more image files |
| `slots` | string[] | No | Ordered slot labels matching each file: `front`, `side`, `close_up` |

**Response (200 OK):**

```json
{
  "uploaded": [
    {
      "attachment_id": "uuid",
      "filename": "photo_001.jpg",
      "slot": "front",
      "size_bytes": 2048000,
      "uploaded_at": "2026-03-30T10:00:00Z"
    },
    {
      "attachment_id": "uuid",
      "filename": "photo_002.jpg",
      "slot": "side",
      "size_bytes": 1800000,
      "uploaded_at": "2026-03-30T10:00:01Z"
    },
    {
      "attachment_id": "uuid",
      "filename": "photo_003.jpg",
      "slot": "close_up",
      "size_bytes": 2200000,
      "uploaded_at": "2026-03-30T10:00:02Z"
    }
  ],
  "skipped": []
}
```

**Error responses:**

| Code | Reason |
|---|---|
| 400 | Invalid file type or size exceeded |
| 401 | Invalid or expired session token |
| 413 | Total payload too large |
| 422 | Missing required fields |

---

## Data Model

### `lead_attachments` Table

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `lead_id` | UUID | FK → `leads.id` |
| `session_id` | UUID | FK → `widget_sessions.id` |
| `workspace_id` | UUID | FK → `workspaces.id` |
| `filename` | string | Original filename (sanitized) |
| `slot` | enum | `front`, `side`, `close_up`, `other` — photo position label |
| `storage_key` | string | Internal storage reference (not exposed to client) |
| `file_size_bytes` | integer | |
| `mime_type` | string | |
| `scan_status` | enum | `pending`, `clean`, `quarantined` |
| `uploaded_at` | timestamp | |
| `deleted_at` | timestamp | Soft delete |

---

## New Step Type: `question_upload`

To be added to the scenario engine alongside existing step types.

**Schema:**

```json
{
  "type": "question_upload",
  "id": "step_photo_upload",
  "message": "To provide you with the most accurate recommendations, our specialists may need to review a few photos.",
  "accepted_types": ["image/jpeg", "image/png", "image/heic"],
  "slots": [
    { "key": "front",    "label": "Front-facing photo",                    "hint": "Face looking straight ahead" },
    { "key": "side",     "label": "Side profile photo",                    "hint": "Left or right side" },
    { "key": "close_up", "label": "Close-up of the area to address",       "hint": "Clear, well-lit close-up" }
  ],
  "max_files": 3,
  "max_size_mb": 10,
  "required": false,
  "save_to": "uploaded_photos",
  "on_skip": {
    "next_step": "step_booking_confirmation"
  },
  "on_complete": {
    "next_step": "step_booking_confirmation"
  }
}
```

---

## Security & Privacy Requirements

1. **No public URLs** — uploaded file storage keys must never be returned to the widget. Only opaque references are used client-side.
2. **Presigned URLs** — when operators access files in the CRM, pre-signed time-limited URLs must be generated server-side (e.g., 15-minute expiry).
3. **Malware scanning** — all uploads must pass a malware scan before being accessible to operators. Files in `pending` or `quarantined` scan state are not displayed in the CRM.
4. **Session validation** — upload requests must be validated against an active, non-expired session tied to the correct `lead_id`.
5. **File type enforcement** — MIME type must be validated server-side; do not rely solely on client-declared content type or file extension.
6. **GDPR / privacy compliance** — users must be shown a clear privacy notice before uploading. The notice must state who will view the photos and how long they are retained.
7. **Workspace isolation** — attachments must be strictly scoped to the workspace; cross-workspace access must return 404.

---

## UI/UX Specifications

### Widget – Upload Component

- Native file picker triggered by "Upload photos" button
- Three named upload slots presented sequentially or as a labelled multi-upload panel: **Front**, **Side**, **Close-up**
- Each slot shows its label and a short hint (e.g., "Face looking straight ahead")
- Inline upload progress bar per slot
- Thumbnail preview per slot after successful upload
- Individual file removal (×) per slot before final submission
- Slots are independent — user can fill some and skip others
- Clear error state for oversized or wrong-format files
- Skip link always visible — user must never feel trapped

### CRM – Lead Detail: Attachments Tab

- New **"Attachments"** tab in the lead detail view (alongside Notes, Timeline, etc.)
- Three labelled slots displayed: **Front**, **Side**, **Close-up** — with a placeholder state if a slot was not uploaded
- Thumbnail per filled slot; click to open full-size preview (lightbox)
- Slot label shown in the lightbox header (e.g., "Front-facing photo")
- Download button per attachment
- Delete button per attachment (with confirmation modal)
- Badge on tab showing how many slots are filled (e.g., `Attachments (2/3)`)

---

## Scope

### In Scope

- New `question_upload` step type in the scenario engine
- Secure file upload API endpoint (`POST /v1/widget/upload`)
- `lead_attachments` database table and model
- CRM lead detail: Attachments section
- Presigned URL generation for operator file access
- Malware scan integration
- Event logging for upload actions
- Privacy notice copy in the chatbot upload prompt

### Out of Scope

- Video uploads
- Document uploads (PDF, DOCX) — considered separately
- Automated AI analysis of uploaded photos
- User-facing photo deletion from the widget post-upload
- Email attachment delivery to consultants (handled by notification system separately)

---

## Acceptance Criteria

1. The photo prompt is shown **only** after basic info is collected and consultation intent is confirmed.
2. Users are prompted to upload 3 specific photos (front, side, close-up); each slot accepts one file up to 10 MB in JPG, PNG, or HEIC format. Files exceeding size or format limits are rejected with a clear inline error.
3. Skipping the upload does not block the conversation from proceeding.
4. Uploaded files are stored securely and are not accessible via public URL.
5. Files appear in the CRM lead detail under Attachments after a successful upload.
6. Operators can view (via presigned URL), download, and delete attachments.
7. Files that fail the malware scan are quarantined and not shown in the CRM.
8. Event log records `photo_upload_initiated`, `photo_upload_completed`, and `photo_upload_skipped`.
9. Cross-workspace file access returns 404.

---

## Dependencies

- Widget session mechanism (`/v1/widget/session`)
- Lead creation and deduplication (`CRM-5.1`)
- Step processor service (`STEP_TYPES_IMPLEMENTATION`)
- Object storage (S3-compatible)
- Malware scanning service
- CRM lead detail view (PRD 5.4)
- Presigned URL generation utility

---

## Non-Functional Requirements

- Upload P95 response under 2 seconds for a single 5 MB file on a standard connection
- Storage must be encrypted at rest (AES-256) and in transit (TLS 1.2+)
- Scan results must complete within 10 seconds of upload
- Strict workspace isolation on all read/write paths
