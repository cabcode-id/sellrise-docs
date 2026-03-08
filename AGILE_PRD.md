# Sellrise Platform — Agile Product Requirements Document (PRD)

**Version:** 1.0  
**Date:** March 8, 2026  
**Author:** Product Owner  
**Purpose:** Translate the Feature Breakdown into Sprint-ready Epics and User Stories with BDD acceptance criteria, cross-referenced against repository documentation.

---

## Priority Levels

- 🔴 **MVP Phase 1** — Must-have for first release
- 🟡 **MVP Phase 2** — Important, follows Phase 1
- 🟢 **Post-MVP** — Future enhancements

---

## Global System Rules (Non-Negotiable)

| Rule | Summary | Reference |
|------|---------|-----------|
| **R1** | Workspace isolation — every query enforces `workspace_id`; return 404 (not 403) on cross-workspace access | `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §2 |
| **R2** | Transcript derived from ordered `lead_events` only; no hidden sources | `EVENT_LOGGING_IMPLEMENTATION.md` |
| **R3** | KB-only answers in MVP; no LLM free-form; fallback to contact form | `FEATURE_BREAKDOWN.md` §6.4 |
| **R4** | Lead dedup by normalized email first, then phone; idempotent | `crm-prds/01-lead-creation-and-deduplication.md` |
| **R5** | Deterministic scenario execution — same inputs produce same next step | `STEP_TYPES_IMPLEMENTATION.md` |

---

## Personas

| Role | Description |
|------|-------------|
| **Visitor** | Anonymous user interacting with the embedded widget |
| **Agent** | Operator managing leads in the CRM |
| **Admin** | Configures workspace, scenarios, KB, users, settings |
| **Viewer** | Read-only access for reporting |

---

## Epic Overview

| # | Epic | Priority | Sprint Track |
|---|------|----------|-------------|
| 1 | Core Platform & Multi-Tenancy | 🔴 | Foundation |
| 2 | Authentication & User Management | 🔴 | Foundation |
| 3 | Widget & Embedding | 🔴 | Widget |
| 4 | Conversation Engine | 🔴 | Intelligence |
| 5 | Lead Management — Mini-CRM | 🔴 | CRM |
| 6 | Knowledge Base | 🔴 | Content |
| 7 | Analytics & Reporting | 🔴/🟡 | Insights |
| 8 | Notifications & Jobs | 🔴/🟡 | Ops |
| 9 | Settings & Administration | 🔴/🟡 | Platform |
| 10 | Booking System | 🔴 | Conversion |

---

## Epic 1: Core Platform & Multi-Tenancy

**Business Value:** Foundation for a multi-tenant SaaS platform where each customer operates in a fully isolated workspace, ensuring data security and scalability.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §1 (Core Platform Features)
- `CONVERSATION_SYSTEM_ARCHITECTURE.md` (Workspace Isolation)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC A

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-1.1: Workspace Creation

**Description:** As an Admin, I want to create a workspace so that I can manage my business leads and conversations in an isolated environment.

**Acceptance Criteria (BDD):**

- **Given** an authenticated user with no existing workspace  
  **When** they `POST /v1/workspaces` with `{ "name": "My Clinic" }`  
  **Then** a workspace is created with a unique `workspace_id` (UUID) and the creator is assigned the `ADMIN` role

- **Given** an authenticated user  
  **When** they `POST /v1/workspaces` without a `name` field  
  **Then** the API returns `400 Bad Request` with a validation error message

- **Given** a workspace is created  
  **When** the API returns the response  
  **Then** the response includes `workspace_id`, `name`, `settings` (default JSON), and `created_at`

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/workspaces` → `{ workspace_id, name, settings, created_by }`
- **Tables:** `Workspaces` (workspace_id UUID PK, name, settings JSONB, created_by FK)
- **Rule R1** applies to all subsequent queries
- No upstream dependencies (foundational)

---

#### US-1.2: Workspace Data Isolation

**Description:** As a platform operator, I want every data query to enforce workspace isolation so that tenants never access each other's data.

**Acceptance Criteria (BDD):**

- **Given** a user authenticated in workspace W1  
  **When** they request a resource (lead, scenario, KB article) belonging to workspace W2 by its ID  
  **Then** the API returns `404 Not Found` (not `403 Forbidden`) to prevent data leakage

- **Given** any API endpoint that returns business data  
  **When** a request is processed  
  **Then** the SQL query includes `WHERE workspace_id = :current_workspace_id`

- **Given** a database table containing business data  
  **When** the schema is inspected  
  **Then** the table includes a `workspace_id` foreign key column with a database index

**Technical Notes/Dependencies:**
- Enforced at middleware/ORM level across all endpoints
- Database indexes on `workspace_id` for every business table
- **Rule R1** is the governing constraint
- Ref: `CONVERSATION_SYSTEM_ARCHITECTURE.md` — workspace security enforcement

---

#### US-1.3: Domain Registration

**Description:** As an Admin, I want to register allowed domains so that the widget only loads on authorized websites.

**Acceptance Criteria (BDD):**

- **Given** an Admin authenticated in workspace W1  
  **When** they `POST /v1/domains` with `{ "domain_name": "example.com" }`  
  **Then** the domain is registered for W1 and returned with `domain_id`, `branding_config` (defaults)

- **Given** a registered domain "example.com" for workspace W1  
  **When** the widget attempts to start a session from "malicious-site.com"  
  **Then** the `POST /v1/widget/session` returns `403 Forbidden`

- **Given** an Admin configuring a domain  
  **When** they set `dev_mode_enabled: true`  
  **Then** the widget also loads on `localhost` for development testing

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/domains`, `GET /v1/domains`, `PATCH /v1/domains/{id}`, `DELETE /v1/domains/{id}`
- **Table:** `Domains` (domain_id, workspace_id FK, domain_name, branding_config JSONB, dev_mode_enabled)
- `dev_mode_enabled` is a per-domain boolean flag, set via `PATCH /v1/domains/{id}`
- Depends on: US-1.1 (Workspace Creation)

### Out of Scope
- White-label branding (multi-brand under different domains)
- Custom subdomain provisioning
- Workspace deletion with cascade cleanup (Admin-only in future)

### Open Questions
- Should there be a limit on domains per workspace in MVP?

---

## Epic 2: Authentication & User Management

**Business Value:** Secure, role-based access control ensuring only authorized users can perform actions within their workspace.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §2 (Authentication & User Management)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC A (US-A1, US-A2, US-A3)

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-2.1: User Login (JWT)

**Description:** As a user, I want to log in with email and password so that I receive a secure access token for API requests.

**Acceptance Criteria (BDD):**

- **Given** a registered user with valid credentials  
  **When** they `POST /v1/auth/login` with `{ "email": "admin@example.com", "password": "s3cret" }`  
  **Then** the API returns `200 OK` with `access_token` (JWT, 60 min TTL) and `refresh_token` (7-day TTL)

- **Given** a user with invalid credentials  
  **When** they `POST /v1/auth/login`  
  **Then** the API returns `401 Unauthorized` with a generic error (no credential enumeration)

- **Given** a valid `refresh_token`  
  **When** the user `POST /v1/auth/refresh` with the token  
  **Then** a new `access_token` is issued and the old refresh token is rotated

- **Given** an expired `refresh_token`  
  **When** the user attempts to refresh  
  **Then** the API returns `401 Unauthorized` and the user must log in again

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/auth/login`, `POST /v1/auth/refresh`, `POST /v1/auth/logout`
- **Tables:** `Users` (user_id, email, password_hash bcrypt), `RefreshTokens` (token_hash HMAC-SHA256, user_id, expires_at)
- JWT payload includes `user_id`, `workspace_id`, `role`
- Depends on: US-1.1 (Workspace exists)

