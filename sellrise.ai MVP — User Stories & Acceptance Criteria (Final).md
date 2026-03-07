# **sellrise.ai MVP — User Stories & Acceptance Criteria (Final)**

## **0) MVP Scope (Locked)**

### **Included**

1. **Web Widget (bubble)**: embed via JS snippet, branding, consent, lead capture, basic validation, fallback.
2. **Scenario Engine**: deterministic, config-driven (state machine), published versions.
3. **Knowledge Base (KB)**: Articles + FAQ CRUD, **KB-only** search answers in widget.
4. **Mini-CRM**: Inbox, Pipeline (kanban), Lead detail (summary + transcript), notes/tags, owner assignment, stage changes.
5. **Booking handoff**: **link-based CTA** only (no internal calendar).
6. **Email notifications**: on hot lead submit + basic retry.
7. **Analytics**: funnel counts + drop-offs (basic), CSV export.

### **Excluded (MVP)**

- Native booking calendar / availability / slot locking / ICS.
- Omnichannel (WhatsApp/IG/FB).
- LLM-generated free-form answers.
- Real-time “live takeover chat” (operator typing into the widget).

---

## **1) Glossary (Single Source of Truth)**

- **Workspace**: isolated tenant under sellrise.ai brand (not white-label). Contains users, domains, scenarios, KB, leads.
- **Domain**: a website domain allowed to load widget (e.g., example.com).
- **Widget Session**: a conversation session started by a visitor on a website. Identified by session_id.
- **Lead**: a persisted entity representing a person/contact. Created/updated when contact details are submitted.
- **Lead Event**: an append-only record of interactions/actions (widget events, answers, scoring, stage changes, notifications). Transcript is derived from events.
- **Scenario**: draft configuration (JSON) editable by Admin.
- **Scenario Version**: immutable published snapshot of a Scenario used by widget sessions.
- **Score**: qualification category (hot|warm|cold) determined by rules.
- **Stage**: pipeline stage (new|qualified|contacted|booked|won|lost) editable by operators.

---

## **2) Global System Rules (Non-negotiable)**

### **R1 — Workspace isolation**

- Every DB table containing business data includes workspace_id.
- Every authenticated API request is scoped to a workspace and **MUST NOT** return or mutate data outside it.

**Acceptance**

- Attempting to access another workspace’s lead via ID returns 404 (not 403) to avoid leakage.

### **R2 — Transcript from events**

- CRM transcript is rendered from ordered lead_events only.
- No “hidden” transcript sources.

**Acceptance**

- If events are deleted, transcript disappears accordingly (events are append-only; deletion should be admin-only/none in MVP).

### **R3 — KB-only answers**

- Widget “FAQ/knowledge” responses come strictly from KB search results (Articles/FAQ).
- If no results → controlled fallback (“Leave contact”).

### **R4 — Lead deduplication & idempotency**

- A lead submit is idempotent by:
    1. normalized email (dedup_email), else
    2. normalized phone (dedup_phone)
- If both missing → create a new lead (but MVP contact form must require email, so practically email is always present).

### **R5 — Deterministic scenario execution**

- Scenario engine must produce the same next step for the same inputs and scenario version.

---

## **3) Personas / Roles**

- **Visitor**: anonymous user interacting with widget.
- **Agent**: operator managing leads inside CRM.
- **Admin**: configures workspace, scenarios, KB, users, settings.
- **Viewer**: read-only access for reporting.

---

# **4) User Stories + Acceptance Criteria**

## **EPIC A — Auth, Users, Workspaces**

### **US-A1: Admin creates a workspace**

**As an Admin**, I can create a workspace to manage a project/business.

**Acceptance Criteria**

- Workspace creation requires name.
- Workspace is created with a new workspace_id.
- Creator user is assigned role admin in that workspace.
- API returns created workspace object.

---

### **US-A2: Admin creates/invites users with roles**

**As an Admin**, I can add users to my workspace and assign roles.

