# PRD 5.4 - Lead Detail View

## Metadata

- Feature ID: `CRM-5.4`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM (Frontend + Backend)`
- Owner: `Product + Frontend + Backend`

## Objective

Provide a complete operational workspace for a single lead, including context, transcript, and actions.

## User Story

As an agent, I review lead details and take actions.

## Scope

In scope:
- Lead profile and qualification summary.
- Source attribution details.
- Transcript rendering from events.
- Notes and tags surfaces.
- Owner and stage actions.

Out of scope:
- Follow-up campaign sequencing.
- Real-time operator takeover.

## Information Architecture

1. Contact Card: name, email, phone, preferred method, timezone.
2. Qualification: procedure, budget, timeframe, location, score.
3. Pipeline: current stage, owner.
4. Source: page URL, referrer, UTM values.
5. Transcript: timeline from `lead_events` in ascending timestamp.
6. Notes: chronological entries with author and timestamp.
7. Tags: searchable labels.

## Functional Requirements

1. Detail endpoint returns all sections required by the view.
2. Transcript is derived from events only, ordered by `created_at`.
3. Stage updates and owner assignment are available from the detail page.
4. Notes can be created and displayed without page breakage.
5. Tags can be added/removed and persisted.
6. All actions write corresponding events when applicable.

## API Contracts

- `GET /v1/leads/{id}`
- `PATCH /v1/leads/{id}`
- `POST /v1/leads/{id}/notes`
- `PATCH /v1/leads/{id}/tags`

## Acceptance Criteria

1. All lead data sections are visible and accurate.
2. Transcript matches ordered event history.
3. Actions (assign, stage, notes, tags) persist correctly.
4. Notes are visible to all users in the same workspace.
5. Tags are searchable and retrievable.

## Dependencies

- Lead creation.
- Event logging.
- Notes and tags storage.

## Non-Functional Requirements

- Detail page should load under 2.5 seconds P95 for leads with up to 1,000 events.
- Event timeline rendering should be virtualized/paginated if needed.

## Test Coverage

- Unit: transcript ordering and formatting.
- Integration: detail read + mutate actions.
- E2E: open detail, add note, change owner, change stage, verify timeline updates.
