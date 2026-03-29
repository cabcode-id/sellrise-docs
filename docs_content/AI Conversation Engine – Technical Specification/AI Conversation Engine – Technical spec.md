# **AI Conversation Engine – Technical Specification**

Version: 1.0

Purpose: Implement a stage-based AI conversation engine for automated lead qualification, product presentation, and meeting booking.

---

# **1. System Purpose**

The AI conversation engine manages customer dialogues using a **structured stage-based dialogue system**.

The goal is to allow the AI to:

- qualify incoming leads
- present products when requested
- schedule meetings
- handle objections
- send follow-ups
- trigger external actions (CRM, calendar, content delivery)

The conversation logic is **data-driven** and defined in a **script configuration file**, not hardcoded.

The developer should focus on building the **conversation engine**, not on writing dialogue logic.

---

# **2. Core Architecture**

The system consists of five main modules.

### **1. Conversation State Store**

Stores the current state of the conversation.

Tracks:

- current stage
- last executed task
- conversation history
- filled slots (customer data)
- executed actions
- used approved phrases

---

### **2. Stage & Task Router**

Determines which stage and task should run next.

The router evaluates:

- current conversation state
- last user message
- slot availability
- stage entry conditions
- task criteria

---

### **3. Prompt Assembler**

Constructs the prompt for the LLM.

The prompt contains:

- System persona rules
- Knowledge base facts
- Script configuration
- Current stage
- Conversation state
- Last user message

---

### **4. LLM Executor**

Calls the LLM and receives structured output.

The LLM must return a structured JSON response.

---

### **5. Action Dispatcher**

Executes backend actions based on tags returned by the model.

Examples:

- send product cards
- create calendar event
- stop script
- send content materials

---

# **3. Conversation Flow Model**

The conversation follows a **stage-based architecture**.

```
Conversation
 ├ Stage
 │   ├ Task
 │   ├ Task
 │   └ Task
 │
 └ Followups
```

Stages represent major phases of the sales dialogue.

Tasks represent specific instructions within a stage.

---

# **4. Conversation Stages**

The conversation uses the following stages.

---

## **Stage 0 — Outbound Message**

Entry condition:

First message in conversation.

Purpose:

Introduce the assistant and request permission to ask questions.

---

## **Stage 1 — Conversation Start**

Entry condition:

Customer intent and basic information are unknown.

Goal:

Understand the customer’s interest and context.

Typical tasks:

- greet the user
- ask about goals
- identify context of inquiry

---

## **Stage 2 — Initial Qualification**

Entry condition:

Customer agreed to continue conversation via chat.

Goal:

Collect basic qualification data.

Required fields:

- business type
- customer role
- interest trigger

---

## **Stage 3 — Full Qualification**

Entry condition:

Customer agreed to answer additional questions.

Goal:

Understand the lead’s current processes and pain points.

Example topics:

- lead acquisition channels
- lead handling process
- lead processing metrics
- operational challenges

---

## **Stage 4 — Meeting Booking**

Entry condition:

- lead qualified
- or user explicitly requested a meeting

Goal:

Schedule a call with a human specialist.

Tasks include:

- offer meeting
- collect date/time
- collect email
- create meeting link
- confirm meeting

---

## **Stage 5 — Conversation Closure**

Entry condition:

- conversation finished
- or user declined further interaction

Goal:

- close the conversation politely
- optionally send resources
- stop the automation script

---

# **5. Task Structure**

Each task contains the following fields.

```
task_id
task_criteria
task_value
approved_phrases
tags
```

---

### **task_id**

Unique identifier.

Example:

```
task_3_1
```

---

### **task_criteria**

Defines when the task should be selected.

Example:

```
IF acquisition_channels_unknown
```

---

### **task_value**

Instruction describing the task objective.

Example:

```
Ask the user how they currently acquire customers
```

---

### **approved_phrases**

List of allowed phrases the AI may use.

The AI must select and adapt one phrase.

Example:

```
How do you currently acquire new customers?
```

---

### **tags**

Optional action triggers.

Examples:

```
#book_demo
#product_request
#stop_script
```

---

# **6. Slot System (Conversation Data)**

Slots store structured information extracted from the conversation.

Example slots:

```
industry
role
interest_trigger
acquisition_channels
lead_handler
lead_tracking
pain_points
lost_leads_cost
meeting_datetime_candidate
email
```

Slots are stored in the conversation state.

---

# **7. Stage Routing Logic**

The router must evaluate stages in priority order.

### **Stage Selection Priority**

1. Explicit user requests
2. Required slots missing
3. Meeting scheduling flow
4. Objection handling
5. Followup logic
6. Continue current stage

---

### **Example Routing Logic**

```
IF user requests product details
    stage = product_presentation

ELSE IF meeting booking started
    stage = meeting_booking

ELSE IF required slots missing
    stage = qualification

ELSE IF user refuses
    stage = objection_handling

ELSE
    stage = current_stage
```

---

# **8. LLM Output Contract**

The LLM must return structured JSON.

Example response:

```
{
  "stage_id": "stage_4",
  "task_id": "task_3_2",
  "assistant_message": "Great, I booked the meeting for you. You will receive a reminder before the call.",
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

# **9. Action System**

Tags trigger backend integrations.

Example action mapping:

| **Tag** | **Action** |
| --- | --- |
| #book_demo | create calendar event |
| #product_request | send product cards |
| #send_case | send case studies |
| #stop_script | stop automation |
| #handover | transfer to human |
| #follow_up_start | start followup sequence |

---

# **10. Followup Engine**

Followups are triggered when the user does not respond.

Two followup sequences exist.

---

### **Sequence A (Stages 1-3)**

1. Check if the user is available
2. Ask for response to previous question
3. Offer value and suggest meeting
4. Send useful resources

---

### **Sequence B (Stages 4-5)**

1. Send helpful information
2. End automation

---

# **11. Output Validator**

All generated messages must pass validation rules.

Rules:

- maximum 1 question per message
- maximum 2 sentences
- avoid repeated phrases
- no hallucinated facts
- respect approved phrase constraints

If validation fails:

The message must be regenerated.

---

# **12. Conversation State Object**

Example structure:

```
{
  "stage_id": "stage_3",
  "task_id": "task_3_1",
  "slots": {
    "industry": "ecommerce",
    "role": "owner",
    "acquisition_channels": null
  },
  "used_phrases": [],
  "used_tags": [],
  "conversation_history": []
}
```

---

# **13. Developer Responsibilities**

The developer must implement:

### **Router**

Select stage and task based on state and input.

### **State Manager**

Persist conversation state.

### **Prompt Builder**

Assemble prompt from system + knowledge base + script.

### **LLM Client**

Send request and parse structured output.

### **Action Dispatcher**

Execute backend actions triggered by tags.

### **Validator**

Ensure response follows system rules.

---

# **14. Definition of Done**

The system is considered complete when:

- stage routing works correctly
- tasks execute according to criteria
- actions trigger backend integrations
- structured LLM output is parsed successfully
- validator enforces message constraints
- logs store stage/task/action execution

---

# **15. Testing Requirements**

Minimum 20 test cases should be implemented.

Example tests:

1. User asks about products → product stage triggered
2. User provides meeting time → meeting booking action triggered
3. User refuses meeting → objection handling stage triggered
4. Missing slot → qualification question asked
5. User silent → followup triggered

---

# **16. Logging**

Each message must log:

```
timestamp
stage_id
task_id
slots
llm_output
actions
validator_result
```

---

# **17. Future Extensions**

Possible improvements:

- CRM integration
- multi-language support
- adaptive followups
- dynamic knowledge base retrieval
- intent classification layer