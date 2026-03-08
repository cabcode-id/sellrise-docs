# PRD 5.5 - Lead Search and Filters

## Metadata

- Feature ID: `CRM-5.5`
- Priority: `MVP Phase 1`
- Release Focus: `MVP 1 (Current)`
- Category: `CRM (Backend)`
- Owner: `Product + Backend + Frontend`

## Objective

Allow agents to quickly find leads using structured filters and free-text search at CRM scale.

## User Story

As an agent, I search and filter leads to find specific contacts.

## Scope

In scope:
- Multi-filter query support.
- Text search by name, email, phone.
- Combined filter logic (`AND`).
- Pagination and performance guardrails.

Out of scope:
- Full-text fuzzy ranking across notes/transcripts.
- Saved views.

## Supported Filters

- `stage`
- `score`
- `owner`
- `created_at` date range (`from`, `to`)
- `procedure` or category

## Search Fields

- Name (case-insensitive)
- Email (case-insensitive)
- Phone

## Functional Requirements

1. API accepts query parameters for all supported filters.
2. Multiple filters combine with `AND` semantics.
3. Search term applies across name/email/phone fields.
4. Results are paginated using `limit` and `offset`.
5. Default sorting is `created_at desc` unless overridden.
6. Workspace scoping is mandatory for all queries.

## API Contract

- Endpoint:

```text
GET /v1/leads?stage=new&score=hot&owner=user_id&search=john&limit=20&offset=0
```

- Response includes filtered list and pagination metadata.

## Acceptance Criteria

1. Filters apply correctly via query params.
2. Search finds matches in name, email, or phone.
3. Combined filters produce expected narrowed result set.
4. Results paginate correctly.
5. P95 response time is under 500ms for 10k leads.

## Dependencies

- Lead persistence model and indexes.
- Owner/user data for owner filter values.

## Non-Functional Requirements

- Add DB indexes to support filter and search paths.
- Query plan remains stable at production volume.

## Test Coverage

- Unit: query builder for mixed filters.
- Integration: filter combinations + pagination behavior.
- Performance: benchmark on seeded dataset >=10k leads.
