# Sellrise AI-Focused PRD for Widget Chatbot

Version: 1.0  
Date: 2026-03-09  
Owner: Product + Engineering  
Status: Draft

---

## 1. Purpose

Define where and how Sellrise implements LLM and automation capabilities for the widget chatbot, while following the current production flow and architecture.

This PRD is implementation-focused and aligned to existing modules in:
- `Sellrise-BackEnd/app/api/v1/widget.py`
- `Sellrise-BackEnd/app/services/conversation_service.py`
- `Sellrise-BackEnd/app/services/step_processor.py`
- `Sellrise-BackEnd/app/services/llm_service.py`
- `Sellrise-BackEnd/app/api/v1/llm.py`

---

## 2. Scope

### In Scope

- AI behavior for `POST /v1/widget/message` (runtime chatbot replies).
- Retrieval-augmented responses from workspace Knowledge Base.
- Structured LLM output mapped to scenario stage/task and action tags.
- Automation tasks tied to conversation flow:
  - slot extraction and context updates
  - action dispatch triggers
  - follow-up scheduling
  - lead scoring triggers
  - hot-lead notifications
- Guardrails for workspace isolation, deterministic fallback, and safety.

### Out of Scope

- Omnichannel runtime expansion (WhatsApp, Instagram) for this document.
- Human live chat takeover UI (beyond action/tag emission).
- Full redesign of scenario editor UX.

---

## 3. Product Goal

Deliver a reliable widget chatbot that:
1. Responds with context-aware, KB-grounded answers.
2. Progresses leads through the configured scenario stages.
3. Triggers automation actions safely and deterministically.
4. Never breaks tenant isolation or baseline widget reliability.

---

## 4. Current Flow Baseline (Must Preserve)

### Widget startup and context setup

- `POST /v1/widget/session`
  - resolves domain to workspace
  - returns published scenario config
  - logs `widget_opened`

### Lead and event lifecycle

- `POST /v1/widget/event`
  - logs interaction events
- `POST /v1/widget/lead`
  - creates/deduplicates lead
  - stores consent
  - updates lead score
  - triggers hot-lead notification when threshold is met

### Chat message runtime

- `POST /v1/widget/message`
  - gets/creates conversation
  - stores user message
  - calls `process_widget_message(...)` orchestrator
  - builds prompt package via `app/utils/scenario_helpers.py::build_prompt_package`
  - calls `LLMService.generate_widget_runtime_response(...)`
  - validates structured output and applies deterministic fallback when needed

This PRD upgrades only the runtime AI path for widget messages without changing the external API contract shape.

---

## 5. AI and Automation Capability Map

| Capability | Type | Where Implemented | Notes |
|---|---|---|---|
| Scenario stage/task routing | Deterministic automation | `app/utils/scenario_helpers.py` + `app/services/widget_message_orchestrator.py` | Deterministic selection is executed before generation |
| Response generation | LLM | `app/services/llm_service.py` via OpenRouter | Runtime model: DeepSeek via OpenRouter |
| Prompt enhancement for admins | LLM (authoring time) | `app/api/v1/llm.py` | Already implemented |
| KB grounding | Retrieval + optional LLM synthesis | `widget.py` + `llm_service.py` | Honor strict KB-only mode |
| Slot extraction and context update | Structured LLM + validation | `app/services/widget_message_orchestrator.py` + `conversation_service.py` | Validated before DB write |
| Action trigger emission | Automation | `app/services/action_dispatcher.py` | Tags: `#product_request`, `#New_meeting_time`, `#stop_script`, `#handover` |
| Follow-up messages | Automation | scheduler/job module | Triggered by silence windows in scenario config |
| Lead score and notifications | Automation | existing lead scoring + notification services | Reuse existing hot-lead pipeline |

---

## 6. Target Runtime Architecture (Widget Chatbot)

### 6.1 Runtime Sequence for `POST /v1/widget/message`

1. Validate workspace and lead ownership.
2. `get_or_create_conversation` for lead+channel.
3. Persist incoming user message.
4. Load published scenario for conversation workspace.
5. Retrieve conversation history and context.
6. Evaluate deterministic stage/task eligibility.
7. Retrieve KB context (top N relevant article/FAQ snippets).
8. Build prompt package:
   - system rules
   - current stage/task instruction
   - slot state
   - relevant KB snippets
   - short message history
