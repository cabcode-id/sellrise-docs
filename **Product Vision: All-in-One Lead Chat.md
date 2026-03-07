# **Product Vision: All-in-One Lead Chat + Qualification + Mini-CRM + Knowledge Base + Booking**

## **One-liner**

A single product that embeds on any website as a chat widget, qualifies leads, captures contact details, stores everything in an internal mini-CRM, uses a controlled knowledge base for answers, and routes hot leads to booking—without stitching together third-party chatbot + CRM + automation tools.

## **Problem**

Most “chatbot stacks” require:

- separate chatbot tool,
- separate CRM,
- multiple integrations/webhooks/zaps,
- fragile workflows and duplicated data.

This creates operational friction, slow iteration, and inconsistent performance across projects.

## **Goal**

Deliver a unified, configurable system:

- deployable in hours,
- reusable across multiple projects/businesses under one brand,
- controlled dialogues (no random LLM answers),
- operators can manage leads and transcripts in one CRM,
- booking supported (phase 1: link, phase 2: native calendar).

## **Non-goals (v1)**

- Full enterprise CRM (marketing automation suites, complex sequences).
- Omnichannel (WhatsApp/IG/FB) in v1.
- Medical advice. The product is lead intake and routing.

---

# **PRD (v1)**

## **Target users**

- **Website visitors**: interact with widget, ask questions, submit contact, book.
- **Operators (Agents)**: review leads, read transcripts, update pipeline, follow up, book calls.
- **Admins**: configure scenarios, knowledge base, routing, branding.

## **Core use cases**

1. Visitor opens widget → answers questions → leaves contact → lead appears in CRM with transcript.
2. Visitor asks FAQ → system answers strictly from KB (no hallucinations).
3. Hot lead → immediate CTA to booking link (v0), later native booking (v1).
4. Operator views “New leads” inbox, assigns owner, moves pipeline stages, adds notes.
5. Admin edits scenarios/KB and publishes changes without code changes.

## **Success metrics**

- Widget open → conversation started rate
- Conversation completion rate
- Contact submit rate
- Hot lead rate
- Booking link click rate (v0)
- Booking completion rate (v1)
- Operator response time (SLA)

---

# **Product Scope**

## **Tenancy model (single brand, multi-workspace)**

This is **not white-label**, but must support **multiple workspaces** internally:

- Workspace = a business/project/client under your brand.
- Each workspace has its own: domains, scenarios, KB, users, leads, pipeline.
- Hard isolation of data between workspaces.

Reason: scaling to new projects without duplicating infrastructure.

---

# **Modules**

## **1) Web Widget (Frontend SDK)**

### **Features**

- Embed via JS snippet on any site.
- Display modes:
    - floating bubble
    - inline embed
- Branding: colors, logo, intro text.
- Language support (EN default; multi-language later).
- Consent UI: contact consent checkbox + privacy link.
- Data capture: name, phone, email, preferred contact method/timezone.
- Graceful fallback: “Leave your details” if chat fails.

### **Performance / UX requirements**

- Lazy-load
- Mobile-first
- Fast initial paint (keep bundle small; target <150–250KB gz as a guideline)

### **Events emitted**

- widget_opened
- chat_started
- step_completed
- lead_submitted
- handoff_clicked
- booking_link_shown
- booking_link_clicked

---

## **2) Conversation Engine (Backend)**

### **2.1 Dialogue Orchestrator (state machine)**

Scenarios are **config**, not hard-coded.

### **Step types**

- message
- question_text
- question_choice
- form
- conditional_branch
- kb_search
- handoff_booking_link
- handoff_operator
- end

### **Requirements**

- Variables stored per session/lead (e.g., lead.intent, lead.budget_range, lead.timeframe, lead.score).
- Validation: email; phone (E.164); required fields.
- Dedup: by email/phone.
- Lead scoring: hot/warm/cold based on rules.

### **2.2 Knowledge Base answering**

Two modes (v1 recommended baseline is strict):

1. **Strict KB** (must-have): search KB and return stored answers/cards only.
2. **LLM assist** (optional later): generate paraphrase strictly grounded in KB (toggle + safety).

**Rule:** For medical-adjacent contexts, answers should be **KB-only** in v1.

### **2.3 Handoff routing rules**

- Hot → show booking CTA + capture contact if needed.
- Warm → capture contact + notify operator.
- Cold → capture email (optional) + notify operator (or deprioritize).

---

## **3) Mini-CRM (Operators work here)**

### **3.1 Lead entity (minimum fields)**

- id, created_at, updated_at
- Source: domain, page_url, referrer, utm_*
- Contact: name, phone, email, preferred_channel, timezone
- Intent: procedure/category, notes
- Qualification: budget_range, timeframe, location, score (hot/warm/cold)
- Pipeline: stage (new → qualified → contacted → booked → won/lost)
- owner_id
- tags

### **3.2 Activity log**

- Full conversation transcript (events + answers)
- Manual notes
- Stage changes
- Assignment changes

### **3.3 CRM UI requirements**

- **Inbox**: new/unassigned leads
- **Pipeline**: kanban by stage
- **Lead detail**:
    - summary card
    - transcript timeline
    - actions: assign owner, change stage, add note, tag, booking CTA