---

#### US-2.2: Role-Based Access Control (RBAC)

**Description:** As an Admin, I want to assign roles to users so that each role has appropriate permissions within the workspace.

**Acceptance Criteria (BDD):**

- **Given** a user with role `VIEWER`  
  **When** they attempt to `PATCH /v1/leads/{id}` (modify a lead)  
  **Then** the API returns `403 Forbidden`

- **Given** a user with role `AGENT`  
  **When** they attempt to `POST /v1/scenarios` (create a scenario)  
  **Then** the API returns `403 Forbidden` (scenarios are Admin-only)

- **Given** a user with role `ADMIN`  
  **When** they perform any CRUD operation on scenarios, KB, users, or settings  
  **Then** the API processes the request successfully

- **Given** any authenticated API request  
  **When** the server decodes the JWT  
  **Then** the `role` claim is validated against the endpoint's permission requirements

**Technical Notes/Dependencies:**
- **Roles:** `ADMIN` (full access), `AGENT` (lead management), `VIEWER` (read-only)
- **Table:** `Users` — role (enum: ADMIN, AGENT, VIEWER), email unique per workspace
- Permission matrix enforced at API middleware level
- Depends on: US-2.1 (Authentication)

---

#### US-2.3: User Invitation

**Description:** As an Admin, I want to create and invite users with specific roles so that my team can access the workspace.

**Acceptance Criteria (BDD):**

- **Given** an Admin in workspace W1  
  **When** they `POST /v1/users` with `{ "email": "agent@clinic.com", "role": "AGENT" }`  
  **Then** a new user is created in W1 with the specified role

- **Given** a user already exists with the same email in workspace W1  
  **When** an Admin tries to create another user with the same email  
  **Then** the API returns `409 Conflict`

- **Given** a non-Admin user  
  **When** they attempt to `POST /v1/users`  
  **Then** the API returns `403 Forbidden`

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/users`, `GET /v1/users`, `GET /v1/users/{id}`, `PATCH /v1/users/{id}`, `DELETE /v1/users/{id}`
- Email uniqueness enforced per-workspace (not globally)
- Depends on: US-2.2 (RBAC), US-1.1 (Workspace)

---

#### US-2.4: Logout

**Description:** As a user, I want to log out so that my refresh token is invalidated and the session ends.

**Acceptance Criteria (BDD):**

- **Given** an authenticated user with a valid refresh token  
  **When** they `POST /v1/auth/logout`  
  **Then** the refresh token is invalidated (deleted/marked used) and the API returns `200 OK`

- **Given** a user who has logged out  
  **When** they attempt to use the old refresh token via `POST /v1/auth/refresh`  
  **Then** the API returns `401 Unauthorized`

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/auth/logout`
- Refresh token removed from `RefreshTokens` table
- Access token remains valid until its 60-min TTL expires (stateless JWT)
- Depends on: US-2.1

### Out of Scope
- Password reset flow (🟡 Phase 2 — requires email sending)
- OAuth/SSO integration (Google, Microsoft)
- Multi-factor authentication (MFA)
- Session management UI (active sessions list)

### Open Questions
- Should Admin be able to force-logout another user?
- Is email verification required on user creation in MVP, or deferred to Phase 2?

---

## Epic 3: Widget & Embedding

**Business Value:** A lightweight, embeddable chat widget that loads on customer websites to capture and qualify leads in real-time.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §3 (Widget & Embedding)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC B, §EPIC D (US-D1, US-D5)
- `ai chatbot ui ux.md` (UI/UX specifications)

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-3.1: Widget Embed Code Generation

**Description:** As an Admin, I want to copy a JS embed snippet so that I can install the widget on my website.

**Acceptance Criteria (BDD):**

- **Given** an Admin with a registered domain  
  **When** they navigate to the widget installation page  
  **Then** a ready-to-copy `<script>` snippet is displayed containing the workspace public key

- **Given** the embed snippet is pasted into a website's HTML  
  **When** the page loads  
  **Then** the widget JS bundle loads asynchronously (< 250KB gzipped) without blocking page render

**Technical Notes/Dependencies:**
- Snippet includes workspace public key for session handshake
- Widget bundle built with Vite/Rollup; iframe-safe
- Depends on: US-1.3 (Domain Registration)

---

#### US-3.2: Widget Display Modes

**Description:** As a Visitor, I want the widget to display as a floating bubble or inline element so that it integrates naturally with the website.

**Acceptance Criteria (BDD):**

- **Given** a website with bubble mode configured  
  **When** the page loads  
  **Then** a floating chat bubble appears at the configured position (default: bottom-right)

- **Given** a website with inline mode configured  
  **When** the page loads  
  **Then** the widget renders inside the specified DOM container element

- **Given** a mobile device viewport  
  **When** the widget is displayed  
  **Then** it is responsive and usable on screens ≥ 320px width

**Technical Notes/Dependencies:**
- Configurable: position, primary color, logo, welcome message
- Branding stored in `Domains.branding_config` (JSONB)
- Depends on: US-3.1 (Embed Code), US-1.3 (Domain config)

---

#### US-3.3: Widget Session Handshake

**Description:** As a Visitor, I want the widget to initialize a session automatically so that I can begin a conversation with the bot.

**Acceptance Criteria (BDD):**

- **Given** a Visitor on a registered domain  
  **When** the widget loads and calls `POST /v1/widget/session` with `{ domain, page_url, referrer, utm_*, timezone }`  
  **Then** the API returns `{ session_id, branding, scenario_version_id, scenario_json, booking_links }`

- **Given** the session is created  
  **When** the response is processed  
  **Then** a `widget_opened` event is logged in `LeadEvents`

- **Given** a Visitor on an unregistered domain  
  **When** the widget calls `POST /v1/widget/session`  
  **Then** the API returns `403 Forbidden` and the widget does not render

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/widget/session`
- **Request:** `{ domain, page_url, referrer, utm_source, utm_medium, utm_campaign, timezone }`
- **Response:** `{ session_id, branding, scenario_version_id, scenario_json, booking_links }`
- **Table:** `WidgetSessions` (session_id UUID, workspace_id, domain, page_url, referrer, utm_parameters, timezone, created_at)
- Depends on: US-1.3 (Domain Validation), US-4.2 (Published Scenario)

---

#### US-3.4: Widget Fallback on API Failure

**Description:** As a Visitor, I want a fallback contact form when the widget API is unavailable so that I can still leave my details.

**Acceptance Criteria (BDD):**

- **Given** the widget session handshake fails (network error or 5xx)  
  **When** the widget detects the failure after retry attempts (exponential backoff)  
  **Then** a static fallback contact form is displayed (name, email, phone, message)

- **Given** a Visitor fills out the fallback form  
  **When** they submit it  
  **Then** the data is sent to `POST /v1/widget/fallback-lead` and a lead is created

- **Given** a mid-conversation API timeout  
  **When** 3 retry attempts fail  
  **Then** the widget displays a friendly error message with the fallback form

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/widget/fallback-lead`
- Retry: 3 attempts with exponential backoff
- Errors logged with correlation ID for debugging
- Depends on: US-3.3 (Session), US-5.1 (Lead Creation)

