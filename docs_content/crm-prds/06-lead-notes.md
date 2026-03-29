# PRD 5.6 - Lead Notes

## Metadata

- Feature ID: `CRM-5.6`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM`
- Owner: `Product + Backend + Frontend`

## Objective

Allow agents to add internal context and follow-up records directly on lead profiles.

## User Story

As an agent, I add notes to track follow-ups and context.

## Scope

In scope:
- Create notes with text content.
- List notes chronologically in lead detail.
- Display note author and timestamp.
- Emit note-related events.

Out of scope:
- Rich text editor.
- Note threading.
- Collaborative real-time editing.

## Functional Requirements

1. Notes can be created through lead detail using `POST /v1/leads/{id}/notes`.
2. Notes list supports chronological rendering.
3. Every note displays author identity and creation timestamp.
4. Creating note logs `note_added` event with note metadata.
5. Optional MVP behavior: only note author can delete their own notes.

## API Contracts

- `POST /v1/leads/{id}/notes`
- `GET /v1/leads/{id}/notes`
- `DELETE /v1/notes/{note_id}` (optional in MVP)

## Acceptance Criteria

1. Agent adds note successfully to a lead.
2. Added note appears in notes list with author and timestamp.
3. Notes are visible in lead detail to workspace users.
4. `note_added` event is recorded.

## Dependencies

- Lead detail view.
- Event logging service.
- RBAC policy.

## Non-Functional Requirements

- Note create request returns under 300ms P95.
- Note content length is validated and sanitized.

## Test Coverage

- Unit: note payload validation.
- Integration: create note -> list note -> event emitted.
- Authorization: role checks and delete permissions.
