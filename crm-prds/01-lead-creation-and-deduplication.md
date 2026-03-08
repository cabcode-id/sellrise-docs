# PRD 5.1 - Lead Creation and Deduplication

## Metadata

- Feature ID: `CRM-5.1`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM`
- Owner: `Product + Backend`

## Objective

Capture visitor contact details from widget conversations and persist leads using idempotent deduplication within workspace boundaries.

## User Story

As a visitor, I submit my contact details and they appear in the CRM.

## Scope

In scope:
- Lead create/upsert from widget submissions.
- Deduplication by normalized email and phone.
- Consent capture with timestamp.
- Qualification and attribution field persistence.
- Initial lead score assignment on create/update.

Out of scope:
- Manual lead import.
- Cross-workspace deduplication.

## Functional Requirements

1. API accepts widget lead payload with `session_id`, contact info, consent, and qualification fields.
2. System normalizes email to lowercase + trimmed string as `dedup_email`.
3. System normalizes phone into E.164 as `dedup_phone` when phone is provided.
4. Upsert resolution order is deterministic:
   - Match by `dedup_email` first.
   - Else match by `dedup_phone`.
   - Else create new lead.
5. Existing matched lead is updated with latest fields and attribution metadata.
6. New lead defaults to stage `new` unless otherwise specified by system rules.
7. Consent timestamp is stored when consent is true.
8. Score is computed via scoring service and persisted.
9. Event logging records `lead_submitted` with key payload data.

## API Contract

- Endpoint: `POST /v1/widget/lead`
- Request body:

```json
{
  "session_id": "uuid",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "+628123456789",
  "consent": true,
  "qualification_data": {
    "procedure": "Dental Implant",
    "budget": "high",
    "timeframe": "1_month",
    "location": "Jakarta"
  }
}
```

- Response body:

```json
{
  "lead_id": "uuid",
  "score": "hot",
  "stage": "new"
}
```

## Data Requirements

- Required: `workspace_id`, `session_id`, `consent`, at least one contact identifier (email or phone, based on widget policy).
- Core fields: `name`, `email`, `phone`, `dedup_email`, `dedup_phone`.
- Qualification fields: `procedure`, `budget`, `timeframe`, `location`.
- Attribution fields: `page_url`, `referrer`, `utm_source`, `utm_medium`, `utm_campaign`, others as available.
- Constraints:
  - Unique index for (`workspace_id`, `dedup_email`) where not null.
  - Optional unique index for (`workspace_id`, `dedup_phone`) where not null.

## Acceptance Criteria

1. Visitor submits contact data via widget and receives successful response.
2. Existing lead is updated if dedup match exists.
3. New lead is created if no dedup match exists.
4. Duplicate `dedup_email` per workspace is prevented by DB constraints.
5. Consent is persisted with timestamp.
6. Score is assigned and returned.

## Dependencies

- Widget session handshake (`/v1/widget/session`).
- Event logging (`lead_events`).
- Lead scoring service.

## Non-Functional Requirements

- P95 API response under 500ms at 10k+ lead scale.
- Strict workspace isolation for all read/write paths.
- Idempotent behavior for repeated submissions.

## Test Coverage

- Unit: email/phone normalization and matching precedence.
- Integration: widget lead submit -> upsert -> event log -> score assignment.
- Security: cross-workspace ID probes return 404.