### Out of Scope
- Omnichannel widget (WhatsApp, Instagram, Facebook, Telegram)
- Real-time live chat takeover by operator
- A/B testing widget variants
- Widget analytics (heatmaps, click tracking)

### Open Questions
- Should the fallback form include a CAPTCHA to prevent spam?
- Maximum retry timeout before showing fallback — 10 seconds or configurable?

---

## Epic 4: Conversation Engine

**Business Value:** A deterministic, config-driven conversation engine that qualifies leads through structured scenarios, ensuring consistent and predictable interactions across all channels.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §4 (Conversation Engine)
- `CONVERSATION_SYSTEM_ARCHITECTURE.md` (One-brain-per-workspace, channel-agnostic)
- `STEP_TYPES_SUMMARY.md` (9 step types implementation)
- `STEP_TYPES_IMPLEMENTATION.md` (Detailed backend/frontend specs)
- `SCENARIO_CONFIGURATION_GUIDE.md` (Admin UI, LLM enhancement)
- `DEFAULT_SCENARIO_TEMPLATE.md` (Default template: 6 stages, 11 slots, 4 actions)
- `AI Conversation Engine – Technical Specification/` (Router algorithm, JSON schema)
- `EVENT_LOGGING_IMPLEMENTATION.md` (Event types and logging)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC C, §EPIC D

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-4.1: Scenario CRUD (Draft Editing)

**Description:** As an Admin, I want to create and edit a scenario draft in JSON format so that I can configure the conversation flow for my workspace.

**Acceptance Criteria (BDD):**

- **Given** an Admin authenticated in workspace W1  
  **When** they `POST /v1/scenarios` with `{ "name": "Lead Qualifier", "config": { ... } }`  
  **Then** a scenario is created with `status: "draft"` and the full config is stored

- **Given** an existing draft scenario  
  **When** the Admin `PATCH /v1/scenarios/{id}` with updated config JSON  
  **Then** the draft is updated and `updated_at` is refreshed

- **Given** invalid scenario JSON (e.g., missing required `version` field)  
  **When** the Admin saves the scenario  
  **Then** the API returns `422 Unprocessable Entity` with specific validation errors

- **Given** an Admin editing a scenario  
  **When** they `POST /v1/llm/enhance` with a prompt and the current config  
  **Then** the LLM returns an enhanced version of the config (via OpenRouter / Claude 3.5 Sonnet)

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/scenarios`, `GET /v1/scenarios`, `GET /v1/scenarios/{id}`, `PATCH /v1/scenarios/{id}`
- **LLM Endpoints:** `POST /v1/llm/enhance`, `POST /v1/llm/generate-config`
- **File Endpoints:** `POST /v1/files/upload`, `DELETE /v1/files/{id}` (LLM grounding files)
- **Tables:** `Scenarios` (scenario_id, workspace_id, name, description, config JSONB, status, created_by, updated_by), `Files` (file_id, workspace_id, filename, storage_path, file_type, file_size)
- Admin-only (RBAC enforced)
- Depends on: US-2.2 (RBAC)
- Ref: `SCENARIO_CONFIGURATION_GUIDE.md` for full admin UI spec

---

#### US-4.2: Scenario Publishing

**Description:** As an Admin, I want to publish a scenario so that it becomes the active, immutable version used by the widget.

**Acceptance Criteria (BDD):**

- **Given** a valid draft scenario  
  **When** the Admin calls `POST /v1/scenarios/{id}/publish`  
  **Then** an immutable `ScenarioVersion` is created with a snapshot of the config and `published_at` timestamp

- **Given** a published scenario version  
  **When** the widget initiates a session  
  **Then** the latest published `scenario_json` is returned in the session handshake response

- **Given** the Admin publishes a new version  
  **When** existing widget sessions are active  
  **Then** active sessions continue using their original scenario version (no mid-conversation changes)

- **Given** a scenario with validation errors  
  **When** the Admin attempts to publish  
  **Then** the API returns `422 Unprocessable Entity` and the scenario remains in draft

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/scenarios/{id}/publish`
- **Table:** `ScenarioVersions` (version_id, scenario_id, config JSONB, published_at)
- One active published version per workspace at a time
- **Rule R5** — deterministic execution guaranteed by immutable versions
- Depends on: US-4.1 (Scenario CRUD)

---

#### US-4.3: Step Execution (Conversation Flow)

**Description:** As a Visitor, I want to complete conversation steps in the widget so that the bot qualifies me and routes me to the appropriate next step.

**Acceptance Criteria (BDD):**

- **Given** a Visitor in an active widget session with a published scenario  
  **When** they complete a step (e.g., answer a `question_text` field)  
  **Then** the engine validates the input and returns the next step based on scenario config

- **Given** a Visitor submits an invalid input (e.g., malformed email in a form step)  
  **When** the step processor validates the input  
  **Then** a validation error is returned and the Visitor remains on the current step

- **Given** a `conditional_branch` step  
  **When** the branch condition evaluates against the current session variables  
  **Then** the engine routes to the correct branch deterministically (**Rule R5**)

- **Given** a `kb_search` step  
  **When** the Visitor enters a search query  
  **Then** the engine queries the KB API and returns results from articles/FAQs only (**Rule R3**)

- **Given** any step completion  
  **When** the answer is processed  
  **Then** the answer is stored in `conversation.context.slots` and a `step_completed` event is logged

**Technical Notes/Dependencies:**
- **Step Types (9):** `message`, `question_text`, `question_choice`, `form`, `conditional_branch`, `kb_search`, `handoff_booking_link`, `handoff_operator`, `end`
- **Backend:** Requires a `StepProcessor` service to execute all step types, validate inputs, and manage state
- **Frontend:** Requires a universal `StepRenderer` component that renders all 9 step types with validation
- **Endpoints:** `POST /v1/steps/init`, `POST /v1/steps/execute`, `POST /v1/steps/process`
- Ref: `STEP_TYPES_IMPLEMENTATION.md` for complete implementation details
- Depends on: US-4.2 (Published Scenario), US-3.3 (Widget Session)

---

#### US-4.4: Event Logging

**Description:** As a platform operator, I want every widget interaction logged as an immutable event so that I can reconstruct the full conversation transcript.

**Acceptance Criteria (BDD):**

- **Given** a Visitor opens the widget  
  **When** the session is created  
  **Then** a `widget_opened` event is logged with `{ domain, page_url, utm_* }`

- **Given** a Visitor submits the lead contact form  
  **When** the lead is created/upserted  
  **Then** a `lead_submitted` event is logged with `{ name, email, phone }`

- **Given** an Agent changes a lead's stage from "new" to "qualified"  
  **When** the `PATCH /v1/leads/{id}` request is processed  
  **Then** a `stage_changed` event is logged with `{ old_stage: "new", new_stage: "qualified" }`

