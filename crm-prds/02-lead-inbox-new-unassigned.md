# PRD 5.2 - Lead Inbox (New and Unassigned)

## Metadata

- Feature ID: `CRM-5.2`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM (Frontend + Backend)`
- Owner: `Product + Frontend + Backend`

## Objective

Provide a default triage view for agents to quickly process newly captured or unowned leads.

## User Story

As an agent, I see new or unassigned leads to process.

## Scope

In scope:
- Inbox listing endpoint and UI view.
- Query filters for `stage = new` or `owner_id is null`.
- Default sort by newest first.
- Pagination.

Out of scope:
- Advanced filter combinations (covered by PRD 5.5).
- Bulk actions.

## Functional Requirements

1. Inbox query includes leads where stage is `new` OR owner is not assigned.
2. Results are ordered by `created_at` descending.
3. List row/cards display key triage fields:
   - Created date/time
   - Name, email, phone
   - Score badge (hot/warm/cold)
   - Procedure or interest
   - Source summary
4. Clicking an item opens lead detail view.
5. Pagination supports `limit` and `offset`.
6. Quick actions support at least view and assign.

## API Contract

- Endpoint: `GET /v1/leads`
- Suggested query:

```text
/v1/leads?inbox=true&limit=20&offset=0&sort=created_at:desc
```

- Response includes lead summary objects with pagination metadata.

## UX Requirements

- Desktop supports table or card list.
- Mobile supports simplified cards with key actions.
- Score badge colors are consistent with pipeline and detail views.

## Acceptance Criteria

1. Inbox only shows leads meeting new/unassigned condition.
2. Results are sorted newest first.
3. Score, contact, and procedure are visible per entry.
4. Row click navigates to lead detail.
5. Pagination works across datasets larger than one page.

## Dependencies

- Lead creation and scoring.
- Lead detail route/page.

## Non-Functional Requirements

- Page load under 2 seconds for first 20 records on typical broadband.
- P95 query latency under 500ms at 10k leads.

## Test Coverage

- Unit: inbox filter predicate.
- Integration: API filter + sort + pagination.
- E2E: inbox list -> click lead -> detail page opens.
