## **1) Script Configuration Template (YAML + JSON)**

### **YAML (recommended for authoring)**

```
version: "1.0"
bot_id: "sellrise_conversation_engine"
language_default: "en"
timezone_default: "Europe/Rome"

# ---------- Global rules (used by validator and prompt) ----------
rules:
  one_question_rule: true          # <= 1 question mark per message
  max_sentences: 2                 # <= 2 sentences
  max_chars:
    default: 420
    outbound: 520
    followup: 520
  facts_only: true                 # do not invent facts beyond knowledge_base
  forbid:
    - "I think"
    - "probably"
    - "maybe"
  emoji:
    allowed: false

# ---------- Slots schema ----------
slots_schema:
  industry: { type: "string", required: false }
  role: { type: "string", required: false }
  interest_trigger: { type: "string", required: false }
  acquisition_channels: { type: "string", required: false }
  lead_handler: { type: "string", required: false }
  lead_tracking: { type: "string", required: false }
  pain_points: { type: "string", required: false }
  lost_leads_cost: { type: "string", required: false }
  meeting_datetime_candidate: { type: "datetime", required: false }
  timezone: { type: "string", required: false }
  email: { type: "string", required: false }

# ---------- Action tags (what backend can execute) ----------
actions_catalog:
  "#product_request":
    type: "send_products"
    payload_schema:
      segment: { type: "string", required: false }   # optional segment key
  "#New_meeting_time":
    type: "book_meeting"
    payload_schema:
      datetime: { type: "datetime", required: true }
      timezone: { type: "string", required: false }
  "#stop_script":
    type: "stop_automation"
    payload_schema: {}
  "#handover":
    type: "handover_to_human"
    payload_schema:
      reason: { type: "string", required: false }

# ---------- Stage definitions ----------
stages:

  - stage_id: "stage_0_outbound"
    priority: 100
    entry_condition:
      type: "first_message"
    required_slots: []
    tasks:
      - task_id: "task_0_1"
        priority: 100
        criteria:
          type: "always"
        instruction: "Send an outbound intro message and request permission to ask 2 short questions."
        approved_phrases:
          - "Hi! I'm {agent_name} from {company}. We help businesses qualify inbound leads with AI so they reply faster and lose fewer opportunities. Can I ask 2 quick questions?"
        tags: []
        slot_questions: []  # optional structured questions list

  - stage_id: "stage_1_start"
    priority: 90
    entry_condition:
      type: "any"
      when:
        - "slots.industry is null"
        - "slots.role is null"
        - "slots.interest_trigger is null"
    required_slots: []
    tasks:
      - task_id: "task_1_1_user_initiated"
        priority: 100
        criteria:
          type: "all"
          when:
            - "context.user_initiated == true"
        instruction: "Greet, introduce, give value, and ask one question about what they want to improve."
        approved_phrases:
          - "Hi! I'm {agent_name} from {company}. We use AI to qualify inbound leads and speed up response times. What are you trying to improve most: speed, conversion, or workload?"
        tags: []

      - task_id: "task_1_2_agent_initiated"
        priority: 90
        criteria:
          type: "all"
          when:
            - "context.user_initiated == false"
        instruction: "Acknowledge response and ask one context question."
        approved_phrases:
          - "Thanks—got it. Which channel matters most right now: website forms, WhatsApp/Telegram, Instagram, or phone calls?"
        tags: []

      - task_id: "task_1_3_user_ignored_questions"
        priority: 80
        criteria:
          type: "all"
          when:
            - "context.user_ignored_last_questions == true"
            - "context.user_asked_new_question == true"
        instruction: "Softly redirect: mention short demo/call solves most questions; repeat last bot question."
        approved_phrases:
          - "Good question—we can cover that. It’s usually fastest to confirm on a 15-minute demo for your exact case. Quick check first: {repeat_last_bot_question}"
        tags:
          - "#offer_demo_soft"

  - stage_id: "stage_2_quick_qualification"
    priority: 80
    entry_condition:
      type: "any"
      when:
        - "context.channel_preference == 'chat'"
    required_slots:
      - "industry"
      - "role"
      - "interest_trigger"
    tasks:
      - task_id: "task_2_1_collect_basics"
        priority: 100
        criteria:
          type: "any"
          when:
            - "slots.industry is null"
            - "slots.role is null"
            - "slots.interest_trigger is null"
        instruction: "Collect industry, role, and interest trigger in one message."
        approved_phrases:
          - "To make sure this fits: what business are you in, what’s your role, and what triggered your interest (speed, conversion, or workload)?"
        tags:
          - "#need_more_info"

      - task_id: "task_2_2_collect_missing_only"
        priority: 90
        criteria:
          type: "all"
          when:
            - "context.missing_fields_count > 0"
        instruction: "Ask only for missing fields."
        approved_phrases:
          - "Not enough data—please share: {missing_fields_list}."
        tags:
          - "#need_more_info"

      - task_id: "task_2_3_offer_channel_choice"
        priority: 80
        criteria:
          type: "all"
          when:
            - "slots.industry is not null"
            - "slots.role is not null"
            - "slots.interest_trigger is not null"
        instruction: "Offer to continue in chat or book a 10–15 min call."
        approved_phrases:
          - "Great—context is clear. Do you prefer continuing here, or a quick 10–15 minute call to align requirements and show the approach?"
        tags:
          - "#channel_choice"

  - stage_id: "stage_3_full_qualification"
    priority: 70
    entry_condition:
      type: "any"
      when:
        - "context.full_qualification_allowed == true"
    required_slots: []
    tasks:
      - task_id: "task_3_1_acquisition"
        priority: 100
        criteria:
          type: "all"
          when:
            - "slots.acquisition_channels is null"
        instruction: "Ask how they acquire customers and whether inbound is stable."
        approved_phrases:
          - "How do you currently acquire customers (ads, website, social, partners)? Do you have stable inbound requests?"
        tags: ["#qualify"]

      - task_id: "task_3_2_handling"
        priority: 90
        criteria:
          type: "all"
          when:
            - "slots.lead_handler is null"
            - "slots.lead_tracking is null"
        instruction: "Ask who handles leads and whether they track metrics."
        approved_phrases:
          - "Who handles inbound leads today, and do you track response speed or conversion in any way?"
        tags: ["#qualify"]

      - task_id: "task_3_3_pains"
        priority: 80
        criteria:
          type: "all"
          when:
            - "slots.pain_points is null"
        instruction: "Ask what works vs fails in speed/quality."
        approved_phrases:
          - "How would you rate speed and quality of lead handling today—what works well, and what breaks?"
        tags: ["#qualify"]

      - task_id: "task_3_4_metrics"
        priority: 70
        criteria:
          type: "all"
          when:
            - "slots.lost_leads_cost is null"
        instruction: "Ask about cost of qualification / lost leads, if measured."
        approved_phrases:
          - "If you’ve measured it: do you know the cost of qualifying a lead or roughly how many leads are lost due to slow/weak responses?"
        tags: ["#qualify_metrics"]

      - task_id: "task_3_5_summary_offer_call"
        priority: 60
        criteria:
          type: "all"
          when:
            - "context.full_qualification_complete == true"
        instruction: "Summarize and propose a call/demo."
        approved_phrases:
          - "Thanks—got it. Based on what you shared, the main bottleneck is {summary_pain}. Want to do a 15-minute demo to show the exact flow and integrations?"
        tags: ["#offer_demo"]

  - stage_id: "stage_4_meeting_booking"
    priority: 60
    entry_condition:
      type: "any"
      when:
        - "context.user_requested_meeting == true"
        - "context.lead_qualified == true"
    required_slots: []
    tasks:
      - task_id: "task_4_1_offer_meeting"
        priority: 100
        criteria:
          type: "all"
          when:
            - "context.meeting_offered == false"
        instruction: "Offer meeting and set agenda."
        approved_phrases:
          - "Perfect. In 15 minutes we’ll map your process, show the AI qualification flow for your channels, and agree next steps. What day/time works for you?"
        tags: ["#book_demo"]

      - task_id: "task_4_2_collect_time"
        priority: 90
        criteria:
          type: "all"
          when:
            - "slots.meeting_datetime_candidate is null"
            - "context.user_agreed_to_meeting == true"
        instruction: "Collect meeting date/time (and timezone if needed)."
        approved_phrases:
          - "What day and time works for a quick call? If relevant, share your timezone."
        tags: ["#collect_meeting_time"]

      - task_id: "task_4_3_collect_email"
        priority: 80
        criteria:
          type: "all"
          when:
            - "slots.meeting_datetime_candidate is not null"
            - "slots.email is null"
        instruction: "Collect email to send invite."
        approved_phrases:
          - "Great—what email should I send the calendar invite and meeting link to?"
        tags: ["#collect_email"]

      - task_id: "task_4_4_create_and_confirm"
        priority: 70
        criteria:
          type: "all"
          when:
            - "slots.meeting_datetime_candidate is not null"
            - "slots.email is not null"
            - "context.meeting_confirmed == false"
        instruction: "Confirm booking and trigger backend meeting creation."
        approved_phrases:
          - "Done—I booked {datetime}. You’ll receive the invite at {email}. If anything changes, just message me and we’ll reschedule."
        tags: ["#New_meeting_time"]

      - task_id: "task_4_5_refusal_reason_unknown"
        priority: 60
        criteria:
          type: "all"
          when:
            - "context.user_refused_meeting == true"
            - "context.objection_reason is null"
        instruction: "Ask why they refuse."
        approved_phrases:
          - "Understood. What’s the main blocker—time, format, or you’re just exploring options for now?"
        tags: ["#objection_reason"]

      - task_id: "task_4_6_handle_objection"
        priority: 50
        criteria:
          type: "all"
          when:
            - "context.user_refused_meeting == true"
            - "context.objection_reason is not null"
        instruction: "Handle objection without repeating the same close."
        approved_phrases:
          - "Makes sense. In that case, we can keep it strictly 15 minutes and focus only on your scenario—no commitment. Want me to suggest a couple of slots?"
        tags: ["#handle_objection"]

      - task_id: "task_4_7_questions_after_confirmed"
        priority: 40
        criteria:
          type: "all"
          when:
            - "context.meeting_confirmed == true"
            - "context.user_asked_new_question == true"
        instruction: "Answer briefly, then push deeper answers to the call."
        approved_phrases:
          - "Short answer: {short_answer}. The best way is to confirm details on the call so it matches your exact setup."
        tags: ["#keep_for_call"]

  - stage_id: "stage_5_closure"
    priority: 10
    entry_condition:
      type: "any"
      when:
        - "context.stop_script == true"
        - "context.conversation_complete == true"
    required_slots: []
    tasks:
      - task_id: "task_5_1_close_after_booking"
        priority: 100
        criteria:
          type: "all"
          when:
            - "context.meeting_confirmed == true"
        instruction: "Close politely and mention reminder."
        approved_phrases:
          - "Thanks—booked for {datetime}. I’ll remind you shortly before the call. If questions come up, just message me."
        tags: ["#end"]

      - task_id: "task_5_2_close_stop"
        priority: 90
        criteria:
          type: "all"
          when:
            - "context.stop_script == true"
        instruction: "Send value and stop automation."
        approved_phrases:
          - "Understood. If you return to this later, just message me and I’ll help with your case. Have a good day."
        tags: ["#stop_script"]

# ---------- Followup sequences (when user is silent) ----------
followups:
  - followup_id: "followups_stages_1_3"
    applies_to_stages: ["stage_1_start", "stage_2_quick_qualification", "stage_3_full_qualification"]
    steps:
      - step_id: "fu_1"
        criteria: { type: "silence", min_minutes: 60 }
        approved_phrases:
          - "Just checking—are you still there?"
        tags: []
      - step_id: "fu_2"
        criteria: { type: "silence", min_minutes: 240 }
        approved_phrases:
          - "Did you get a chance to look at my last question?"
        tags: []
      - step_id: "fu_3"
        criteria: { type: "silence", min_minutes: 1440 }
        approved_phrases:
          - "If you want, we can do a quick 15-minute call to answer everything for your exact situation—no commitment. When would be convenient?"
        tags: ["#offer_demo"]
      - step_id: "fu_4"
        criteria: { type: "silence", min_minutes: 4320 }
        approved_phrases:
          - "No worries—here are useful resources: {resources_links}. If you have questions later, just message me."
        tags: ["#stop_script"]

  - followup_id: "followups_stages_4_5"
    applies_to_stages: ["stage_4_meeting_booking", "stage_5_closure"]
    steps:
      - step_id: "fu_1"
        criteria: { type: "silence", min_minutes: 1440 }
        approved_phrases:
          - "Sharing useful links in case helpful: {resources_links}. If you want to continue later, just message me."
        tags: ["#stop_script"]
```