- **Given** events exist for a lead  
  **When** the transcript is rendered in the CRM  
  **Then** events are ordered by `created_at` ascending (**Rule R2**)

- **Given** an event is created  
  **When** any attempt is made to modify or delete it  
  **Then** the operation is rejected (events are append-only and immutable)

**Technical Notes/Dependencies:**
- **Event Types:** `widget_opened`, `chat_started`, `step_completed`, `lead_submitted`, `booking_link_shown`, `booking_link_clicked`, `stage_changed`, `owner_assigned`, `note_added`
- **Endpoint:** `POST /v1/widget/event` — `{ session_id, event_type, event_data, ts_client }`
- **Table:** `LeadEvents` (event_id, workspace_id, lead_id nullable, session_id, event_type, event_data JSONB, created_at)
- **Rule R2** governs transcript rendering
- Ref: `EVENT_LOGGING_IMPLEMENTATION.md` for auto-logging points
- Depends on: US-3.3 (Widget Session)

---

#### US-4.5: Centralized Conversation System

**Description:** As a platform operator, I want one AI "brain" per workspace processing all conversations so that the AI persona is consistent across channels.

**Acceptance Criteria (BDD):**

- **Given** a workspace with a published scenario  
  **When** a message arrives from any channel (web, WhatsApp, Telegram)  
  **Then** the same scenario processes the message with consistent behavior

- **Given** a Visitor sends a message via `POST /v1/widget/message`  
  **When** the message is processed  
  **Then** a `Message` record is created with `role: "user"` and a bot response with `role: "assistant"` is generated and stored

- **Given** an active conversation  
  **When** the context is inspected  
  **Then** it contains full state: current stage, current task, filled slots, and variables (JSONB)

- **Given** two leads in the same workspace on different channels  
  **When** both send messages simultaneously  
  **Then** each conversation maintains its own isolated context (no context mixing)

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/widget/message`, `GET /v1/conversations`, `GET /v1/conversations/{id}`, `PATCH /v1/conversations/{id}`, `GET /v1/conversations/{id}/messages`
- **Tables:** `Conversations` (conversation_id, workspace_id, lead_id, scenario_id, channel, status, current_stage_id, context JSONB, last_message_at), `Messages` (message_id, conversation_id, role, content, metadata JSONB, is_delivered, is_read, created_at)
- Channels: `web`, `whatsapp`, `telegram`, `instagram`, `messenger`, `email`
- Requires database migration for `Conversations` and `Messages` tables
- Ref: `CONVERSATION_SYSTEM_ARCHITECTURE.md` for full schema and API design
- Depends on: US-4.2 (Published Scenario), US-5.1 (Lead Creation)

### Out of Scope
- LLM-powered AI Conversation Engine with stage/task router (🟢 Post-MVP, §4.6)
- Visual Bot Builder / Flow Editor (🟢 Post-MVP, §7)
- A/B testing of scenarios
- Scenario templates library (🟢 Post-MVP, §7.5)
- `handoff_operator` step type (live chat takeover — 🟢 Post-MVP)

### Open Questions
- Should the scenario simulator (§4.2) be accessible to Agents, or Admin-only?
- How should the system handle a published scenario being unpublished while active sessions exist?

---

## Epic 5: Lead Management — Mini-CRM

**Business Value:** Lightweight CRM enabling operators to manage, qualify, track, and follow up on leads captured through the widget, increasing conversion rates.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §5 (Lead Management Mini-CRM)
- `crm-prds/01-lead-creation-and-deduplication.md`
- `crm-prds/02-lead-inbox-new-unassigned.md`
- `crm-prds/03-pipeline-kanban-view.md`
- `crm-prds/04-lead-detail-view.md`
- `crm-prds/05-lead-search-and-filters.md`
- `crm-prds/06-lead-notes.md`
- `crm-prds/07-lead-assignment.md`
- `crm-prds/08-lead-scoring.md`
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC E, §EPIC F

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-5.1: Lead Creation & Deduplication

**Description:** As a Visitor, I want to submit my contact details through the widget so that a lead is created (or updated if I already exist) without duplication.

**Acceptance Criteria (BDD):**

- **Given** a Visitor submits contact details `{ name, email, phone, consent: true }`  
  **When** `POST /v1/widget/lead` is called  
  **Then** the system normalizes the email (lowercase, trim) and phone (E.164), checks for duplicates, and creates or upserts the lead

- **Given** an existing lead with `dedup_email = "john@example.com"` in workspace W1  
  **When** a new submission arrives with email `"John@Example.com "`  
  **Then** the existing lead is updated (upserted) — no duplicate is created

- **Given** a submission with no matching email but matching `dedup_phone`  
  **When** the dedup check runs  
  **Then** the lead is matched by phone and updated

- **Given** a submission with no matching email or phone  
  **When** the dedup check runs  
  **Then** a new lead is created with auto-calculated score (HOT/WARM/COLD)

- **Given** any lead submission  
  **When** the lead is created or updated  
  **Then** a `lead_submitted` event is logged, consent is recorded with timestamp, and the API returns `{ lead_id, score, stage }`

**Technical Notes/Dependencies:**
- **Endpoint:** `POST /v1/widget/lead` — `{ session_id, name, email, phone, consent, qualification_data }`
- **Table:** `Leads` (lead_id UUID, workspace_id, session_id, name, email, phone, dedup_email unique/workspace, dedup_phone, consent, consent_timestamp, procedure, budget, timeframe, location, source_data JSONB, score enum, stage enum, owner_id, tags array, created_at, updated_at)
- **Rule R4** governs deduplication logic
- Ref: `crm-prds/01-lead-creation-and-deduplication.md`
- Depends on: US-3.3 (Widget Session), US-4.4 (Event Logging)

---

#### US-5.2: Lead Inbox (New/Unassigned)

**Description:** As an Agent, I want to see a list of new and unassigned leads so that I can quickly pick up leads that need attention.

**Acceptance Criteria (BDD):**

- **Given** an Agent authenticated in workspace W1  
  **When** they navigate to the Inbox view  
  **Then** leads are displayed where `stage = 'new'` OR `owner_id IS NULL`, sorted by `created_at` DESC

- **Given** leads in the inbox  
  **When** the list is rendered  
  **Then** each card shows: created date, name, email, phone, score badge (Hot/Warm/Cold), procedure, source domain

- **Given** an Agent clicks on a lead card  
  **When** the click is processed  
  **Then** the Lead Detail View (US-5.4) opens

**Technical Notes/Dependencies:**
- **Endpoint:** `GET /v1/leads?stage=new&owner_id=null&sort=-created_at`
- Uses `Leads` table with filters
- Ref: `crm-prds/02-lead-inbox-new-unassigned.md`
- Depends on: US-5.1 (Lead Creation)

---

#### US-5.3: Pipeline Kanban View

**Description:** As an Agent, I want to view leads in a Kanban board organized by stage so that I can manage my pipeline visually.

**Acceptance Criteria (BDD):**

- **Given** an Agent opens the Pipeline view  
  **When** leads are loaded  
  **Then** leads are displayed in columns: NEW → QUALIFIED → CONTACTED → BOOKED → WON → LOST

- **Given** an Agent drags a lead card from "NEW" to "QUALIFIED"  
  **When** the drop completes  
  **Then** `PATCH /v1/leads/{id}` is called with `{ "stage": "qualified" }` and a `stage_changed` event is logged

- **Given** a user with role `VIEWER`  
  **When** they attempt to drag a lead to another column  
  **Then** the drag is disabled (read-only access)

- **Given** a mobile device  
  **When** the Pipeline is displayed  
  **Then** a dropdown stage selector replaces drag-and-drop

**Technical Notes/Dependencies:**
- **Endpoint:** `PATCH /v1/leads/{id}` — `{ stage }` triggers `stage_changed` event
- **Stages:** `new`, `qualified`, `contacted`, `booked`, `won`, `lost`
- RBAC: `VIEWER` cannot modify; `AGENT` and `ADMIN` can
- Ref: `crm-prds/03-pipeline-kanban-view.md`
- Depends on: US-5.1 (Lead Creation), US-2.2 (RBAC)

---

#### US-5.4: Lead Detail View

**Description:** As an Agent, I want to view a lead's full details — contact info, qualification, transcript, notes, and tags — so that I have complete context before reaching out.

**Acceptance Criteria (BDD):**

- **Given** an Agent opens a lead's detail view  
  **When** `GET /v1/leads/{id}` returns the data  
  **Then** the view shows sections: Contact Card, Qualification, Pipeline stage, Source/UTM, Transcript (from events), Notes, Tags

- **Given** a lead with logged events  
  **When** the Transcript section renders  
  **Then** events are displayed chronologically (ordered by `created_at`) as a timeline (**Rule R2**)

- **Given** an Agent viewing a lead  
  **When** they assign themselves as owner via the UI  
  **Then** `PATCH /v1/leads/{id}` with `{ "owner_id": "<user_id>" }` is called and an `owner_assigned` event is logged

**Technical Notes/Dependencies:**
- **Endpoints:** `GET /v1/leads/{id}`, `PATCH /v1/leads/{id}`, `POST /v1/leads/{id}/notes`, `PATCH /v1/leads/{id}/tags`
- Combines data from `Leads`, `LeadEvents`, `LeadNotes` tables
- Ref: `crm-prds/04-lead-detail-view.md`
- Depends on: US-5.1, US-5.6 (Notes), US-5.7 (Assignment), US-4.4 (Events)

---

#### US-5.5: Lead Search & Filters

**Description:** As an Agent, I want to search and filter leads by multiple criteria so that I can quickly find specific leads.

**Acceptance Criteria (BDD):**

- **Given** an Agent on the leads list  
  **When** they apply filters `{ stage: "qualified", score: "hot", owner: "<user_id>" }`  
  **Then** only leads matching ALL criteria (AND logic) are returned

- **Given** an Agent types "john" in the search box  
  **When** the search executes  
  **Then** leads matching name, email, or phone (case-insensitive) are returned

- **Given** a large dataset (thousands of leads)  
  **When** `GET /v1/leads?limit=20&offset=0` is called  
  **Then** results are paginated and the response includes total count

- **Given** filters are applied  
  **When** the Agent changes the date range  
  **Then** the results update to include only leads created within the specified range

**Technical Notes/Dependencies:**
- **Endpoint:** `GET /v1/leads?stage=&score=&owner=&search=&from=&to=&limit=20&offset=0`
- Database indexes on `stage`, `score`, `owner_id`, `created_at` for performance
- Ref: `crm-prds/05-lead-search-and-filters.md`
- Depends on: US-5.1 (Lead Creation)

---

#### US-5.6: Lead Notes

**Description:** As an Agent, I want to add notes to a lead so that I can document follow-up actions and observations.

**Acceptance Criteria (BDD):**

- **Given** an Agent viewing a lead's detail page  
  **When** they `POST /v1/leads/{id}/notes` with `{ "content": "Called, no answer. Will retry tomorrow." }`  
  **Then** the note is saved with `author_id`, `created_at`, and a `note_added` event is logged

- **Given** a lead with multiple notes  
  **When** `GET /v1/leads/{id}/notes` is called  
  **Then** notes are returned chronologically (newest last) with author name and timestamp

- **Given** an Agent who authored a note  
  **When** they `DELETE /v1/notes/{note_id}`  
  **Then** the note is deleted (only own notes, optional in MVP)

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/leads/{id}/notes`, `GET /v1/leads/{id}/notes`, `DELETE /v1/notes/{note_id}`
- **Table:** `LeadNotes` (note_id UUID, lead_id FK, workspace_id FK, author_id FK, content TEXT, created_at, updated_at)
- Ref: `crm-prds/06-lead-notes.md`
- Depends on: US-5.4 (Lead Detail), US-2.2 (RBAC)

