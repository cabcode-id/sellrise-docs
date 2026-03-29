# AGILE PRD Implementation Audit (FE + BE)

Audit date: 2026-03-09  
Reference source: `sellrise-docs/AGILE_PRD.md`  
Audit scope:
- Frontend: `Front-End- Sellrise`
- Backend: `SellriseBE`

---

## Status Summary

- Total user stories: **35**
- **Implemented**: 34
- **Partial**: 1
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
| US-1.2 | Workspace Data Isolation | ⚠️ | ✅ | **Implemented** | Fully enforced: `/v1/steps/*` endpoints now scope all conversation and scenario lookups by `workspace_id`. All authenticated admin/CRM endpoints return 404 on cross-workspace access. Rule R1 compliance verified. |
| US-1.3 | Domain Registration | ✅ | ✅ | **Implemented** | Domain CRUD is fully implemented in `domains.py` with branding overrides (`brand_name`, `brand_logo_url`, `brand_primary_color`); widget session handshake rejects unregistered/inactive domains with `403` per PRD. |
| US-2.1 | User Login (JWT) | ✅ | ✅ | **Implemented** | Login, refresh rotation, logout, and me endpoints are available. |
| US-2.2 | RBAC | ✅ | ✅ | **Implemented** | RBAC is enforced across all modules (Admin/Agent/Viewer) using `require_admin`/`require_agent` dependencies; workspace isolation is strictly applied. |
| US-2.3 | User Invitation | ✅ | ✅ | **Implemented** | BE user creation now enforces email uniqueness per-workspace (not global), preserving `409` only for duplicates in the same workspace. |
| US-2.4 | Logout | ✅ | ✅ | **Implemented** | Refresh token invalidation + cookie clear are implemented. |
| US-3.1 | Widget Embed Code Generation | ✅ | ✅ | **Implemented** | FE can generate embed snippets; BE public widget endpoints are available. |
| US-3.2 | Widget Display Modes | ✅ | ✅ | **Implemented** | Bubble + inline modes are available in FE preview/settings. |
| US-3.3 | Widget Session Handshake | ✅ | ✅ | **Implemented** | `/v1/widget/session` is available and used for widget initialization. |
| US-3.4 | Widget Fallback on API Failure | ✅ | ✅ | **Implemented** | FE fallback form renders on session timeout/failure; BE `POST /v1/widget/fallback-lead` now enforces strict domain-based workspace resolution (no first-workspace fallback), creates deduped leads, logs `fallback_lead_submitted`, and triggers hot-lead notification when applicable. |
| US-4.1 | Scenario CRUD (Draft Editing) | ✅ | ✅ | **Implemented** | Create/list/get/patch/delete exist; DELETE endpoint now prevents deletion of published scenarios. |
| US-4.2 | Scenario Publishing | ✅ | ✅ | **Implemented** | Publish endpoint exists with unpublish-others + version bump behavior. |
| US-4.3 | Step Execution (Conversation Flow) | ✅ | ✅ | **Implemented** | BE now has both `/v1/steps/*` execution APIs and `POST /v1/scenarios/{id}/simulate` (LLM-powered, reads scenario config/rules/stages/slots, calls OpenRouter). FE `ScenarioSimulator.jsx` calls simulate endpoint and receives `{ reply, slots, stage_id }`. |
| US-4.4 | Event Logging | ✅ | ✅ | **Implemented** | `/v1/widget/event` and lead/session event persistence are implemented. |
| US-4.5 | Centralized Conversation System | ⚠️ | ⚠️ | **Partial** | Conversation + step-execution modules exist, but `POST /v1/widget/message` still uses a placeholder bot reply and requires final production-wiring to the scenario processor. |
| US-5.1 | Lead Creation & Deduplication | ✅ | ✅ | **Implemented** | Dedup by email then phone is implemented in lead service. |
| US-5.2 | Lead Inbox (New/Unassigned) | ✅ | ✅ | **Implemented** | FE inbox + BE lead list/filter are working. |
| US-5.3 | Pipeline Kanban View | ✅ | ✅ | **Implemented** | FE kanban + BE stage patch are available. |
| US-5.4 | Lead Detail View | ✅ | ✅ | **Implemented** | Lead detail + events + notes are available. |
| US-5.5 | Lead Search & Filters | ✅ | ✅ | **Implemented** | Stage/score/search filters exist in both BE and FE. |
| US-5.6 | Lead Notes | ✅ | ✅ | **Implemented** | Notes endpoint + add/view notes UI are implemented. |
| US-5.7 | Lead Assignment | ✅ | ✅ | **Implemented** | `user_id` assignment is fully supported; user management ecosystem (list/get users) now exists in `users.py`. |
| US-5.8 | Lead Scoring | ✅ | ✅ | **Implemented** | Rule-based scoring + score update event are implemented. |
| US-6.1 | KB Articles CRUD | ✅ | ✅ | **Implemented** | Articles CRUD is available in FE/BE. |
| US-6.2 | KB FAQ CRUD | ✅ | ✅ | **Implemented** | FAQ CRUD is available in FE/BE. |
| US-6.3 | KB Full-Text Search | ✅ | ✅ | **Implemented** | ILIKE search for articles/FAQs is available (basic full-text). |
| US-6.4 | KB-Only Answer Mode | ✅ | ✅ | **Implemented** | `LLMService.enhance_kb_answer` now supports `strict=True` mode; enabled via `kb_only` in scenario config. Replaces general assistant fallback with strict "I don't know" when context is missing. |
| US-7.1 | Funnel Metrics Dashboard | ✅ | ✅ | **Implemented** | FE Analytics wiring complete: `analyticsService.js` now calls all BE endpoints (summary, funnel, sources, dropoff). |
| US-7.2 | CSV Export | ✅ | ✅ | **Implemented** | FE already exports CSV client-side and BE now also provides `/v1/analytics/export` for server-side export. |
| US-8.1 | Hot Lead Email Notification | ✅ | ✅ | **Implemented** | Email notification service includes retry logic, exponential backoff, and proper event logging; lead fetch is now workspace-scoped (`lead_id` + `workspace_id`) for tenant safety. |
| US-8.2 | Notification Event Logging | ✅ | ✅ | **Implemented** | `notification_sent` and `notification_failed` event types now logged to LeadEvents after email attempts. |
| US-9.1 | Workspace Settings | ✅ | ✅ | **Implemented** | Workspace CRUD and Notification Settings (`hot_lead_recipients`, etc.) are now fully available in `workspaces.py` and `notification_settings.py`. |
| US-9.2 | Team Management | ✅ | ✅ | **Implemented** | FE team management (invite/role/deactivate) and BE `/v1/users` management endpoints are now available. |
| US-10.1 | Booking Link Handoff | ✅ | ✅ | **Implemented** | BE now provides `/v1/booking/tokens` (creation, GET, consumption) for end-to-end booking tracking. Tokens are short-lived and single-use. |

---

## Most Critical Gaps (Recommended Priority)

1. **Widget Message Wiring (US-4.5)**  
   The primary remaining gap: `POST /v1/widget/message` needs to be wired to the actual scenario engine (similar to how `simulate` works) to provide production-ready responses instead of placeholders.

2. **Alembic Migrations**  
   Ensure all new model fields (like `notification_config` in `Workspace`) are reflected in actual database migrations.

---

## Additional Notes

- Main integration gap now is API contract mismatch: FE still uses `/v1/scenarios/{id}/simulate` (now implemented in BE) while the production widget flow uses `/v1/steps/*` and `/v1/widget/message`.
- `POST /v1/widget/fallback-lead` endpoint now matches FE contract (domain-based workspace resolution, dedup, event logging).
- `POST /v1/scenarios/{id}/simulate` endpoint now matches FE `ScenarioSimulator.jsx` contract (LLM-powered with scenario config context).
- Test coverage has improved significantly: dedicated test files now exist for workspaces/users/domains/notification settings/widget flows, in addition to RBAC tests.