**Acceptance Criteria**

- Admin can create user with: email, role.
- Roles allowed: admin, agent, viewer.
- email must be unique within workspace.
- Created user can log in and see only that workspace.
- Role permissions enforced:
    - Viewer: can read leads, KB (optional), analytics; cannot modify leads/stages/owners/KB/scenarios.
    - Agent: can modify leads (stage/owner/tags/notes), cannot publish scenarios or edit KB (optional policy).
    - Admin: full access.

---

### **US-A3: Login / auth protection**

**As a User**, I can log in to access CRM/admin.

**Acceptance Criteria**

- Login requires email + password.
- Authenticated requests require access token.
- Unauthorized requests return 401.
- Invalid token returns 401.
- Logging out invalidates refresh token or session.

---

## **EPIC B — Domains & Widget Installation**

### **US-B1: Admin registers allowed domains**

**As an Admin**, I can register a domain to allow widget sessions from that site.

**Acceptance Criteria**

- Admin can add domain string (e.g., example.com).
- System stores domains per workspace.
- Widget session creation checks domain:
    - If domain not allowed → returns 403 with error code domain_not_allowed.
    - (Dev mode override optional, but must be explicit config.)
- Domain list is editable (add/remove).

---

### **US-B2: Admin copies embed snippet**

**As an Admin**, I can copy a JS snippet to install the widget.

**Acceptance Criteria**

- UI displays a snippet that:
    - loads widget JS bundle
    - passes a workspace_public_key (or equivalent) and optional domain_id
- After installing snippet, widget can create session successfully.

---

## **EPIC C — Scenario Management (Draft / Publish)**

### **US-C1: Admin creates/edits scenario draft JSON**

**As an Admin**, I can create a scenario and edit its draft JSON.

**Acceptance Criteria**

- Admin can create scenario with name.
- Admin can edit draft_json in UI.
- Draft JSON is validated against schema:
    - required fields exist
    - entry_step_id references existing step
    - every step id unique
    - next references valid step ids where applicable
- If invalid, show specific errors and prevent publish.

---

### **US-C2: Admin can simulate scenario draft**

**As an Admin**, I can simulate a draft scenario to verify flow.

**Acceptance Criteria**

- Simulator shows:
    - current step
    - rendered UI for that step type (basic)
    - current variable store (vars)
- Admin can provide inputs and move forward.
- Simulator uses the same engine logic as production (shared module).

---

### **US-C3: Admin publishes immutable scenario version**

**As an Admin**, I can publish a scenario so widget uses it.

**Acceptance Criteria**

- Publishing creates scenario_version with:
    - incremental version
    - published_json snapshot
    - published_at, published_by
- Published version cannot be edited (immutable).
- Widget sessions always fetch **latest published** version.
- If no published version exists:
    - widget returns fallback flow (contact form) OR explicit error with controlled UX.

---

## **EPIC D — Widget Session & Conversation**

### **US-D1: Widget session handshake**

**As a Visitor**, when widget loads it starts a session and receives config.

**Acceptance Criteria**

- Widget calls POST /v1/widget/session with:
    - domain, page_url, referrer, utm_*, timezone_guess
- API returns:
    - session_id
    - branding config
    - scenario_version_id + published_scenario_json
    - booking_links dictionary (key → url)
- System records event:
    - widget_opened (workspace-scoped)
- Widget renders entry_step_id as first step.

---

### **US-D2: Visitor completes steps, engine resolves next step**

**As a Visitor**, I answer questions and progress through the scenario.

**Acceptance Criteria**

- For each step completion:
    - Widget sends POST /v1/widget/event with step_completed including:
        - session_id, step_id, answer, ts_client (optional)
- Engine resolves next step deterministically from scenario + stored vars.
- Widget enforces validation:
    - email format
    - required fields
- Validation errors:
    - do not advance flow
    - show user-friendly message

---

### **US-D3: Lead submission (contact + consent)**

