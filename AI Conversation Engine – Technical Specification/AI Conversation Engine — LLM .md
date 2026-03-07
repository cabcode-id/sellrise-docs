# **AI Conversation Engine — LLM Integration & Logic Specification**

Version: 1.0

Language: English

Purpose: Define the full architecture and operational logic for the AI-driven conversation system used for lead qualification, product presentation, objection handling, and meeting booking.

---

# **1. System Overview**

The AI Conversation Engine is a **stage-based dialogue system** designed to automate structured conversations with potential customers.

The AI assistant performs the following functions:

- greet and engage users
- qualify incoming leads
- present products when requested
- schedule consultations
- handle objections
- send follow-ups
- trigger backend actions

The conversation logic is implemented using a **Stage → Task → Action** model.

The backend controls conversation routing, while the LLM generates responses and structured outputs.

---

# **2. Core System Architecture**

The system consists of the following modules.

### **1. Conversation State Store**

Maintains the conversation state.

Stores:

- current stage
- current task
- slot values
- conversation history
- executed tags/actions
- used approved phrases

Example state structure:

```
{
  "stage_id": "stage_3",
  "task_id": "task_3_1",
  "slots": {},
  "used_phrases": [],
  "used_tags": [],
  "history": []
}
```

---

### **2. Stage & Task Router**

Responsible for determining which stage and task should execute next.

The router evaluates:

- user intent
- filled slots
- stage entry conditions
- task criteria

Routing priority:

1. Explicit user request
2. Required slot missing
3. Meeting booking flow
4. Objection handling
5. Follow-up logic
6. Continue current stage

---

### **3. Prompt Builder**

Constructs the final prompt for the LLM.

Prompt components:

```
SYSTEM
Master System Prompt

KNOWLEDGE BASE
Product and business information

SCRIPT
Current stage and task

STATE
Slots + conversation history

USER MESSAGE
Latest user message
```

---

### **4. LLM Executor**

Calls the language model and receives structured output.

The LLM must always return a JSON object matching the defined schema.

---

### **5. Action Dispatcher**

Executes backend actions triggered by tags.

Example actions:

| **Tag** | **Action** |
| --- | --- |
| #product_request | send product cards |
| #New_meeting_time | create calendar event |
| #send_case | send case study |
| #handover | transfer to human |
| #stop_script | stop automation |

---

# **3. Dialogue Model**

Conversation follows a stage-driven structure.

```
Conversation
 ├ Stage
 │   ├ Task
 │   ├ Task
 │   └ Task
 │
 └ Followups
```

Stages represent phases of the conversation.

Tasks represent instructions executed within a stage.

---

# **4. Conversation Stages**

## **Stage 0 — Outbound Message**

Purpose:

Initiate conversation and request permission to ask questions.

Example message:

“Hi, I’m Maria from Lead.Click. We help businesses automate inbound lead qualification using AI. May I ask a few quick questions about your business?”

---

## **Stage 1 — Conversation Start**

Goal:

Understand the user’s initial intent.

Typical actions:

- greet user
- introduce service
- ask about goals

---

## **Stage 2 — Initial Qualification**

Goal:

Collect basic user data.

Required information:

- business type
- user role
- interest trigger

---

## **Stage 3 — Full Qualification**

Goal:

Understand user’s current lead management process.

Typical questions:

- how they acquire leads
- who handles leads
- lead processing metrics
- operational challenges

---

## **Stage 4 — Meeting Booking**

Goal:

Schedule a call with a specialist.

Flow:

1. Offer meeting
2. Ask for date/time
3. Request email
4. Create meeting link
5. Confirm meeting

---

## **Stage 5 — Conversation Closure**

Goal:

End conversation politely or stop automation.

---

# **5. Task Structure**

Each task includes the following properties.

```
task_id
task_criteria
task_instruction
approved_phrases
tags
```

Example:

```
{
  "task_id": "task_3_1",
  "task_criteria": "acquisition_channels_unknown",
  "task_instruction": "Ask how the user currently acquires customers.",
  "approved_phrases": [
    "How do you currently acquire new customers?"
  ],
  "tags": []
}
```

