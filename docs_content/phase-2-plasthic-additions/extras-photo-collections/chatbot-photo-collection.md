# PRD - Chatbot Photo Collection Routing (Sellrise Proxy -> Plasthic)

## Metadata

- Feature ID: `CHAT-PHOTO-ROUTING-02`
- Priority: `High`
- Release Focus: `Phase 2 - Plasthic Additions`
- Category: `Integration / Widget / Backend`
- Owner: `Product + Sellrise BE + Plasthic BE + FE`

---

## Problem Statement

Today chatbot image files can land in Sellrise storage, while downstream Plasthic workflows (email variant, clinical review) need Plasthic to have those photos as source-of-truth.

This creates two risks:

1. Split-brain state between Sellrise file state and Plasthic medical-photo state.
2. Pressure to expose arbitrary external upload endpoints in widget config, which increases security risk.

---

## Product Goal

Enable one configurable upload routing mode in Sellrise so photo uploads can be proxied server-to-server to Plasthic with minimum setup, while preserving a local fallback mode.

### Success Criteria

- Workspace can choose upload mode without code deployment.
- In Plasthic mode, photo binary is persisted in Plasthic `photos` table with `uploaded_via = chatbot`.
- No patient service token is exposed to browser clients.
- Upload failures do not break the chat session; user gets retry or skip path.

---

## Scope

### In Scope

1. New routing behavior in Sellrise backend for chatbot photo uploads.
2. Workspace-level configuration for photo upload destination mode.
3. Reuse Plasthic existing upload API endpoint for patient photos.
4. Event logging and observability for routed uploads.

### Out of Scope

1. Arbitrary endpoint templating from frontend widget settings.
2. Direct browser upload to unknown third-party endpoints.
3. New Plasthic database tables.
4. Full admin media gallery redesign.

---

## Architecture Decision

### Selected Model (Safe Minimum Setup)

- Browser uploads only to Sellrise widget backend.
- Sellrise backend decides target based on workspace configuration.
- If route is `patient_service_proxy`, Sellrise forwards multipart upload to Plasthic API.
- If route is `sellrise_local`, Sellrise stores as current behavior.

### Rejected Model

- Browser reads configurable endpoint/payload and uploads directly.

Reason for rejection:

- Hard to secure secrets.
- Hard to validate destination trust boundaries.
- Higher risk of data exfiltration and broken contracts.

---

## High-Level Flow

1. User reaches photo step in chatbot.
2. Widget sends file(s) to Sellrise photo upload endpoint with `workspace_id`, `lead_id`, and metadata.
3. Sellrise resolves upload mode from workspace settings.
4. Sellrise resolves `patient_id` via `lead.external_identities`.
5. If mode is `patient_service_proxy`, Sellrise forwards each file to Plasthic photo API.
6. Sellrise records lead event `photo_uploaded` with route details and result.
7. Widget receives unified response and continues flow.

---

## Configuration Design

Use workspace settings object (backend-controlled):

```json
{
  "photo_upload": {
    "mode": "sellrise_local",
    "timeout_ms": 15000,
    "retry_count": 1
  },
  "patient_service": {
    "enabled": true,
    "base_url": "https://plasthic.example.com/api/v1",
    "auth_token": "server_side_secret"
  }
}
```

### Allowed Modes

- `sellrise_local`: keep existing storage flow.
- `patient_service_proxy`: forward to Plasthic.

### Validation Rules

1. `patient_service_proxy` requires `patient_service.enabled`, `base_url`, and `auth_token`.
2. Unknown mode must fail closed and fallback to `sellrise_local` with warning log.
3. `auth_token` never returned to widget session payload.

---

## API Contract Changes

## 1) Sellrise Endpoint (widget-facing)

Use existing widget upload surface or a dedicated chatbot-photo route. Recommended dedicated route for clearer semantics:

- Method: `POST`
- Path: `/v1/widget/photo-upload`
- Auth: existing public widget constraints + workspace validation
- Content-Type: `multipart/form-data`

Request fields:

