# PRD 5: Compliance & Data Lifecycle

## Overview

| Field | Value |
|-------|-------|
| **Modules** | 5 (Email Validation) + Compliance from Module 1 |
| **System** | System 2: Phlastic Patient Service (new VPS) |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 1 day (email validation) + included in Module 1 (compliance endpoints) |
| **Dependencies** | Module 1 |
| **Priority** | P0 — Legal requirement (Australian Privacy Act 1988) |

## Goal

Ensure full compliance with the **Australian Privacy Act 1988 (APP)** through robust data lifecycle management — including email validation, data export, secure deletion, and an immutable audit log.

---

## Australian Privacy Act 1988 — Compliance Matrix

| Principle | Requirement | Implementation |
|-----------|-------------|----------------|
| APP 1 — Management | Transparency about data handling | Privacy notice shown before data collection |
| APP 3 — Collection | Only collect necessary data, with consent | Consent checkbox + timestamp mandatory before ANY data storage |
| APP 5 — Notification | Tell patient what's collected and why | Consent text explains: what, why, who sees it |
| APP 6 — Use/Disclosure | Use only for stated purpose | Data used only for consultation process |
| APP 8 — Cross-border | Protect data if it leaves AU | VPS in AU region (Sydney preferred) |
| APP 11 — Security | Protect from misuse, loss, unauthorized access | AES-256 at rest + TLS 1.2+ in transit + audit log |
| APP 12 — Access | Individual can request their data | `GET /patients/:id/export` endpoint |
| APP 13 — Correction | Individual can correct their data | `PUT /patients/:id` endpoint |

**Full compliance technical checklist:**

- [ ] VPS in Australian region (DigitalOcean SYD / AWS ap-southeast-2)
- [ ] Database encrypted at rest (AES-256)
- [ ] All API traffic over TLS 1.2+
- [ ] Consent collected before ANY patient data stored (timestamp + IP + consent text version)
- [ ] Access log: every read/write logged (who, when, what) in `patient_events`
- [ ] Data deletion endpoint: permanently delete patient + all associated data
- [ ] Data export endpoint: export all data for a single patient (JSON)
- [ ] No patient PII in Sellrise DB
- [ ] Photos stored encrypted, accessible only via authenticated API
- [ ] API keys rotatable, minimum 32 characters
- [ ] `consent_records` table is NEVER deleted (legal requirement)

---

## Module 5: Email Validation

### Goal

Validate patient emails automatically and asynchronously — without blocking patient creation.

### Validation Trigger

```
1. Patient created (POST /patients) → email_status = "unknown"
2. Response sent immediately (201) — validation does NOT block creation
3. Background async job starts:
   a. Format check (regex): is this a valid email format?
   b. MX record check (DNS lookup): does the domain have mail servers?
   c. (Optional) External API check if configured
4. After checks complete → update email_status:
   - "valid"   — format OK + MX record exists
   - "invalid" — bad format OR no MX record
   - "unknown" — couldn't check (DNS timeout or failure)
```

**Also triggers on email update:** If patient email is changed via `PUT /patients/:id`, re-run email validation automatically.

### Email Status Values

| Status | Meaning | CRM Badge |
|--------|---------|-----------|
| `valid` | Format OK + MX record found | 🟢 Green |
| `invalid` | Bad format OR no MX record | 🔴 Red |
| `unknown` | Check not yet complete or DNS timed out | ⚫ Gray |

### External Validation Service (Optional — Phase 2)

Support configurable external validation API (ZeroBounce, NeverBounce, AbstractAPI):

```json
{
  "email_validation": {
    "external_service": "zerobounce",
    "api_key": "zb_xxx",
    "enabled": false
  }
}
```

- For now: implement format + MX check only
- External service is a bonus (implement only if free tier / API key is configured)

### Email Validation Acceptance Tests

- [ ] `valid@gmail.com` → `email_status = "valid"`
- [ ] `test@nonexistent-domain-xyz123abc.com` → `email_status = "invalid"`
- [ ] `not-an-email` → `email_status = "invalid"`
- [ ] `user@` → `email_status = "invalid"`
- [ ] Validation runs async — patient creation returns 201 before validation completes
- [ ] `email_status` readable via `GET /patients/:id`
- [ ] `email_status` filterable via `GET /patients?email_status=valid`
- [ ] Updating patient email via `PUT /patients/:id` triggers re-validation
- [ ] DNS timeout handled gracefully (returns `"unknown"`, not server error)

---

## Data Lifecycle: Soft Delete

**Endpoint:** `DELETE /patients/:id`