---

# **6. Slot System**

Slots store structured data extracted from conversation.

Example slots:

```
industry
role
interest_trigger
acquisition_channels
lead_handler
lead_tracking
pain_points
meeting_datetime_candidate
email
```

Slots are updated via LLM output.

---

# **7. Master System Prompt**

The Master System Prompt defines the LLM’s behavior.

---

### **MASTER SYSTEM PROMPT**

```
You are an AI assistant responsible for executing a structured conversation task.

You are not responsible for routing the conversation. The backend router has already selected the current stage and task.

Your job is to perform the task and generate a response according to the rules.

Always follow these rules:

1. Respond with exactly one message.
2. Maximum two sentences.
3. Ask at most one question.
4. Do not invent information that is not present in the knowledge base.
5. If approved_phrases are provided, select one and adapt it slightly if needed.
6. Do not repeat the same phrase previously used in the conversation.
7. Never include action tags inside the user-facing message.
8. If required information is missing, ask a clarifying question.
9. Always return output in valid JSON according to the output schema.

If you cannot complete the task because information is missing, return a clarifying question.
```

---

# **8. LLM Input Structure**

Example LLM input:

```
{
  "meta": {
    "bot_id": "sellrise_ai",
    "version": "1.0"
  },
  "rules": {},
  "knowledge_base": {},
  "script": {},
  "state": {},
  "runtime": {},
  "last_user_message": {}
}
```

---

# **9. LLM Output Format**

The model must return structured JSON.

Example:

```
{
  "stage_id": "stage_4",
  "task_id": "task_4_3",
  "assistant_message": "Great, I scheduled your meeting. You'll receive the invite shortly.",
  "slots_update": {
    "meeting_datetime_candidate": "2026-03-07T14:30:00+01:00"
  },
  "actions": [
    {
      "type": "book_meeting",
      "tag": "#New_meeting_time",
      "payload": {
        "datetime": "2026-03-07T14:30:00+01:00"
      }
    }
  ]
}
```

---

# **10. Intent Detection**

User messages should be classified into intents.

Supported intents:

```
product_request
meeting_request
question
objection
information
stop
```

Detection methods:

- keyword heuristics
- lightweight LLM classifier

---

# **11. Slot Extraction**

Entities extracted from messages should populate slots.

Examples:

### **Email**

Regex detection.

### **Datetime**

Convert phrases like:

```
"Tomorrow at 3"
```

Into:

```
2026-03-06T15:00
```

---

# **12. Follow-up System**

Follow-ups are triggered when the user becomes inactive.

### **Stages 1–3**

Sequence:

1. Availability check
2. Reminder about previous question
3. Value reminder + meeting suggestion
4. Send useful resources

---

### **Stages 4–5**

Sequence:

1. Send helpful information
2. Stop automation

---

# **13. Validator**

Every LLM output must be validated.

Rules:

- maximum one question
- maximum two sentences
- valid JSON
- tags must not appear in message
- facts must exist in knowledge base

If validation fails:

1. regenerate
2. fallback to approved phrase

---

# **14. Error Handling**

Possible failures:

- invalid JSON
- missing fields
- incorrect action tag

Recovery steps:

```
1 repair prompt
2 regenerate response
3 fallback phrase
```

---

# **15. Logging Requirements**

Each interaction must log:

```
timestamp
conversation_id
stage_id
task_id
slots
llm_prompt
llm_output
actions
validator_result
```

---

# **16. Testing**

At least 20 test cases should be implemented.

Examples:

1. User requests product → product stage triggered
2. User sends meeting time → meeting booking triggered
3. User refuses meeting → objection handling triggered
4. Missing slot → qualification question asked
5. User silent → follow-up triggered

---

# **17. Definition of Done**

System is complete when:

- routing works correctly
- tasks execute based on criteria
- LLM outputs valid JSON
- actions trigger integrations
- validator enforces rules
- logs record all events