**As a Visitor**, I submit contact details and consent.

**Acceptance Criteria**

- Contact form requires:
    - name (required)
    - email (required)
    - consent (required)
    - phone optional
- Widget calls POST /v1/widget/lead with:
    - session_id
    - contact fields
    - captured qualification fields (procedure/timeframe/budget etc.)
    - consent: { contact: true }
    - optionally answers map (step_id → value)
- API behavior:
    - upserts lead by dedup rules
    - stores consent record with timestamp
    - stores lead_submitted event
- Response returns:
    - lead_id, score, stage
- Widget shows completion message.

---

### **US-D4: Booking link CTA step**

**As a Visitor**, if I reach booking step, I see a CTA button.

**Acceptance Criteria**

- Step type handoff_booking_link displays:
    - text + button label (configurable)
    - URL resolved by booking_link_key
- System logs:
    - booking_link_shown when step is rendered
    - booking_link_clicked when CTA clicked
- If URL missing:
    - widget shows fallback message and continues flow.

---

### **US-D5: Widget fallback on API failure**

**As a Visitor**, if chat cannot run, I can still leave contact details.

**Acceptance Criteria**

- If /widget/session fails:
    - widget shows fallback lead form with consent
    - submission still reaches backend (dedicated fallback endpoint or retry session)
- If backend times out mid-flow:
    - widget shows “Please leave your details” fallback
- Backend logs error with correlation id.

---

## **EPIC E — Leads, Dedup, Events**

### **US-E1: Lead deduplication**

**As the System**, I prevent duplicate leads.

**Acceptance Criteria**

- Dedup normalization:
    - email trimmed + lowercased
    - phone normalized (strip spaces, + optional; best effort MVP)
- Upsert behavior:
    - if dedup_email match → update existing lead
    - else if dedup_phone match → update existing lead
    - else create new lead
- Upsert never creates two leads with same dedup_email in same workspace (DB constraint).

---

### **US-E2: Event logging is complete and ordered**

**As an Agent**, I see a correct transcript.

**Acceptance Criteria**

- For a completed conversation, lead_events must include at minimum:
    - widget_opened
    - chat_started (when first step displayed or first user action; define consistently)
    - step_completed events for answered steps
    - lead_submitted
    - booking_link_shown/clicked if applicable
- Lead detail renders transcript ordered by created_at ASC.
- Each event payload includes enough data to render:
    - bot message text OR step definition reference
    - visitor answer

---

## **EPIC F — Mini-CRM (Operator UI)**

### **US-F1: Inbox view for new/unassigned leads**

**As an Agent**, I see new/unassigned leads to process.

**Acceptance Criteria**

- Inbox default filter:
    - stage = 'new' OR owner_id IS NULL (choose one and document; recommended: both)
- List shows:
    - created_at
    - name/email/phone
    - score
    - procedure
    - page_url (or short source)
- Clicking row opens lead detail.

---

### **US-F2: Pipeline (Kanban)**

**As an Agent**, I manage leads by stage visually.

**Acceptance Criteria**

- Stages displayed as columns:
    - new, qualified, contacted, booked, won, lost
- Stage change by drag/drop OR dropdown.
- Stage change creates:
    - lead.stage updated
    - lead_event: stage_changed with old/new values
- Viewer cannot change stage.

---

### **US-F3: Lead detail view + actions**

**As an Agent**, I review lead and take actions.

**Acceptance Criteria**

- Lead detail shows:
    - contact card
    - qualification fields
    - score
    - owner
    - transcript (from events)
    - notes list
- Actions:
    - assign owner (self or other agent)
    - change stage
    - add note
    - add tags (simple string array)
- All actions log lead_events:
    - owner_assigned
    - stage_changed
    - note_added
    - tags_updated

---

### **US-F4: Search and filters**

**As an Agent**, I search/filter leads.

**Acceptance Criteria**

- Filters:
    - stage, score, owner, date range, procedure
- Search query matches:
    - name, email, phone (case-insensitive)
