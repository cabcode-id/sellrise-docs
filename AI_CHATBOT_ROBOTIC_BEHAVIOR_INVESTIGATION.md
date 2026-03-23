# AI Chatbot — Robotic Behavior & Scenario Drift Investigation

**Date:** 2026-03-23  
**Scope:** `Sellrise-BackEnd/` — full LLM pipeline from widget endpoint to response delivery  
**Symptoms reported:**
1. AI chatbot feels **robotic / unnatural**
2. Sometimes **does not follow the scenario** fully
3. Sometimes **spits out strange outputs**

---

## Executive Summary

After auditing the full runtime pipeline (`widget.py` → `widget_message_orchestrator.py` → `scenario_helpers.py` → `llm_service.py` → `llm_output.py`), **23 root causes** were identified across 6 categories. The most impactful issues are:

- The LLM is forced into **JSON-only mode** on every turn, making the model focus on structured fields rather than producing a natural reply.
- **Only the "main" prompt** from the scenario is injected; persona-specific prompts (e.g. `qualification`, `meeting_booking`) are **silently dropped** in the widget runtime path.
- Conversation **history is truncated to 200 chars per message** and capped at 6 messages, causing the model to lose vital context mid-conversation.
- There are **no anti-repetition, tone, or persona instructions** in the prompt, so the model falls back to its default dry assistant voice.
- **KB search is keyword-based** (`ILIKE`), and irrelevant results get injected into the prompt, confusing the model's grounding.

---

## Category 1: Prompt Engineering Deficiencies (Robotic Feel)

### 1.1 — Only `prompts.main` is included in the widget runtime system prompt

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()` (line ~430)

The scenario config supports multiple prompt sections (e.g. `main`, `qualification`, `meeting_booking`), but `build_prompt_package` only reads `prompts.main`:

```python
prompts = scenario_config.get("prompts", scenario_config.get("system_prompts", {}))
main_prompt = prompts.get("main", "")
if main_prompt:
    system_parts.append(main_prompt)
```

**Meanwhile**, `LLMService._build_system_prompt()` (used in the standalone path, not the widget runtime) correctly concatenates **all** prompt entries:

```python
ordered_keys = (["main"] if "main" in prompts else []) + [
    k for k in prompts if k != "main"
]
combined = "\n\n".join(prompts[k].strip() for k in ordered_keys if prompts.get(k, "").strip())
```

**Impact:** The widget chatbot never receives persona/tone prompts beyond `main`. Any custom voice defined in `qualification`, `meeting_booking`, etc. is silently discarded.

### 1.2 — No persona, tone, or language instructions

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()`

The default fallback system prompt is:

```python
"You are a helpful AI assistant for lead qualification and customer support. "
"Be professional, concise, and helpful."
```

There are **no instructions** for:
- Conversational tone or warmth
- Required language/locale (the bot may reply in English when the user writes in Indonesian, or vice versa)
- Personality traits (friendly, empathetic, casual, etc.)
- How to handle greetings or small talk naturally

**Impact:** The model defaults to its generic "assistant" personality — dry, formal, and robotic.

