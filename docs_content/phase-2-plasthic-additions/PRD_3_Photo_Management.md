# PRD 3: Secure Photo Management

## Overview

| Field | Value |
|-------|-------|
| **Module** | 3 — Photo Upload in Chatbot |
| **System** | System 1: Sellrise (scenario engine + widget) |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 2–3 days |
| **Dependencies** | Module 1 (Phlastic API) + Module 2 (webhook, for `external_patient_id`) |
| **Priority** | P1 — Required for pre-consultation photo collection |

## Goal

Implement a secure, encrypted photo upload and retrieval pipeline for patient medical photos. Photos go **directly from the user's browser to the Phlastic Patient Service** — the Sellrise backend never touches the photo files.

---

## Architecture Context

```
Browser (chatbot widget)
        │
        │  POST /api/v1/patients/{id}/photos
        │  (multipart/form-data)
        │  X-API-Key: pk_xxx
        ↓
Phlastic Patient Service (AU VPS)
        │
        ↓
/data/photos/{patient_id}/{photo_id}.ext
(encrypted disk — NOT web-accessible)
```

Sellrise backend logs only a `photo_uploaded` event with `photo_id` reference — **no photo binary data** ever touches Sellrise servers.

---

## What to Build

### 1. New Scenario Step Type: `photo_upload`

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

This step type is available in the scenario builder just like text/buttons/form steps.

**Constraint:** The `photo_upload` step **MUST come AFTER the lead submission step** in the scenario. The patient must exist in Phlastic DB before photos can be uploaded (we need the `patient_id`). If the scenario builder places `photo_upload` before the lead submission step, display a warning.

---

### 2. Widget UI for Photo Upload Step

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

**UI components:**
- File picker + camera option on mobile devices
- Photo type selector (dropdown per photo): `face_front`, `face_side`, `face_45`, `body`, `other`
- Consent checkbox — **must be checked** before upload button is active
- Upload progress bar per photo
- Success (✅) / failure (❌ + retry button) indicators per photo
- "Skip" option if step is `required: false`

---

### 3. Upload Flow (Browser → Phlastic Directly)

```
1. User selects photo(s) + checks consent checkbox + picks photo type per photo
2. Widget reads patient_service config from workspace settings
   - base_url: e.g. https://patients.phlastic.com.au/api/v1
   - api_key: pk_xxx (used as X-API-Key header)
3. Widget retrieves external_patient_id from the current lead record
   (lead must already be submitted — photo step must come AFTER contact form)
4. For each selected photo:
   a. Widget sends: POST /api/v1/patients/{patient_id}/photos
      - Headers: X-API-Key: pk_xxx
      - Body: multipart/form-data
           file          = <binary>
           type          = face_front (or selected type)
           uploaded_via  = chatbot
           consent_given = true
   b. Show per-photo progress bar during upload
   c. On success (201) → show green checkmark ✅
   d. On failure → show retry button ❌
5. After all photos uploaded (or user clicks Skip) → proceed to next chatbot step
6. Sellrise logs event: photo_uploaded { photo_id: "uuid" } — NO photo binary
```

---

### 4. File Validation Rules

| Rule | Limit |
|------|-------|
| Max files per step | 5 |
| Max size per file | 10 MB |
| Accepted MIME types | `image/jpeg`, `image/png`, `image/heic` |

Validation happens **client-side** (immediate feedback) AND **server-side** (Phlastic API returns 413 / 415 errors if violated).

---

### 5. Fallback: Patient Not Yet Created

If for some reason the lead hasn't been submitted yet when the user reaches the photo step:

1. Queue photos in browser memory (as Blob objects)
2. After lead submission + webhook completes + `patient_id` received → upload queued photos
3. This is a fallback edge case, not the primary flow

---

### 6. Photo Storage Rules (Phlastic Service)

- Files stored at: `/data/photos/{patient_id}/{photo_id}.{ext}`
- **Directory is NOT web-accessible** — served only through authenticated API
- File names are UUIDs — no PII in file paths
- No original filename in the path (stored in `photos.original_filename` DB field only)

---

### 7. Access Control

| Access Type | Method |
|-------------|--------|
| Upload | `X-API-Key` in header (from workspace `patient_service.api_key`) |
| Download | `X-API-Key` OR patient JWT — **no public URLs ever** |
| Public access | ❌ — never accessible without authentication |

---

## Photo Types Reference

| Type | Description |
|------|-------------|
| `face_front` | Front-facing face photo |
| `face_side` | Side profile |
| `face_45` | 45-degree angle |
| `body` | Full body or area of interest |
| `other` | Any other medical photo |
| `avatar` | Profile avatar (cabinet only) |

---

## Acceptance Tests

- [ ] `photo_upload` step type available in scenario builder
- [ ] Widget shows file picker + consent checkbox when reaching this step
- [ ] Consent checkbox must be checked before upload button is active (button disabled until checked)
- [ ] Upload goes **directly to Phlastic API** (verify in browser Network tab: request goes to Phlastic domain, NOT Sellrise domain)
- [ ] Upload request includes `X-API-Key` header
- [ ] Max 5 files enforced (widget prevents adding more)
- [ ] Max 10 MB per file enforced (widget shows error for larger files)
- [ ] Only JPEG, PNG, HEIC accepted (widget shows error for other file types)
- [ ] Mobile browser shows camera option in native file picker
- [ ] Upload progress bar shown per photo during upload
- [ ] Successful upload shows confirmation (✅) in chat
- [ ] Failed upload shows retry button (❌)
- [ ] "Skip" button works if step is `required: false`
- [ ] Consent record created in Phlastic `consent_records` table (type: `photo_sharing`)
- [ ] Sellrise logs `photo_uploaded` event (with `photo_id`, without photo binary data)
- [ ] Scenario builder warns if `photo_upload` step is placed before the lead submission step
- [ ] Photos stored in non-web-accessible directory on Phlastic VPS
- [ ] `GET /patients/:id/photos/:pid` without API key → 401 (no public access)
