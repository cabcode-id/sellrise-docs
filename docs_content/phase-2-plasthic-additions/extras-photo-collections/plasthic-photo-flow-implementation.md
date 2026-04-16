# Plasthic Photo Flow Implementation (Sellrise -> Plasthic Reverse Signal)

## Context

New requirement for acknowledgement email:

- If user has not uploaded photos in chatbot: include extra line in email.
- If user already uploaded photos in chatbot: do not include extra line.

This implementation makes **Plasthic backend the source of truth** for photo existence.

## Final Architecture Decision

1. Browser uploads photos directly to Plasthic (`/api/v1/patients/{patient_id}/photos`).
2. Sellrise sends final enquiry completion callback to Plasthic.
3. Plasthic checks its own `photos` table to decide email variant.
4. Plasthic sends acknowledgement email with conditional line.

Reason:

- Prevent split-brain state between systems.
- Avoid storing photo binaries in Sellrise.
- Keep medical-photo privacy controls centralized in Plasthic.

## Readiness Check

### Backend readiness (already available)

- Photo storage model exists in Plasthic: `photos` table with `original_filename`, `file_path`, `mime_type`, `uploaded_via`.
- File storage is non-public path-based storage (not static public URL).
- Photo routes are protected by staff authentication in current API.
- Sellrise already stores and exposes linked `patient_id` (`external_identities`) for widget upload flow.

### Database readiness (already available)

The current schema already records filename metadata:

- `photos.original_filename`
- `photos.file_path` (internal path, not exposed)
- `photos.uploaded_via` (`chatbot|crm|cabinet`)
- `photos.created_at`

No new table is required for this feature.

## Required Implementation Changes

## 1) PlasthicBE: Add completion endpoint

Add endpoint:

- `POST /api/v1/integrations/sellrise/enquiry-completed`

Auth:

- Bearer service token (same integration token model used today).

Payload (recommended):

```json
{
  "sellrise_lead_id": "uuid-or-string",
  "sellrise_workspace_id": "uuid-or-string",
  "patient_id": "uuid",
  "conversation_id": "uuid",
  "completed_at": "2026-04-16T10:00:00Z",
  "image_uploaded_in_chatbot": true
}
```

Behavior:

1. Resolve patient (prefer `patient_id`, fallback to `sellrise_lead_id + sellrise_workspace_id`).
2. Count photos for patient (recommended filter: `uploaded_via = 'chatbot'`).
3. Compute `include_photo_reply_prompt = (chatbot_photo_count == 0)`.
4. Send acknowledgement email with this flag.
5. Log event for observability and idempotency.

Idempotency:

- Use `conversation_id` as idempotency key.
- Repeated callbacks for same key must not re-send email.

## 2) PlasthicBE: Email template conditional line

Template to update:

- `app/email_templates/customer_enquiry_acknowledgement.html`

Add conditional block:

```html
{% if include_photo_reply_prompt %}
<p style="margin:16px 0 0 0;font-size:15px;color:#3f3f3f;line-height:1.7;">
  If you have not yet uploaded your pictures, please reply to this email with your images so our team can review them as part of your enquiry.
</p>
{% endif %}
```

Render inputs required:

- `customer_name`
- `procedure`
- `include_photo_reply_prompt` (boolean)

## 3) PlasthicBE: secure photo access guarantees

Keep these guarantees enforced:

- No public static file serving from patient media directory.
- Download/list photo endpoints require valid JWT and role authorization.
- Never expose internal `file_path` in external response payloads.
- Keep storage location outside publicly served directories.

Recommended hardening:

- Add explicit tests for `401/403` on photo endpoints without auth.
- Add audit events for `download_photo` and `list_photos` access.

## 4) Sellrise-BackEnd: send reverse completion callback

At chatbot completion (same moment currently used for lead-submitted finalization):

1. Resolve linked `patient_id` from lead `external_identities`.
2. Send signed callback to Plasthic completion endpoint.
3. Non-blocking behavior: callback failures must not block lead completion.
4. Retry policy: `1s -> 5s -> 30s`.

Do not push photo binary from Sellrise.

## 5) Sellrise widget: keep direct upload and event logging

Current direction remains valid:

- Widget uploads directly to Plasthic.
- Optional event `photo_uploaded` remains in Sellrise for analytics only.

Important:

- Final email decision must be made by Plasthic DB state, not Sellrise event count.

## Optional DB Optimization

No schema redesign is required.

Recommended index for faster lookup when traffic grows:

```sql
CREATE INDEX IF NOT EXISTS idx_photos_patient_uploaded_via_created
ON photos (patient_id, uploaded_via, created_at DESC);
```

## No Admin Panel Constraint

Current status:

- Plasthic has no full admin UI to browse photos.

Operational plan now:

1. Keep photos accessible only through authenticated API (already aligned with security requirement).
2. Use staff-authenticated API clients for internal review workflows.
3. Add lightweight internal viewer later (phase follow-up), not required to deliver conditional email feature.

## Acceptance Criteria

- Completion callback accepted by Plasthic with valid service token.
- When patient has zero chatbot photos, email includes additional upload-request line.
- When patient has at least one chatbot photo, email omits that line.
- Repeated callback for same conversation does not send duplicate email.
- Photo files cannot be accessed without authentication.
- API response never leaks storage internal path.

## Delivery Checklist

- [ ] Implement completion endpoint in PlasthicBE
- [ ] Implement idempotency guard for acknowledgement email
- [ ] Add conditional block in acknowledgement template
- [ ] Add email rendering path with `include_photo_reply_prompt`
- [ ] Add callback sender from Sellrise-BackEnd
- [ ] Add integration tests (with and without uploaded photos)
- [ ] Add auth regression tests for photo access
- [ ] Add operational runbook entry for support team

## Rollout Sequence

1. Deploy Plasthic completion endpoint + template changes first.
2. Verify endpoint manually from staging Sellrise token.
3. Deploy Sellrise callback sender.
4. Run E2E scenario tests:
   - Lead with no photo
   - Lead with at least one photo
5. Monitor callback success and email variant counts for 48 hours.