---

#### US-5.7: Lead Assignment

**Description:** As an Agent, I want to assign a lead to myself or another team member so that ownership and responsibility are clear.

**Acceptance Criteria (BDD):**

- **Given** an unassigned lead  
  **When** an Agent `PATCH /v1/leads/{id}` with `{ "owner_id": "<user_id>" }`  
  **Then** the lead's `owner_id` is updated and an `owner_assigned` event is logged with `{ old_owner_id: null, new_owner_id: "<user_id>" }`

- **Given** a lead assigned to Agent A  
  **When** Agent B reassigns it to themselves  
  **Then** the `owner_assigned` event logs both old and new owner IDs

- **Given** an assigned lead  
  **When** an Agent sets `{ "owner_id": null }`  
  **Then** the lead returns to the unassigned inbox

**Technical Notes/Dependencies:**
- **Endpoint:** `PATCH /v1/leads/{id}` — `{ owner_id }`
- `owner_id` references `Users` table
- Ref: `crm-prds/07-lead-assignment.md`
- Depends on: US-5.1, US-2.3 (Users exist)

---

#### US-5.8: Lead Scoring

**Description:** As a platform operator, I want leads to be automatically scored (HOT/WARM/COLD) upon creation so that Agents can prioritize high-value leads.

**Acceptance Criteria (BDD):**

- **Given** a lead submission with high budget + urgent timeframe + clear need  
  **When** the scoring logic runs  
  **Then** the lead is scored as `HOT`

- **Given** a lead submission with medium budget or longer timeframe  
  **When** the scoring logic runs  
  **Then** the lead is scored as `WARM`

- **Given** a lead submission with low budget or no clear timeline  
  **When** the scoring logic runs  
  **Then** the lead is scored as `COLD`

- **Given** an existing lead whose qualification data is updated  
  **When** `PATCH /v1/leads/{id}` includes new `budget` or `timeframe`  
  **Then** the score is recalculated and a `score_assigned` event is logged

**Technical Notes/Dependencies:**
- Scoring runs automatically on `POST /v1/widget/lead` and `PATCH /v1/leads/{id}`
- Score stored in `Leads.score` (enum: HOT, WARM, COLD)
- Configurable scoring rules deferred to Post-MVP
- Ref: `crm-prds/08-lead-scoring.md`
- Depends on: US-5.1 (Lead Creation)

### Out of Scope
- Lead merge UI (manual merge of duplicates)
- Lead import/export (bulk CSV import)
- Lead activity automation (auto-assign rules, round-robin)
- CRM integration push (HubSpot, Salesforce — 🟢 Post-MVP, §10.2)
- Send follow-up email/SMS from lead detail

