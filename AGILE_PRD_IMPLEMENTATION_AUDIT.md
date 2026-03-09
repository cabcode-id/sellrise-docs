# AGILE PRD Implementation Audit (FE + BE)

Audit date: 2026-03-09  
Reference source: `sellrise-docs/AGILE_PRD.md`  
Audit scope:
- Frontend: `Front-End- Sellrise`
- Backend: `SellriseBE`

---

## Status Summary

- Total user stories: **35**
- **Implemented**: 26
- **Partial**: 9
- **Missing**: 0

> Definitions:
> - **Implemented**: available end-to-end for core requirements.
> - **Partial**: implementation exists, but is incomplete and/or not fully aligned with PRD specifications.
> - **Missing**: no usable implementation is available yet.

---

## User Story Matrix

| US | Title | FE | BE | Status | Notes |
|---|---|---|---|---|---|
| US-1.1 | Workspace Creation | ✅ | ✅ | **Implemented** | BE `POST /v1/workspaces` now enforces `name`, initializes default `settings` JSON, and returns `workspace_id` + standard workspace fields; creator is assigned `ADMIN`. |
| US-1.2 | Workspace Data Isolation | ⚠️ | ✅ | **Implemented** | Most BE queries already enforce `workspace_id`; 404 cross-workspace behavior is applied in core data endpoints. |
| US-1.3 | Domain Registration | ✅ | ⚠️ | **Partial** | Domain CRUD exists, but PRD fields/contracts (`domain_name`, `dev_mode_enabled`) are not fully aligned. |
| US-2.1 | User Login (JWT) | ✅ | ✅ | **Implemented** | Login, refresh rotation, logout, and me endpoints are available. |
| US-2.2 | RBAC | ✅ | ⚠️ | **Partial** | Role enforcement exists in core endpoints; coverage is incomplete for user/workspace modules because those endpoints are missing. |
| US-2.3 | User Invitation | ✅ | ✅ | **Implemented** | BE user creation now enforces email uniqueness per-workspace (not global), preserving `409` only for duplicates in the same workspace. |
| US-2.4 | Logout | ✅ | ✅ | **Implemented** | Refresh token invalidation + cookie clear are implemented. |
| US-3.1 | Widget Embed Code Generation | ✅ | ✅ | **Implemented** | FE can generate embed snippets; BE public widget endpoints are available. |
| US-3.2 | Widget Display Modes | ✅ | ✅ | **Implemented** | Bubble + inline modes are available in FE preview/settings. |
| US-3.3 | Widget Session Handshake | ✅ | ✅ | **Implemented** | `/v1/widget/session` is available and used for widget initialization. |
| US-3.4 | Widget Fallback on API Failure | ✅ | ✅ | **Implemented** | Fallback form + `/v1/widget/fallback-lead` are implemented. |
| US-4.1 | Scenario CRUD (Draft Editing) | ✅ | ✅ | **Implemented** | Create/list/get/patch/delete exist; DELETE endpoint now prevents deletion of published scenarios. |
| US-4.2 | Scenario Publishing | ✅ | ✅ | **Implemented** | Publish endpoint exists with unpublish-others + version bump behavior. |
| US-4.3 | Step Execution (Conversation Flow) | ⚠️ | ⚠️ | **Partial** | BE now has `/v1/steps/*` execution APIs, but FE still calls `/v1/scenarios/{id}/simulate` and the flow is not yet fully wired end-to-end. |
| US-4.4 | Event Logging | ✅ | ✅ | **Implemented** | `/v1/widget/event` and lead/session event persistence are implemented. |
| US-4.5 | Centralized Conversation System | ⚠️ | ⚠️ | **Partial** | Conversation + step-execution modules now exist (`/v1/conversations`, `/v1/steps`), but widget message handling still contains TODO/placeholder behavior and is not fully production-wired. |
| US-5.1 | Lead Creation & Deduplication | ✅ | ✅ | **Implemented** | Dedup by email then phone is implemented in lead service. |
| US-5.2 | Lead Inbox (New/Unassigned) | ✅ | ✅ | **Implemented** | FE inbox + BE lead list/filter are working. |
| US-5.3 | Pipeline Kanban View | ✅ | ✅ | **Implemented** | FE kanban + BE stage patch are available. |
| US-5.4 | Lead Detail View | ✅ | ✅ | **Implemented** | Lead detail + events + notes are available. |
| US-5.5 | Lead Search & Filters | ✅ | ✅ | **Implemented** | Stage/score/search filters exist in both BE and FE. |
| US-5.6 | Lead Notes | ✅ | ✅ | **Implemented** | Notes endpoint + add/view notes UI are implemented. |
| US-5.7 | Lead Assignment | ✅ | ⚠️ | **Partial** | `user_id` assignment is possible, but full user management ecosystem is not complete yet. |
| US-5.8 | Lead Scoring | ✅ | ✅ | **Implemented** | Rule-based scoring + score update event are implemented. |
| US-6.1 | KB Articles CRUD | ✅ | ✅ | **Implemented** | Articles CRUD is available in FE/BE. |
| US-6.2 | KB FAQ CRUD | ✅ | ✅ | **Implemented** | FAQ CRUD is available in FE/BE. |
| US-6.3 | KB Full-Text Search | ✅ | ✅ | **Implemented** | ILIKE search for articles/FAQs is available (basic full-text). |
| US-6.4 | KB-Only Answer Mode | ⚠️ | ⚠️ | **Partial** | KB search endpoint exists, but strict “KB-only” answer enforcement in conversation engine is still incomplete. |
| US-7.1 | Funnel Metrics Dashboard | ✅ | ⚠️ | **Partial** | Summary dashboard exists; aggregation is still basic and not fully aligned with PRD time dimensions. |
| US-7.2 | CSV Export | ✅ | ✅ | **Implemented** | FE already exports CSV client-side and BE now also provides `/v1/analytics/export` for server-side export. |
| US-8.1 | Hot Lead Email Notification | ✅ | ✅ | **Implemented** | Email notification service now includes retry logic, exponential backoff, and proper event logging. |
| US-8.2 | Notification Event Logging | ✅ | ✅ | **Implemented** | `notification_sent` and `notification_failed` event types now logged to LeadEvents after email attempts. |
| US-9.1 | Workspace Settings | ⚠️ | ⚠️ | **Partial** | Workspace update API exists and notification settings endpoints are added, but the full PRD settings model is not fully standardized yet. |
| US-9.2 | Team Management | ✅ | ✅ | **Implemented** | FE team management (invite/role/deactivate) and BE `/v1/users` management endpoints are now available. |
| US-10.1 | Booking Link Handoff | ⚠️ | ⚠️ | **Partial** | `booking_links` are passed from scenario, but booking token/handoff endpoint flow is still incomplete. |

---

## Most Critical Gaps (Recommended Priority)

1. **US-4.3 + US-4.5 + FE/BE wiring**  
   Align FE with current BE conversation execution APIs (`/v1/steps/*` or provide `/v1/scenarios/{id}/simulate` compatibility), then complete deterministic end-to-end runtime.

2. **US-6.4 (strict KB-only enforcement)**  
   Implement deterministic conversation engine with strict KB-only mode enforcement.

3. **US-10.1**  
   Complete end-to-end booking handoff (token generation, token consumption endpoint, lead status update).

---

## Additional Notes

- BE now supports previously missing modules (`/v1/workspaces`, `/v1/users`, `/v1/llm/*`, `/v1/files/upload`, `/v1/analytics/export`, `/v1/steps/*`, `/v1/conversations/*`).
- Main integration gap now is API contract mismatch: FE still uses `/v1/scenarios/{id}/simulate` while BE execution flow is exposed under `/v1/steps/*` and `/v1/widget/message`.
- Test coverage has improved significantly: dedicated test files now exist for workspaces/users/domains/notification settings/widget flows, in addition to RBAC tests.
