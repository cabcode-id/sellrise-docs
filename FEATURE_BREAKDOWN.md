# Sellrise Platform — Complete Feature Breakdown

**Version:** 1.0  
**Last Updated:** March 8, 2026  
**Purpose:** Organize all PRD requirements into clear, actionable feature specifications

---

## Table of Contents

1. [Feature Overview & Priorities](#feature-overview--priorities)
2. [Core Platform Features](#1-core-platform-features)
3. [Authentication & User Management](#2-authentication--user-management)
4. [Widget & Embedding](#3-widget--embedding)
5. [Conversation Engine](#4-conversation-engine)
6. [Lead Management (Mini-CRM)](#5-lead-management-mini-crm)
7. [Knowledge Base](#6-knowledge-base)
8. [Scenario/Bot Builder](#7-scenariobot-builder)
9. [Analytics & Reporting](#8-analytics--reporting)
10. [Notifications & Jobs](#9-notifications--jobs)
11. [Integrations](#10-integrations)
12. [Settings & Administration](#11-settings--administration)
13. [Booking System](#12-booking-system)

---

## Feature Overview & Priorities

### Priority Levels

- 🔴 **MVP Phase 1** (Core functionality, must-have for first release)
- 🟡 **MVP Phase 2** (Important but can follow Phase 1)
- 🟢 **Post-MVP** (Nice to have, future enhancements)

### MVP Scope Summary

**Included in MVP:**
- Multi-tenant workspace architecture
- Web widget (bubble + inline)
- Scenario engine (config-driven, deterministic)
- Knowledge Base (articles + FAQs, search)
- Mini-CRM (inbox, pipeline, lead detail, notes)
- Link-based booking handoff
- Email notifications
- Basic analytics (funnel, CSV export)

**Excluded from MVP:**
- Native booking calendar
- Omnichannel (WhatsApp/Instagram/Facebook)
- LLM free-form answers (KB-only in MVP)
- Real-time operator chat takeover
- Advanced automations (sequences, drip campaigns)

### ✅ Recently Implemented Features (March 2026)

**Scenario Configuration & AI Enhancement** (Section 4.1)
- Comprehensive admin UI for creating/managing AI conversation scenarios
- LLM-enhanced editing via OpenRouter (Claude 3.5 Sonnet)
- Three enhancement levels: prompt-level, JSON-level, full scenario
- Default conversation engine template with 6 stages, 11 slots, 4 actions
- System prompts editor with templates and individual enhancement
- File attachment component for LLM grounding (PDF, TXT, MD, JSON, CSV, DOCX)
- Interactive scenario preview simulator
- Draft/publish workflow with validation
- Frontend: 450+ line main page + 4 components
- Backend: LLM endpoints, file upload with workspace isolation
- Documentation: User guide + template reference

**Centralized Conversation System** (Section 4.5)
- One AI "brain" per workspace handles all channels (web, WhatsApp, Telegram, Instagram, etc.)
- Complete workspace isolation preventing context mixing between admins
- Conversation and Message models with stateful context tracking
- Service layer with workspace security enforcement
- API endpoints for admin conversation management
- Widget endpoint for public message sending
- Channel-agnostic processing with consistent AI persona
- Full audit trail of all messages
- Database models ready for migration
- Comprehensive architecture documentation

**Status**: Database migration pending (`alembic revision --autogenerate -m "add_conversations_and_messages"`)
- Real-time operator chat takeover
- Advanced automations (sequences, drip campaigns)

---

## 1. Core Platform Features

### 1.1 Multi-Tenant Architecture

**Priority:** 🔴 MVP Phase 1  
**Category:** Infrastructure

**Description:**  
Foundation for supporting multiple isolated workspaces under single Sellrise brand.

**Requirements:**
- Hard data isolation between workspaces
- All business tables include `workspace_id` foreign key
- Every API request filtered by `workspace_id`
- Return 404 (not 403) for unauthorized access to prevent data leakage

**Acceptance Criteria:**
- ✅ User from Workspace A cannot see Workspace B data
- ✅ All queries automatically filter by `workspace_id`
- ✅ Database constraints prevent cross-workspace references
- ✅ API responds with 404 for non-existent or unauthorized resources

**Technical Notes:**
- Implement in ORM base model
- Add database indexes on `workspace_id`
- Middleware validates workspace context

**Dependencies:** None (foundational)

---

### 1.2 Workspace Management

**Priority:** 🔴 MVP Phase 1  
**Category:** Infrastructure

**Description:**  
Create and manage isolated tenant workspaces.

**User Story:**  
> **As an Admin**, I can create a workspace to manage a project/business.

**Requirements:**
- Workspace creation with name
- Creator automatically assigned admin role
- Workspace settings (JSON config)
- Allowed domains registration

**Acceptance Criteria:**
- ✅ Admin creates workspace with unique name
- ✅ Creator receives admin role automatically
- ✅ Workspace ID generated (UUID)
- ✅ Workspace isolated from others

**API Endpoints:**
```
POST   /v1/workspaces
GET    /v1/workspaces/{id}
PATCH  /v1/workspaces/{id}
DELETE /v1/workspaces/{id}
```

**Dependencies:** None (foundational)

---

## 2. Authentication & User Management

### 2.1 User Authentication (JWT)

**Priority:** 🔴 MVP Phase 1  
**Category:** Auth

**Description:**  
Secure login system with JWT access/refresh tokens.

**User Story:**  
> **As a User**, I can log in to access the CRM/admin interface.

**Requirements:**
- Email + password login
- JWT access tokens (60 min expiry)
- Refresh tokens (7 day expiry, stored as hash)
- Token validation middleware
- Logout (invalidate refresh token)

**Acceptance Criteria:**
- ✅ User logs in with email/password
- ✅ Returns access_token + refresh_token
- ✅ Unauthorized requests return 401
- ✅ Invalid/expired tokens return 401
- ✅ Logout invalidates refresh token

**API Endpoints:**
```
POST /v1/auth/login
POST /v1/auth/refresh
POST /v1/auth/logout
```

**Technical Notes:**
- Use bcrypt for password hashing
- Store refresh tokens as HMAC-SHA256 hash
- Access token includes: user_id, workspace_id, role

**Dependencies:** Workspace Management

---

### 2.2 Role-Based Access Control (RBAC)

**Priority:** 🔴 MVP Phase 1  
**Category:** Auth

**Description:**  
Three-tier permission system for workspace users.

**User Story:**  
> **As an Admin**, I can add users to my workspace and assign roles.

**Roles:**
- **ADMIN**: Full access (scenarios, KB, users, settings)
- **AGENT**: Lead management (update stage, owner, notes), cannot edit scenarios/KB
- **VIEWER**: Read-only (leads, analytics)

**Requirements:**
- User creation with role assignment
- Email unique within workspace
- Permission enforcement at API level
- Role-based UI visibility

**Acceptance Criteria:**
- ✅ Admin can create/invite users with roles
- ✅ Viewer cannot modify leads/stages/KB
- ✅ Agent cannot publish scenarios or edit KB
- ✅ Admin has full access
- ✅ API returns 403 for unauthorized role actions

**API Endpoints:**
```
POST   /v1/users
GET    /v1/users
GET    /v1/users/{id}
PATCH  /v1/users/{id}
DELETE /v1/users/{id}
```

**Dependencies:** User Authentication

---

### 2.3 Password Reset

**Priority:** 🟡 MVP Phase 2  
**Category:** Auth

**Description:**  
Self-service password reset via email.

**User Story:**  
> **As a User**, I can reset my password if I forget it.

**Requirements:**
- Generate secure reset token (1-hour expiry)
- Send reset email with link
- Token validation on reset
- Update password

**Acceptance Criteria:**
- ✅ User requests reset with email
- ✅ Receives email with reset link
- ✅ Link expires after 1 hour
- ✅ Token used only once
- ✅ Password updated successfully

**API Endpoints:**
```
POST /v1/auth/forgot-password
POST /v1/auth/reset-password
```

**Dependencies:** User Authentication, Notifications

---

## 3. Widget & Embedding

### 3.1 Widget Embed Code Generation

**Priority:** 🔴 MVP Phase 1  
**Category:** Widget

**Description:**  
Generate JavaScript snippet for embedding widget on client websites.

**User Story:**  
> **As an Admin**, I can copy a JS snippet to install the widget on my website.

**Requirements:**
- Generate unique snippet per workspace
- Include workspace public key
- Load widget JS bundle
- Configure domain restrictions

**Acceptance Criteria:**
- ✅ Admin copies embed snippet from UI
- ✅ Snippet loads widget JS bundle
- ✅ Snippet passes workspace_public_key
- ✅ Widget creates session successfully after installation

**Example Snippet:**
```html
<script>
  (function(w,d,s,o,f,js,fjs){
    w['SellriseWidget']=o;w[o]=w[o]||function(){(w[o].q=w[o].q||[]).push(arguments)};
    js=d.createElement(s),fjs=d.getElementsByTagName(s)[0];
    js.id=o;js.src=f;js.async=1;fjs.parentNode.insertBefore(js,fjs);
  }(window,document,'script','sellrise','https://cdn.sellrise.ai/widget.js'));
  sellrise('init', { workspace: 'YOUR_WORKSPACE_KEY' });
</script>
```

**Dependencies:** Workspace Management, Domain Registration

---

### 3.2 Domain Registration & Validation

**Priority:** 🔴 MVP Phase 1  
**Category:** Widget

**Description:**  
Register allowed domains to prevent unauthorized widget usage.

**User Story:**  
> **As an Admin**, I can register a domain to allow widget sessions from that site.

**Requirements:**
- Add/remove allowed domains
- Validate domain on widget session creation
- Support development mode (localhost)
- Store branding config per domain

**Acceptance Criteria:**
- ✅ Admin adds domain (e.g., example.com)
- ✅ Widget session validates origin domain
- ✅ Returns 403 if domain not allowed
- ✅ Domain list is editable
- ✅ Dev mode override for testing

**API Endpoints:**
```
POST   /v1/domains
GET    /v1/domains
PATCH  /v1/domains/{id}
DELETE /v1/domains/{id}
```

**Dependencies:** Workspace Management

---

### 3.3 Widget Display Modes

**Priority:** 🔴 MVP Phase 1  
**Category:** Widget (Frontend)

**Description:**  
Multiple widget display options for different use cases.

**Requirements:**
- **Bubble mode**: Floating button (bottom-right)
- **Inline mode**: Embedded in page element
- Configurable position, colors, logo
- Mobile-responsive design
- Lazy-load for performance (<150-250KB bundle)

**Acceptance Criteria:**
- ✅ Bubble widget appears bottom-right
- ✅ Inline widget renders in container
- ✅ Colors/logo configurable via admin
- ✅ Mobile-optimized (touch-friendly)
- ✅ Fast initial paint (<3 sec)

**Technical Notes:**
- Vite/Rollup build for widget
- Shadow DOM to prevent CSS conflicts
- Lazy-load images and heavy components

**Dependencies:** Widget Embed, Branding Config

---

### 3.4 Widget Session Handshake

**Priority:** 🔴 MVP Phase 1  
**Category:** Widget (Backend)

**Description:**  
Initialize widget session and return configuration.

**User Story:**  
> **As a Visitor**, when the widget loads it starts a session and receives config.

**Requirements:**
- Create session on widget load
- Return session_id, branding, scenario config
- Track widget_opened event
- Validate domain

**Acceptance Criteria:**
- ✅ Widget calls POST /v1/widget/session
- ✅ Returns session_id, branding, scenario config
- ✅ Logs widget_opened event
- ✅ Returns latest published scenario version
- ✅ Fallback if no published scenario

**API Endpoint:**
```
POST /v1/widget/session
Body: { domain, page_url, referrer, utm_*, timezone }
Response: { session_id, branding, scenario_version_id, scenario_json }
```

**Dependencies:** Domain Validation, Scenario Publishing

---

### 3.5 Widget Fallback Handling

**Priority:** 🔴 MVP Phase 1  
**Category:** Widget

**Description:**  
Graceful degradation if backend fails or scenario unavailable.

**User Story:**  
> **As a Visitor**, if chat cannot run, I can still leave contact details.

**Requirements:**
- Fallback contact form on session failure
- Fallback on mid-flow timeout
- Retry logic with exponential backoff
- Log errors with correlation ID

**Acceptance Criteria:**
- ✅ Session failure shows fallback form
- ✅ Timeout shows "Please leave details" message
- ✅ Submission reaches backend via dedicated endpoint
- ✅ Errors logged with correlation ID

**API Endpoint:**
```
POST /v1/widget/fallback-lead
```

**Dependencies:** Widget Session, Lead Submission

---

## 4. Conversation Engine

### 4.1 Scenario Configuration (Draft/Publish)

**Priority:** 🔴 MVP Phase 1  
**Category:** Scenario Engine  
**Status:** ✅ **IMPLEMENTED**

**Description:**  
Comprehensive admin interface for creating and managing LLM-powered conversation scenarios with AI-enhanced editing, system prompts, file attachments, and interactive preview.

**User Story:**  
> **As an Admin**, I can create, edit, and publish AI conversation scenarios with LLM enhancement to improve prompts and configurations.

**Features Implemented:**

1. **Scenario JSON Editor**
   - Visual JSON editor with real-time validation
   - Format button for clean JSON output
   - AI enhancement button for entire config improvement
   - Error highlighting with specific messages
   - Draft/publish workflow

2. **System Prompts Editor**
   - Manage multiple system prompts (main, qualification, outbound, meeting_booking, followup)
   - Template library with pre-configured prompts
   - Individual AI enhancement buttons per prompt
   - Add/remove custom prompts
   - LLM-powered prompt improvement via OpenRouter (Claude 3.5 Sonnet)

3. **File Attachment for LLM Grounding**
   - Upload documents (PDF, TXT, MD, JSON, CSV, DOCX)
   - 10MB file size limit per file
   - Workspace-isolated file storage
   - Drag-and-drop support
   - Files provide context to LLM during conversations

4. **Default Template System**
   - Comprehensive conversation engine template auto-loaded for new scenarios
   - 6 conversation stages (outbound → start → qualification → full qualification → meeting booking → closure)
   - 11 data collection slots (industry, role, interest_trigger, email, etc.)
   - 4 action triggers (product_request, meeting booking, stop_script, handover)
   - Strict communication rules (1 question/message, 2 sentences max, character limits)
   - Follow-up sequences for silence handling
   - 5 specialized system prompts

5. **Interactive Scenario Preview**
   - Step-by-step conversation simulator
   - Variable state tracking
   - Supports multiple step types (message, question_text, question_choice, form, end)
   - Test scenarios before publishing

6. **LLM Enhancement (3 Types)**
   - **Prompt-level**: Individual system prompt enhancement
   - **JSON-level**: Entire configuration enhancement (structure, consistency, rules)
   - **Full scenario**: Holistic enhancement including prompts + config

**Acceptance Criteria:**
- ✅ Admin creates scenario with name and description
- ✅ Default comprehensive template auto-loaded
- ✅ JSON editor with real-time validation (entry_step_id, steps, references)
- ✅ System prompts editor with templates and AI enhancement
- ✅ File upload component with workspace isolation
- ✅ Interactive preview simulator
- ✅ Three levels of AI enhancement via OpenRouter API
- ✅ Save as draft functionality
- ✅ Publish to make scenario active
- ✅ Widget uses latest published scenario
- ✅ Sidebar navigation integrated
- ✅ Error handling and success notifications

**Frontend Files:**
- Main page: `/src/pages/ScenarioConfiguration/ScenarioConfiguration.jsx` (450+ lines)
- JSON editor: `/src/pages/ScenarioConfiguration/components/JsonEditor.jsx`
- System prompts: `/src/pages/ScenarioConfiguration/components/SystemPromptEditor.jsx`
- File attachment: `/src/pages/ScenarioConfiguration/components/FileAttachment.jsx`
- Preview: `/src/pages/ScenarioConfiguration/components/ScenarioPreview.jsx`
- API service: `/src/services/api.js` (updated with scenarios, KB, LLM, file endpoints)
- Routing: `/src/App.jsx` (added /scenarios route)
- Navigation: `/src/layout/Sidebar/Sidebar.jsx` (added Scenarios menu item)

**Backend Files:**
- API endpoints: `/app/api/v1/scenarios.py` (list, create, get, update, publish)
- LLM enhancement: `/app/api/v1/llm.py` (OpenRouter integration with 3 enhancement types)
- File upload: `/app/api/v1/files.py` (upload with workspace isolation, delete)
- Router registration: `/app/main.py` (llm and files routes registered)
- Models: `/app/models/scenario.py` (existing)
- Schemas: `/app/schemas/scenario.py` (existing)

**Documentation:**
- User guide: `/docs/SCENARIO_CONFIGURATION_GUIDE.md`
- Template reference: `/docs/DEFAULT_SCENARIO_TEMPLATE.md`
- Technical spec: `/docs/AI Conversation Engine – Technical Specification/config script template.md`

**API Endpoints:**
```
POST   /v1/scenarios              - Create scenario
GET    /v1/scenarios              - List scenarios
GET    /v1/scenarios/{id}         - Get scenario details
PATCH  /v1/scenarios/{id}         - Update scenario
POST   /v1/scenarios/{id}/publish - Publish scenario

POST   /v1/llm/enhance            - Enhance prompt or config
POST   /v1/llm/generate-config    - Generate scenario from description

POST   /v1/files/upload           - Upload file
DELETE /v1/files/{id}             - Delete file
```

**Environment Variables Required:**
```env
# Backend
OPENROUTER_API_KEY=your-openrouter-api-key
UPLOAD_DIR=./uploads

# Frontend
VITE_API_BASE_URL=http://localhost:8000
```

**Default Template Structure:**
```json
{
  "version": "1.0",
  "bot_id": "sellrise_conversation_engine",
  "rules": {
    "one_question_rule": true,
    "max_sentences": 2,
    "max_chars": { "default": 420, "outbound": 520, "followup": 520 },
    "facts_only": true,
    "forbid": ["I think", "probably", "maybe"],
    "emoji": { "allowed": false }
  },
  "slots_schema": { /* 11 slots */ },
  "actions_catalog": { /* 4 actions */ },
  "stages": [ /* 6 stages with tasks */ ],
  "followups": [ /* 2 followup sequences */ ],
  "system_prompts": { /* 5 prompts */ },
  "llm_config": {
    "model": "anthropic/claude-3.5-sonnet",
    "temperature": 0.7,
    "max_tokens": 2000
  }
}
```

**Dependencies:** Workspace Management, RBAC (Admin-only), OpenRouter API

---

### 4.2 Scenario Simulator/Preview

**Priority:** 🟡 MVP Phase 2  
**Category:** Scenario Engine  
**Status:** ✅ **IMPLEMENTED**

**Description:**  
Interactive preview simulator to test scenario flow before publishing, integrated into the Scenario Configuration interface.

**User Story:**  
> **As an Admin**, I can simulate a draft scenario to verify the flow works correctly before publishing.

**Features Implemented:**
- Interactive conversation preview panel
- Step-by-step execution through scenario
- Variable state tracking and display
- Supports multiple step types:
  - `message`: Display bot messages
  - `question_text`: Free-text input with validation
  - `question_choice`: Multiple choice selection
  - `form`: Multi-field forms
  - `end`: Conversation termination
- Real-time conversation history display
- Variable inspector showing current state
- Reset functionality to restart simulation

**Acceptance Criteria:**
- ✅ Simulator integrated in scenario configuration page
- ✅ Shows current step UI correctly
- ✅ Displays conversation history
- ✅ Tracks and displays variables/state
- ✅ Admin provides inputs and advances through flow
- ✅ Handles validation (required fields, formats)
- ✅ Shows error messages for invalid inputs
- ✅ Reset button restarts simulation
- ✅ Uses same rendering logic as production widget

**Implementation Details:**
- Component: `/src/pages/ScenarioConfiguration/components/ScenarioPreview.jsx`
- Integrated in main scenario configuration page
- Real-time preview as configuration is edited
- No backend required (client-side simulation)
- Compatible with both simple and stage-based scenarios

**UI Components:**
- Simulator panel with conversation display
- Current step renderer
- Variable inspector sidebar
- Input controls for each step type
- Reset and navigation controls

**Step Type Handling:**
```javascript
// Supported step types
- message: Display text, auto-advance
- question_text: Text input with validation
- question_choice: Button choices
- form: Multi-field with validation
- end: Show completion message
```

**Dependencies:** Scenario Configuration (4.1)

---

### 4.3 Step Types Implementation

**Priority:** 🔴 MVP Phase 1  
**Category:** Scenario Engine (Frontend)

**Description:**  
Core step types for conversation flow.

**Step Types:**
- `message`: Display text to visitor
- `question_text`: Free-text input
- `question_choice`: Multiple choice
- `form`: Multi-field form
- `conditional_branch`: Logic branching based on variables
- `kb_search`: Knowledge base lookup
- `handoff_booking_link`: Show booking CTA
- `handoff_operator`: Transfer to human (future)
- `end`: Terminate conversation

**Requirements:**
- Render each step type in widget
- Validate inputs (email, phone, required fields)
- Store answers in session state
- Progress to next step deterministically

**Acceptance Criteria:**
- ✅ Each step type renders correctly
- ✅ Validation prevents invalid progression
- ✅ Answers stored and accessible
- ✅ Same inputs → same next step (deterministic)

**Dependencies:** Widget Session, Scenario Configuration

---

### 4.4 Event Logging

**Priority:** 🔴 MVP Phase 1  
**Category:** Scenario Engine (Backend)

**Description:**  
Append-only log of all conversation interactions.

**User Story:**  
> **As an Agent**, I see a correct transcript in the CRM.

**Event Types:**
- `widget_opened`
- `chat_started`
- `step_completed`
- `lead_submitted`
- `booking_link_shown`
- `booking_link_clicked`
- `stage_changed`
- `owner_assigned`
- `note_added`

**Requirements:**
- Log every significant interaction
- Store in `lead_events` table
- Include event_data (JSON payload)
- Ordered by timestamp
- Never delete (append-only)

**Acceptance Criteria:**
- ✅ Widget logs events via API
- ✅ Events include enough data to render transcript
- ✅ Transcript rendered from events ordered by created_at
- ✅ Deleting events (if ever) removes from transcript

**API Endpoint:**
```
POST /v1/widget/event
Body: { session_id, event_type, event_data, ts_client }
```

**Dependencies:** Widget Session

---

### 4.5 Centralized Conversation & Message System

**Priority:** 🔴 MVP Phase 1  
**Category:** Conversation Infrastructure  
**Status:** ✅ **IMPLEMENTED**

**Description:**  
Centralized conversation management system that ensures all lead conversations across all channels (web, WhatsApp, Telegram, Instagram, etc.) are processed by a single AI "brain" per workspace, with complete workspace isolation to prevent context mixing between admins.

**User Story:**  
> **As an Admin**, I want all my lead conversations to be handled by one consistent AI persona, regardless of which channel they use, and I never want my conversations mixed with other admins' workspaces.

**Architecture Principles:**

1. **One Brain Per Project (Workspace)**
   - Each workspace has ONE published scenario (the "brain")
   - All conversations in that workspace use the same scenario
   - Consistent AI behavior across all channels

2. **Workspace Isolation (Critical)**
   - Every conversation, message, and lead scoped to workspace_id
   - Database queries enforce workspace checks
   - Context cannot leak between workspaces
   - One admin's data never visible to another

3. **Channel-Agnostic Processing**
   - Same AI handles web, WhatsApp, Telegram, Instagram, etc.
   - Channel tracked but doesn't change logic
   - Consistent persona across all channels

4. **Stateful Conversations**
   - Full context maintained (slots, stage, task, variables)
   - State persists across messages
   - No context lost between messages

**Database Schema:**

Conversations Table:
- `workspace_id` (FK, indexed) - Workspace isolation
- `lead_id` (FK, indexed) - Which lead
- `scenario_id` (FK) - The "brain" processing this conversation
- `channel` (indexed) - web, whatsapp, telegram, instagram, messenger, email
- `channel_identifier` - Phone number, username, etc.
- `status` (indexed) - active, snoozed, closed, handed_off
- `current_stage_id` - Current stage in flow
- `current_task_id` - Current task being executed
- `context` (JSONB) - Full conversation state (slots, variables, used_phrases, etc.)
- `last_message_at`, `last_user_message_at`, `last_bot_message_at` - Timestamps
- `message_count` - Total messages in conversation

Messages Table:
- `conversation_id` (FK, indexed) - Parent conversation
- `role` (indexed) - user, assistant, system
- `content` - Message text
- `external_message_id` - Channel-specific ID (WhatsApp/Telegram/etc.)
- `metadata` (JSONB) - stage_id, task_id, actions, attachments, processing_time
- `is_delivered`, `delivered_at`, `is_read`, `read_at` - Delivery tracking
- `is_error`, `error_message` - Error handling

**API Endpoints:**

Public (Widget):
- `POST /v1/widget/message` - Send message from lead, get bot response

Admin (Authenticated):
- `GET /v1/conversations` - List conversations (workspace-filtered)
- `GET /v1/conversations/{id}` - Get conversation with messages
- `PATCH /v1/conversations/{id}` - Update status or context
- `GET /v1/conversations/{id}/messages` - Get all messages

**Service Layer:**
- `get_or_create_conversation()` - One active conversation per lead+channel
- `add_message()` - Add message and update conversation state
- `get_conversation_messages()` - Retrieve messages with workspace check
- `update_conversation_status()` - Close, snooze, hand off
- `get_active_conversations()` - List for admin inbox

**Security Features:**
- ✅ Workspace isolation enforced at database, service, and API layers
- ✅ All queries include workspace_id checks
- ✅ Lead verification before conversation creation
- ✅ JWT tokens restrict to user's workspace
- ✅ 404 responses prevent data leakage

**Acceptance Criteria:**
- ✅ Models created: `Conversation`, `Message`
- ✅ Relationships added to `Workspace`, `Lead` models
- ✅ Service layer with workspace isolation
- ✅ API endpoints with authentication
- ✅ Widget endpoint for public message sending
- ✅ Router registered in main.py
- ✅ All conversations use workspace's published scenario
- ✅ Context maintained across messages
- ✅ Channel-agnostic processing (same brain for all channels)
- ✅ Admin cannot see conversations from other workspaces

**Files Implemented:**
- Models: `/app/models/conversation.py`, `/app/models/message.py`
- Service: `/app/services/conversation_service.py`
- API: `/app/api/v1/conversations.py` (admin), `/app/api/v1/widget.py` (updated)
- Schemas: `/app/schemas/conversation.py`
- Documentation: `/docs/CONVERSATION_SYSTEM_ARCHITECTURE.md`

**Next Steps:**
1. Create database migration: `alembic revision --autogenerate -m "add_conversations_and_messages"`
2. Run migration: `alembic upgrade head`
3. Implement scenario processor for full LLM integration
4. Test multi-channel flows (web → WhatsApp → Telegram)
5. Build admin conversation inbox UI

**Dependencies:** Workspace Management, Scenario Configuration, Lead Management

---

### 4.6 AI Conversation Engine (LLM-Powered)

**Priority:** 🟢 Post-MVP  
**Category:** Scenario Engine (Advanced)

**Description:**  
Stage-based AI dialogue system with LLM integration for dynamic responses.

**Architecture:**
1. Conversation State Store
2. Stage & Task Router
3. Prompt Assembler
4. LLM Executor
5. Action Dispatcher

**Stages:**
- Stage 0: Outbound Message
- Stage 1: Conversation Start
- Stage 2: Initial Qualification
- Stage 3: Full Qualification
- Stage 4: Meeting Booking
- Stage 5: Closure

**Requirements:**
- Config-driven stage/task definitions
- LLM generates responses based on script
- Structured JSON output from LLM
- Action tags trigger backend integrations
- Slot system stores conversation data
- Facts-only responses (no hallucination)

**Constraints:**
- One question per message
- Max 2 sentences
- Max character limits (420 default, 520 outbound/followup)
- Approved phrases only
- No emojis (configurable)

**Technical Notes:**
- Separates routing logic (backend) from response generation (LLM)
- Uses YAML config templates
- See: `AI Conversation Engine – Technical Specification/`

**Dependencies:** Scenario Engine (basic), Knowledge Base

---

## 5. Lead Management (Mini-CRM)

Detailed PRDs for each CRM functionality are available in `docs/crm-prds/README.md`.

### 5.1 Lead Creation & Deduplication

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM

**Description:**  
Create leads with idempotent deduplication logic.

**User Story:**  
> **As a Visitor**, I submit my contact details and they appear in the CRM.

**Dedup Rules:**
1. Normalize email (lowercase, trim) → `dedup_email`
2. Normalize phone (E.164 format) → `dedup_phone`
3. Match by `dedup_email` first, else `dedup_phone`, else create new

**Requirements:**
- Lead fields: name, email, phone, consent
- Qualification fields: procedure, budget, timeframe, location
- Source tracking: page_url, referrer, utm_*
- Auto-score on creation (HOT/WARM/COLD)

**Acceptance Criteria:**
- ✅ Visitor submits contact via widget
- ✅ Backend upserts lead by dedup rules
- ✅ Existing lead updated if match found
- ✅ New lead created if no match
- ✅ DB constraint prevents duplicate `dedup_email` in workspace
- ✅ Consent captured with timestamp

**API Endpoint:**
```
POST /v1/widget/lead
Body: { session_id, name, email, phone, consent, qualification_data }
Response: { lead_id, score, stage }
```

**Dependencies:** Widget Session, Event Logging

---

### 5.2 Lead Inbox (New/Unassigned)

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM (Frontend + Backend)

**Description:**  
Default view for new leads requiring attention.

**User Story:**  
> **As an Agent**, I see new/unassigned leads to process.

**Requirements:**
- Filter: stage='new' OR owner_id IS NULL
- List view with key fields
- Sorting by created_at (newest first)
- Click to open lead detail

**Display Fields:**
- Created date/time
- Name, email, phone
- Score badge (Hot/Warm/Cold)
- Procedure/interest
- Source (page_url or short domain)

**Acceptance Criteria:**
- ✅ Inbox shows new/unassigned leads only
- ✅ Sorted by newest first
- ✅ Displays score, contact, procedure
- ✅ Clicking row opens lead detail
- ✅ Pagination supported

**UI Design:**
- Table or card layout
- Color-coded score badges
- Quick actions (assign, view)

**Dependencies:** Lead Creation, Lead Detail

---

### 5.3 Pipeline (Kanban View)

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM (Frontend + Backend)

**Description:**  
Visual pipeline for managing leads by stage.

**User Story:**  
> **As an Agent**, I manage leads by stage visually.

**Stages (Columns):**
- NEW
- QUALIFIED
- CONTACTED
- BOOKED
- WON
- LOST

**Requirements:**
- Drag-and-drop to change stage
- Dropdown stage selector (mobile)
- Stage change logs event
- Viewer role cannot change stage

**Acceptance Criteria:**
- ✅ Stages displayed as vertical columns
- ✅ Leads draggable between columns
- ✅ Stage change updates lead.stage
- ✅ Logs stage_changed event with old/new values
- ✅ Viewer cannot change stage (403)

**API Endpoint:**
```
PATCH /v1/leads/{id}
Body: { stage: "qualified" }
```

**Dependencies:** Lead Creation, RBAC

---

### 5.4 Lead Detail View

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM (Frontend + Backend)

**Description:**  
Comprehensive view of individual lead with actions.

**User Story:**  
> **As an Agent**, I review lead details and take actions.

**Display Sections:**
1. **Contact Card**: Name, email, phone, preferred contact method, timezone
2. **Qualification**: Procedure, budget, timeframe, location, score
3. **Pipeline**: Current stage, owner
4. **Source**: Page URL, referrer, UTM parameters
5. **Transcript**: Conversation timeline from events
6. **Notes**: Agent notes (chronological)
7. **Tags**: Lead categorization

**Actions:**
- Assign owner (self or other agent)
- Change stage
- Add note
- Add/remove tags
- Send follow-up (future)

**Acceptance Criteria:**
- ✅ All lead data displayed
- ✅ Transcript rendered from events (ordered)
- ✅ Actions log appropriate events
- ✅ Notes visible to all workspace users
- ✅ Tags searchable

**API Endpoints:**
```
GET    /v1/leads/{id}
PATCH  /v1/leads/{id}
POST   /v1/leads/{id}/notes
PATCH  /v1/leads/{id}/tags
```

**Dependencies:** Lead Creation, Event Logging, Notes

---

### 5.5 Lead Search & Filters

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM (Backend)

**Description:**  
Search and filter leads by multiple criteria.

**User Story:**  
> **As an Agent**, I search/filter leads to find specific contacts.

**Filters:**
- Stage (dropdown)
- Score (dropdown: Hot/Warm/Cold)
- Owner (dropdown: users)
- Date range (created_at)
- Procedure/category

**Search:**
- Name (case-insensitive)
- Email (case-insensitive)
- Phone

**Requirements:**
- Combine multiple filters (AND logic)
- Pagination (offset/limit)
- Performance for thousands of leads

**Acceptance Criteria:**
- ✅ Filters applied via query params
- ✅ Search matches name/email/phone
- ✅ Results paginated
- ✅ Fast queries (<500ms for 10k leads)

**API Endpoint:**
```
GET /v1/leads?stage=new&score=hot&owner=user_id&search=john&limit=20&offset=0
```

**Dependencies:** Lead Creation

---

### 5.6 Lead Notes

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM

**Description:**  
Agent annotations and follow-up notes on leads.

**User Story:**  
> **As an Agent**, I add notes to track follow-ups and context.

**Requirements:**
- Create note with text content
- Display notes chronologically
- Show author and timestamp
- Edit/delete own notes (optional)

**Acceptance Criteria:**
- ✅ Agent adds note to lead
- ✅ Note shows author, timestamp, content
- ✅ Notes listed in lead detail
- ✅ Logs note_added event

**API Endpoints:**
```
POST   /v1/leads/{id}/notes
GET    /v1/leads/{id}/notes
DELETE /v1/notes/{note_id}
```

**Dependencies:** Lead Detail, RBAC

---

### 5.7 Lead Assignment

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM

**Description:**  
Assign leads to team members for ownership.

**User Story:**  
> **As an Agent**, I assign leads to myself or other agents.

**Requirements:**
- Assign owner (user_id)
- Unassign (set owner_id to null)
- Log owner_assigned event
- Filter by assigned owner

**Acceptance Criteria:**
- ✅ Agent assigns lead to user
- ✅ Logs owner_assigned event with old/new owner
- ✅ Lead detail shows current owner
- ✅ Inbox filters by unassigned

**API Endpoint:**
```
PATCH /v1/leads/{id}
Body: { owner_id: "user_uuid" }
```

**Dependencies:** Lead Detail, Users

---

### 5.8 Lead Scoring

**Priority:** 🔴 MVP Phase 1  
**Category:** CRM (Backend)

**Description:**  
Automated lead qualification scoring.

**Scoring Logic:**
- **HOT**: High budget + urgent timeframe + clear need
- **WARM**: Medium budget or longer timeframe
- **COLD**: Low budget or no clear timeline

**Requirements:**
- Score calculated on lead submission
- Re-score when qualification data updated
- Configurable scoring rules (future)

**Acceptance Criteria:**
- ✅ Lead scored automatically on creation
- ✅ Score stored in lead.score
- ✅ Logs score_assigned event
- ✅ Re-scoring when data changes

**Technical Notes:**
- Rule-based logic in `lead_scoring.py`
- Can be done via scenario conditional OR server-side service

**Dependencies:** Lead Creation

---

## 6. Knowledge Base

### 6.1 KB Articles (CRUD)

**Priority:** 🔴 MVP Phase 1  
**Category:** Knowledge Base

**Description:**  
Create and manage help articles for AI responses.

**User Story:**  
> **As an Admin**, I create/update articles for the knowledge base.

**Fields:**
- Title
- Body (Markdown)
- Category
- Tags (array)
- Language (default: EN)
- Updated_at, updated_by

**Acceptance Criteria:**
- ✅ Admin creates article with title, body, category
- ✅ Markdown rendering supported
- ✅ Tags for categorization
- ✅ Edit/delete supported
- ✅ Only Admin can edit (RBAC)
- ✅ Audit trail (updated_by, updated_at)

**API Endpoints:**
```
POST   /v1/kb/articles
GET    /v1/kb/articles
GET    /v1/kb/articles/{id}
PATCH  /v1/kb/articles/{id}
DELETE /v1/kb/articles/{id}
```

**Dependencies:** RBAC (Admin-only)

---

### 6.2 KB FAQs (CRUD)

**Priority:** 🔴 MVP Phase 1  
**Category:** Knowledge Base

**Description:**  
Manage frequently asked questions.

**User Story:**  
> **As an Admin**, I create/update FAQ items.

**Fields:**
- Question
- Answer
- Category
- Tags (array)
- Language

**Acceptance Criteria:**
- ✅ Admin creates FAQ with question/answer
- ✅ Edit/delete supported
- ✅ Tags for categorization
- ✅ Only Admin can edit
- ✅ Audit trail

**API Endpoints:**
```
POST   /v1/kb/faq
GET    /v1/kb/faq
GET    /v1/kb/faq/{id}
PATCH  /v1/kb/faq/{id}
DELETE /v1/kb/faq/{id}
```

**Dependencies:** RBAC (Admin-only)

---

### 6.3 KB Search (Full-Text)

**Priority:** 🔴 MVP Phase 1  
**Category:** Knowledge Base (Backend)

**Description:**  
Search articles and FAQs for visitor queries.

**User Story:**  
> **As a Visitor**, I get relevant answers from the knowledge base.

**Requirements:**
- Full-text search across articles and FAQs
- Ranking by relevance
- Return type (article/faq), title/question, snippet
- Empty result fallback: "Please leave contact details"

**Acceptance Criteria:**
- ✅ Widget queries KB via API
- ✅ Returns ranked results
- ✅ Includes snippet/preview
- ✅ Empty results return empty array
- ✅ Widget shows fallback on empty

**API Endpoint:**
```
GET /v1/widget/kb/search?q=cargo+bike+price&limit=5
Response: [
  { type: "article", id: "...", title: "...", snippet: "..." },
  { type: "faq", id: "...", question: "...", answer_preview: "..." }
]
```

**Technical Notes:**
- PostgreSQL full-text search (MVP)
- Use `ts_vector` and `ts_rank`
- Future: Elasticsearch or vector embeddings

**Dependencies:** KB Articles, KB FAQs

---

### 6.4 KB-Only Answer Mode (MVP Constraint)

**Priority:** 🔴 MVP Phase 1  
**Category:** Knowledge Base (Policy)

**Description:**  
Widget responses strictly from KB, no LLM free-form answers.

**Requirements:**
- Widget step type `kb_search` queries KB API
- Display search results as cards/links
- No LLM-generated answers in MVP
- Fallback to contact form if no results

**Acceptance Criteria:**
- ✅ Widget uses KB search for answers
- ✅ No free-form LLM responses
- ✅ Empty results → fallback message
- ✅ KB answers only (no hallucinations)

**Dependencies:** KB Search, Widget

---

## 7. Scenario/Bot Builder

### 7.1 Bot Builder UI (Visual Flow Editor)

**Priority:** 🟢 Post-MVP  
**Category:** Bot Builder (Frontend)

**Description:**  
Visual interface for designing conversation flows.

**Layout:**
- **Left Sidebar**: Stage tree (collapsible)
- **Center Canvas**: Visual flow diagram
- **Right Panel**: Properties/configuration
- **Bottom Console**: Testing simulator

**Features:**
- Drag-and-drop stages/tasks
- Visual connections between steps
- Inline configuration
- Real-time validation
- Testing console

**UI Components:**
- Stage nodes
- Task cards
- Action triggers
- Condition builders

**Technical Notes:**
- Use React Flow or custom canvas
- See: `Bot Builder — Pixel-Level UI Layout.md`

**Dependencies:** Scenario Configuration, Scenario Simulator

---

### 7.2 Stage Configuration

**Priority:** 🟢 Post-MVP  
**Category:** Bot Builder

**Description:**  
Configure conversation stages in visual editor.

**Stage Properties:**
- Stage name
- Stage description
- Entry condition (when to enter this stage)
- Required slots
- Fallback behavior
- Follow-up rules

**Acceptance Criteria:**
- ✅ Admin configures stage via UI
- ✅ Entry conditions validated
- ✅ Required slots listed
- ✅ Fallback defined

**Dependencies:** Bot Builder UI

---

### 7.3 Task Configuration

**Priority:** 🟢 Post-MVP  
**Category:** Bot Builder

**Description:**  
Configure individual tasks within stages.

**Task Properties:**
- Task name
- Task criteria (when to execute)
- Instruction (AI prompt)
- Approved phrases (allowed responses)
- Action tags (backend triggers)
- Slot updates

**Acceptance Criteria:**
- ✅ Admin configures task via UI
- ✅ Criteria validated
- ✅ Approved phrases listed
- ✅ Actions selected from catalog

**Dependencies:** Bot Builder UI, Stage Configuration

---

### 7.4 Action Catalog

**Priority:** 🟢 Post-MVP  
**Category:** Bot Builder

**Description:**  
Predefined backend actions triggered by conversation.

**Actions:**
- `#product_request`: Send product cards
- `#New_meeting_time`: Create calendar event
- `#send_case`: Send case study
- `#handover`: Transfer to human operator
- `#stop_script`: Stop automation

**Requirements:**
- Define action types and payloads
- Map actions to backend handlers
- Validate action payloads
- Log action execution

**Dependencies:** Bot Builder UI, Action Dispatcher

---

### 7.5 Template Library

**Priority:** 🟢 Post-MVP  
**Category:** Bot Builder

**Description:**  
Pre-built scenario templates for common use cases.

**Templates:**
- Lead qualification
- Product consultation
- Customer support
- Event registration
- Demo booking

**Requirements:**
- Admin selects template
- Template populates scenario config
- Customizable after import
- Save custom templates

**Dependencies:** Bot Builder UI, Scenario Configuration

---

## 8. Analytics & Reporting

### 8.1 Funnel Metrics Dashboard

**Priority:** 🔴 MVP Phase 1  
**Category:** Analytics (Backend + Frontend)

**Description:**  
Conversion funnel visualization.

**User Story:**  
> **As an Admin**, I see funnel metrics by date range.

**Metrics:**
- Widget opened
- Chat started
- Lead submitted
- Qualified (hot leads)
- Booking link clicked

**Requirements:**
- Date range filter (from/to)
- Workspace-scoped counts
- Conversion rates calculated in UI
- Visual funnel chart

**Acceptance Criteria:**
- ✅ Endpoint returns counts for date range
- ✅ All counts workspace-scoped
- ✅ UI calculates conversion rates
- ✅ Funnel chart displays drop-offs

**API Endpoint:**
```
GET /v1/analytics/funnel?from=2026-03-01&to=2026-03-31
Response: {
  widget_opened: 1500,
  chat_started: 1200,
  lead_submitted: 450,
  qualified_hot: 180,
  booking_link_clicked: 90
}
```

**Dependencies:** Event Logging

---

### 8.2 Lead Source Analytics

**Priority:** 🟡 MVP Phase 2  
**Category:** Analytics

**Description:**  
Track lead volume by source.

**Dimensions:**
- Page URL
- Referrer
- UTM source/medium/campaign
- Domain

**Requirements:**
- Group leads by source
- Date range filter
- Chart visualization

**Acceptance Criteria:**
- ✅ Shows lead count by source
- ✅ Supports date range
- ✅ Visualized as bar/pie chart

**Dependencies:** Lead Creation, Analytics Dashboard

---

### 8.3 Stage Drop-Off Analysis

**Priority:** 🟡 MVP Phase 2  
**Category:** Analytics

**Description:**  
Identify where conversations fail.

**Requirements:**
- Count step completions
- Calculate drop-off percentage per step
- Visualize as funnel or bar chart

**Acceptance Criteria:**
- ✅ Shows step completion rates
- ✅ Highlights high drop-off steps
- ✅ Helps optimize scenarios

**Dependencies:** Event Logging, Analytics Dashboard

---

### 8.4 CSV Export

**Priority:** 🔴 MVP Phase 1  
**Category:** Analytics

**Description:**  
Export lead data for external analysis.

**User Story:**  
> **As an Admin**, I export leads/funnel data to CSV.

**Export Fields:**
- Created date
- Name, email, phone
- Procedure, timeframe, budget, score
- Stage, owner
- UTM parameters, referrer, page_url

**Acceptance Criteria:**
- ✅ Export respects filters and date range
- ✅ Workspace-scoped only
- ✅ CSV formatted correctly
- ✅ Download triggered via API

**API Endpoint:**
```
GET /v1/leads/export?format=csv&from=2026-03-01&to=2026-03-31
```

**Dependencies:** Lead Search & Filters

---

## 9. Notifications & Jobs

### 9.1 Hot Lead Email Notification

**Priority:** 🔴 MVP Phase 1  
**Category:** Notifications

**Description:**  
Immediate email alert when hot lead submits.

**User Story:**  
> **As an Agent**, I receive email notifications for hot leads.

**Requirements:**
- Trigger on lead_submitted with score=hot
- Send to configured recipients (workspace setting)
- Include lead details and CRM link
- Retry on failure (3 attempts, exponential backoff)

**Email Content:**
- Lead name, email, phone
- Procedure, timeframe, budget
- Source (page_url)
- Direct link to lead detail

**Acceptance Criteria:**
- ✅ Email sent when hot lead submits
- ✅ Email includes lead details and CRM link
- ✅ Retry policy: 3 attempts with backoff
- ✅ Logs notification_sent or notification_failed event

**Technical Notes:**
- Use job queue (Redis + worker)
- Email service: SendGrid, AWS SES, or SMTP

**Dependencies:** Lead Creation, Lead Scoring

---

### 9.2 Lead Assignment Notification

**Priority:** 🟡 MVP Phase 2  
**Category:** Notifications

**Description:**  
Notify agent when lead assigned to them.

**Requirements:**
- Trigger on owner_assigned event
- Email to assigned agent
- Include lead summary and link

**Acceptance Criteria:**
- ✅ Agent receives email on assignment
- ✅ Email includes lead details
- ✅ Link to lead detail

**Dependencies:** Lead Assignment, Notifications

---

### 9.3 SLA Reminder (Optional MVP+)

**Priority:** 🟢 Post-MVP  
**Category:** Notifications

**Description:**  
Alert if hot lead untouched for threshold period.

**User Story:**  
> **As the System**, I remind operators if hot lead is untouched for X minutes.

**Requirements:**
- Configurable threshold (default: 30 min)
- "Untouched" = no operator events after lead creation
- Send reminder email
- Log sla_reminder event

**Acceptance Criteria:**
- ✅ Threshold configurable per workspace
- ✅ Reminder sent after threshold
- ✅ Only if no operator actions taken
- ✅ Logs event

**Dependencies:** Lead Creation, Notifications, Job Scheduler

---

## 10. Integrations

### 10.1 Calendar Integration (Future)

**Priority:** 🟢 Post-MVP  
**Category:** Integrations

**Description:**  
Sync bookings with external calendars.

**Providers:**
- Google Calendar
- Microsoft Outlook
- Calendly

**Requirements:**
- OAuth authentication
- Create/update/cancel events
- ICS file generation
- Two-way sync

**Dependencies:** Booking System (Native)

---

### 10.2 CRM Integration (Future)

**Priority:** 🟢 Post-MVP  
**Category:** Integrations

**Description:**  
Sync leads with external CRMs.

**Providers:**
- HubSpot
- Salesforce
- Pipedrive
- Custom via API

**Requirements:**
- OAuth or API key authentication
- Lead push on submission
- Field mapping configuration
- Bi-directional sync (optional)

**Dependencies:** Lead Creation

---

### 10.3 Messaging Channels (Future)

**Priority:** 🟢 Post-MVP  
**Category:** Integrations

**Description:**  
Omnichannel conversation support.

**Channels:**
- WhatsApp
- Instagram DM
- Facebook Messenger
- Telegram

**Requirements:**
- Channel authentication
- Unified conversation interface
- Message routing
- Channel-specific formatting

**Dependencies:** Conversation Engine

---

## 11. Settings & Administration

### 11.1 Workspace Settings

**Priority:** 🔴 MVP Phase 1  
**Category:** Settings

**Description:**  
Configure workspace-level settings.

**Settings:**
- Workspace name
- Branding (logo, colors)
- Notification recipients
- Timezone, language defaults
- Booking links (per workspace or per procedure)

**Acceptance Criteria:**
- ✅ Admin updates workspace settings
- ✅ Settings persisted in JSON field
- ✅ Changes reflected in widget/emails

**API Endpoint:**
```
PATCH /v1/workspaces/{id}/settings
```

**Dependencies:** Workspace Management

---

### 11.2 Team Management

**Priority:** 🔴 MVP Phase 1  
**Category:** Settings

**Description:**  
Manage workspace users and roles.

**Features:**
- Invite users (email invite)
- Assign/change roles
- Deactivate users
- View user activity

**Acceptance Criteria:**
- ✅ Admin invites users via email
- ✅ Admin changes user roles
- ✅ Admin deactivates users
- ✅ User list shows roles and status

**Dependencies:** RBAC, User Management

---

### 11.3 Branding Configuration

**Priority:** 🟡 MVP Phase 2  
**Category:** Settings

**Description:**  
Customize widget appearance per domain.

**Settings:**
- Primary color
- Logo (URL or upload)
- Welcome message
- Chat bubble icon
- Position (bottom-right, bottom-left, etc.)

**Acceptance Criteria:**
- ✅ Admin configures branding
- ✅ Widget reflects branding settings
- ✅ Per-domain customization

**Dependencies:** Domain Registration, Widget

---

### 11.4 Notification Settings

**Priority:** 🟡 MVP Phase 2  
**Category:** Settings

**Description:**  
Configure notification preferences.

**Settings:**
- Hot lead recipients (emails)
- Notification channels (email, SMS, Slack)
- Frequency (immediate, digest)
- SLA thresholds

**Acceptance Criteria:**
- ✅ Admin configures recipients
- ✅ Notifications sent to configured emails
- ✅ Channels enabled/disabled

**Dependencies:** Notifications

---

## 12. Booking System

### 12.1 Booking Link Handoff (MVP)

**Priority:** 🔴 MVP Phase 1  
**Category:** Booking

**Description:**  
Simple CTA with external booking link.

**User Story:**  
> **As a Visitor**, when I reach the booking step, I see a CTA button with link.

**Requirements:**
- Step type: `handoff_booking_link`
- Display text + button label (configurable)
- URL resolved by booking_link_key
- Log events: booking_link_shown, booking_link_clicked

**Acceptance Criteria:**
- ✅ Widget displays booking CTA
- ✅ Button opens external booking URL
- ✅ Logs booking_link_shown when rendered
- ✅ Logs booking_link_clicked when clicked
- ✅ Fallback message if URL missing

**Configuration Example:**
```json
{
  "type": "handoff_booking_link",
  "text": "Ready to schedule a consultation?",
  "button_label": "Book Now",
  "booking_link_key": "default_booking_link"
}
```

**Dependencies:** Scenario Configuration, Event Logging

---

### 12.2 Native Booking Calendar (Post-MVP)

**Priority:** 🟢 Post-MVP  
**Category:** Booking

**Description:**  
Internal calendar system with availability and slot booking.

**Entities:**
- TeamMember (operators, timezone, working hours)
- Service (consultation type, duration, buffers)
- Availability rules + exceptions
- Booking (lead_id, member_id, datetime, status, notes)
- Tokenized links (reschedule/cancel)

**Flow:**
1. Visitor selects timezone
2. Shows available date/time slots
3. Visitor confirms booking
4. Slot locked (2-5 min)
5. Confirmation email + ICS file
6. Operator notification
7. CRM updates stage to "Booked"

**Requirements:**
- Availability management
- Slot locking mechanism
- Email confirmations with ICS
- Reschedule/cancel via token link
- Timezone conversion
- Buffer time between bookings

**Technical Notes:**
- Implement slot locking (Redis TTL)
- ICS generation (RFC 5545)
- Calendar sync (Google/Outlook)

**Dependencies:** Booking Link Handoff, Calendar Integration

---

## Implementation Roadmap

### Phase 1: MVP Core (Months 1-2)

**Goal:** Basic functional platform

**Features:**
1. ✅ Multi-tenant architecture
2. ✅ Authentication & RBAC
3. ✅ Widget embed + session
4. ✅ Basic scenario engine (config-driven)
5. ✅ Lead creation + deduplication
6. ✅ Lead inbox + pipeline
7. ✅ Lead detail view
8. ✅ Knowledge Base (CRUD + search)
9. ✅ Event logging
10. ✅ Hot lead notifications
11. ✅ Basic analytics (funnel)
12. ✅ Booking link handoff

### Phase 2: MVP Polish (Month 3)

**Goal:** Enhanced usability

**Features:**
1. ✅ Scenario simulator/preview
2. ✅ Lead search & filters
3. ✅ Password reset
4. ✅ Branding configuration
5. ✅ Source analytics
6. ✅ Drop-off analysis
7. ✅ Assignment notifications
8. ✅ CSV export

### Phase 3: Advanced Features (Months 4-6)

**Goal:** Competitive differentiation

**Features:**
1. ✅ Bot Builder UI (visual flow editor)
2. ✅ AI Conversation Engine (LLM-powered)
3. ✅ Native booking calendar
4. ✅ Template library
5. ✅ CRM integrations
6. ✅ Advanced analytics

### Phase 4: Scale & Expand (Months 7+)

**Goal:** Enterprise-ready

**Features:**
1. ✅ Omnichannel (WhatsApp, Instagram)
2. ✅ Real-time operator takeover
3. ✅ Advanced automations
4. ✅ A/B testing scenarios
5. ✅ Custom reporting
6. ✅ White-label option
7. ✅ API marketplace

---

## Technical Dependencies

### Critical Path

```
Workspace Management
  → User Authentication
    → RBAC
      → Domain Registration
        → Widget Session
          → Scenario Configuration
            → Lead Creation
              → Lead Inbox/Pipeline
                → Lead Detail
```

### Parallel Tracks

**Track 1: Core Platform**
- Authentication
- RBAC
- Workspace Management

**Track 2: Widget**
- Widget Embed
- Session Handshake
- Event Logging

**Track 3: CRM**
- Lead Creation
- Inbox/Pipeline
- Detail View
- Search/Filters

**Track 4: Content**
- Knowledge Base Articles
- Knowledge Base FAQs
- KB Search

**Track 5: Intelligence**
- Scenario Configuration
- AI Conversation Engine (future)
- Lead Scoring

---

## Testing Requirements

### MVP Test Suite

**Unit Tests:**
- Scenario validation
- Lead deduplication logic
- Scoring rules
- KB search ranking
- JWT token generation/validation

**Integration Tests:**
- Widget session → events → lead creation
- RBAC permission enforcement
- Workspace isolation
- KB search integration

**End-to-End Tests:**
- Complete visitor flow (widget → lead submission)
- Agent workflow (inbox → detail → pipeline)
- Admin workflow (create scenario → publish)

**Manual QA Checklist:**
- ✅ Embed widget on test page
- ✅ Complete conversation flow
- ✅ Lead appears in CRM with correct transcript
- ✅ Hot lead triggers email
- ✅ Booking link clicked logged
- ✅ Workspace isolation verified
- ✅ Mobile responsive
- ✅ Cross-browser (Chrome, Safari, Firefox)

---

## Success Metrics

### Platform Metrics

- **Widget Load Time**: <3 sec
- **API Response Time**: <500ms (p95)
- **Uptime**: 99.9%
- **Database Performance**: <100ms queries (p95)

### Business Metrics

- **Widget Open Rate**: >15%
- **Conversation Start Rate**: >60%
- **Lead Submission Rate**: >30%
- **Hot Lead Rate**: >20%
- **Booking Click Rate**: >15%

### User Satisfaction

- **Agent Time to Response**: <5 min (hot leads)
- **Admin Setup Time**: <30 min (first scenario)
- **Widget Mobile Experience**: >4.5/5 rating

---

## Document Version History

- **v1.0** (2026-03-08): Initial comprehensive feature breakdown from all PRD sources

---

## Related Documents

- `sellrise.ai MVP — User Stories & Acceptance Criteria (Final).md`
- `**Product Vision: All-in-One Lead Chat.md`
- `AI Conversation Engine – Technical Specification/`
- `Bot Builder — Pixel-Level UI Layout.md`
- `ai chatbot ui ux.md`
- `list of screens.md`
- `Sellrise-BackEnd/copilot-instructions.md`
- `Sellrise-Front-End/copilot-instructions.md`