- Pagination supported (MVP: offset/limit is fine)
- Performance acceptable for at least thousands of leads/workspace.

---

## **EPIC G — Knowledge Base**

### **US-G1: Admin CRUD KB Articles**

**As an Admin**, I create/update articles.

**Acceptance Criteria**

- Fields: title, body_md, tags, category, language
- Create/edit/delete supported
- Updated_at and updated_by captured
- Only Admin can edit (recommended).

---

### **US-G2: Admin CRUD KB FAQ**

**As an Admin**, I create/update FAQ items.

**Acceptance Criteria**

- Fields: question, answer, tags, language
- Create/edit/delete supported
- Updated_at and updated_by captured
- Only Admin can edit (recommended).

---

### **US-G3: KB search API**

**As a Visitor**, I get relevant KB results.

**Acceptance Criteria**

- GET /v1/widget/kb/search?q=... returns:
    - array of results with type (article/faq), id, title/question, snippet/answer_preview
- Ranking: basic full-text ranking is acceptable (Postgres FTS).
- If no results:
    - return empty list
    - widget shows fallback: “Please leave contact.”

---

## **EPIC H — Notifications & Jobs**

### **US-H1: Hot lead notification email**

**As an Agent**, I receive email on hot lead.

**Acceptance Criteria**

- On lead submit, if score=hot:
    - send email to configured recipients
- Email includes:
    - lead contact
    - procedure, timeframe, budget
    - direct link to lead detail
- Retry policy:
    - at least 3 retries with backoff
- Log events:
    - notification_sent or notification_failed

---

### **US-H2: SLA reminder (Optional MVP+)**

**As the System**, remind if hot lead untouched for X minutes.

**Acceptance Criteria**

- Configurable threshold (default e.g. 30 min)
- “Untouched” means:
    - no operator events (stage_changed/owner_assigned/note_added) after lead creation
- Sends reminder email + logs event

(You can push this after MVP if time is tight.)

---

## **EPIC I — Analytics**

### **US-I1: Funnel metrics endpoint**

**As an Admin**, I see funnel metrics by date range.

**Acceptance Criteria**

- Endpoint supports from and to (inclusive) and returns counts:
    - widget_opened
    - chat_started
    - lead_submitted
    - qualified_hot (derived: lead_submitted with score=hot OR explicit event)
    - booking_link_clicked
- All counts are workspace-scoped.
- UI can compute conversion rates from counts.

---

### **US-I2: CSV export**

**As an Admin**, I export leads/funnel data.

**Acceptance Criteria**

- Leads export includes:
    - created_at, name, email, phone
    - procedure, timeframe, budget, score
    - stage, owner
    - utm/referrer/page_url
- Export respects filters and date range.
- Workspace-scoped only.

---

# **5) Test Requirements (MVP)**

## **Automated tests (minimum)**

1. Scenario validation:
    - missing entry_step_id
    - invalid next references
    - invalid step types
2. Scenario engine determinism:
    - same inputs produce same path
3. Lead dedup:
    - same email updates existing
4. RBAC:
    - viewer cannot mutate lead
5. KB search:
    - results and empty fallback
6. Widget endpoints smoke:
    - session → events → lead submit

## **Manual QA checklist (release gate)**

- Embed widget on a real test page; verify mobile + desktop.
- Complete flow end-to-end:
    - lead appears in CRM
    - transcript correct order
    - hot lead triggers email
    - booking link clicked event logged
- Verify workspace isolation:
    - user from workspace A cannot see workspace B leads.

---

# **6) Implementation Notes (to avoid ambiguity)**

- **Define chat_started** consistently (recommended): log chat_started when first step is rendered, not when button opens.
- **Stages are editable** but initial set is fixed for MVP: new, qualified, contacted, booked, won, lost.
- **Scoring** in MVP:
    - can be done via scenario conditional logic OR server rule engine.
    - whichever is used, must set leads.score and log score_assigned event.