### Open Questions
- Should scoring use conditional logic (if/else rules) or a rules engine (weighted scoring)?
- Are pipeline stages editable per workspace, or fixed to the 6 predefined stages?
- Should `VIEWER` role see lead email/phone, or are those masked for privacy?

---

## Epic 6: Knowledge Base

**Business Value:** A self-serve knowledge base that powers the widget's FAQ/search step, answering visitor questions from curated content without LLM hallucination.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §6 (Knowledge Base)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC G

**Priority:** 🔴 MVP Phase 1

### User Stories

#### US-6.1: KB Articles CRUD

**Description:** As an Admin, I want to create, edit, and delete knowledge base articles so that the widget can provide accurate answers to visitor questions.

**Acceptance Criteria (BDD):**

- **Given** an Admin in workspace W1  
  **When** they `POST /v1/kb/articles` with `{ "title": "Pricing", "body": "## Plans\n...", "category": "General", "tags": ["pricing"] }`  
  **Then** the article is created with `article_id`, `created_by`, `created_at`

- **Given** an existing article  
  **When** the Admin `PATCH /v1/kb/articles/{id}` with updated body  
  **Then** the article is updated, `updated_at` and `updated_by` are set

- **Given** an Agent (non-Admin) user  
  **When** they attempt to `POST /v1/kb/articles`  
  **Then** the API returns `403 Forbidden` (Admin-only CRUD)

- **Given** articles exist in a workspace  
  **When** `GET /v1/kb/articles` is called  
  **Then** all articles for the workspace are returned (workspace-scoped)

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/kb/articles`, `GET /v1/kb/articles`, `GET /v1/kb/articles/{id}`, `PATCH /v1/kb/articles/{id}`, `DELETE /v1/kb/articles/{id}`
- **Table:** `KBArticles` (article_id UUID, workspace_id FK, title, body TEXT/Markdown, category, tags ARRAY, language default 'EN', created_by FK, updated_by FK, created_at, updated_at)
- Depends on: US-2.2 (RBAC, Admin-only)

---

#### US-6.2: KB FAQ CRUD

**Description:** As an Admin, I want to manage FAQ entries (question-answer pairs) so that common visitor questions are answered instantly.

**Acceptance Criteria (BDD):**

- **Given** an Admin  
  **When** they `POST /v1/kb/faq` with `{ "question": "What are your hours?", "answer": "Mon-Fri 9-5", "category": "General" }`  
  **Then** the FAQ entry is created with a unique `faq_id`

- **Given** a FAQ entry exists  
  **When** the Admin `DELETE /v1/kb/faq/{id}`  
  **Then** the FAQ is removed and no longer appears in search results

- **Given** FAQ entries exist  
  **When** `GET /v1/kb/faq` is called  
  **Then** all FAQs for the workspace are returned with question, answer, category, and tags

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/kb/faq`, `GET /v1/kb/faq`, `GET /v1/kb/faq/{id}`, `PATCH /v1/kb/faq/{id}`, `DELETE /v1/kb/faq/{id}`
- **Table:** `KBFAQs` (faq_id UUID, workspace_id FK, question, answer, category, tags ARRAY, language, created_by FK, updated_by FK, created_at, updated_at)
- Depends on: US-2.2 (RBAC, Admin-only)

---

#### US-6.3: KB Full-Text Search

**Description:** As a Visitor, I want to search the knowledge base from the widget so that I get relevant answers to my questions.

**Acceptance Criteria (BDD):**

- **Given** a Visitor in an active widget session at a `kb_search` step  
  **When** they enter a search query (e.g., "cargo bike price")  
  **Then** `GET /v1/widget/kb/search?q=cargo+bike+price&limit=5` returns matching articles and FAQs ranked by relevance

- **Given** search results are returned  
  **When** the widget renders them  
  **Then** each result shows `type` (article/faq), `title`/`question`, and a text snippet/preview

- **Given** no search results match the query  
  **When** the API returns an empty array  
  **Then** the widget displays a fallback message: "No results found. Please leave your contact details." (**Rule R3**)

- **Given** search is executed  
  **When** the Postgres query runs  
  **Then** it uses `ts_vector` and `ts_rank` for full-text ranking

**Technical Notes/Dependencies:**
- **Endpoint:** `GET /v1/widget/kb/search?q=&limit=5`
- **Response:** `[{ type, id, title/question, snippet/answer_preview }]`
- PostgreSQL full-text search with `ts_vector` indexes on `articles.body` and `faqs.question`
- **Rule R3** — KB-only answers, no LLM generation
- Depends on: US-6.1 (Articles), US-6.2 (FAQs)

---

#### US-6.4: KB-Only Answer Mode

**Description:** As a platform operator, I want the widget to only surface verified KB content so that visitors never receive hallucinated or unverified answers.

**Acceptance Criteria (BDD):**

- **Given** the `kb_search` step type is configured in a scenario  
  **When** the step executes  
  **Then** it queries only the KB Search API — no LLM-generated answers are included

- **Given** a KB search returns zero results  
  **When** the fallback is triggered  
  **Then** the widget offers a contact form to capture the visitor's details

- **Given** any widget response in MVP  
  **When** the response content is inspected  
  **Then** all answers trace back to a KB article or FAQ entry (auditable)

**Technical Notes/Dependencies:**
- Enforced at step processor level
- No LLM free-form answers in MVP scope
- **Rule R3** is the governing constraint
- Depends on: US-6.3 (KB Search), US-4.3 (Step Execution)

### Out of Scope
- LLM-enhanced KB answers (summarization, rephrasing — 🟢 Post-MVP)
- KB versioning and change history
- Multi-language KB (beyond default language field)
- KB analytics (most-searched queries, unanswered queries)
- Rich media in articles (images, videos)

### Open Questions
- Should KB search support fuzzy matching or synonyms in MVP?
- Is there a minimum number of articles/FAQs required before the `kb_search` step activates?
- Should the KB search endpoint be rate-limited for public widget access?

---

## Epic 7: Analytics & Reporting

**Business Value:** Data-driven insights into funnel conversion and lead quality, enabling Admins to optimize scenarios and Agents to focus on high-value leads.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §8 (Analytics & Reporting)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC I

**Priority:** 🔴 MVP Phase 1 (Funnel + CSV), 🟡 Phase 2 (Source analytics, drop-off)

### User Stories

#### US-7.1: Funnel Metrics Dashboard

**Description:** As an Admin, I want to see funnel metrics (widget opened → booking clicked) so that I can measure conversion rates and identify drop-off points.

**Acceptance Criteria (BDD):**

- **Given** an Admin on the Analytics page  
  **When** they request funnel data with `GET /v1/analytics/funnel?from=2026-03-01&to=2026-03-31`  
  **Then** the API returns counts: `{ widget_opened, chat_started, lead_submitted, qualified_hot, booking_link_clicked }`

- **Given** funnel data is returned  
  **When** the dashboard renders  
  **Then** conversion rates between each stage are calculated and displayed (e.g., "30% of visitors submitted a lead")

- **Given** no date range is specified  
  **When** the dashboard loads  
  **Then** default to the last 30 days

- **Given** the funnel data query  
  **When** it executes  
  **Then** it is workspace-scoped (only counts events for the current workspace)

