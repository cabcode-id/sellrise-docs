# Step Types Implementation Guide

## Overview

This document describes the complete implementation of the **Step Types** feature for the Sellrise conversation engine, including backend processing, LLM integration, and frontend rendering.

## Architecture

### Backend Components

1. **Step Processor Service** (`app/services/step_processor.py`)
   - Core logic for executing scenario steps
   - Input validation
   - State management (conversation context/slots)
   - Deterministic next-step resolution

2. **LLM Service** (`app/services/llm_service.py`)
   - Integration with OpenAI and Anthropic APIs
   - Context-aware response generation
   - Knowledge base answer enhancement
   - Structured output parsing

3. **Step Execution API** (`app/api/v1/step_execution.py`)
   - `/v1/steps/init` - Initialize conversation
   - `/v1/steps/execute` - Execute specific step
   - `/v1/steps/process` - High-level message processing with LLM

### Frontend Components

1. **StepRenderer** (`src/components/StepRenderer.jsx`)
   - Universal step rendering component
   - Handles all step types
   - Built-in validation
   - Responsive design

2. **ScenarioPreview** (`src/pages/ScenarioConfiguration/components/ScenarioPreview.jsx`)
   - Enhanced to use StepRenderer
   - Real-time scenario testing
   - Variable tracking

---

## Step Types Reference

### 1. Message

Display informational text to the visitor.

**Configuration:**
```json
{
  "type": "message",
  "content": "Welcome to our consultation assistant!",
  "next": "step_2"
}
```

**Behavior:**
- Displays message
- No user input required
- Automatically continues to next step or waits for trigger

---

### 2. Question (Text Input)

Collect free-text input from the visitor.

**Configuration:**
```json
{
  "type": "question_text",
  "question": "What is your email address?",
  "variable": "email",
  "validation": {
    "type": "email",
    "required": true,
    "error_message": "Please enter a valid email address"
  },
  "next": "step_3"
}
```

**Validation Types:**
- `email` - Email format validation
- `phone` - Phone number validation
- `url` - URL format validation
- `number` - Numeric validation
- `min_length` - Minimum character length
- `max_length` - Maximum character length
- `pattern` - Custom regex pattern

**Behavior:**
- Prompts user for input
- Validates input before proceeding
- Stores answer in conversation context under `slots.{variable}`
- Shows error message if validation fails

---

### 3. Question (Multiple Choice)

Present predefined choices to the visitor.

**Configuration:**
```json
{
  "type": "question_choice",
  "question": "How can we help you today?",
  "choices": ["Product Info", "Book Demo", "Technical Support"],
  "variable": "inquiry_type",
  "next": {
    "Product Info": "step_products",
    "Book Demo": "step_booking",
    "Technical Support": "step_support"
  }
}
```

**Behavior:**
- Displays question with clickable choices
- Conditional branching based on selection
- Stores choice in `slots.{variable}`
- Can use simple next (string) or conditional next (object)

---

### 4. Form (Multi-Field)

Collect multiple fields in a single submission.

**Configuration:**
```json
{
  "type": "form",
  "title": "Contact Information",
  "fields": [
    {
      "name": "full_name",
      "label": "Full Name",
      "type": "text",
      "required": true,
      "placeholder": "John Doe"
    },
    {
      "name": "email",
      "label": "Email Address",
      "type": "email",
      "required": true,
      "validation": {
        "type": "email"
      }
    },
    {
      "name": "phone",
      "label": "Phone Number",
      "type": "tel",
      "required": false,
      "validation": {
        "type": "phone"
      }
    },
    {
      "name": "message",
      "label": "Message",
      "type": "textarea",
      "required": false,
      "placeholder": "Tell us more..."
    }
  ],
  "next": "step_4"
}
```

**Field Types:**
- `text` - Single-line text
- `email` - Email input
- `tel` - Phone number
- `number` - Numeric input
- `url` - URL input
- `textarea` - Multi-line text
- `date` - Date picker (future)
- `select` - Dropdown (future)

**Behavior:**
- Displays all fields together
- Validates all fields before submission
- Shows field-specific errors
- Stores all fields in conversation context

---

### 5. Conditional Branch

Logic branching based on conversation variables (internal step, not rendered).

**Configuration:**
```json
{
  "type": "conditional_branch",
  "conditions": [
    {
      "expression": "slots.budget > 10000",
      "next": "step_high_value"
    },
    {
      "expression": "slots.timeframe == 'urgent'",
      "next": "step_urgent"
    },
    {
      "expression": "slots.inquiry_type == 'Technical Support'",
      "next": "step_support"
    }
  ],
  "default": "step_default"
}
```

**Supported Expressions:**
- Comparisons: `==`, `!=`, `>`, `<`, `>=`, `<=`
- Logical operators: `and`, `or`, `not` (future)
- Access: `slots.variable_name`, `context.variable_name`