- Search + filters (stage, score, date, procedure, owner, tags)
- SLA indicator: time since created / time since last operator action

---

## **4) Knowledge Base (KB)**

### **Entities**

- Articles: title, body (markdown), category, language, tags
- FAQ: question, answer
- Offer cards: procedure/package, “price from”, what’s included, duration/notes

### **Requirements**

- Search (v1: Postgres full-text is fine)
- Versioning/audit:
    - updated_at
    - updated_by
    - history (at least scenario/KB publish events)

---

## **5) Scenario Admin (Config & Control)**

### **Requirements**

- JSON editor (MVP) with:
    - schema validation
    - preview/simulator
    - versioning (draft vs published)
    - publish action
- Reusable blocks library:
    - “Collect contact”
    - “Budget question”
    - “Timeframe question”
    - “Booking handoff”
- Routing/scoring configuration:
    - scoring rules
    - required fields
    - assignment rules (v1: manual; later round-robin)

---

## **6) Notifications (v1 minimal)**

- Email notification to operators for:
    - new hot lead
    - new lead assigned
- Optional: confirmation email to lead (“We received your request”)
- SLA reminders (scheduled jobs):
    - hot lead untouched for X minutes → ping operator

---

## **7) Analytics (v1)**

- Funnel dashboard:
    - opened → started → lead_submitted → qualified_hot → booking_click
- Drop-off by step
- Volume by source (utm/referrer/page)
- CSV export

---

# **Booking Roadmap**

## **v0 (MVP): booking link**

- Booking link stored per workspace (optionally per procedure)
- Scenario step handoff_booking_link shows CTA
- Track booking_link_clicked

## **v1: native booking (internal calendar)**

### **Entities**

- TeamMember (operator), timezone, working hours
- Service (consultation type), duration, buffers
- Availability rules + exceptions
- Booking: lead_id, member_id, datetime, status, notes
- Tokenized links for reschedule/cancel

### **Flow**

- Visitor selects timezone → date/time slot → confirms
- Slot locking (2–5 min lock)
- Confirmation email + ICS
- Operator notification
- CRM updates stage automatically to “Booked”

---

# **Non-Functional Requirements (NFR)**

## **Security**

- HTTPS only
- Auth: JWT + refresh tokens
- RBAC by role
- Audit log for scenario/KB/pipeline changes
- Rate limiting (widget + API)

## **Privacy / Medical-adjacent constraints**

- Minimize PII
- Consent capture with timestamp
- Avoid collecting sensitive health details in v1 flows
- Data retention policies per workspace (configurable later)
- Lead export/delete mechanisms

## **Reliability**

- Idempotent lead submit
- Job queue for notifications
- Graceful widget fallback on backend timeouts

---

# **Data Model (Minimum Tables)**

- workspaces
- users
- domains
- scenarios
- scenario_versions
- leads
- lead_events
- notes
- kb_articles
- kb_faq
- consents

Booking (for v1 native):

- team_members
- services
- availability_rules
- availability_exceptions
- bookings
- booking_tokens
- slot_locks

---

# **API Surface (Minimum)**

## **Widget-facing**

- POST /v1/widget/session (create session; return published config + branding)
- POST /v1/widget/event (log events)
- POST /v1/widget/lead (submit lead + answers)
- GET /v1/widget/kb/search?q=... (KB search)
- POST /v1/widget/handoff/click (track CTA clicks)

## **Admin/CRM**

- Leads:
    - GET /v1/leads
    - GET /v1/leads/{id}
    - PATCH /v1/leads/{id} (stage/owner/tags)
- Notes:
    - POST /v1/leads/{id}/notes
- KB:
    - CRUD articles/FAQ
- Scenarios:
    - CRUD scenarios + publish/version
- Analytics:
    - funnel + dropoffs endpoints
- Booking (v1):
    - availability query
    - create/reschedule/cancel booking

---

# **MVP Release Definition (what “done” means)**

1. Widget embeds and works on mobile/desktop.
2. Admin can create a scenario (JSON), publish it, and preview it.
3. Visitor completes qualification and submits contact.
4. Lead appears in CRM with full transcript + score.
5. Operator can assign, change stage, add notes.
6. Booking link handoff is supported and tracked.
7. Email notification triggers on hot leads.
8. Basic funnel analytics available.

---

# **Recommended MVP Stack (implementation-neutral)**

- Backend: Node (Nest/Express) or Python (FastAPI)
- DB: Postgres
- Cache/Queue: Redis (rate limits + jobs)
- Admin UI: Next.js
- Widget build: Vite/Rollup
- KB Search: Postgres full-text (v1)

---

# **Open Questions (optional, for the developer kickoff)**

- Do we need real-time live takeover chat in MVP, or is transcript + follow-up enough?
- What is the minimum operator workflow for “contacted” (email only, or in-app messaging later)?
- Any mandatory compliance constraints specific to the initial vertical (medical tourism)?

---

If you want, I can also output:

- **User stories + acceptance criteria** (Linear/Jira-ready),
- a **scenario JSON schema** and example flows (qualification + booking),
- and a first-pass **DB schema** (Postgres DDL).