**Technical Notes/Dependencies:**
- **Endpoint:** `GET /v1/analytics/funnel?from=&to=`
- Queries `LeadEvents` table, counts by `event_type`
- Conversion rates calculated in the frontend
- Depends on: US-4.4 (Event Logging)

---

#### US-7.2: CSV Export

**Description:** As an Admin, I want to export leads as a CSV file so that I can analyze data externally or share with stakeholders.

**Acceptance Criteria (BDD):**

- **Given** an Admin with leads in their workspace  
  **When** they call `GET /v1/leads/export?format=csv&from=2026-03-01&to=2026-03-31`  
  **Then** a CSV file is downloaded containing: created_at, name, email, phone, procedure, timeframe, budget, score, stage, owner, utm_source, utm_medium, utm_campaign, referrer, page_url

- **Given** filters are active (e.g., score=hot)  
  **When** the export is triggered  
  **Then** the CSV contains only leads matching the active filters

- **Given** the export runs  
  **When** the CSV is generated  
  **Then** it is workspace-scoped (only includes leads from the current workspace)

**Technical Notes/Dependencies:**
- **Endpoint:** `GET /v1/leads/export?format=csv&from=&to=&stage=&score=`
- Uses `Leads` table with same filter logic as US-5.5
- Depends on: US-5.5 (Lead Search & Filters)

### Out of Scope
- Lead source analytics breakdown (🟡 Phase 2, §8.2)
- Stage drop-off analysis by step (🟡 Phase 2, §8.3)
- Real-time dashboards / WebSocket updates
- Custom report builder
- Scheduled email reports

### Open Questions
- Should the CSV export have a row limit for performance (e.g., max 10,000 rows)?
- Is the funnel chart a bar chart, funnel shape, or configurable?

---

## Epic 8: Notifications & Jobs

**Business Value:** Timely email alerts ensure hot leads are followed up immediately, reducing response time and increasing conversion.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §9 (Notifications & Jobs)
- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md` §EPIC H
- `EVENT_LOGGING_IMPLEMENTATION.md` (notification events)

**Priority:** 🔴 MVP Phase 1 (Hot lead email), 🟡 Phase 2 (Assignment notifications)

### User Stories

#### US-8.1: Hot Lead Email Notification

**Description:** As an Admin/Agent, I want to receive an email notification when a hot lead is submitted so that I can follow up immediately.

**Acceptance Criteria (BDD):**

- **Given** a lead is submitted with `score = HOT`  
  **When** the lead creation completes  
  **Then** an email notification is queued and sent to the configured recipients (workspace setting)

- **Given** the email content  
  **When** it is rendered  
  **Then** it includes: lead name, email, phone, procedure, timeframe, budget, source page_url, and a direct link to the lead detail in the CRM

- **Given** the email send fails  
  **When** the notification job retries  
  **Then** it retries up to 3 times with exponential backoff and logs `notification_failed` event on final failure

- **Given** the email is sent successfully  
  **When** the job completes  
  **Then** a `notification_sent` event is logged for the lead

**Technical Notes/Dependencies:**
- **Background Job:** Redis + worker queue (Bull/Celery)
- **Table:** `NotificationJobs` (job_id, workspace_id, lead_id, recipient_email, status, attempt_count, last_attempt_at, next_retry_at)
- Recipients configured via workspace settings (`PATCH /v1/workspaces/{id}/settings`)
- Depends on: US-5.1 (Lead Creation), US-5.8 (Lead Scoring), US-9.1 (Workspace Settings)

---

#### US-8.2: Notification Event Logging

**Description:** As a platform operator, I want notification delivery status logged as events so that I can audit notification reliability.

**Acceptance Criteria (BDD):**

- **Given** a notification email is sent successfully  
  **When** the job completes  
  **Then** a `notification_sent` event is appended to the lead's event log

- **Given** a notification fails after all retry attempts  
  **When** the final attempt fails  
  **Then** a `notification_failed` event is appended with error details

**Technical Notes/Dependencies:**
- Event types: `notification_sent`, `notification_failed`
- Appended to `LeadEvents` table (immutable, **Rule R2**)
- Depends on: US-8.1, US-4.4 (Event Logging)

### Out of Scope
- Lead assignment notification email (🟡 Phase 2, §9.2)
- SLA reminder notifications (🟢 Post-MVP, §9.3)
- SMS/Slack/webhook notification channels
- Notification preferences per user
- Digest emails (daily/weekly summary)

### Open Questions
- Should notification recipients be per-workspace or per-user preference?
- What email service provider will be used (SendGrid, SES, Resend)?
- Should the notification include a "claim this lead" button for agents?

---

## Epic 9: Settings & Administration

**Business Value:** Central configuration hub allowing Admins to customize their workspace — branding, team, notifications, and booking links — without developer intervention.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §11 (Settings & Administration)

**Priority:** 🔴 MVP Phase 1 (Workspace settings, Team), 🟡 Phase 2 (Branding, Notification config)

### User Stories

#### US-9.1: Workspace Settings

**Description:** As an Admin, I want to configure workspace-level settings so that the platform behavior matches my business requirements.

**Acceptance Criteria (BDD):**

- **Given** an Admin in workspace W1  
  **When** they `PATCH /v1/workspaces/{id}/settings` with `{ "timezone": "Asia/Jakarta", "notification_emails": ["ops@clinic.com"], "booking_links": { "default": "https://calendly.com/clinic" } }`  
  **Then** the settings are saved in the workspace's `settings` (JSONB) column

- **Given** workspace settings include a `booking_links` dictionary  
  **When** the widget encounters a `handoff_booking_link` step  
  **Then** the `booking_link_key` resolves to the correct URL from the settings

- **Given** a non-Admin user  
  **When** they attempt to `PATCH /v1/workspaces/{id}/settings`  
  **Then** the API returns `403 Forbidden`

**Technical Notes/Dependencies:**
- **Endpoint:** `PATCH /v1/workspaces/{id}/settings`
- **Settings fields:** workspace name, branding (logo, colors), timezone, language, notification_emails, booking_links
- Stored in `Workspaces.settings` (JSONB)
- Depends on: US-1.1 (Workspace), US-2.2 (RBAC)

---

#### US-9.2: Team Management

**Description:** As an Admin, I want to manage team members — invite, assign roles, and deactivate — so that the right people have the right access.

**Acceptance Criteria (BDD):**

- **Given** an Admin in workspace W1  
  **When** they `POST /v1/users` with `{ "email": "new-agent@clinic.com", "role": "AGENT" }`  
  **Then** a new user is created in the workspace (same as US-2.3)

- **Given** an Admin viewing the team list  
  **When** they `PATCH /v1/users/{id}` with `{ "role": "VIEWER" }`  
  **Then** the user's role is updated and their permissions change immediately

- **Given** an Admin  
  **When** they `DELETE /v1/users/{id}` (deactivate)  
  **Then** the user is deactivated and can no longer log in; their assigned leads remain but `owner_id` is preserved for history

**Technical Notes/Dependencies:**
- **Endpoints:** `POST /v1/users`, `GET /v1/users`, `PATCH /v1/users/{id}`, `DELETE /v1/users/{id}`
- Uses `Users` table
- Depends on: US-2.2 (RBAC), US-2.3 (User Invitation)

### Out of Scope
- Per-domain branding customization (🟡 Phase 2, §11.3)
- Notification channel configuration (email/SMS/Slack — 🟡 Phase 2, §11.4)
- Audit log for settings changes
- Workspace billing and subscription management

### Open Questions
- Should deactivated users be soft-deleted or hard-deleted?
- Can an Admin change their own role (e.g., downgrade to Agent)?
- Are workspace settings changes versioned for rollback?

---

## Epic 10: Booking System

**Business Value:** Convert qualified leads into booked appointments by presenting a clear call-to-action within the conversation flow.

**Repo Docs Reference:**
- `FEATURE_BREAKDOWN.md` §12 (Booking System)
- `STEP_TYPES_SUMMARY.md` (`handoff_booking_link` step type)
- `STEP_TYPES_IMPLEMENTATION.md` (Step type configuration)

**Priority:** 🔴 MVP Phase 1 (Link-based only)

### User Stories

#### US-10.1: Booking Link Handoff

**Description:** As a Visitor, I want to see a booking button during the conversation so that I can schedule an appointment after being qualified.

**Acceptance Criteria (BDD):**

- **Given** the conversation reaches a `handoff_booking_link` step  
  **When** the step is rendered in the widget  
  **Then** the Visitor sees a configurable text message and a CTA button with a configurable label

- **Given** the step config `{ "type": "handoff_booking_link", "booking_link_key": "default_booking_link" }`  
  **When** the URL is resolved  
  **Then** it looks up `booking_link_key` from workspace settings' `booking_links` dictionary

- **Given** the booking CTA is displayed  
  **When** the widget renders it  
  **Then** a `booking_link_shown` event is logged

- **Given** the Visitor clicks the booking button  
  **When** the click is processed  
  **Then** the external booking URL opens in a new tab and a `booking_link_clicked` event is logged

- **Given** the `booking_link_key` is missing from workspace settings  
  **When** the step attempts to resolve the URL  
  **Then** a fallback message is displayed (e.g., "Please contact us to schedule an appointment.")

**Technical Notes/Dependencies:**
- Step config example: `{ "type": "handoff_booking_link", "text": "Ready to schedule?", "button_label": "Book Now", "booking_link_key": "default_booking_link" }`
- Booking URLs stored in `Workspaces.settings.booking_links` (JSONB)
- Events: `booking_link_shown`, `booking_link_clicked`
- Ref: `STEP_TYPES_SUMMARY.md` for step type behavior
- Depends on: US-4.3 (Step Execution), US-9.1 (Workspace Settings), US-4.4 (Event Logging)

### Out of Scope
- Native booking calendar with availability slots (🟢 Post-MVP, §12.2)
- Calendar integration (Google Calendar, Outlook — 🟢 Post-MVP, §10.1)
- ICS file generation
- Slot locking mechanism
- Booking confirmation emails
- Reschedule/cancel via token links

### Open Questions
- Should multiple booking links be supported (e.g., per procedure type)?
- Should the booking link open in a new tab, iframe, or in-widget modal?

---

## Cross-Epic Dependency Map

### Foundation Chain (must be built first)
```
Epic 1: US-1.1 (Workspace) → US-1.2 (Isolation) → US-1.3 (Domains)
    ↓
