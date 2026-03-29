# Sellrise CRM PRDs

This folder slices the CRM scope from `docs/FEATURE_BREAKDOWN.md` into individual Product Requirement Documents (PRDs), one feature per file.

## CRM Feature PRDs

1. `01-lead-creation-and-deduplication.md`
2. `02-lead-inbox-new-unassigned.md`
3. `03-pipeline-kanban-view.md`
4. `04-lead-detail-view.md`
5. `05-lead-search-and-filters.md`
6. `06-lead-notes.md`
7. `07-lead-assignment.md`
8. `08-lead-scoring.md`

## Current Delivery Focus

`MVP 1 (Now)`

- The CRM PRDs in this folder are all prioritized for `MVP Phase 1` delivery.
- Build and QA should execute these files in order (`01` -> `08`) unless dependency constraints require a different sequence.
- Any `MVP Phase 2` or `Post-MVP` expansion should be added as separate sections under each PRD to keep MVP scope locked.

## Scope

- Source section: `5. Lead Management (Mini-CRM)`
- Source document: `docs/FEATURE_BREAKDOWN.md`
- Included items: Features `5.1` to `5.8`

## Notes

- These PRDs preserve MVP intent and acceptance criteria from the feature breakdown.
- API contracts should stay aligned with backend implementation under `Sellrise-BackEnd/app/api/v1/`.
