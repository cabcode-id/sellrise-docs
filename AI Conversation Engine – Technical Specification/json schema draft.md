Below are **Draft-07 JSON Schema** definitions for strict validation of LLM_INPUT and LLM_OUTPUT.

You can drop these into:

- schemas/llm_input.schema.json
- schemas/llm_output.schema.json

They are intentionally **strict** (additionalProperties: false) to prevent silent drift.

---

## **LLM_INPUT**

## **— JSON Schema (Draft-07)**

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://sellrise.ai/schemas/llm_input.schema.json",
  "title": "LLM_INPUT",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "meta",
    "rules",
    "knowledge_base",
    "script",
    "state",
    "runtime",
    "last_user_message"
  ],
  "properties": {
    "meta": {
      "type": "object",
      "additionalProperties": false,
      "required": ["bot_id", "version", "language_default"],
      "properties": {
        "bot_id": { "type": "string", "minLength": 1 },
        "version": { "type": "string", "minLength": 1 },
        "language_default": { "type": "string", "minLength": 2, "maxLength": 10 },
        "timezone_default": { "type": "string", "minLength": 1 }
      }
    },

    "rules": {
      "type": "object",
      "additionalProperties": false,
      "required": ["one_question_rule", "max_sentences", "facts_only", "emoji"],
      "properties": {
        "one_question_rule": { "type": "boolean" },
        "max_sentences": { "type": "integer", "minimum": 1, "maximum": 10 },
        "facts_only": { "type": "boolean" },
        "forbid": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "uniqueItems": true
        },
        "max_chars": {
          "type": "object",
          "additionalProperties": false,
          "required": ["default"],
          "properties": {
            "default": { "type": "integer", "minimum": 50, "maximum": 4000 },
            "outbound": { "type": "integer", "minimum": 50, "maximum": 4000 },
            "followup": { "type": "integer", "minimum": 50, "maximum": 4000 }
          }
        },
        "emoji": {
          "type": "object",
          "additionalProperties": false,
          "required": ["allowed"],
          "properties": {
            "allowed": { "type": "boolean" }
          }
        }
      }
    },

    "knowledge_base": {
      "type": "object",
      "description": "Facts-only payload. No rules, no logic. Can be any JSON object.",
      "additionalProperties": true
    },

    "script": {
      "type": "object",
      "additionalProperties": false,
      "required": ["stages", "actions_catalog"],
      "properties": {
        "stages": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/definitions/Stage" }
        },
        "actions_catalog": {
          "type": "object",
          "additionalProperties": { "$ref": "#/definitions/ActionCatalogItem" }
        },
        "followups": {
          "type": "array",
          "items": { "$ref": "#/definitions/FollowupSequence" }
        }
      }
    },

    "state": {
      "type": "object",
      "additionalProperties": false,
      "required": ["stage_id", "task_id", "slots", "context", "used_phrases", "used_tags", "history"],
      "properties": {
        "stage_id": { "type": ["string", "null"] },
        "task_id": { "type": ["string", "null"] },
        "slots": {
          "type": "object",
          "description": "Current slot values. Flexible keys, but values must be JSON primitives/arrays/objects.",
          "additionalProperties": true
        },
        "context": { "$ref": "#/definitions/Context" },
        "used_phrases": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "uniqueItems": true
        },
        "used_tags": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "uniqueItems": true
        },
        "history": {
          "type": "array",
          "items": { "$ref": "#/definitions/HistoryMessage" }
        }
      }
    },

    "runtime": {
      "type": "object",
      "additionalProperties": false,
      "required": ["now_iso", "channel"],
      "properties": {
        "now_iso": { "type": "string", "format": "date-time" },
        "channel": {
          "type": "string",
          "enum": ["whatsapp", "telegram", "instagram", "webchat", "sms", "email", "other"]
        },
        "user_locale": { "type": "string", "minLength": 2, "maxLength": 20 }
      }
    },

    "last_user_message": {
      "type": "object",
      "additionalProperties": false,
      "required": ["text"],
      "properties": {
        "text": { "type": "string", "minLength": 1 },
        "message_id": { "type": "string" },
        "timestamp_iso": { "type": "string", "format": "date-time" }
      }
    }
  },

  "definitions": {
    "Context": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "user_initiated",
        "user_ignored_last_questions",
        "user_asked_new_question",
        "channel_preference",
        "full_qualification_allowed",
        "full_qualification_complete",
        "user_requested_products",
        "user_requested_meeting",
        "user_agreed_to_meeting",
        "user_refused_meeting",
        "objection_reason",
        "meeting_offered",
        "meeting_confirmed",
        "stop_script",
        "conversation_complete",
        "last_bot_question"
      ],
      "properties": {
        "user_initiated": { "type": "boolean" },
        "user_ignored_last_questions": { "type": "boolean" },
        "user_asked_new_question": { "type": "boolean" },
        "channel_preference": { "type": ["string", "null"], "enum": ["chat", "call", null] },
        "full_qualification_allowed": { "type": "boolean" },
        "full_qualification_complete": { "type": "boolean" },
        "user_requested_products": { "type": "boolean" },
        "user_requested_meeting": { "type": "boolean" },
        "user_agreed_to_meeting": { "type": "boolean" },
        "user_refused_meeting": { "type": "boolean" },
        "objection_reason": { "type": ["string", "null"] },
        "meeting_offered": { "type": "boolean" },
        "meeting_confirmed": { "type": "boolean" },
        "stop_script": { "type": "boolean" },
        "conversation_complete": { "type": "boolean" },
        "last_bot_question": { "type": ["string", "null"] }
      }
    },

    "HistoryMessage": {
      "type": "object",
      "additionalProperties": false,
      "required": ["role", "text", "timestamp_iso"],
      "properties": {
        "role": { "type": "string", "enum": ["user", "assistant", "system"] },
        "text": { "type": "string" },
        "timestamp_iso": { "type": "string", "format": "date-time" }
      }
    },

    "Stage": {
      "type": "object",
      "additionalProperties": false,
      "required": ["stage_id", "priority", "entry_condition", "tasks"],
      "properties": {
        "stage_id": { "type": "string", "minLength": 1 },
        "priority": { "type": "integer", "minimum": 0, "maximum": 1000 },
        "entry_condition": { "$ref": "#/definitions/Condition" },
        "required_slots": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "uniqueItems": true
        },
        "tasks": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/definitions/Task" }
        }
      }
    },

    "Task": {
      "type": "object",
      "additionalProperties": false,
      "required": ["task_id", "priority", "criteria", "instruction"],
      "properties": {
        "task_id": { "type": "string", "minLength": 1 },
        "priority": { "type": "integer", "minimum": 0, "maximum": 1000 },
        "criteria": { "$ref": "#/definitions/Condition" },
        "instruction": { "type": "string", "minLength": 1 },
        "approved_phrases": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "minItems": 0
        },
        "tags": {
          "type": "array",
          "items": { "type": "string", "pattern": "^#" },
          "uniqueItems": true
        }
      }
    },

    "Condition": {
      "type": "object",
      "additionalProperties": false,
      "required": ["type"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["always", "any", "all", "first_message", "silence"]
        },
        "when": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 }
        },
        "min_minutes": { "type": "integer", "minimum": 1 }
      },
      "allOf": [
        {
          "if": { "properties": { "type": { "const": "silence" } } },
          "then": { "required": ["min_minutes"] }
        }
      ]
    },

    "ActionCatalogItem": {
      "type": "object",
      "additionalProperties": false,
      "required": ["type", "payload_schema"],
      "properties": {
        "type": { "type": "string", "minLength": 1 },
        "payload_schema": {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "additionalProperties": false,
            "required": ["type"],
            "properties": {
              "type": { "type": "string", "minLength": 1 },
              "required": { "type": "boolean" }
            }
          }
        }
      }
    },

    "FollowupSequence": {
      "type": "object",
      "additionalProperties": false,
      "required": ["followup_id", "applies_to_stages", "steps"],
      "properties": {
        "followup_id": { "type": "string", "minLength": 1 },
        "applies_to_stages": {
          "type": "array",
          "items": { "type": "string", "minLength": 1 },
          "minItems": 1,
          "uniqueItems": true
        },
        "steps": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/definitions/FollowupStep" }
        }
      }
    },

    "FollowupStep": {
      "type": "object",
      "additionalProperties": false,
      "required": ["step_id", "criteria", "approved_phrases"],
      "properties": {
        "step_id": { "type": "string", "minLength": 1 },
        "criteria": { "$ref": "#/definitions/Condition" },
        "approved_phrases": {
          "type": "array",
          "minItems": 1,
          "items": { "type": "string", "minLength": 1 }
        },
        "tags": {
          "type": "array",
          "items": { "type": "string", "pattern": "^#" },
          "uniqueItems": true
        }
      }
    }
  }
}
```

---

## **LLM_OUTPUT**

## **— JSON Schema (Draft-07)**

This schema enforces:

- exactly one message string
- stage_id + task_id required
- actions strictly shaped
- tags must start with #
- no extra fields

```
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://sellrise.ai/schemas/llm_output.schema.json",
  "title": "LLM_OUTPUT",
  "type": "object",
  "additionalProperties": false,
  "required": ["stage_id", "task_id", "assistant_message", "slots_update", "actions"],
  "properties": {
    "stage_id": { "type": "string", "minLength": 1 },
    "task_id": { "type": "string", "minLength": 1 },

    "assistant_message": {
      "type": "string",
      "minLength": 1,
      "description": "User-facing message only. Must not contain tags or JSON."
    },

    "slots_update": {
      "type": "object",
      "description": "Only slots the model is confident about. Empty object allowed.",
      "additionalProperties": true
    },

    "actions": {
      "type": "array",
      "items": { "$ref": "#/definitions/Action" }
    },

    "debug": {
      "type": "object",
      "additionalProperties": false,
      "description": "Optional diagnostics; safe to disable in production.",
      "properties": {
        "selected_approved_phrase_index": { "type": "integer", "minimum": 0 },
        "reasoning_short": { "type": "string", "maxLength": 400 }
      }
    }
  },

  "definitions": {
    "Action": {
      "type": "object",
      "additionalProperties": false,
      "required": ["type", "tag", "payload"],
      "properties": {
        "type": {
          "type": "string",
          "enum": [
            "send_products",
            "book_meeting",
            "stop_automation",
            "handover_to_human",
            "send_materials",
            "noop"
          ]
        },
        "tag": { "type": "string", "pattern": "^#" },
        "payload": {
          "type": "object",
          "description": "Action payload shape should match actions_catalog[tag].payload_schema.",
          "additionalProperties": true
        }
      }
    }
  }
}
```

---

## **Implementation Notes (strict validation)**

### **1) Validate LLM output twice**

- JSON parse
- JSON Schema validate (llm_output.schema.json)
- then run your **business validator** (1 question, 2 sentences, etc.)

### **2) Make tags canonical**

Even if the model outputs #new_meeting_time vs #New_meeting_time, normalize or reject (your choice). For strictness, I recommend **reject and repair once**.

### **3) Enforce “tags not in assistant_message”**

Schema can’t fully guarantee that. Add a simple regex check:

- reject if message contains #\w+