Epic 2: US-2.1 (Auth/JWT) → US-2.2 (RBAC) → US-2.3 (Users)
```

### Parallel Sprint Tracks

```
Track A — Widget & Conversations:
  US-1.3 → US-3.1 → US-3.2 → US-3.3 → US-3.4
                                  ↓
  US-4.1 → US-4.2 → US-4.3 → US-4.4 → US-4.5

Track B — CRM:
  US-5.1 → US-5.2
         → US-5.3
         → US-5.4 → US-5.6 → US-5.7
         → US-5.5
         → US-5.8

Track C — Content:
  US-6.1 → US-6.3 → US-6.4
  US-6.2 ↗

Track D — Insights & Ops:
  US-7.1 (depends on US-4.4)
  US-7.2 (depends on US-5.5)
  US-8.1 → US-8.2 (depends on US-5.8)

Track E — Configuration:
  US-9.1 (depends on US-1.1)
  US-9.2 (depends on US-2.2)
  US-10.1 (depends on US-4.3, US-9.1)
```

### Suggested Sprint Sequence

| Sprint | Focus | Key Deliverables |
|--------|-------|-----------------|
| **Sprint 1** | Foundation | Epic 1 (Workspace, Isolation, Domains) + Epic 2 (Auth, RBAC, Users) |
| **Sprint 2** | Widget + Engine | Epic 3 (Widget embed, session, fallback) + Epic 4 (Scenario CRUD, publishing, step execution) |
| **Sprint 3** | CRM + Content | Epic 5 (Lead CRUD, inbox, pipeline, detail, search, notes, assignment, scoring) + Epic 6 (KB articles, FAQ, search) |
| **Sprint 4** | Insights + Ops + Booking | Epic 7 (Funnel, CSV) + Epic 8 (Notifications) + Epic 9 (Settings, Team) + Epic 10 (Booking link) |

---

## Global Out of Scope (All Epics)

The following modules are explicitly excluded from this PRD iteration:

| Module | Section | Priority | Reason |
|--------|---------|----------|--------|
| Scenario/Bot Builder (Visual) | §7 | 🟢 Post-MVP | JSON editor sufficient for MVP |
| Integrations (Calendar, CRM, Channels) | §10 | 🟢 Post-MVP | Link-based booking and manual CRM in MVP |
| AI Conversation Engine (LLM-powered) | §4.6 | 🟢 Post-MVP | Deterministic step engine for MVP; LLM-driven conversation later |
| Omnichannel Messaging | §10.3 | 🟢 Post-MVP | Web widget only in MVP |
| Password Reset | §2.3 | 🟡 Phase 2 | Deferred to Phase 2 |
| Branding Configuration | §11.3 | 🟡 Phase 2 | Default branding in MVP |
| Lead Source Analytics | §8.2 | 🟡 Phase 2 | Basic funnel sufficient for MVP |
| Stage Drop-Off Analysis | §8.3 | 🟡 Phase 2 | Basic funnel sufficient for MVP |
| Assignment Notification | §9.2 | 🟡 Phase 2 | Hot lead notification only in MVP |
| SLA Reminders | §9.3 | 🟢 Post-MVP | Deferred |
| Native Booking Calendar | §12.2 | 🟢 Post-MVP | Link-based handoff in MVP |

---

## Global Open Questions

1. **Scoring Logic:** Should MVP use conditional logic (if/else) or a weighted rules engine for lead scoring?
2. **Pipeline Stages:** Are the 6 stages fixed, or should workspaces be able to customize them?
3. **Event: `chat_started`:** Define the exact trigger — first step displayed vs. first user interaction.
4. **Widget CAPTCHA:** Should the fallback form include anti-spam protection?
5. **Data Retention:** What is the retention policy for events and conversations?
6. **Rate Limiting:** What rate limits apply to public widget endpoints (`/v1/widget/*`)?
7. **File Storage:** What S3-compatible service will be used for file uploads (LLM grounding)?
8. **Email Provider:** Which service for transactional emails (SendGrid, SES, Resend)?
9. **Monitoring:** What observability stack is needed for MVP (error tracking, APM)?
10. **GDPR:** How does the consent mechanism comply with regional privacy regulations?