```
Logic:
1. Set is_deleted=true, deleted_at=now()
2. Anonymize PII fields:
   - name      → "[deleted]"
   - email     → "deleted_{id}@deleted"
   - phone     → null
   - date_of_birth  → null
   - location  → null
   - medical_notes → null
   (other non-PII fields retained for analytics)
3. Keep consent_records intact (legal requirement)
4. Create patient_event: type=data_deleted, created_by=<staff_id>

Response: 204 No Content
```

**Behavior:** Soft-deleted patients are excluded from `GET /patients` by default. No hard removal from DB — data anonymized but record retained for referential integrity.

---

## Data Lifecycle: Hard Delete (Permanent Erasure)

**Endpoint:** `POST /patients/:id/hard-delete`

```
Logic:
1. Delete all photo files from disk / S3 storage
2. Delete all records from related tables:
   - photos (DB records)
   - patient_events
   - contracts
   - rewards
   - reviews
   - services
   - journey_entries
3. Delete patient record from patients table
4. DO NOT delete consent_records (legal requirement — kept forever)
5. Create one final consent_record:
   type=data_processing, given=false
   (records the withdrawal/deletion for legal audit trail)

Response: 204 No Content
```

**Important:** `consent_records` uses a FK with **NO CASCADE** — these records survive even after the patient record is deleted.

---

## Data Export (Right of Access — APP 12)

**Endpoint:** `GET /patients/:id/export`

```json
// Response 200
{
  "patient": { "...all patient fields..." },
  "photos": [
    {
      "...metadata...",
      "file_base64": "iVBORw0KGgoAAAANSUhEUg..." // base64 encoded binary
    }
  ],
  "events": [ "...all patient_events records..." ],
  "consent_records": [ "...all consent_records records..." ],
  "contracts": [ "...all contracts metadata..." ],
  "exported_at": "2026-03-30T16:00:00Z"
}
```

Creates `data_exported` event in `patient_events`.

---

## Audit Log

**Endpoint:** `GET /audit-log?patient_id={id}`

Returns all `patient_events` for the given patient, including:
- Who performed the action (`created_by`)
- When it happened (`created_at`)
- What was done (`event_type` + `event_data`)
- From where (`ip_address`)

Every read and write operation on patient data must generate an audit event:

| Operation | Event type |
|-----------|------------|
| Patient created | `patient_updated` |
| Patient data updated | `patient_updated` |
| Stage changed | `stage_changed` |
| Note added | `note_added` |
| Photo uploaded | `photo_uploaded` |
| Photo deleted | `photo_deleted` |
| Data exported | `data_exported` |
| Soft deleted | `data_deleted` |
| Hard deleted | `data_deleted` (in consent_records) |
| Owner changed | `owner_changed` |

---

## Consent Records Schema

The `consent_records` table is an **immutable legal audit trail**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | yes | Primary key |
| patient_id | UUID | yes | FK → patients.id — **NO cascade delete** |
| consent_type | enum | yes | `data_processing`, `photo_sharing`, `marketing` |
| consent_text | text | yes | Full verbatim text patient agreed to |
| consent_version | string | yes | Version identifier (e.g. `"phlastic_v1"`) |
| given | boolean | yes | `true` = consented, `false` = withdrawn |
| ip_address | string | no | Patient's IP at time of consent |
| user_agent | string | no | Browser user agent |
| created_at | datetime | yes | Immutable — never updated |

---

## Acceptance Tests

### Email Validation
- [ ] `valid@gmail.com` → `email_status = "valid"`
- [ ] `test@nonexistent-domain-xyz123abc.com` → `email_status = "invalid"`
- [ ] `not-an-email` → `email_status = "invalid"`
- [ ] Validation runs async — `POST /patients` returns 201 immediately
- [ ] Updating patient email triggers re-validation
- [ ] DNS timeout → `email_status = "unknown"` (no server error)

### Soft Delete
- [ ] `DELETE /patients/:id` sets `is_deleted=true` and anonymizes PII
- [ ] Soft-deleted patient does NOT appear in `GET /patients` list
- [ ] `consent_records` for the patient are kept intact after soft-delete

### Hard Delete
- [ ] `POST /patients/:id/hard-delete` permanently removes patient record
- [ ] All associated photos deleted from disk/S3
- [ ] `consent_records` are NOT deleted (verified in DB after operation)
- [ ] Final `consent_record` with `given=false` created as deletion record

### Data Export
- [ ] `GET /patients/:id/export` returns complete JSON including photos (base64), events, consent records
- [ ] Export creates `data_exported` event in audit log

### Audit Log
- [ ] `GET /audit-log?patient_id=xxx` returns all events for the patient
- [ ] Every read/write operation creates an audit event with `who`, `when`, `what`, `ip_address`
- [ ] Audit log is read-only — events cannot be deleted or modified
