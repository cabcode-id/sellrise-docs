# PRD 13: Qualified Lead Email Automation

## Overview

| Field | Value |
|-------|-------|
| Module | 13 - Qualified Lead Email Automation |
| Systems | System 1: Sellrise CRM, System 2: Phlastic Patient Service |
| Owner | Iga Narendra (CV Diantha) |
| Priority | P0 |
| Dependencies | PRD 1 (Patient Service), PRD 2 (Webhook Pipeline), PRD 6/7 (Invite/Auth flow) |

---

## Goal

When chatbot data collection is completed and a lead agrees to in-person consultation, Sellrise must classify the lead as hot and trigger Phlastic backend to send two emails automatically:

1. Customer email: send thank you email to the lead and invite account creation.
2. Internal email: send new patient enquiry email to the sales team.

This module ensures fast follow-up and immediate lead routing without manual action.

---

## Business Flow

1. Visitor completes chatbot flow.
2. Sellrise creates/updates lead in CRM.
3. Sellrise detects explicit user agreement for in-person consultation.
4. Sellrise classifies lead as hot.
5. Sellrise calls Phlastic backend email automation endpoint.
6. Phlastic backend validates request and sends:
   - customer email using app/email_templates/thank_you_for_enquiry.html
   - internal email using app/email_templates/new_patient_enquiry.html
7. Phlastic backend logs email outcomes and returns status to Sellrise.

---

## Trigger Rule

Automation runs only when all conditions are true:

1. Lead exists in Sellrise.
2. Lead explicitly agreed to in-person consultation.
3. Lead classification is hot.
4. Lead email exists and is valid format.
5. Lead consent allows operational follow-up communication.
6. Automation for workspace is enabled.

Primary classification rule:
- in_person_consultation_agreed == true -> lead_status = "Hot Lead"

Optional secondary signal (informational only, not required for send):
- qualification_score may still be stored for analytics.

---

## Integration Contract

### Endpoint (Phlastic)

- Method: POST
- Path: /api/v1/automations/qualified-lead-email
- Auth: service JWT (actor_type=system/service)

### Request Payload (Sellrise -> Phlastic)

{
  "sellrise_workspace_id": "ws_xxx",
  "sellrise_lead_id": "lead_abc123",
  "patient_id": "uuid-if-available",
   "in_person_consultation_agreed": true,
   "consultation_preference": "in_person",
   "consultation_agreed_at": "2026-04-22T10:20:00Z",
  "qualification_score": 87,
   "qualification_level": "high",
  "customer_name": "Jane Citizen",
  "customer_email": "jane@example.com",
  "customer_phone": "+61412345678",
  "country": "Australia",
  "procedure": "Rhinoplasty",
  "travel_time": "Within 3 months",
   "lead_status": "Hot Lead",
  "consent_given": true,
  "crm_lead_url": "https://app.sellrise.ai/workspaces/ws_xxx/leads/lead_abc123"
}

### Response Payload (Phlastic -> Sellrise)

{
  "success": true,
  "request_id": "uuid",
  "customer_email": {
    "sent": true,
    "provider": "smtp",
    "message_id": "optional"
  },
  "sales_email": {
    "sent": true,
    "provider": "smtp",
    "recipient_count": 2
  }
}

---

## Templates To Use

1. Customer template:
- app/email_templates/thank_you_for_enquiry.html

2. Internal sales template:
- app/email_templates/new_patient_enquiry.html

Template source is frontend-owned HTML. Backend must not rebuild layout in Python. Backend only injects placeholders.

---

## Required Template Variables

### 1) thank_you_for_enquiry.html

Minimum required placeholders:
- {{customer_name}}

Recommended additions for account invitation:
- {{account_creation_link}}

Implementation note:
- replace CTA href="#" links with {{account_creation_link}} where relevant.

### 2) new_patient_enquiry.html