9. Call LLM with structured output contract.
10. Validate LLM response schema.
11. Apply allowed updates:
   - `reply_text`
   - `slots_update` (typed and validated)d
   - `stage_id` / `task_id` (only if valid in scenario)
   - `actions` (only from approved catalog)
12. Persist assistant message + metadata.
13. Dispatch actions asynchronously.
14. Return widget response payload.

### 6.2 Structured LLM Output Contract

LLM runtime must return JSON with at least:

```json
{
  "reply_text": "string",
  "stage_id": "string",
  "task_id": "string",
  "slots_update": {},
  "actions": [
    {
      "tag": "#product_request",
      "payload": {}
    }
  ],
  "confidence": 0.0,
  "needs_human": false,
  "kb_citations": []
}
```

Rules:
- No free-form action names.
- `actions[].tag` must exist in scenario `actions_catalog`.
- `stage_id` and `task_id` must be valid references in active scenario config.
- If schema invalid, fallback to deterministic safe response.

### 6.3 Fallback Hierarchy

When LLM or parsing fails:
1. Return deterministic stage-approved fallback phrase.
2. If no phrase available, use safe generic response.
3. Log failure event with reason for observability.
4. Never drop or lose user message.

### 6.4 LLM Provider Policy (Mandatory)

- All runtime LLM inference for widget chatbot must execute in backend only.
- Provider gateway is OpenRouter (no direct browser-to-model calls).
- Default runtime model is DeepSeek via OpenRouter (`deepseek/deepseek-chat`).
- Model selection is controlled server-side and must not be overrideable from widget client payload.
- If DeepSeek is temporarily unavailable, backend uses configured server-side fallback model and logs provider/model switch.

---

## 7. Implementation Plan (Where to Build)

### 7.1 Backend Endpoints

- Update `Sellrise-BackEnd/app/api/v1/widget.py` in `widget_message`:
  - replace placeholder response path with orchestrator call
  - keep response model `WidgetMessageResponse`

### 7.2 New Service Layer

Create `Sellrise-BackEnd/app/services/widget_message_orchestrator.py`:
- `process_widget_message(...)` entrypoint
- scenario loading and validation
- stage/task resolution
- KB retrieval function
- prompt assembly
- LLM execution + schema validation
- fallback decisioning
- action dispatch hooks

### 7.3 Existing Services Integration

- `Sellrise-BackEnd/app/services/conversation_service.py`
  - ensure message metadata can store `actions`, `slots_update`, `kb_citations`, `latency_ms`

- `Sellrise-BackEnd/app/services/llm_service.py`
  - add method for strict structured JSON generation
  - enforce low-temperature mode for factual/KB-only answers
  - add OpenRouter provider path as default runtime execution path
  - set default runtime model to `deepseek/deepseek-chat`

- `Sellrise-BackEnd/app/api/v1/llm.py`
  - set default model constant to DeepSeek on OpenRouter
  - keep model value configurable only through backend environment/config

- `Sellrise-BackEnd/app/services/step_processor.py`
  - reuse validation and deterministic routing logic
  - avoid duplicate branching logic in orchestrator

### 7.4 Action Dispatcher

Create `Sellrise-BackEnd/app/services/action_dispatcher.py`:
- map tags to async handlers
- enforce allowlist from scenario config
- log `action_dispatched` / `action_failed` events

### 7.5 Follow-Up Automation

Create or extend jobs module for follow-up scheduling:
- use scenario-defined silence windows
- cancel pending follow-up when user replies
- append automated follow-up as assistant message with metadata flag `is_followup=true`

---

## 8. LLM Policy and Guardrails

### 8.1 Tenant and Security

- Every data read and write requires `workspace_id` scoped query.
- Cross-workspace access returns 404.
- LLM prompt must only include data from current workspace.

### 8.2 KB and Truthfulness

- MVP runtime default: KB-grounded responses for factual questions.
- If KB evidence is missing, respond with approved uncertainty template and CTA.
- Do not invent metrics, pricing, or product claims.

### 8.3 Output Safety

- Response length and style must comply with scenario rules:
  - max sentences
  - max chars
  - one-question rule
  - forbidden phrases
- sanitize generated text before returning to widget.