### 1.3 — No anti-repetition instructions

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()`

The `conversation.context` tracks `used_phrases` and `used_tags`, but:
- These arrays are **never populated** by the orchestrator — they stay empty `[]` forever.
- They are **never included** in the prompt.
- There is **no "do not repeat yourself"** instruction in the system prompt.

**Impact:** The bot repeats the same phrasing across turns, making it feel like a script rather than a dynamic conversation.

### 1.4 — No few-shot examples of natural replies

**File:** `app/schemas/llm_output.py` — `LLM_OUTPUT_SCHEMA_PROMPT`

The schema prompt tells the model **what structure** to output but gives **zero examples** of what a natural `reply_text` looks like:

```python
LLM_OUTPUT_SCHEMA_PROMPT = """You MUST respond with valid JSON matching this exact schema:
{
  "reply_text": "Your response text to the user",
  ...
}
```

**Impact:** Without concrete examples of warm, human-like replies, the model produces mechanical JSON-focused text.

### 1.5 — Approved phrases are underutilized

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()` (line ~475)

```python
if approved_phrases:
    system_parts.append(f"Reference phrases (adapt naturally): {json.dumps(approved_phrases)}")
```

Dumping a JSON array and saying "adapt naturally" is weak guidance. The model often ignores this or quotes the phrases literally.

---

## Category 2: Forced JSON Output Mode (Strange Outputs)

### 2.1 — LLM is always called with `json_mode=True`

**File:** `app/services/widget_message_orchestrator.py` — `_call_llm_with_fallback()` (line ~636)

```python
raw_result = await llm_service.generate_widget_runtime_response(
    system_prompt=system_prompt,
    user_prompt=user_prompt,
    max_tokens=settings.OPENROUTER_MAX_TOKENS,
    temperature=settings.LLM_TEMPERATURE,
    json_mode=True,  # ← ALWAYS forced
)
```

**Impact:**
- The model must produce valid JSON on every single turn, even for simple greetings.
- When the model's JSON generation fails (brackets mismatch, trailing comma, etc.), the fallback chain kicks in and may produce **raw text fragments or generic fallback** — which are the "strange outputs."
- `json_mode` with DeepSeek models is less reliable than with GPT-4 — DeepSeek sometimes wraps JSON in markdown code blocks or adds preamble text before the JSON.

### 2.2 — Fallback accepts raw non-JSON text as reply

**File:** `app/services/widget_message_orchestrator.py` — `_extract_text_from_raw()` (line ~723)

```python
# If it's just plain text (not JSON at all), use it directly
stripped = raw_content.strip()
if stripped and not stripped.startswith("{") and len(stripped) < 2000:
    return stripped
```

When JSON parsing fails, this function returns literally **whatever the LLM outputted** — which could be partial JSON, self-debugging text, or schema error messages from the model. This is a direct source of "strange outputs."

### 2.3 — Markdown code fence cleanup is incomplete

**File:** `app/schemas/llm_output.py` — `validate_llm_output()` (line ~81)

```python
if content.startswith("```json"):
    content = content[7:]
if content.startswith("```"):
    content = content[3:]
if content.endswith("```"):
    content = content[:-3]
```

This only handles the specific patterns `` ```json `` and `` ``` `` at the very start/end. It does not handle:
- ```` ```JSON ```` (uppercase)
- Leading/trailing whitespace before the fence
- Models that emit `json\n{...}\n` without the backticks
- Nested code blocks or multiple code blocks

---

## Category 3: Context & History Truncation (Scenario Drift)

### 3.1 — Conversation history truncated to 200 chars per message

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()` (line ~523)

```python
for msg in history[-6:]:  # Last 6 messages for context
    role_label = "User" if msg["role"] == "user" else "Assistant"
    user_parts.append(f"  {role_label}: {msg['content'][:200]}")
```

**Impact:** If a bot response is 400+ chars (which the `max_tokens=1000` setting easily produces), the model only sees the **first half** of its own previous reply. It loses context about what it already said, what it already asked, and what it already promised — leading to repeated questions & scenario drift.

### 3.2 — Only 6 messages of history (3 turns)

The `history[-6:]` slice means only the last ~3 exchanges are visible. For multi-stage scenarios that span 10+ turns, the model has no memory of earlier stages. This is why it "doesn't follow the scenario fully" — it literally can't see what happened earlier.

### 3.3 — History fetched with `limit=10` but further sliced to 6

**File:** `app/services/widget_message_orchestrator.py` (line ~324):
```python
history = await _get_conversation_history(db, conversation.id, workspace_id, limit=10)
```
**File:** `app/utils/scenario_helpers.py` (line ~523):
```python
for msg in history[-6:]:
```

10 messages are fetched from DB but only 6 are used. This is wasteful and limits context.

---

## Category 4: Scenario Routing & State Machine Issues

### 4.1 — Hardcoded stage IDs in autofill logic

**File:** `app/services/widget_message_orchestrator.py` — `_autofill_slots_from_previous_turn()` (line ~233)

```python
if previous_stage_id == "1_first_response":
    if slots.get("procedure") is None:
        slots_update["procedure"] = user_message.strip()

if previous_stage_id == "3_collect_basic_info":
    extracted = _extract_basic_info_from_text(user_message)
```

These hardcoded stage IDs (`1_first_response`, `3_collect_basic_info`) only work for **one specific scenario**. If the workspace uses a different scenario with different stage IDs, this autofill logic is entirely dead code, and slot extraction fails silently.

**Impact:** For non-matching scenarios, the LLM is solely responsible for slot extraction via `slots_update` in its JSON output. If it misses a slot, the deterministic router can't advance — the bot gets **stuck** asking the same question.

### 4.2 — Task stays `None` when no criteria match

**File:** `app/utils/scenario_helpers.py` — `resolve_stage_and_task()` (line ~338)

When no task within a stage matches criteria, `selected_task` is `None` and `selected_task_id` is `None`. This means:
- No `task_instruction` is added to the prompt.
- The model is operating in a matched **stage** but with zero task-level guidance.
- The model doesn't know what specific information to collect next.

**Impact:** The bot produces generic filler responses instead of advancing the scenario.

### 4.3 — Stage priority ties are non-deterministic

**File:** `app/utils/scenario_helpers.py` — `resolve_stage_and_task()` (line ~305)

```python
sorted_stages = sorted(
    stages_dict.values(),
    key=lambda s: s.get("priority", 0),
    reverse=True,
)
```

If two stages share the same priority and both have their `entry_condition` met, Python's `sorted()` is stable but the source order from a `dict` depends on insertion order. This can cause **inconsistent stage selection** across restarts or scenario edits.

### 4.4 — `first_message` condition never re-fires after first assignment

**File:** `app/utils/scenario_helpers.py` — `resolve_stage_and_task()` (line ~310)

```python
if cond_type == "first_message":
    if is_first_message or current_stage_id is None:
        selected_stage = stage
        selected_stage_id = sid
        break
    continue
```

Once a `current_stage_id` is set, stages with `first_message` type are permanently skipped via `continue`. However, if the first message's slot extraction fails and `current_stage_id` is already set, the bot can never return to the greeting stage even if it should.

---

## Category 5: LLM Configuration & Model Issues

### 5.1 — Primary and fallback model are identical

**File:** `app/config.py` (line ~52)

```python
OPENROUTER_MODEL: str = "deepseek/deepseek-v3.2"
OPENROUTER_FALLBACK_MODEL: str = "deepseek/deepseek-v3.2"
```

There is no actual fallback. If DeepSeek has a bad response or JSON compliance issue, it retries with the same model, producing the same class of error.

### 5.2 — `max_tokens=1000` is excessive for chat messages

**File:** `app/config.py` (line ~53)

```python
OPENROUTER_MAX_TOKENS: int = 1000
```

For a chatbot that should send concise 1-3 sentence messages, 1000 tokens (~750 words) is far too generous. The model fills the token budget with lengthy, multi-paragraph responses that feel unnatural for a chat widget.

### 5.3 — `temperature=0.3` combined with no personality produces bland output

While 0.3 is good for factual KB answers, it produces very formulaic and repetitive phrasing for conversational turns. There's no mechanism to raise temperature for greeting/small-talk stages vs. lower it for factual/KB stages.

### 5.4 — Singleton LLM service never reloads config

**File:** `app/services/llm_service.py` — `get_widget_llm_service()` (line ~581)

```python
_widget_llm_service_instance = None

def get_widget_llm_service() -> LLMService:
    global _widget_llm_service_instance
    if _widget_llm_service_instance is None:
        provider = OpenRouterProvider()
        _widget_llm_service_instance = LLMService(provider=provider)
    return _widget_llm_service_instance
```

Once instantiated, the model, API key, and all settings are frozen for the lifetime of the process. Env var changes or model updates require a full server restart.

---

## Category 6: Knowledge Base & Grounding Issues

### 6.1 — KB search is naive keyword `ILIKE`, not semantic

**File:** `app/services/widget_message_orchestrator.py` — `retrieve_kb_context()` (line ~539)

```python
pattern = f"%{token}%"
# Search articles
articles_result = await db.execute(
    select(KBArticle)
    .where(
        KBArticle.workspace_id == workspace_id,
        KBArticle.is_published == True,
        or_(KBArticle.title.ilike(pattern), KBArticle.body.ilike(pattern)),
    )
    .limit(limit)
)
```

Each tokenized keyword is searched independently via `ILIKE`. This returns results that match **any single word** anywhere, leading to:
- **False positives**: irrelevant articles inject noise into the prompt.
- **False negatives**: semantically related content with different wording is missed.

**Impact:** When irrelevant KB content is injected, the model may reference it in its reply, producing answers that seem off-topic or "strange."

### 6.2 — KB snippets are truncated to 300 chars

```python
"snippet": body[:300].strip() + ("..." if len(body) > 300 else ""),
```

The model receives incomplete article content, sometimes cutting off mid-sentence. It may then hallucinate the rest or produce a half-answer.

### 6.3 — No relevance threshold — all KB results are injected

Every KB hit is included in the prompt regardless of match quality. There is no minimum score threshold or relevance filter. A single-token match (score=1) is treated the same as a multi-token match (score=5).

---

## Category 7: Booking Protocol Overrides (Edge Case)

### 7.1 — Booking protocol override is always injected when action exists in catalog

**File:** `app/utils/scenario_helpers.py` — `build_prompt_package()` (line ~485)

```python
if "#New_meeting_time" in catalog_dict:
    booking_rules = "\n\nCRITICAL Booking Protocol:\n"
    booking_rules += "- When a user wants to book, schedule, or mentions an appointment..."
    booking_rules += "- IMMEDIATELY ask for ALL missing information in ONE single message..."
    booking_rules += "- This protocol overrides any 'one question per message' or 'conciseness' rules..."
    system_parts.append(booking_rules)
```

This CRITICAL block is injected on **every single message** even when the user is not talking about booking. The aggressive phrasing ("IMMEDIATELY", "overrides", "CRITICAL") can confuse the model's behavior for unrelated conversation turns.

---

## Summary of Fixes (Priority Order)

| # | Issue | Severity | Fix |
|---|-------|----------|-----|
| 1 | Only `prompts.main` injected in widget runtime | **Critical** | Use the same all-prompts concatenation logic as `LLMService._build_system_prompt()` |
| 2 | JSON mode forced on every turn | **Critical** | Consider a 2-pass approach: natural text generation first, then structured extraction; or improve JSON prompt with concrete examples |
| 3 | History truncated at 200 chars / 6 messages | **High** | Increase to 400+ chars and 10-12 messages; use token budgeting instead of char limits |
| 4 | No persona/tone/language instructions | **High** | Add configurable persona block and language directive to system prompt |
| 5 | No anti-repetition mechanism | **High** | Track and inject `used_phrases` into prompt; add "vary your phrasing" instruction |
| 6 | Primary = fallback model | **High** | Set a meaningfully different fallback (e.g., `openai/gpt-4o-mini`) |
| 7 | `max_tokens=1000` too high | **Medium** | Reduce to 300-400 for chat; make it configurable per scenario stage |                  
| 8 | Hardcoded stage IDs in autofill | **Medium** | Make slot autofill generic using `_infer_slots_from_task_criteria` for all stages |
| 9 | KB search is keyword-only | **Medium** | Add PostgreSQL `tsvector` or embed-based semantic search |
| 10 | No relevance threshold for KB results | **Medium** | Add minimum `_score` threshold before including in prompt |
| 11 | Task `None` leaves model with no guidance | **Medium** | Add stage-level fallback instruction when no task matches |
| 12 | Booking protocol always injected | **Low** | Only inject when booking intent is detected or the current stage is booking-related |
| 13 | Raw text fallback accepts anything | **Low** | Add sanity checks: min length, no raw JSON fragments, language detection |
| 14 | Markdown fence cleanup incomplete | **Low** | Use regex to strip all code fence variants |
| 15 | Phrase variation tracking never used | **Low** | Implement `used_phrases` population in the orchestrator response loop |

---

## Appendix: File Reference Map

| File | Role |
|------|------|
| `app/api/v1/widget.py` | Widget REST endpoint — entry point for all chat messages |
| `app/services/widget_message_orchestrator.py` | Main runtime — ties scenario + KB + LLM together |
| `app/utils/scenario_helpers.py` | Stage/task routing, prompt building, tokenization |
| `app/services/llm_service.py` | LLM provider abstraction (OpenRouter/DeepSeek, OpenAI, Anthropic) |
| `app/schemas/llm_output.py` | Structured output contract & validation |
| `app/services/step_processor.py` | Deterministic step execution (message, form, branch — legacy path) |
| `app/services/conversation_service.py` | Conversation CRUD, initial context creation |
| `app/services/action_dispatcher.py` | Async action handler dispatch |
| `app/services/lead_service.py` | Slot → CRM sync, dedup |
| `app/config.py` | LLM config defaults (model, temperature, max_tokens) |
