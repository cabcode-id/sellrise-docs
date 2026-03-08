# PRD 5.3 - Pipeline (Kanban View)

## Metadata

- Feature ID: `CRM-5.3`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM (Frontend + Backend)`
- Owner: `Product + Frontend + Backend`

## Objective

Enable agents to manage lead progress visually across pipeline stages.

## User Story

As an agent, I manage leads by stage visually.

## Scope

In scope:
- Kanban board with fixed stage columns.
- Drag-and-drop stage changes (desktop).
- Stage selector fallback for mobile.
- Event logging for every stage transition.
- RBAC enforcement.

Out of scope:
- Custom stage configuration.
- Stage automation workflows.

## Pipeline Stages

- `new`
- `qualified`
- `contacted`
- `booked`
- `won`
- `lost`

## Functional Requirements

1. Board renders all default columns in stable order.
2. Leads can be moved between columns via drag-and-drop.
3. Mobile devices use dropdown/select stage update.
4. Stage update calls `PATCH /v1/leads/{id}` with new stage value.
5. Backend validates stage transitions against allowed enum values.
6. Each successful stage change appends `stage_changed` event with old/new values and actor metadata.
7. Viewer role cannot change stage and receives `403`.

## API Contract

- Endpoint: `PATCH /v1/leads/{id}`
- Request:

```json
{
  "stage": "qualified"
}
```

- Response returns updated lead object including current stage.

## Acceptance Criteria

1. Stages are visible as columns.
2. Dragging a lead updates stage and board placement.
3. Mobile stage selector updates stage successfully.
4. `stage_changed` event is logged with previous and new stage.
5. Viewer attempts are blocked with `403`.

## Dependencies

- Lead creation.
- RBAC.
- Event logging.

## Non-Functional Requirements

- Drag action to persisted stage update should complete under 800ms P95.
- Board should remain usable with 500 leads loaded across columns.

## Test Coverage

- Unit: stage enum validation.
- Integration: patch stage -> event creation.
- E2E: drag lead between columns and verify persistence after reload.