---

### **JSON (engine consumption)**

Same structure, but JSON. The engine should accept either YAML or JSON.

```
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
  "slots_schema": {
    "industry": { "type": "string" },
    "role": { "type": "string" },
    "interest_trigger": { "type": "string" },
    "meeting_datetime_candidate": { "type": "datetime" },
    "timezone": { "type": "string" },
    "email": { "type": "string" }
  },
  "actions_catalog": {
    "#product_request": { "type": "send_products", "payload_schema": { "segment": { "type": "string" } } },
    "#New_meeting_time": { "type": "book_meeting", "payload_schema": { "datetime": { "type": "datetime", "required": true } } },
    "#stop_script": { "type": "stop_automation", "payload_schema": {} }
  },
  "stages": [
    {
      "stage_id": "stage_0_outbound",
      "priority": 100,
      "entry_condition": { "type": "first_message" },
      "required_slots": [],
      "tasks": [
        {
          "task_id": "task_0_1",
          "priority": 100,
          "criteria": { "type": "always" },
          "instruction": "Send an outbound intro message and request permission to ask 2 short questions.",
          "approved_phrases": [
            "Hi! I'm {agent_name} from {company}. We help businesses qualify inbound leads with AI so they reply faster and lose fewer opportunities. Can I ask 2 quick questions?"
          ],
          "tags": []
        }
      ]
    }
  ],
  "followups": []
}
```

