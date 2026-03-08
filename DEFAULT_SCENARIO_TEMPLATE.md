# Default Scenario Template

This document describes the default conversation engine template used in Sellrise's Scenario Configuration interface.

## Overview

The default template implements a **stage-based, task-driven conversation flow** for AI-powered lead qualification. It guides prospects through qualification, meeting booking, and follow-up sequences while enforcing strict communication rules for professional, concise interactions.

## Template Structure

### 1. Global Rules

**Purpose**: Enforce consistent, professional communication standards

```json
{
  "one_question_rule": true,      // Max 1 question per message
  "max_sentences": 2,              // Max 2 sentences per message
  "max_chars": {
    "default": 420,                // Standard messages
    "outbound": 520,               // Initial outreach
    "followup": 520                // Follow-up messages
  },
  "facts_only": true,              // Never invent information
  "forbid": ["I think", "probably", "maybe"],
  "emoji": { "allowed": false }
}
```

**Key Principles**:
- One question at a time for clarity
- Concise responses (2 sentences max)
- Character limits prevent overwhelming messages
- Fact-based only (no speculation or assumptions)
- Professional tone (no casual language or emoji)

### 2. Slots Schema

**Purpose**: Define data collection fields for lead qualification

```json
{
  "industry": "string",
  "role": "string",
  "interest_trigger": "string",
  "acquisition_channels": "string",
  "lead_handler": "string",
  "lead_tracking": "string",
  "pain_points": "string",
  "lost_leads_cost": "string",
  "meeting_datetime_candidate": "datetime",
  "timezone": "string",
  "email": "string"
}
```

**Collection Flow**:
1. **Basic qualification**: industry, role, interest_trigger
2. **Deep qualification**: acquisition_channels, lead_handler, pain_points
3. **Meeting booking**: meeting_datetime_candidate, timezone, email

### 3. Actions Catalog

**Purpose**: Define backend actions triggered by conversation tags

| Tag | Type | Purpose |
|-----|------|---------|
| `#product_request` | send_products | Send product information/catalog |
| `#New_meeting_time` | book_meeting | Create calendar booking |
| `#stop_script` | stop_automation | End automated conversation |
| `#handover` | handover_to_human | Transfer to human agent |

### 4. Conversation Stages

#### Stage 0: Outbound (`stage_0_outbound`)
- **Priority**: 100 (highest)
- **Entry condition**: First message (outbound initiation)
- **Goal**: Introduce company and request permission
- **Example**: "Hi! I'm {agent_name} from {company}. We help businesses qualify inbound leads with AI so they reply faster and lose fewer opportunities. Can I ask 2 quick questions?"

#### Stage 1: Start (`stage_1_start`)
- **Priority**: 90
- **Entry condition**: Missing industry, role, or interest_trigger
- **Tasks**:
  - **User-initiated**: Greet, introduce value, ask improvement goal
  - **Agent-initiated**: Acknowledge response, ask channel preference
- **Goal**: Begin qualification and establish context

#### Stage 2: Quick Qualification (`stage_2_quick_qualification`)
- **Priority**: 80
- **Entry condition**: User prefers chat channel
- **Required slots**: industry, role, interest_trigger
- **Goal**: Collect basic information and offer channel choice (chat vs. call)

#### Stage 3: Full Qualification (`stage_3_full_qualification`)
- **Priority**: 70
- **Entry condition**: Full qualification allowed
- **Tasks**: Collect acquisition channels, handling process, pain points, metrics
- **Goal**: Deep understanding of lead's situation and needs
- **Outcome**: Summarize findings and propose demo

#### Stage 4: Meeting Booking (`stage_4_meeting_booking`)
- **Priority**: 60
- **Entry condition**: User requested meeting OR lead is qualified
- **Tasks**:
  - Offer meeting with clear agenda
  - Collect date/time and timezone
  - Collect email for calendar invite
  - Confirm booking and trigger `#New_meeting_time` action
  - Handle objections without repeating same close
- **Goal**: Secure a confirmed meeting

#### Stage 5: Closure (`stage_5_closure`)
- **Priority**: 10 (lowest)
- **Entry condition**: Stop script triggered OR conversation complete
- **Tasks**:
  - Close after successful booking
  - Soft close with value if user wants to stop
