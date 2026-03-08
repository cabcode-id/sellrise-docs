# Conversation System — Centralized AI Processing Architecture

## Overview

The Sellrise conversation system ensures that **all lead conversations are processed through a single, centralized AI "brain" per workspace**, regardless of the channel (web, WhatsApp, Telegram, Instagram, etc.). This architecture prevents context mixing between different administrators/workspaces and provides consistent, stateful conversations.

## Architecture Principles

### 1. **One Brain Per Project (Workspace)**
- Each workspace has ONE published scenario that acts as the "brain"
- All conversations in that workspace use the same scenario
- The scenario defines conversation flow, rules, prompts, and actions
- Admins configure this brain via the Scenario Configuration UI

### 2. **Workspace Isolation (Critical)**
- Every conversation, message, and lead is scoped to a workspace
- Database queries enforce workspace_id checks
- Context cannot leak between workspaces
- One admin's conversations never mix with another's

### 3. **Channel-Agnostic Processing**
- Same AI brain handles web widget, WhatsApp, Telegram, Instagram, etc.
- Channel is tracked but doesn't change conversation logic
- Messages from all channels feed into the same conversation context
- Responses maintain consistent persona across channels

### 4. **Stateful Conversations**
- Each conversation maintains full context (slots, stage, task, variables)
- Conversation state persists across messages
- No context is lost between messages
- Follow-up messages continue from previous state

## Database Schema

### Conversations Table

```python
class Conversation:
    id: UUID                    # Unique conversation ID
    workspace_id: UUID          # Workspace isolation (FK: workspaces.id)
    lead_id: UUID               # Which lead this conversation belongs to
    scenario_id: UUID           # The "brain" processing this conversation
    
    # Channel tracking
    channel: str                # web, whatsapp, telegram, instagram, messenger, email
    channel_identifier: str     # Phone number, username, etc.
    
    # State management
    status: str                 # active, snoozed, closed, handed_off
    current_stage_id: str       # Current stage in scenario flow
    current_task_id: str        # Current task being executed
    context: JSONB              # Full conversation state (slots, variables, etc.)
    
    # Timestamps
    last_message_at: datetime
    last_user_message_at: datetime
    last_bot_message_at: datetime
    closed_at: datetime
    
    # Metadata
    message_count: int
```

**Key Indexes:**
- `(lead_id, channel, status)` - Fast lookup of active conversation per lead+channel
- `(workspace_id, status, last_message_at)` - Admin inbox view
- `workspace_id` - Workspace isolation enforcement

### Messages Table

```python
class Message:
    id: UUID                    # Unique message ID
    conversation_id: UUID       # Parent conversation
    
    # Message content
    role: str                   # user, assistant, system
    content: str                # Message text
    
    # Channel integration
    external_message_id: str    # WhatsApp/Telegram/etc. message ID
    
    # Processing metadata
    metadata: JSONB             # stage_id, task_id, actions, attachments, etc.
    
    # Delivery tracking
    is_delivered: bool
    delivered_at: datetime
    is_read: bool
    read_at: datetime
    
    # Error handling
    is_error: bool
    error_message: str
```

**Key Indexes:**
- `(conversation_id, created_at)` - Chronological message retrieval
- `(is_delivered, is_error, created_at)` - Retry queue for failed messages

## How It Works

### Message Flow (Widget Example)

```
1. User sends message via web widget
   ↓
2. POST /v1/widget/message
   {
     workspace_id: "abc-123",
     lead_id: "lead-456",
     channel: "web",
     message: "Hi, I'm interested in your services"
   }
   ↓
3. get_or_create_conversation()
   - Checks if active conversation exists for lead+channel
   - If not, creates new conversation with workspace's published scenario
   - Workspace isolation enforced: lead MUST belong to workspace
   ↓
4. add_message() - User message
   - Saves user message to messages table
   - Updates conversation.last_user_message_at
   - Increments conversation.message_count
   ↓
5. Process through scenario (TODO: Implement full router)
   - Load scenario config from conversation.scenario_id
   - Load conversation context (slots, stage, task)
   - Run router algorithm (see config script template.md)
   - Call LLM with proper system prompts + context
   - Extract actions (book meeting, send products, etc.)
   - Update slots and conversation state
   ↓
6. add_message() - Bot response
   - Saves assistant message with metadata
   - Metadata includes: stage_id, task_id, actions_triggered
   - Updates conversation.current_stage_id, current_task_id
   - Updates conversation.context with new slots
   ↓
7. Return response to widget
   {
     conversation_id: "conv-789",
     message_id: "msg-012",
     bot_reply: "Thanks for your interest! What industry are you in?",
     current_stage: "stage_1_start",
     current_task: "task_1_1_user_initiated"
   }
```

### Multi-Channel Support

**Scenario: Lead starts on web, continues on WhatsApp**

```
Day 1 - Web Widget:
  - Conversation #1 created (lead_123, channel=web, scenario=brain_xyz)
  - Messages: "Hi" → "Hello! What industry are you in?"
  - Context saved: { slots: {}, stage: "stage_1_start" }

Day 2 - WhatsApp:
  - Conversation #2 created (lead_123, channel=whatsapp, scenario=brain_xyz)
  - Uses SAME scenario (brain_xyz) as web conversation
  - Fresh context (separate conversation)
  - Messages: "I'm in healthcare" → "Great! Tell me about your role..."
  
Result:
  - Same AI brain handles both channels
  - Consistent persona and flow
  - Separate conversation threads (web vs whatsapp)
  - No context pollution between channels
```

## Workspace Isolation Implementation

