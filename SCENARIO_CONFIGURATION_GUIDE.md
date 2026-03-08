# Scenario Configuration - LLM-Enhanced Conversation Builder

This feature allows administrators to create, edit, and manage AI-powered conversation scenarios with LLM enhancement capabilities using OpenRouter.

## Features

### 1. **Scenario JSON Editor**
- Visual JSON editor with real-time validation
- Validates scenario structure (entry_step_id, steps, references)
- Format button for clean JSON output
- Error highlighting and messages

### 2. **System Prompts Editor**
- Manage multiple system prompts (main, qualification, closing, custom)
- Template library for common prompt types
- AI-powered prompt enhancement via OpenRouter
- Add/remove custom prompts dynamically

### 3. **File Attachment for LLM Grounding**
- Upload documents (PDF, TXT, MD, JSON, CSV, DOCX)
- 10MB file size limit per file
- Workspace-isolated file storage
- Files provide context to LLM during conversations

### 4. **LLM Enhancement**
- Enhance system prompts with AI suggestions
- Generate complete scenario configurations from descriptions
- Powered by Claude 3.5 Sonnet via OpenRouter
- One-click enhancement for prompts and configurations

### 5. **Scenario Preview**
- Interactive preview of conversation flow
- Step-by-step simulation
- Variable state tracking
- Test scenarios before publishing

### 6. **Draft/Publish Workflow**
- Save scenarios as drafts
- Publish to make active
- Version tracking
- Only published scenarios are used by the widget

## Setup

### Backend Setup

1. **Install Dependencies**
   ```bash
   cd Sellrise-BackEnd
   pip install -r requirements.txt
   ```

2. **Configure Environment Variables**
   
   Copy `.env.example` to `.env` and configure:
   ```bash
   cp .env.example .env
   ```

   Add your OpenRouter API key:
   ```env
   OPENROUTER_API_KEY=your-openrouter-api-key-here
   UPLOAD_DIR=./uploads
   ```

3. **Get OpenRouter API Key**
   - Sign up at https://openrouter.ai/
   - Create an API key
   - Add to `.env` file

4. **Run Database Migrations**
   ```bash
   alembic upgrade head
   ```

5. **Start Backend Server**
   ```bash
   uvicorn app.main:app --reload
   ```

### Frontend Setup

1. **Install Dependencies**
   ```bash
   cd Sellrise-Front-End
   npm install
   ```

