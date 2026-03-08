## **1) Flow Diagram — Architecture (Router → LLM → Actions → State)**

Use this diagram in ARCHITECTURE_FLOW.md (Mermaid). Most devs can render it in GitHub / Notion.

```
flowchart TD
  A[Incoming User Message] --> B[Load Conversation State]
  B --> C[Preprocess / Parse]
  C --> C1[Intent Detection]
  C --> C2[Entity/Slot Extraction]
  C1 --> D[Stage Selection (Router)]
  C2 --> D
  D --> E[Task Selection (Router)]
  E --> F[Prompt Assembly]
  F --> G[LLM Call]
  G --> H[Parse JSON Output]
  H --> I[Schema Validate (LLM_OUTPUT)]
  I -->|fail| J[Repair Prompt + Re-run LLM (1x)]
  J --> H
  I -->|pass| K[Business Validator]
  K -->|fail| J2[Repair Prompt + Re-run LLM (1x)]
  J2 --> H
  K -->|pass| L[Apply slots_update to State]
  L --> M[Normalize + Deduplicate Actions]
  M --> N[Action Dispatcher]
  N --> O[External Integrations]
  O --> P[Update State with action results]
  P --> Q[Persist State]
  Q --> R[Send assistant_message to User]
  R --> S[Log event: stage/task/slots/actions/validator]
```

### **Notes (implementation-critical)**

- **Router owns** stage/task selection. LLM is a task executor.
- **Two validation layers**:
    1. JSON Schema validation (hard)
    2. Business validator (1 question, 2 sentences, no tags in message, facts-only)
- **Actions are separate from message**. Never embed tags into user-facing text.
- **Persist state after actions**, since action results can affect future routing.

---

## **2) Acceptance Tests (30) — Routing + Output + Actions**

Format below is intended to be copy-pasted into acceptance_tests.md.

Each test includes:

- **Given** (state + context)
- **When** (user message)
- **Expect** (stage/task + required behavior + actions)

> Assumption: you implement router stages similar to:
> 
- stage_0_outbound
- stage_1_start
- stage_2_quick_qualification
- stage_3_full_qualification
- stage_4_meeting_booking
- stage_5_closure
- followup sequences

You can rename stage/task IDs; keep behavior consistent.

---

### **Group A — Stage 0 / Start (3 tests)**

**T01 — First message outbound**

- Given: state.stage_id = null, empty history
- When: system triggers outbound send
- Expect: stage_0_outbound / task_0_1
- Expect: message asks permission to ask 2 quick questions
- Actions: none

**T02 — User initiates conversation**

- Given: empty slots (industry/role/interest_trigger = null)
- When: user: “Hi, can you help?”
- Expect: stage_1_start / task_1_1_user_initiated
- Expect: greeting + value + 1 question about goal

**T03 — Agent initiated, user responds**

- Given: context.user_initiated = false, user replies to outbound
- When: user: “Ok, what is this about?”
- Expect: stage_1_start / task_1_2_agent_initiated
- Expect: acknowledge + 1 context question

---

### **Group B — Quick Qualification (7 tests)**

**T04 — Missing basics, collect all**

- Given: channel_preference='chat', industry/role/interest_trigger = null
- When: user: “Sure, ask”
- Expect: stage_2_quick_qualification / task_2_1_collect_basics
- Expect: asks for industry+role+trigger in one message

**T05 — Partial basics, ask missing only**

- Given: industry="ecommerce", role=null, interest_trigger="speed"
- When: user: “We’re ecommerce, speed matters”
- Expect: stage_2_quick_qualification / task_2_2_collect_missing_only
- Expect: asks only for role
- Must contain: “Not enough data—please share: role” (or equivalent)
- Actions: #need_more_info optional

**T06 — Basics complete → offer channel choice**

- Given: industry, role, interest_trigger are filled
- When: user: “Ok”
- Expect: stage_2_quick_qualification / task_2_3_offer_channel_choice
- Expect: offer continue in chat vs short call

**T07 — User ignores bot question and asks unrelated**

- Given: bot asked qualification question last; user ignored it
- When: user: “How much does it cost?”
- Expect: stage_1_start / task_1_3_user_ignored_questions OR a dedicated redirect task
- Expect: short answer if price exists in KB, then repeat last bot question
- Must not hallucinate pricing if missing in KB

**T08 — User chooses “call”**

- Given: in stage 2
- When: user: “Better to call”
- Expect: router sets context.channel_preference='call'
- Expect stage selected: stage_4_meeting_booking (meeting flow)

**T09 — User says “stop”**

- Given: any stage, active
- When: user: “Stop messaging me”
- Expect: stage: stage_5_closure / task_5_2_close_stop
- Expect actions include: #stop_script

**T10 — Facts-only enforcement**

- Given: KB does not include pricing, no case studies, no claims
- When: user: “What results do you guarantee?”
- Expect: assistant asks a clarifying question or states “I can’t claim results; depends…”
- Must not invent numbers/percentages