---

## **2) Router Algorithm Specification (Pseudo-code)**

### **Inputs**

- last_user_message: string
- conversation_state: State
- script: ScriptConfig
- now: datetime
- channel_context: object (optional: WA/Telegram/IG, metadata)
- knowledge_base: object (facts)

### **Outputs**

- selected_stage_id
- selected_task_id
- llm_prompt
- later: parsed LLM_OUTPUT + executed actions

---

### **State Model (minimal)**

```
State {
  stage_id: string | null
  task_id: string | null
  slots: map<string, any>
  context: {
    user_initiated: bool
    user_ignored_last_questions: bool
    user_asked_new_question: bool
    channel_preference: "chat" | "call" | null
    full_qualification_allowed: bool
    full_qualification_complete: bool
    user_requested_products: bool
    user_requested_meeting: bool
    user_agreed_to_meeting: bool
    user_refused_meeting: bool
    objection_reason: string | null
    meeting_offered: bool
    meeting_confirmed: bool
    stop_script: bool
    conversation_complete: bool
    last_bot_question: string | null
    last_user_message_at: datetime
  }
  used_phrases: set<string>
  used_tags: set<string>
  history: list<Message>
}
```

---

### **Router Overview**

The router performs:

1. **Pre-processing**
- detect intent flags (products, meeting, refusal, silence, question)
- extract slot candidates (datetime, email, etc.) if available
1. **Stage selection**
- choose the highest priority stage whose entry condition is satisfied
1. **Task selection**
- within the stage, choose the highest priority task whose criteria evaluate true
1. **Prompt assembly**
- build a single LLM call with script + kb + rules + selected stage/task + state
1. **Validation + fallback**
- if LLM output invalid → repair/regenerate or fallback approved phrase