2. **Configure Environment Variables**
   
   Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```

   Configure API URL:
   ```env
   VITE_API_BASE_URL=http://localhost:8000
   ```

3. **Start Frontend**
   ```bash
   npm run dev
   ```

## Usage

### Accessing the Feature

Navigate to `/scenarios` in the admin panel after logging in with an Admin role.

### Creating a New Scenario

1. Click "New Scenario" or select "+ New Scenario" from the dropdown
2. Enter scenario name and description
3. **Default Template Loaded**: A comprehensive conversation engine template is automatically loaded with:
   - **6 conversation stages** (outbound → start → qualification → full qualification → meeting booking → closure)
   - **Strict communication rules** (1 question/message, 2 sentences max, character limits)
   - **11 data collection slots** (industry, role, interest_trigger, email, etc.)
   - **4 action triggers** (product_request, meeting booking, stop_script, handover)
   - **Follow-up sequences** for silence handling
   - **5 system prompts** pre-configured for different conversation phases
4. Configure system prompts:
   - Use pre-loaded templates or write custom prompts
   - Click "Enhance" to improve prompts with AI
5. Edit JSON configuration:
   - Modify stages, tasks, and conversation flows
   - Adjust rules (character limits, question limits)
   - Add/remove slots and actions
   - Click "Enhance with AI" in the JSON editor to improve entire config
6. Upload grounding files (optional):
   - Add PDFs, documents, or text files
   - These provide context to the LLM
7. Click "Save Draft" to save
8. Click "Publish" to make the scenario active

> **📖 See Full Template Documentation**: `/docs/DEFAULT_SCENARIO_TEMPLATE.md` for detailed explanation of the default structure, stages, rules, and customization guide.

### Using AI Enhancement

#### Enhance System Prompts
1. Enter or select a system prompt
2. Click the "Enhance" button next to the prompt field
3. AI will improve the prompt for clarity and effectiveness
4. Review and save

#### Enhance Configuration
1. Create a basic scenario configuration
2. Click "Enhance with AI" in the toolbar
3. AI will suggest improvements to the conversation flow
4. Review changes and save

### Scenario Configuration Structure

The default template uses a **stage-based, task-driven** conversation flow (see `/docs/DEFAULT_SCENARIO_TEMPLATE.md` for full details):

```json
{
  "version": "1.0",
  "bot_id": "sellrise_conversation_engine",
  "language_default": "en",
  "timezone_default": "Europe/Rome",
  
  "rules": {
    "one_question_rule": true,
    "max_sentences": 2,
    "max_chars": { "default": 420, "outbound": 520, "followup": 520 },
    "facts_only": true,
    "forbid": ["I think", "probably", "maybe"],
    "emoji": { "allowed": false }
  },
  
  "slots_schema": {
    "industry": { "type": "string", "required": false },
    "role": { "type": "string", "required": false },
    "email": { "type": "string", "required": false }
    // ... 8 more slots
  },
  
  "actions_catalog": {
    "#product_request": { "type": "send_products", "payload_schema": {...} },
    "#New_meeting_time": { "type": "book_meeting", "payload_schema": {...} },
    "#stop_script": { "type": "stop_automation", "payload_schema": {} },
    "#handover": { "type": "handover_to_human", "payload_schema": {...} }
  },
  
  "stages": [
    {
      "stage_id": "stage_0_outbound",
      "priority": 100,
      "entry_condition": { "type": "first_message" },
      "tasks": [
        {
          "task_id": "task_0_1",
          "priority": 100,
          "criteria": { "type": "always" },
          "instruction": "Send an outbound intro message...",
          "approved_phrases": ["Hi! I'm {agent_name}..."],
          "tags": []
        }
      ]
    }
    // ... 5 more stages (start, qualification, full_qualification, meeting_booking, closure)
  ],
  
  "followups": [
    {
      "followup_id": "followups_stages_1_3",
      "applies_to_stages": ["stage_1_start", "stage_2_quick_qualification", "stage_3_full_qualification"],
      "steps": [
        { "step_id": "fu_1", "criteria": { "type": "silence", "min_minutes": 60 }, "approved_phrases": ["..."] }
        // ... more followup steps
      ]
    }
  ],
  
  "system_prompts": {
    "main": "You are an AI conversation agent for lead qualification...",
    "qualification": "Your goal is to efficiently qualify leads...",
    "outbound": "When initiating outbound contact...",
    "meeting_booking": "When booking meetings...",
    "followup": "For follow-up messages after silence..."
  },
  
  "llm_config": {
    "model": "anthropic/claude-3.5-sonnet",
    "temperature": 0.7,
    "max_tokens": 2000
  }
}
```

**Key Concepts**:
- **Stages**: High-level conversation phases with priority-based selection
- **Tasks**: Specific actions within stages with conditional execution
- **Rules**: Enforce message quality (character limits, question limits, professional tone)
- **Slots**: Data fields to collect from leads
- **Actions**: Backend triggers (booking meetings, sending products, stopping automation)
- **Approved Phrases**: Template messages that comply with rules
- **Follow-ups**: Automated messages triggered by user silenceFor legacy/simple scenarios, you can still use the step-based format:

```json
{
  "entry_step_id": "step_1",
  "system_prompts": {
    "main": "You are a helpful AI assistant...",
    "qualification": "Ask qualifying questions...",
    "closing": "Guide towards booking..."
  },
  "steps": {
    "step_1": {
      "type": "message",
      "content": "Welcome message",
      "next": "step_2"
    },
    "step_2": {
      "type": "question_choice",
      "question": "How can we help?",
      "choices": ["Option A", "Option B"],
      "next": {
        "Option A": "step_3",
        "Option B": "step_4"
      }
    }
  },
  "llm_config": {
    "model": "anthropic/claude-3.5-sonnet",
    "temperature": 0.7,
    "max_tokens": 500
  }
}
```

### Supported Step Types

- **message**: Display text to visitor
- **question_text**: Free-text input
- **question_choice**: Multiple choice selection
- **form**: Multi-field form
- **conditional_branch**: Logic branching
- **kb_search**: Knowledge base lookup
- **handoff_booking_link**: Booking CTA
- **end**: Terminate conversation

## API Endpoints

### Scenarios
- `GET /v1/scenarios` - List all scenarios
- `POST /v1/scenarios` - Create scenario
- `GET /v1/scenarios/{id}` - Get scenario details
- `PATCH /v1/scenarios/{id}` - Update scenario
- `POST /v1/scenarios/{id}/publish` - Publish scenario

### LLM Enhancement
- `POST /v1/llm/enhance` - Enhance prompts or config
- `POST /v1/llm/generate-config` - Generate scenario from description

### File Upload
- `POST /v1/files/upload` - Upload file
- `DELETE /v1/files/{id}` - Delete file

## File Structure

```
Sellrise-Front-End/
├── src/
│   ├── pages/
│   │   └── ScenarioConfiguration/
│   │       ├── ScenarioConfiguration.jsx    # Main page
│   │       ├── index.js
│   │       └── components/
│   │           ├── JsonEditor.jsx           # JSON editor with validation
│   │           ├── SystemPromptEditor.jsx   # System prompt manager
│   │           ├── FileAttachment.jsx       # File upload component
│   │           └── ScenarioPreview.jsx      # Interactive preview
│   └── services/
│       └── api.js                           # API client

Sellrise-BackEnd/
├── app/
│   └── api/
│       └── v1/
│           ├── llm.py                       # LLM enhancement endpoints
│           ├── files.py                     # File upload endpoints
│           └── scenarios.py                 # Scenario CRUD (existing)
└── uploads/                                 # File upload directory
```

## Security Notes

- Only Admin role can create/edit/publish scenarios
- File uploads are workspace-isolated
- File types and sizes are validated
- OpenRouter API key is server-side only
- Uploaded files are stored in workspace-specific directories

## Troubleshooting

### "OpenRouter API key not configured"
- Ensure `OPENROUTER_API_KEY` is set in backend `.env`
- Restart backend server after adding the key

### File upload fails
- Check file size (max 10MB)
- Verify file type is supported
- Ensure `UPLOAD_DIR` exists and is writable

### JSON validation errors
- Ensure `entry_step_id` exists in `steps` object
- All step references must be valid
- Each step must have required fields for its type

### Enhancement not working
- Check OpenRouter API key is valid
- Check backend logs for API errors
- Ensure you have credits in your OpenRouter account

## Future Enhancements

- Visual flow diagram editor
- A/B testing for scenarios
- Analytics per scenario
- Template marketplace
- Multi-language support
- Collaborative editing

## Support

For issues or questions, please contact the development team or create an issue in the repository.
