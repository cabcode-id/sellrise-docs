# PRD 5.8 - Lead Scoring

## Metadata

- Feature ID: `CRM-5.8`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM (Backend)`
- Owner: `Product + Backend`

## Objective

Automatically classify lead quality into hot, warm, or cold to help teams prioritize follow-up.

## User Story

As an operator, I want leads scored automatically so I can prioritize high-intent opportunities.

## Scope

In scope:
- Rule-based scoring on lead create.
- Re-scoring when qualification fields change.
- Persisted score on lead record.
- Score event logging.

Out of scope:
- ML-based scoring models.
- Workspace-level custom scoring UI (future).

## Scoring Rules (MVP)

- `hot`: high budget + urgent timeframe + clear need.
- `warm`: medium budget or longer timeframe with intent.
- `cold`: low budget or unclear/no timeline.

## Functional Requirements

1. Scoring logic runs on lead creation.
2. Scoring logic re-runs when qualification attributes are updated.
3. Resulting score is stored in `lead.score`.
4. Score transitions log `score_assigned` event with previous/new score.
5. Scoring service is deterministic for same input payload.

## Implementation Notes

- Source service: `Sellrise-BackEnd/app/services/lead_scoring.py`.
- Keep rules isolated so future configurability can be added without changing API contracts.

## Acceptance Criteria

1. New leads are scored automatically.
2. Score is persisted and visible in inbox, pipeline, and lead detail.
3. Score updates after qualification data changes.
4. Score event is logged for auditability.

## Dependencies

- Lead creation/update flow.
- Event logging infrastructure.

## Non-Functional Requirements

- Scoring function executes within 20ms per lead on average.
- Scoring service has comprehensive unit test coverage.

## Test Coverage

- Unit: rule matrix for budget/timeframe/need combinations.
- Integration: create/update lead triggers scoring and event creation.
- Regression: deterministic output for fixed fixtures.
