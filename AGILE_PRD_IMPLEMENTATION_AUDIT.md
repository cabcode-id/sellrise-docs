# AGILE PRD Implementation Audit (FE + BE)

Audit date: 2026-03-09  
Reference source: `sellrise-docs/AGILE_PRD.md`  
Audit scope:
- Frontend: `Front-End- Sellrise`
- Backend: `SellriseBE`

---

## Status Summary

- Total user stories: **35**
- **Implemented**: 19
- **Partial**: 11
- **Missing**: 5

> Definitions:
> - **Implemented**: available end-to-end for core requirements.
> - **Partial**: implementation exists, but is incomplete and/or not fully aligned with PRD specifications.
> - **Missing**: no usable implementation is available yet.

---

## User Story Matrix

| US | Title | FE | BE | Status | Notes |
|---|---|---|---|---|---|
| US-1.1 | Workspace Creation | ❌ | ❌ | **Missing** | FE already has `workspaceService`, but BE does not yet provide `/v1/workspaces`. |
| US-1.2 | Workspace Data Isolation | ⚠️ | ✅ | **Implemented** | Most BE queries already enforce `workspace_id`; 404 cross-workspace behavior is applied in core data endpoints. |
| US-1.3 | Domain Registration | ✅ | ⚠️ | **Partial** | Domain CRUD exists, but PRD fields/contracts (`domain_name`, `dev_mode_enabled`) are not fully aligned. |
| US-2.1 | User Login (JWT) | ✅ | ✅ | **Implemented** | Login, refresh rotation, logout, and me endpoints are available. |
| US-2.2 | RBAC | ✅ | ⚠️ | **Partial** | Role enforcement exists in core endpoints; coverage is incomplete for user/workspace modules because those endpoints are missing. |
| US-2.3 | User Invitation | ⚠️ | ❌ | **Missing** | FE is ready (`/v1/users`), but BE has no users router yet. |
| US-2.4 | Logout | ✅ | ✅ | **Implemented** | Refresh token invalidation + cookie clear are implemented. |
| US-3.1 | Widget Embed Code Generation | ✅ | ✅ | **Implemented** | FE can generate embed snippets; BE public widget endpoints are available. |
| US-3.2 | Widget Display Modes | ✅ | ✅ | **Implemented** | Bubble + inline modes are available in FE preview/settings. |
| US-3.3 | Widget Session Handshake | ✅ | ✅ | **Implemented** | `/v1/widget/session` is available and used for widget initialization. |
| US-3.4 | Widget Fallback on API Failure | ✅ | ✅ | **Implemented** | Fallback form + `/v1/widget/fallback-lead` are implemented. |
| US-4.1 | Scenario CRUD (Draft Editing) | ✅ | ⚠️ | **Partial** | Create/list/get/patch exist; delete is still missing in BE although FE already prepares delete calls. |
| US-4.2 | Scenario Publishing | ✅ | ✅ | **Implemented** | Publish endpoint exists with unpublish-others + version bump behavior. |
| US-4.3 | Step Execution (Conversation Flow) | ⚠️ | ❌ | **Missing** | FE calls `/v1/scenarios/{id}/simulate`, but BE endpoint is not implemented yet. |
| US-4.4 | Event Logging | ✅ | ✅ | **Implemented** | `/v1/widget/event` and lead/session event persistence are implemented. |
| US-4.5 | Centralized Conversation System | ⚠️ | ❌ | **Missing** | No centralized conversation runtime engine yet as required by PRD. |
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
| US-7.2 | CSV Export | ✅ | ⚠️ | **Partial** | CSV export is FE client-side; BE export endpoint is still missing. |
| US-8.1 | Hot Lead Email Notification | ⚠️ | ⚠️ | **Partial** | Notification trigger exists, but service is still stub-like / SMTP-config dependent. |
| US-8.2 | Notification Event Logging | ❌ | ❌ | **Missing** | No structured notification event logs yet (sent/failed/retry). |
| US-9.1 | Workspace Settings | ⚠️ | ❌ | **Partial** | Settings UI exists; workspace settings API is not available yet. |
| US-9.2 | Team Management | ⚠️ | ❌ | **Partial** | Team management UI exists; users/workspace management API is missing. |
| US-10.1 | Booking Link Handoff | ⚠️ | ⚠️ | **Partial** | `booking_links` are passed from scenario, but booking token/handoff endpoint flow is still incomplete. |

---

## Most Critical Gaps (Recommended Priority)

1. **US-1.1 + US-2.3 + US-9.2**  
   Build `/v1/workspaces` and `/v1/users` APIs so Settings/Team features can work fully.

2. **US-4.3 + US-4.5 + US-6.4**  
   Implement `/v1/scenarios/{id}/simulate` + deterministic conversation engine + strict KB-only mode enforcement.

3. **US-8.2 + US-8.1 hardening**  
   Add notification event logs (sent, failed, retry) and notification observability.

4. **US-10.1**  
   Complete end-to-end booking handoff (token generation, token consumption endpoint, lead status update).

5. **US-7.2 server-side export**  
   Add BE CSV export endpoint (for larger datasets + safer access control).

---

## Additional Notes

- FE has several service calls not yet supported by BE (`/v1/workspaces`, `/v1/users`, `/v1/scenarios/{id}/simulate`, LLM/upload endpoints).
- Several FE pages are still placeholders (e.g., advanced settings/security/notification preferences).
- Automated tests are currently strongest in RBAC, while other test files (`test_widget.py`, `test_leads.py`, `test_scenarios.py`) are still empty.