- `workspace_id` (UUID, required)
- `lead_id` (UUID, required)
- `file` (one or many files)
- `photo_type` (optional, default `other`)
- `consent_given` (boolean, optional)

Unified response example:

```json
{
  "uploaded": [
    {
      "id": "d8e...",
      "name": "front.jpg",
      "mime_type": "image/jpeg",
      "size": 232143,
      "destination": "patient_service_proxy"
    }
  ],
  "errors": []
}
```

## 2) Sellrise -> Plasthic Proxy Call (server-to-server)

- Method: `POST`
- Path: `/patients/{patient_id}/photos`
- Auth: `Authorization: Bearer <patient_service.auth_token>`
- Form fields:
  - `upload` (file)
  - `photo_type`
  - `uploaded_via = chatbot`
  - `consent_given`

Plasthic endpoint already exists and should be reused.

---

## Functional Requirements

## Sellrise Backend

1. Add upload routing decision layer before file persistence.
2. Resolve `patient_id` from lead external identities.
3. In `patient_service_proxy` mode:
   - forward file with timeout control,
   - capture returned photo identifier,
   - log success/failure event with correlation id.
4. In failure cases:
   - return structured error to widget,
   - allow retry path,
   - do not crash session.
5. Keep compatibility: if config missing, default to `sellrise_local`.

## Sellrise Frontend

6. No arbitrary destination field in browser config.
7. Continue uploading to Sellrise API only.
8. Render clear state messages: uploading, success, failed with retry.

## Plasthic Backend

9. Reuse existing patient photo upload endpoint and validation.
10. Persist records with `uploaded_via = chatbot`.
11. Keep current authorization policy for service/staff JWT.

---

## Non-Functional Requirements

1. Security: No patient service token in browser responses, logs, or local storage.
2. Reliability: At least one retry on transient upstream failure.
3. Performance: P95 proxy upload latency <= 4s per photo under normal load.
4. Auditability: every proxy attempt logged with `workspace_id`, `lead_id`, `patient_id`, `destination`, status.

---

## Risk Register and Mitigations

1. Risk: Missing `patient_id` when webhook mapping not ready.
   Mitigation: fail gracefully, return actionable error, keep chat progression option.
2. Risk: Plasthic unavailable.
   Mitigation: retry once, emit alert, provide skip path to user.
3. Risk: accidental secret leakage via session payload.
   Mitigation: strip `patient_service.auth_token` from any widget response object.
4. Risk: mixed behavior across workspaces.
   Mitigation: explicit mode field with default and admin-visible config state.

---

## Migration and Rollout Plan

1. Add backend config schema support and safe defaults.
2. Implement Sellrise proxy upload path behind mode flag.
3. Validate against staging Plasthic endpoint.
4. Enable `patient_service_proxy` for one pilot workspace.
5. Monitor 48h:
   - upload success rate,
   - upstream timeout rate,
   - photo presence in Plasthic.
6. Expand to all Plasthic workspaces.

---

## Acceptance Criteria

1. With `sellrise_local`, uploads behave exactly as current behavior.
2. With `patient_service_proxy`, photo appears in Plasthic linked to patient with `uploaded_via = chatbot`.
3. Widget cannot access patient service auth token.
4. Upload failures return deterministic error payload and preserve chat continuity.
5. Logs clearly show route decision and final outcome.

---

## Implementation Checklist

- [ ] Add `photo_upload.mode` handling in Sellrise workspace settings logic.
- [ ] Add/adjust widget photo upload endpoint to route by mode.
- [ ] Implement Sellrise proxy uploader service to Plasthic photo API.
- [ ] Ensure `patient_id` resolution from `external_identities` before proxying.
- [ ] Add structured events and error metrics.
- [ ] Add integration tests for both modes and failure fallback.
- [ ] Confirm `auth_token` never returned from widget session payload.

---

## Open Questions

1. Should missing `patient_id` trigger automatic fallback to `sellrise_local`, or hard-fail with retry only?
2. Should proxy mode support batch uploads in one request, or sequential single-file forwarding only?
3. Do we need idempotency keys for duplicate user re-submits of same photo file?

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