**Behavior:**
- Evaluates conditions in order
- Jumps to first matching condition's next step
- Falls back to default if no conditions match
- Does not render UI (instant evaluation)

---

### 6. Knowledge Base Search

Search knowledge base articles and FAQs.

**Configuration:**
```json
{
  "type": "kb_search",
  "prompt": "What would you like to know about our services?",
  "max_results": 3,
  "next": "step_after_kb",
  "no_results_next": "step_fallback"
}
```

**Behavior:**
- Prompts user for search query
- Searches both articles and FAQs
- Displays results with snippets
- Can optionally enhance with LLM if `use_llm: true` in scenario config
- Different next step for no results scenario

---

### 7. Handoff (Booking Link)

Display booking CTA with external link.

**Configuration:**
```json
{
  "type": "handoff_booking_link",
  "text": "Ready to schedule your consultation?",
  "button_label": "Book Now",
  "booking_url": "https://calendly.com/your-company/consultation",
  "next": "step_confirmation"
}
```

**Behavior:**
- Displays message and CTA button
- Opens booking link in new tab
- Logs `booking_link_shown` and `booking_link_clicked` events
- Can continue conversation after booking initiated

---

### 8. Handoff (Human Operator)

Transfer conversation to human operator (future feature).

**Configuration:**
```json
{
  "type": "handoff_operator",
  "message": "Let me connect you with a specialist...",
  "department": "sales"
}
```

**Behavior:**
- Marks conversation for operator takeover
- Displays waiting message
- Updates conversation status (future: `TRANSFERRED_TO_HUMAN`)
- Ends automated flow

---

### 9. End

Terminate the conversation.

**Configuration:**
```json
{
  "type": "end",
  "message": "Thank you for your time! We'll be in touch shortly."
}
```

**Behavior:**
- Displays farewell message
- Marks conversation as complete
- Sets `context.conversation_complete = true`
- No next step

---

## API Endpoints

### Initialize Conversation

**Endpoint:** `POST /v1/steps/init`

**Request:**
```json
{
  "workspace_id": "uuid",
  "lead_id": "uuid",
  "channel": "web",
  "channel_identifier": null,
  "metadata": {}
}
```

**Response:**
```json
{
  "conversation_id": "uuid",
  "scenario_id": "uuid",
  "initial_message": "Welcome! How can I help you?",
  "initial_step_id": "step_1"
}
```

---

### Execute Step

**Endpoint:** `POST /v1/steps/execute`

**Request:**
```json
{
  "conversation_id": "uuid",
  "step_id": "step_2",
  "user_input": "john@example.com",  // For text/kb_search steps
  "form_data": {                      // For form steps
    "name": "John",
    "email": "john@example.com"
  }
}
```

**Response:**
```json
{
  "conversation_id": "uuid",
  "step_id": "step_2",
  "next_step_id": "step_3",
  "response_data": {
    "type": "question_text_response",
    "answer": "john@example.com",
    "variable": "email"
  },
  "conversation_complete": false,
  "context": {
    "slots": {
      "email": "john@example.com"
    },
    "context": {
      "current_step_id": "step_3"
    }
  }
}
```

---

### Process Message (High-Level)

**Endpoint:** `POST /v1/steps/process`

**Request:**
```json
{
  "workspace_id": "uuid",
  "lead_id": "uuid",
  "channel": "web",
  "message": "I'm interested in your premium package",
  "session_id": "session_123"
}
```

**Response:**
```json
{
  "conversation_id": "uuid",
  "message_id": "uuid",
  "bot_reply": "Great! Our premium package includes...",
  "response_type": "message",
  "response_data": {
    "type": "message",
    "content": "Great! Our premium package includes..."
  },
  "current_step_id": "step_5",
  "conversation_complete": false
}
```

---

## LLM Integration

### Environment Variables

Runtime default is OpenRouter. Configure at least:

```bash
OPENROUTER_API_KEY=...
OPENROUTER_MODEL=deepseek/deepseek-chat
OPENROUTER_FALLBACK_MODEL=anthropic/claude-3.5-sonnet
```

### Enable LLM in Scenario

Add to scenario config:

```json
{
  "use_llm": true,
  "system_prompts": {
    "main": "You are a helpful assistant for lead qualification...",
    "qualification": "Your goal is to efficiently qualify leads...",
    "meeting_booking": "When booking meetings, be specific..."
  },
  "rules": {
    "one_question_rule": true,
    "max_sentences": 2,
    "max_chars": { "default": 420 },
    "emoji": { "allowed": false },
    "facts_only": true
  }
}
```

Compatibility note: backend runtime still accepts legacy keys (`prompts`, `global_rules`) for older scenarios.

### LLM Usage Examples

**Generate Dynamic Response:**
```python
from app.services.llm_service import get_llm_service

llm_service = get_llm_service()

response = await llm_service.generate_response(
    conversation_context=conversation.context,
    scenario_config=scenario.config,
    user_message="What's your refund policy?",
    kb_results=kb_search_results
)
```