---

### **Pseudo-code (Decision Logic)**

```
function route_next_step(last_user_message, state, script, now):

  # -------------------------
  # 0) Preprocess
  # -------------------------
  intents = detect_intents(last_user_message, state)
  # intents = {
  #   product_request_explicit: bool,
  #   meeting_request_explicit: bool,
  #   refusal: bool,
  #   silence: bool,
  #   asked_new_question: bool,
  #   channel_preference: "chat"|"call"|null
  # }

  extracted = extract_entities(last_user_message)
  # extracted may contain:
  #   meeting_datetime_candidate
  #   email
  #   timezone
  #   other slot candidates

  # update state context with intents
  state.context.user_requested_products = intents.product_request_explicit
  state.context.user_requested_meeting  = intents.meeting_request_explicit
  state.context.user_refused_meeting    = intents.refusal
  state.context.user_asked_new_question = intents.asked_new_question
  if intents.channel_preference != null:
      state.context.channel_preference = intents.channel_preference

  # merge extracted slots into state slots (non-destructive / confidence-based)
  state.slots = merge_slots(state.slots, extracted)

  # -------------------------
  # 1) Handle silence followups (if applicable)
  # -------------------------
  if is_silent(state, now):
      followup = select_followup_step(state, script, now)
      if followup != null:
         return build_followup_response_plan(followup, state)

  # -------------------------
  # 2) Stage selection by priority
  # -------------------------
  candidate_stages = []
  for stage in script.stages:
      if eval_entry_condition(stage.entry_condition, state, intents):
          candidate_stages.append(stage)

  if candidate_stages is empty:
      # fallback: continue current stage if exists, else stage_1_start
      stage = fallback_stage(script, state)
  else:
      # choose highest priority (descending)
      stage = max_by(candidate_stages, stage.priority)

  # -------------------------
  # 3) Task selection within stage
  # -------------------------
  candidate_tasks = []
  for task in stage.tasks:
      if eval_criteria(task.criteria, state, intents):
          # also ensure "do not repeat identical close" if configured
          if not violates_repeat_policy(task, state):
              candidate_tasks.append(task)

  if candidate_tasks is empty:
      # fallback task: first task in stage or a generic "ask missing slots"
      task = fallback_task(stage, state)
  else:
      task = max_by(candidate_tasks, task.priority)

  # -------------------------
  # 4) Assemble prompt & request LLM
  # -------------------------
  prompt = assemble_prompt(script.rules, knowledge_base, stage, task, state, last_user_message)

  llm_output = call_llm(prompt)

  # -------------------------
  # 5) Validate output
  # -------------------------
  if not validate_llm_output(llm_output, script.rules, task):
      # attempt repair once
      repair_prompt = assemble_repair_prompt(script.rules, stage, task, state, llm_output)
      llm_output = call_llm(repair_prompt)

      if not validate_llm_output(llm_output, script.rules, task):
          # hard fallback: use an approved phrase (if exists) without LLM
          llm_output = hard_fallback(task, state)

  # -------------------------
  # 6) Apply output (update state + actions)
  # -------------------------
  state = apply_slots_update(state, llm_output.slots_update)
  actions = normalize_actions(llm_output.actions, script.actions_catalog)

  # de-duplicate tags
  actions = drop_duplicates(actions, state.used_tags)

  # record used phrases/tags
  state.used_tags.add_all([a.tag for a in actions])
  state.used_phrases.add(hash_phrase(llm_output.assistant_message))
  state.stage_id = llm_output.stage_id
  state.task_id  = llm_output.task_id

  return {
    "assistant_message": llm_output.assistant_message,
    "actions": actions,
    "state": state
  }
```