- **Goal**: Polite exit with clear next steps

### 5. Follow-up Sequences

**Stages 1-3 (Active Qualification)**:
- **60 minutes silence**: "Just checking—are you still there?"
- **4 hours silence**: "Did you get a chance to look at my last question?"
- **24 hours silence**: Offer 15-minute demo call
- **3 days silence**: Send resources and soft close with `#stop_script`

**Stages 4-5 (Booking/Closure)**:
- **24 hours silence**: Share resources and offer to continue later with `#stop_script`

### 6. System Prompts

The template includes five specialized system prompts:

1. **Main**: General behavior rules and constraints
2. **Qualification**: Focus on efficient data collection
3. **Outbound**: Guidelines for initial outreach
4. **Meeting Booking**: Specific instructions for booking process
5. **Followup**: Rules for follow-up message timing and tone

### 7. LLM Configuration

```json
{
  "model": "anthropic/claude-3.5-sonnet",
  "temperature": 0.7,
  "max_tokens": 2000
}
```

## Usage in Scenario Configuration

### Creating a New Scenario

1. Click **"+ New Scenario"** in the dropdown
2. The editor loads with this default template
3. Customize stages, tasks, slots, and rules as needed
4. Use **"Enhance with AI"** buttons to improve specific sections
5. Save as draft or publish

### Customization Options

- **Add/remove stages**: Adjust conversation flow
- **Modify rules**: Change character limits, question limits, etc.
- **Update slots**: Define what data to collect
- **Edit approved phrases**: Customize message templates
- **Add actions**: Define new backend triggers
- **Adjust follow-up timing**: Change silence thresholds

### AI Enhancement

Three levels of AI enhancement available:

1. **Prompt-level**: Individual system prompt enhancement
2. **JSON-level**: Entire configuration enhancement (structure, consistency)
3. **Full scenario**: Holistic enhancement including prompts + config

## Best Practices

### Do's ✅
- Keep approved phrases under character limits
- Use one question per message for clarity
- Define clear entry conditions for stages
- Set appropriate task priorities (100 = highest)
- Test conversation flow with preview simulator
- Use tags to trigger backend actions
- Attach knowledge base files for fact grounding

### Don'ts ❌
- Don't invent information not in knowledge base
- Don't use speculation phrases ("I think", "probably")
- Don't repeat identical closing attempts
- Don't exceed character limits (breaks UX)
- Don't ask multiple questions in one message
- Don't skip required slots before advancing stages

## Technical Notes

### Priority System
- Higher priority = selected first when conditions met
- Stage priority: 100 (outbound) → 10 (closure)
- Task priority within stage: 100 (most specific) → 40 (fallback)

### Condition Evaluation
- **any**: At least one condition must be true
- **all**: All conditions must be true
- **always**: Always executes (no conditions)

### Variable Interpolation
Approved phrases support variables:
- `{agent_name}`: Bot's name
- `{company}`: Company name
- `{datetime}`: Formatted meeting time
- `{email}`: User's email
- `{summary_pain}`: Extracted pain point summary
- `{resources_links}`: Knowledge base links

## Example Customization

### Adding a Product Demo Stage

```json
{
  "stage_id": "stage_product_demo",
  "priority": 65,
  "entry_condition": {
    "type": "any",
    "when": ["context.user_requested_products == true"]
  },
  "required_slots": [],
  "tasks": [
    {
      "task_id": "task_show_products",
      "priority": 100,
      "criteria": { "type": "always" },
      "instruction": "Show product catalog tailored to their industry",
      "approved_phrases": [
        "Here's what works best for {slots.industry}. Which interests you most?"
      ],
      "tags": ["#product_request"]
    }
  ]
}
```

## Reference

- **Original spec**: `/docs/AI Conversation Engine – Technical Specification/config script template.md`
- **JSON schema**: See technical specification for full validation rules
- **Router algorithm**: See technical specification for decision logic

---

**Last Updated**: March 8, 2026  
**Version**: 1.0  
**Template Type**: Lead Qualification & Meeting Booking