### 8.4 Configuration and Secrets

- Required environment variable: `OPENROUTER_API_KEY` (backend).
- Optional environment variables:
  - `OPENROUTER_MODEL=deepseek/deepseek-chat`
  - `OPENROUTER_FALLBACK_MODEL=<server_defined_model>`
- API keys never exposed to frontend or widget embed script.

---

## 9. Non-Functional Requirements

- P95 `POST /v1/widget/message` latency: <= 2500 ms with LLM enabled.
- Fallback response success rate during provider outage: >= 99.9%.
- Invalid structured output parse rate: <= 2% after stabilization.
- Zero cross-workspace leakage incidents.
- Full traceability: all AI decisions logged with correlation IDs.

---

## 10. Analytics and Observability

Log and monitor these events:
- `widget_message_received`
- `widget_message_replied`
- `llm_called`
- `llm_failed`
- `llm_parse_failed`
- `fallback_used`
- `action_dispatched`
- `action_failed`
- `followup_scheduled`
- `followup_sent`

Metrics dashboard should include:
- LLM success/failure rate
- median and P95 latency
- fallback rate
- action execution success rate
- conversion to meeting/booked state

---

## 11. Delivery Phases

### Phase 1: Runtime Wiring (Must Have)

- Wire widget message endpoint to orchestrator.
- Deterministic stage/task selection + safe fallback.
- Basic LLM response generation with strict schema validation.
- Persist enriched message metadata.

### Phase 2: KB-First Intelligence

- Implement retrieval pipeline for article/FAQ grounding.
- Enforce KB-only mode option per scenario.
- Add citation metadata in message records.

### Phase 3: Automation Expansion

- Action dispatcher and execution logging.
- Follow-up scheduler based on silence criteria.
- lead scoring trigger refinement from AI-derived events.

### Phase 4: Optimization and Controls

- latency/cost optimization
- configurable model policy by workspace
- prompt/version experiment support (A/B)

---

## 12. Acceptance Criteria (BDD)

### AC-1 Widget runtime AI response

- Given a valid widget message request
- When the message is processed
- Then the system returns a non-placeholder response generated from scenario + context
- And assistant message is persisted with stage/task metadata

### AC-2 Safe fallback

- Given the LLM provider is unavailable or output is invalid
- When processing widget message
- Then system returns deterministic fallback response
- And records `fallback_used` with failure reason

### AC-3 Workspace isolation

- Given lead or conversation from a different workspace
- When accessed through widget runtime
- Then API returns 404 and no data is exposed

### AC-4 Action allowlist enforcement

- Given LLM proposes an action tag not defined in `actions_catalog`
- When validating output
- Then action is rejected and logged
- And no side effect is executed

### AC-5 KB-only compliance

- Given scenario is configured with KB-only strict mode
- When user asks a factual question outside available KB
- Then bot responds with approved uncertainty template
- And does not hallucinate unsupported claims

### AC-6 Backend OpenRouter DeepSeek enforcement

- Given a widget message request
- When runtime LLM inference is executed
- Then the call is made from backend to OpenRouter
- And the selected model is DeepSeek by default
- And no client-side payload can override provider or model

---

## 13. Testing Strategy

- Unit tests:
  - structured output validator
  - stage/task resolver
  - fallback selector
  - action allowlist
- Integration tests:
  - `POST /v1/widget/message` success path
  - provider failure path
  - KB-only mode behavior
  - cross-workspace access checks
  - OpenRouter DeepSeek path assertion (provider/model in runtime metadata)
- Load tests:
  - concurrent widget message throughput
  - P95 latency under realistic prompt/context sizes

---

## 14. Open Decisions

- exact retry policy for transient LLM/network failures
- whether to store full prompt/response bodies or redacted traces only
- whether follow-up jobs run in existing scheduler or separate worker queue

---

## 15. Definition of Done

- Widget runtime uses orchestrator (no placeholder path remains).
- All acceptance criteria pass in automated test suite.
- Metrics and logs visible in operations dashboard.
- Rollback switch available to deterministic fallback-only mode.
- Documentation updated in:
  - `docs/CONVERSATION_SYSTEM_ARCHITECTURE.md`
  - `docs/STEP_TYPES_IMPLEMENTATION.md`
  - this PRD (`docs/AI_WIDGET_CHATBOT_PRD.md`)

