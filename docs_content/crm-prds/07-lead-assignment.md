# PRD 5.7 - Lead Assignment

## Metadata

- Feature ID: `CRM-5.7`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM`
- Owner: `Product + Backend + Frontend`

## Objective

Create clear ownership of leads by assigning them to team members and tracking ownership history.

## User Story

As an agent, I assign leads to myself or other agents.

## Scope

In scope:
- Assign owner via `owner_id`.
- Unassign owner (`owner_id = null`).
- Log ownership changes.
- Surface owner in lead detail and list filters.

Out of scope:
- Auto-routing assignment logic.
- Round-robin balancing.

## Functional Requirements

1. Assignment uses `PATCH /v1/leads/{id}` with `owner_id`.
2. Assigned user must belong to same workspace and be active.
3. Unassigning owner is supported.
4. Successful assignment emits `owner_assigned` event with previous and new owner IDs.
5. Inbox and search filters support owner-based filtering.

## API Contract

- Endpoint: `PATCH /v1/leads/{id}`
- Request body examples:

```json
{
  "owner_id": "user_uuid"
}
```

```json
{
  "owner_id": null
}
```

## Acceptance Criteria

1. Agent can assign lead to a valid user.
2. Owner field updates correctly on lead.
3. Ownership change is event-logged with old/new values.
4. Unassigned leads appear in inbox based on inbox predicate.

## Dependencies

- User management and RBAC.
- Lead detail/inbox views.
- Event logging.

## Non-Functional Requirements

- Assignment updates should complete under 300ms P95.
- Unauthorized/cross-workspace assignment attempts are blocked.

## Test Coverage

- Unit: owner validation rules.
- Integration: assignment update + event emission.
- Security: cross-workspace user assignment returns 404/403 per policy.