**Enhance KB Answer:**
```python
answer = await llm_service.enhance_kb_answer(
    question="What's your refund policy?",
    kb_results=[...],
    max_tokens=200
)
```

---

## Frontend Usage

### Using StepRenderer Component

```jsx
import { StepRenderer } from '../components';

function ChatWidget() {
  const [currentStep, setCurrentStep] = useState(null);

  const handleStepSubmit = (submission) => {
    const { type, value } = submission;
    
    // Send to backend
    api.post('/v1/steps/execute', {
      conversation_id: conversationId,
      step_id: currentStep.id,
      user_input: type === 'text' ? value : undefined,
      form_data: type === 'form' ? value : undefined
    }).then(response => {
      // Update to next step
      const nextStep = scenario.steps[response.next_step_id];
      setCurrentStep(nextStep);
    });
  };

  return (
    <StepRenderer 
      step={currentStep}
      onSubmit={handleStepSubmit}
      disabled={loading}
    />
  );
}
```

---

## Testing

### Unit Tests (Backend)

```python
import pytest
from app.services.step_processor import StepProcessor, StepValidationError

@pytest.mark.asyncio
async def test_question_text_validation(db_session):
    # Test email validation
    step = {
        "type": "question_text",
        "question": "What's your email?",
        "variable": "email",
        "validation": {"type": "email", "required": True}
    }
    
    processor = StepProcessor(db_session, conversation)
    
    # Valid email
    result, next_id = await processor.execute_step(step, "test@example.com")
    assert result["answer"] == "test@example.com"
    
    # Invalid email should raise error
    with pytest.raises(StepValidationError):
        await processor.execute_step(step, "invalid-email")
```

### Integration Tests

```python
@pytest.mark.asyncio
async def test_full_conversation_flow(client, db_session):
    # Initialize conversation
    response = await client.post("/v1/steps/init", json={
        "workspace_id": workspace_id,
        "lead_id": lead_id,
        "channel": "web"
    })
    conversation_id = response.json()["conversation_id"]
    
    # Process through steps
    response = await client.post("/v1/steps/process", json={
        "workspace_id": workspace_id,
        "lead_id": lead_id,
        "message": "john@example.com"
    })
    
    assert response.status_code == 200
    assert response.json()["bot_reply"]
```

---

## Best Practices

### Scenario Design

1. **Keep steps focused** - One question/action per step
2. **Validate early** - Use validation rules to catch errors
3. **Provide clear error messages** - Help users fix input issues
4. **Use conditional branching** - Personalize flow based on answers
5. **Design for mobile** - Keep forms short, choices clear

### Performance

1. **Minimize LLM calls** - Use deterministic steps when possible
2. **Cache KB results** - Store common queries
3. **Batch validations** - Validate forms all at once
4. **Lazy load** - Only fetch steps as needed

### Security

1. **Validate all inputs** - Never trust client data
2. **Sanitize outputs** - Prevent XSS in rendered content
3. **Rate limit** - Protect LLM endpoints
4. **Workspace isolation** - Always filter by workspace_id

---

## Troubleshooting

### Common Issues

**Issue: Step validation fails with no clear error**
- Check validation rules match input type
- Verify regex patterns are correct
- Test validation logic in isolation

**Issue: LLM responses are too long**
- Adjust `max_tokens` parameter
- Add character limits to `rules.max_chars`
- Use structured output for consistency

**Issue: Conversation context lost**
- Ensure `conversation_id` is persisted
- Check context updates are committed to DB
- Verify session management in frontend

**Issue: Steps not rendering correctly**
- Verify step type is spelled correctly
- Check all required fields are present
- Review browser console for errors

---

## Next Steps

### MVP Phase 1
- ✅ All core step types implemented
- ✅ Basic LLM integration
- ✅ Widget rendering

### Phase 2 (Future)
- [ ] Advanced conditional logic (AND/OR operators)
- [ ] Slot filling and validation
- [ ] Multi-language support
- [ ] A/B testing for steps
- [ ] Analytics per step
- [ ] Visual flow builder UI

---

## Related Documentation

- [FEATURE_BREAKDOWN.md](../../docs/FEATURE_BREAKDOWN.md) - Feature requirements
- [AI Conversation Engine – Technical Spec](../../docs/AI%20Conversation%20Engine%20%E2%80%93%20Technical%20Specification/AI%20Conversation%20Engine%20%E2%80%93%20Technical%20spec.md) - LLM engine architecture
- [Scenario Configuration Guide](../../docs/SCENARIO_CONFIGURATION_GUIDE.md) - Scenario JSON schema

---

## Support

For questions or issues with step types implementation:
1. Check this documentation
2. Review test cases in `tests/`
3. Inspect browser console for frontend errors
4. Check backend logs for API errors
5. Review conversation context in database

---

**Last Updated:** 2026-03-08  
**Version:** 1.0.0