---

## 16. Implementation Audit — 2026-03-10

This section records the current implementation status against this PRD so the next engineer can quickly understand what is already live, what is partially implemented, and what still blocks the intended CRM/widget behavior.

### 16.1 Already aligned with this PRD

The following items are already implemented and broadly aligned with the target architecture:

- `POST /v1/widget/message` is already wired to a backend orchestrator service and no longer returns a static placeholder response.
- `SellriseBE/app/services/widget_message_orchestrator.py` already performs:
  - published scenario loading
  - deterministic stage/task resolution
  - KB retrieval
  - prompt assembly
  - OpenRouter-backed LLM execution
  - structured output validation
  - deterministic fallback handling
  - action dispatch invocation
- `SellriseBE/app/services/llm_service.py` already defaults widget runtime inference to OpenRouter with DeepSeek (`deepseek/deepseek-chat`) and supports a server-side fallback model.
- `SellriseBE/app/api/v1/llm.py` already uses the backend OpenRouter configuration and defaults to the configured DeepSeek model.
- `SellriseBE/app/services/conversation_service.py` already persists assistant message metadata including `stage_id`, `task_id`, and `slots_update` into conversation context.
- `SellriseBE/app/services/action_dispatcher.py` exists and enforces action allowlisting against `actions_catalog`.
- `SellriseBE/app/services/followup_scheduler.py` exists and can schedule/cancel follow-up messages with `is_followup=true` metadata.
- Integration tests already exist for the major runtime AI path (`tests/test_widget_message.py`) including success path, fallback path, KB strict mode, and action allowlist enforcement.

### 16.2 Partially aligned / still missing

The following areas are not yet fully aligned with the intent of this PRD:

1. **Lead enrichment from runtime conversation is incomplete**
  - The orchestrator stores `slots_update` only in conversation context / message metadata.
  - There is currently no backend step that maps extracted slots such as `procedure`, `budget_range`, `timeframe`, `preferred_channel`, or `timezone` back into the `leads` table after chat messages are processed.

2. **Frontend widget does not submit qualification fields when creating a lead**
  - The browser widget currently sends only minimal contact payload (`name`, `email`, `phone`, `consent_given`) to `POST /v1/widget/lead`.
  - Backend schema/service support richer lead fields, but the widget client is not sending them.

3. **Scenario defaults do not target CRM qualification fields**
  - The default scenario configuration in frontend authoring currently focuses on slots such as `industry`, `role`, and `interest_trigger`.
  - It does not default to CRM-oriented slots like `procedure`, `budget_range`, `timeframe`, or `preferred_channel`, so runtime extraction is not naturally optimized for the lead table.

4. **Lead scoring is only partially fed by runtime chat**
  - Current score calculation can use `lead.procedure` / `lead.budget_range`, but runtime chat is not consistently writing those values back to the lead record.
  - The widget frontend also does not currently emit structured `answer_given` events for each qualification answer, so event-based scoring remains weaker than intended.

5. **Session-to-lead source propagation is incomplete**
  - The widget session endpoint captures source context such as domain, page URL, UTM, and timezone.
  - That context is not consistently propagated into the later lead creation payload, so some lead records can still end up with null source fields.

### 16.3 Root cause analysis — why lead fields are still null in CRM

Observed issue: lead rows are still not populating values such as `procedure`, `budget_range`, `timeframe`, `preferred_channel`, and related qualification data.

This happens because of a flow gap across multiple layers, not because the database schema is missing.

#### Root cause A — backend supports the fields, but widget lead creation does not send them

`WidgetLeadRequest` and `get_or_create_lead(...)` already support:

- `timezone`
- `preferred_channel`
- `procedure`
- `budget_range`
- `timeframe`

However, the current embeddable widget only submits:

- `workspace_id`
- `session_id`
- `name`
- `email`
- `phone`
- `consent_given`

As a result, `POST /v1/widget/lead` usually creates the lead without qualification fields, even though the backend would accept them.

#### Root cause B — runtime AI stores extracted slots only in conversation context

The runtime orchestrator already accepts `slots_update` from structured LLM output and merges it into `conversation.context.slots`.

But after that merge, there is no synchronization step that says:

