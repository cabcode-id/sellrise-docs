# PRD 2: Webhook & Data Pipeline

## Overview

| Field | Value |
|-------|-------|
| **Module** | 2 — Sellrise → Phlastic Webhook Pipeline |
| **System** | System 1: Sellrise (existing codebase) |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 2 days |
| **Dependencies** | Module 1 (Phlastic Patient Service must be running) |
| **Priority** | P0 — Required for all lead data to flow into Phlastic |

## Goal

Integrate Sellrise (System 1) with Phlastic Patient Service (System 2) to ensure seamless, reliable data flow from the chatbot to the patient database — while maintaining complete PII isolation (zero PII in Sellrise DB).

---

## Architecture Context

```
[Sellrise Chatbot] → lead_submitted event → [Webhook Handler]
                                                    ↓
                                     POST /api/v1/patients
                                    (Phlastic Patient Service)
                                            ↓
                                 Phlastic DB stores patient PII
                                            ↓
                               Returns patient_id to Sellrise
                                            ↓
                    Sellrise stores external_patient_id on lead record
```

**The rule:** Sellrise DB stores only `external_patient_id` (a UUID reference). Zero patient PII in Sellrise.

---

## What to Build

### 1. Workspace Configuration: Patient Service Settings

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

**`profile_mapping`** maps Sellrise chatbot step IDs → Phlastic patient fields. This makes the integration configurable per scenario — different chatbots can map different answers.

---

### 2. New Field on Sellrise `leads` Table

```sql
ALTER TABLE leads ADD COLUMN external_patient_id VARCHAR(255) NULL;
```

This is the **only** new column added to Sellrise DB for this integration. No patient PII.

---

### 3. Webhook Logic

Fires on `lead_submitted` event in Sellrise:

```
1. lead_submitted event fires in Sellrise
2. Check: does this workspace have patient_service.enabled = true?
3. If no  → stop (normal Sellrise behavior, no webhook fired)
4. If yes → build payload:
   a. Take lead data (name, email, phone from contact form)
   b. Map chatbot answers to patient fields using profile_mapping config
   c. Include UTM, source_url, referrer from session
   d. Include consent data from chatbot
5. POST payload to {base_url}/patients
6. On success (201 or 200):
   a. Save returned patient_id as external_patient_id on lead record
   b. Log: success, timestamp, patient_id
7. On failure (4xx / 5xx):
   a. Retry up to 3 times with exponential backoff: 1s → 5s → 30s
   b. Log each attempt: URL, status code, response body, timestamp
   c. After 3 failures: log final failure, mark webhook as failed
   d. Lead is still saved in Sellrise — webhook failure does NOT block lead creation
```

---

### 4. Webhook Payload (Sellrise → Phlastic)

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
      {
        "step_id": "step_procedure",
        "question": "What procedure are you interested in?",
        "answer": "Rhinoplasty"
      },
      {
        "step_id": "step_budget",
        "question": "What is your budget range?",
        "answer": "$10,000-15,000"
      },
      {
        "step_id": "step_timeline",
        "question": "When would you like the procedure?",
        "answer": "3-6 months"
      }
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

---

### 5. Upsert Logic (Deduplication)

If Phlastic returns `200` (patient already exists with same email + workspace):
- Do NOT create a duplicate patient
- Phlastic merges/updates the record
- Sellrise updates `external_patient_id` with the existing patient ID
- Sellrise creates a `patient_updated` event

---

### 6. Webhook Call Log

Each webhook attempt is stored and visible in Sellrise admin:

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

This log must be viewable in Sellrise admin panel for debugging.

---

### 7. Retry Strategy (Failure Handling)

| Attempt | Delay Before Retry | Action on Failure |
|---------|-------------------|-------------------|
| 1st | Immediate | Log attempt |
| 2nd | 1 second | Log attempt |
| 3rd | 5 seconds | Log attempt |
| 4th (final) | 30 seconds | Log as `failed`, stop retrying |

After all retries exhausted:
- Webhook is marked `failed`
- Lead is still successfully saved in Sellrise (webhook failure is non-blocking)
- Error logged with full details: URL, status code, response body, timestamps

---

## Data Flow Diagram

```
Lead submitted in Sellrise chatbot
         │
         ▼
[lead_submitted event]
         │
         ├─ Check: patient_service.enabled?
         │         No → stop, normal flow
         │         Yes ↓
         ▼
[Build webhook payload]
  - name, email, phone (from contact form)
  - chatbot answers → mapped via profile_mapping
  - UTM, source_url, referrer from session
  - consent data
         │
         ▼
POST {base_url}/patients
         │
   ┌─────┴──────┐
   │            │
  201/200      4xx/5xx
   │            │
   ▼            ▼
Save           Retry (up to 3x)
external_      1s → 5s → 30s
patient_id     │
on lead        ▼ (after 3 failures)
               Log failure
               Lead still saved
```

---

## Acceptance Tests

- [ ] Completed chatbot session with `consent=true` → patient auto-created in Phlastic DB
- [ ] Completed chatbot session with `consent=false` → webhook NOT fired (no data sent to Phlastic)
- [ ] `sellrise_lead_id` correctly links the two records (Sellrise lead ↔ Phlastic patient)
- [ ] Sellrise lead record has `external_patient_id` populated after successful webhook
- [ ] `profile_mapping` config correctly maps chatbot step answers to Phlastic patient fields
- [ ] Repeat submission with same email + same workspace → upsert in Phlastic (no duplicate patient)
- [ ] Webhook failure → retries 3 times with 1s / 5s / 30s backoff
- [ ] After 3 failures → logged as failed, lead still saved in Sellrise (webhook non-blocking)
- [ ] Webhook call log visible in Sellrise admin (timestamp, status code, response time, patient_id)
- [ ] Workspace WITHOUT `patient_service` config → no webhook fired, normal Sellrise behavior
- [ ] Webhook payload includes all UTM / source / referrer data from session
- [ ] Webhook sends `consent_given`, `consent_text_version`, `consent_timestamp`, `consent_ip`