### Database Level
```python
# EVERY query includes workspace check
async def get_conversation_messages(
    db: AsyncSession,
    conversation_id: uuid.UUID,
    workspace_id: uuid.UUID  # ← REQUIRED
):
    # Verify conversation belongs to workspace
    conv_result = await db.execute(
        select(Conversation).where(
            Conversation.id == conversation_id,
            Conversation.workspace_id == workspace_id  # ← ENFORCED
        )
    )
    if not conv_result.scalar_one_or_none():
        raise ValueError("Conversation not found")
    
    # Continue with query...
```

### API Level
```python
@router.get("/{conversation_id}")
async def get_conversation(
    conversation_id: uuid.UUID,
    current_user: User = Depends(require_agent)  # ← JWT enforces user
):
    # current_user.workspace_id automatically limits scope
    messages = await get_conversation_messages(
        conversation_id=conversation_id,
        workspace_id=current_user.workspace_id  # ← Cross-workspace access impossible
    )
```

### Service Level
```python
async def get_or_create_conversation(
    workspace_id: uuid.UUID,  # ← ALWAYS required
    lead_id: uuid.UUID,
    ...
):
    # First verify lead belongs to workspace
    lead = await db.execute(
        select(Lead).where(
            Lead.id == lead_id,
            Lead.workspace_id == workspace_id  # ← Security check
        )
    )
    if not lead.scalar_one_or_none():
        raise ValueError(f"Lead {lead_id} not found in workspace {workspace_id}")
    
    # Continue...
```

## API Endpoints

### Public Endpoints (Widget)

**POST /v1/widget/message**
- Send message from lead
- No auth required (public widget)
- Requires workspace_id in body
- Returns bot response

```json
POST /v1/widget/message
{
  "workspace_id": "abc-123",
  "lead_id": "lead-456",
  "channel": "web",
  "message": "Hi there!",
  "session_id": "session-789"
}

Response:
{
  "conversation_id": "conv-111",
  "message_id": "msg-222",
  "bot_reply": "Hello! How can I help you today?",
  "current_stage": "stage_1_start",
  "current_task": "task_1_1_user_initiated"
}
```

### Admin Endpoints (Authenticated)

**GET /v1/conversations**
- List all conversations in workspace
- Filtered by user's workspace_id
- Optional filters: status, channel, lead_id

**GET /v1/conversations/{id}**
- Get conversation details with messages
- Only accessible if conversation.workspace_id == user.workspace_id

**PATCH /v1/conversations/{id}**
- Update conversation status or context
- Workspace isolation enforced

**GET /v1/conversations/{id}/messages**
- Get all messages for a conversation
- Workspace isolation enforced

## Security & Isolation Guarantees

### ✅ Cannot Happen
1. ❌ Admin A sees conversations from Admin B's workspace
2. ❌ Messages from workspace X leak into workspace Y's context
3. ❌ Lead from workspace A processed by scenario from workspace B
4. ❌ Cross-workspace conversation queries succeed

### ✅ Guaranteed
1. ✅ All conversations in workspace use the same published scenario
2. ✅ Conversation context maintained across messages
3. ✅ Channel doesn't affect AI logic (same brain)
4. ✅ Database enforces workspace_id on all queries
5 ✅ API JWT tokens restrict to user's workspace

## Migration Path

### Step 1: Create Tables
```bash
cd Sellrise-BackEnd
alembic revision --autogenerate -m "Add conversations and messages tables"
alembic upgrade head
```

### Step 2: Update Widget Integration
```javascript
// Old: POST /v1/widget/event
await fetch('/v1/widget/event', {
  body: JSON.stringify({ event_type: 'message', data: {...} })
})

// New: POST /v1/widget/message
await fetch('/v1/widget/message', {
  body: JSON.stringify({
    workspace_id: widgetSession.workspace_id,
    lead_id: leadId,
    channel: 'web',
    message: userMessage
  })
})
```

### Step 3: Implement Scenario Processor
```python
# TODO: In /app/services/scenario_processor.py
async def process_message(
    conversation: Conversation,
    user_message: str
) -> Dict[str, Any]:
    """
    1. Load scenario config
    2. Load conversation context
    3. Run router algorithm (select stage + task)
    4. Build LLM prompt with system prompts + context
    5. Call LLM
    6. Parse response
    7. Execute actions
    8. Return bot reply + updated context
    """
    pass
```

## Benefits

### For Developers
- Clear separation of concerns (conversation vs message)
- Type-safe models with SQLAlchemy
- Easy to add new channels (just change channel field)
- Comprehensive audit trail (all messages stored)

### For Admins
- Single place to configure AI behavior (scenario config)
- View all conversations across channels in one inbox
- Consistent AI persona regardless of channel
- Full conversation history and context

### For Leads
- Seamless experience across channels
- AI remembers previous context
- Consistent quality regardless of channel
- No repeated questions

## Next Steps

1. **Create migration**: `alembic revision --autogenerate -m "add_conversations_and_messages"`
2. **Implement scenario processor**: Full router algorithm + LLM integration
3. **Test multi-channel**: Send messages from web, WhatsApp, Telegram
4. **Build admin inbox**: View/manage conversations in frontend
5. **Add channel integrations**: WhatsApp Business API, Telegram Bot API, etc.

## Reference Files

- **Models**: `/app/models/conversation.py`, `/app/models/message.py`
- **Service**: `/app/services/conversation_service.py`
- **API**: `/app/api/v1/conversations.py`, `/app/api/v1/widget.py`
- **Schemas**: `/app/schemas/conversation.py`
- **Config Template**: `/docs/AI Conversation Engine – Technical Specification/config script template.md`