---

### **Group C — Full Qualification (6 tests)**

**T11 — Ask acquisition channels**

- Given: full_qualification_allowed=true, slots.acquisition_channels=null
- When: user: “Ok, ask more”
- Expect: stage_3_full_qualification / task_3_1_acquisition

**T12 — Ask lead handling + tracking**

- Given: acquisition filled; lead_handler and lead_tracking null
- When: user: “We get leads from ads”
- Expect: stage_3_full_qualification / task_3_2_handling

**T13 — Ask pain points**

- Given: lead_handler/tracking filled; pain_points null
- When: user: “Sales team handles them, we track in CRM”
- Expect: stage_3_full_qualification / task_3_3_pains

**T14 — Ask metrics (if measured)**

- Given: pain_points filled; lost_leads_cost null
- When: user: “Main issue is slow response”
- Expect: stage_3_full_qualification / task_3_4_metrics
- Expect: phrasing includes “If you’ve measured it…”

**T15 — Summary + offer demo**

- Given: context.full_qualification_complete=true
- When: user: “Ok”
- Expect: stage_3_full_qualification / task_3_5_summary_offer_call
- Expect: summary + propose 15-min demo (1 question max)

**T16 — One-question / two-sentences compliance**

- Given: any qualification task
- When: normal message
- Expect: output validator passes:
    - <= 1 ?
    - <= 2 sentences
    - no tags in message

---

### **Group D — Meeting Booking (8 tests)**

**T17 — User explicitly requests meeting**

- Given: any stage
- When: user: “Let’s schedule a demo”
- Expect: stage_4_meeting_booking / task_4_1_offer_meeting OR task_4_2_collect_time
- Expect: asks for date/time

**T18 — Collect meeting time**

- Given: user agreed; meeting_datetime_candidate=null
- When: user: “Tomorrow 2pm”
- Expect: slot extraction sets candidate datetime
- Expect: stage_4_meeting_booking / task_4_3_collect_email (next step)
- Must not book yet without email (if required by your flow)

**T19 — Collect email**

- Given: meeting_datetime_candidate filled; email null
- When: user: “my email is test@acme.com”
- Expect: stage_4_meeting_booking / task_4_4_create_and_confirm
- Expect actions: #New_meeting_time with datetime payload

**T20 — Meeting confirmation message**

- Given: booking action executed successfully
- When: (next bot message)
- Expect: confirmation text includes datetime + reminder (no tags in text)

**T21 — User proposes time; outside allowed schedule**

- Given: you enforce schedule windows in backend
- When: user: “Sunday 23:00”
- Expect: backend flags invalid slot
- Expect: bot requests choosing within allowed window (dedicated task)
- Must not emit #New_meeting_time

**T22 — User refuses meeting, reason unknown**

- Given: meeting offered
- When: user: “No, I don’t want a call”
- Expect: task_4_5_refusal_reason_unknown
- Expect: asks 1 question: what’s the blocker?

**T23 — User refuses meeting, reason known**

- Given: context.objection_reason="no time"
- When: user: “No time”
- Expect: task_4_6_handle_objection
- Expect: closes objection without repeating previously used close

**T24 — User asks questions after booking**

- Given: meeting_confirmed=true
- When: user: “Can you integrate with HubSpot?”
- Expect: task_4_7_questions_after_confirmed
- Expect: short answer if in KB; otherwise “need more data”; push deeper to call

---

### **Group E — Product Presentation (3 tests)**

### **(if enabled in your bot)**

**T25 — Explicit product request triggers product stage**

- Given: any stage
- When: user: “Send me product options and photos”
- Expect: product stage triggered (or task that emits #product_request)
- Expect actions: #product_request

**T26 — Product request but missing required slots**

- Given: product stage requires some slots (e.g., segment)
- When: user: “Show options”
- Expect: bot asks for missing slot in 1 question
- Must not emit #product_request until required info present (if your design requires it)

**T27 — Product cards sent; ask follow-up**

- Given: #product_request already executed
- When: user: “Ok”
- Expect: bot asks what they liked / any questions (1 question)

---

### **Group F — Refusal / Stop / Closure (3 tests)**

**T28 — Hard refusal triggers stop automation**

- Given: any stage
- When: user: “Not interested. Stop.”
- Expect: closure stage + #stop_script

**T29 — Soft refusal triggers value-only message**

- Given: refusal but not “stop” wording
- When: user: “Maybe later”
- Expect: value resource message; optional #stop_script depending on policy

**T30 — No hallucinations policy**

- Given: KB missing specifics (pricing, guarantees, client counts)
- When: user requests them
- Expect: assistant does NOT invent; asks clarifying or says not available

---

## **Practical note for implementation**

To automate these tests, represent each test as JSON fixtures:

- state_before.json
- message.json
- expected.json (stage_id, task_id, expected_action_tags)