- if `slots_update.procedure` exists, write it to `lead.procedure`
- if `slots_update.budget_range` exists, write it to `lead.budget_range`
- if `slots_update.timeframe` exists, write it to `lead.timeframe`
- if `slots_update.preferred_channel` exists, write it to `lead.preferred_channel`
- if `slots_update.timezone` exists, write it to `lead.timezone`

So the conversation can become richer while the CRM lead row remains null.

#### Root cause C — default scenario slot schema is not aligned to lead qualification columns

The default scenario authoring config currently defaults to slots such as:

- `industry`
- `role`
- `interest_trigger`
- `acquisition_channels`
- `meeting_datetime_candidate`

This means the default AI/runtime flow is biased toward generic discovery, not CRM lead qualification fields. Even if chat works well, the extracted data may not correspond to the columns displayed in lead management.

#### Root cause D — no integration test currently protects this requirement end-to-end

Current tests validate AI response, fallback behavior, KB strict mode, and action allowlist behavior.

There is not yet a dedicated integration test asserting this full sequence:

1. user answers qualification questions in widget chat
2. orchestrator extracts `procedure` / `budget_range` / `timeframe`
3. lead row is updated in `leads`
4. score is recomputed from the updated lead data and/or emitted answer events

Because that regression test is missing, this gap can remain unnoticed while runtime chat itself appears healthy.

### 16.4 Required fixes

The following fixes should be treated as implementation work required to fulfill the CRM outcome expected by this PRD.

#### Fix 1 — add lead synchronization after `slots_update`

Add a backend synchronization step in `SellriseBE/app/services/widget_message_orchestrator.py` (or a dedicated helper/service) that:

- loads the current lead by `lead_id` and `workspace_id`
- maps known slot keys to lead columns
- updates only allowed lead fields
- recomputes lead score after qualification updates
- optionally logs `answer_given` or equivalent qualification events for observability

Recommended mapping for MVP:

| Slot key | Lead column |
|---|---|
| `procedure` | `lead.procedure` |
| `budget_range` | `lead.budget_range` |
| `timeframe` | `lead.timeframe` |
| `preferred_channel` | `lead.preferred_channel` |
| `timezone` | `lead.timezone` |
| `email` | `lead.email` (only if valid / empty or explicitly allowed) |

#### Fix 2 — expand widget client payload for lead creation

Update the embeddable widget client so `POST /v1/widget/lead` can submit:

- `timezone`
- `preferred_channel`
- `procedure`
- `budget_range`
- `timeframe`
- `domain`
- `page_url`
- `utm_source`
- `utm_medium`
- `utm_campaign`

At minimum, `timezone`, `domain`, `page_url`, and UTM fields should come from browser/session context even before the AI extraction path is completed.

#### Fix 3 — align default scenario slots with CRM requirements

Update the default scenario template / scenario authoring defaults so CRM-focused projects can collect the following slots by default when lead qualification is desired:

- `procedure`
- `budget_range`
- `timeframe`
- `preferred_channel`
- `timezone`

This does not mean removing `industry` / `role` / `interest_trigger`, but the lead qualification slots must be first-class if the CRM is expected to show them.

#### Fix 4 — add regression tests for lead field population

Add automated tests covering:

- `POST /v1/widget/lead` persists qualification fields when they are present in the request
- widget message runtime with `slots_update` writes mapped values into `leads`
- lead score changes after `procedure` / `budget_range` are captured
- source context (`domain`, `page_url`, `utm_*`, `timezone`) is preserved from widget session / lead submission

#### Fix 5 — keep documentation and implementation terminology consistent

Normalize naming between browser payload, backend schema, and runtime context. Current examples use both `session_id` and `session_token`; future implementation should standardize one canonical field name and document how session data is attached to lead creation/update.

### 16.5 Current conclusion for maintainers

Current runtime AI/chatbot work is **partially implemented and already much closer to the PRD than the original baseline text suggests**.

However, the CRM symptom in production/testing is expected today because:

- the widget frontend is not sending qualification/source fields during lead creation,
- runtime AI extraction is not syncing captured slots back into the `leads` table,
- and default scenario slots are not yet optimized for CRM qualification columns.

If future engineers only inspect the UI or only inspect the database schema, this issue is easy to miss. The real problem is the missing bridge between **conversation slot capture** and **persistent lead enrichment**.