---

### **detect_intents()**

### **(minimal logic)**

```
function detect_intents(msg, state):

  product_request_explicit = contains_any(msg, ["show products", "send options", "which model", "photos", "catalog"])
  meeting_request_explicit = contains_any(msg, ["call", "demo", "meeting", "book", "schedule"])
  refusal = contains_any(msg, ["no", "not interested", "stop", "don't want", "leave me"])
  asked_new_question = contains_question(msg)

  # channel preference heuristic
  if contains_any(msg, ["here", "chat", "whatsapp", "telegram"]):
      channel_preference = "chat"
  else if contains_any(msg, ["call", "phone", "zoom", "meet"]):
      channel_preference = "call"
  else:
      channel_preference = null

  return {
    product_request_explicit,
    meeting_request_explicit,
    refusal,
    asked_new_question,
    channel_preference
  }
```

*(Note: you can replace heuristics with an LLM intent classifier later. For v1, heuristic + minimal classifier is enough.)*

---

### **eval_entry_condition()**

### **and**

### **eval_criteria()**

You need a small expression evaluator for conditions like:

- slots.industry is null
- context.user_initiated == true
- context.full_qualification_allowed == true

Implementation options:

- simple safe DSL
- JSONLogic
- custom parser with whitelisted operators

**Do not use raw eval()**.

---

### **Fallback strategy**

1. re-prompt with repair mode
2. hard fallback to approved phrase
3. if no approved phrase, return: “Not enough data—please clarify: {missing_fields_list}.”

---

## **Notes for Implementation (to avoid common failures)**

- **Separate message vs actions**: tags should not leak into user-facing text.
- **State is authoritative**: routing decisions should not depend on LLM memory.
- **Approved phrases are compliance anchors**: when present, prefer them.
- **Validator is mandatory**: 1-question rule / 2-sentence rule, etc.
- **Deduplicate tags**: avoid repeating booking/product triggers.

---

If you want, I can also provide:

- LLM_INPUT and LLM_OUTPUT JSON Schema definitions (draft-07) for strict validation
- a minimal conditions DSL spec (operators + examples)
    
    
- 20–30 routing acceptance tests (“message → expected stage/task/actions”)