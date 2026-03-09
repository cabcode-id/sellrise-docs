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
  - currently returns placeholder bot response
  - TODO exists for scenario processor + LLM wiring

This PRD upgrades only the runtime AI path for widget messages without changing the external API contract shape.

---

## 5. AI and Automation Capability Map

| Capability | Type | Where Implemented | Notes |
|---|---|---|---|
| Scenario stage/task routing | Deterministic automation | `app/services/step_processor.py` + new orchestrator | Keep deterministic selection before generation |
| Response generation | LLM | `app/services/llm_service.py` via OpenRouter | Runtime model: DeepSeek via OpenRouter |
| Prompt enhancement for admins | LLM (authoring time) | `app/api/v1/llm.py` | Already implemented |
| KB grounding | Retrieval + optional LLM synthesis | `widget.py` + `llm_service.py` | Honor strict KB-only mode |
| Slot extraction and context update | Structured LLM + validation | new orchestrator + `conversation_service.py` | Must validate schema before write |
| Action trigger emission | Automation | new action dispatcher service | Tags: `#product_request`, `#New_meeting_time`, `#stop_script`, `#handover` |
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