Required placeholders:
- {{lead_status}}
- {{customer_name}}
- {{customer_email}}
- {{customer_phone}}
- {{country}}
- {{procedure}}
- {{travel_time}}
- {{crm_lead_url}}

Implementation note:
- replace "View & Respond in CRM" button href="#" with {{crm_lead_url}}.

---

## Backend Processing Requirements

1. Validate auth token and workspace.
2. Validate mandatory fields and consent flag.
   - in_person_consultation_agreed must be true
   - consultation_preference must equal in_person
3. Normalize fallback values:
   - customer_phone -> Not provided
   - country -> Not provided
   - travel_time -> Not specified
   - procedure -> Not specified
4. Build account creation link for customer email:
   - If patient_id/account exists: use cabinet login URL.
   - If account not created: generate invite token and use registration URL.
5. Render both HTML templates by placeholder substitution.
6. Send two emails asynchronously:
   - customer email to customer_email
   - internal email to configured sales recipients list
7. Return combined result without blocking lead creation.
8. Log automation event for audit.

---

## Configuration

Add or confirm in Phlastic backend environment:

- LEAD_NOTIFICATION_RECIPIENTS
- CABINET_BASE_URL
- SUPPORT_EMAIL
- EMAIL_PROVIDER and provider credentials

Add or confirm in Sellrise workspace settings:

- qualified_lead_email_automation.enabled
- hot_lead_rule.in_person_consultation_required
- hot_lead_rule.required_value = true
- patient_service.base_url
- patient_service.auth_token

---

## Reliability and Idempotency

1. Sellrise retries on non-2xx with backoff: 1s, 5s, 30s.
2. Request must include idempotency key:
   - email_auto:{workspace_id}:{lead_id}:{consultation_agreed_at}
3. Phlastic must deduplicate by idempotency key within 24h.
4. Duplicate webhook should not resend the same two emails.
5. If one email fails:
   - mark partial_success
   - return per-channel result
   - do not rollback successful channel

---

## Audit and Observability

For every automation request store:
- request_id
- sellrise_workspace_id
- sellrise_lead_id
- patient_id
- idempotency_key
- trigger_timestamp
- customer_email_status
- sales_email_status
- provider response summary
- error (if any)

Suggested event type in patient_events:
- qualified_lead_email_automation

---

## Security and Compliance

1. Service-to-service JWT required.
2. No public endpoint access.
3. No sensitive PII in logs beyond required identifiers and masked email where possible.
4. Respect consent-driven communication rules.
5. Maintain audit records per Privacy Act requirements.

---

## Acceptance Criteria

- [ ] Lead agreeing to in-person consultation is marked as hot lead in Sellrise.
- [ ] Hot lead triggers exactly one automation request from Sellrise.
- [ ] Phlastic sends thank_you_for_enquiry.html to the customer.
- [ ] Phlastic sends new_patient_enquiry.html to all sales recipients.
- [ ] Customer email includes valid account creation/login link.
- [ ] Sales email includes valid CRM lead URL.
- [ ] Placeholder substitution works for all required variables.
- [ ] Duplicate webhook with same idempotency key does not resend emails.
- [ ] Partial failure is reported with per-email status.
- [ ] Full success and failure are logged with request_id.
- [ ] Lead creation in Sellrise remains non-blocking if automation fails.

---

## Out of Scope (This Module)

- Contract and payment email flow.
- Patient account creation UX redesign.
- Marketing drip campaigns.
- Multi-language templates.

---

## Implementation Notes

1. Reuse existing app/email_templates HTML files as canonical layouts.
2. Create backend helper to render HTML templates from disk and inject placeholders safely.
3. Keep subject lines configurable; defaults:
   - Customer: Thank you for contacting Plasthic
   - Sales: New Plasthic Lead - {{customer_name}}
4. Add integration test for:
   - in-person consultation agreement trigger sends two emails
   - idempotency prevents duplicate sends
   - missing required variable fails validation with clear error
