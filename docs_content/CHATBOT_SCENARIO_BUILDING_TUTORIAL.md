# Step-by-Step Tutorial: Build a Chatbot Scenario in Sellrise

This tutorial explains how to build a chatbot flow using the current Sellrise frontend scenario builder.

Scope for this guide:

- Build one complete scenario from scratch
- Configure tabs in the Scenario Configuration editor
- Save, simulate, and publish

## 1. Open the Scenario Builder

1. Log in to Sellrise.
2. In the left sidebar, click **Scenarios**.
3. Click **New Scenario**.

You will see a modal with:

- **Name** (required)
- **Description** (optional)
- **Initial Config (JSON)** (optional)

If you leave Initial Config empty, Sellrise starts from a default stage-based template.

## 2. Create the Scenario Draft

1. Enter a scenario name, for example: `Lead Qualification - Clinic`.
2. Add a short description of the business objective.
3. Click **Create**.

After creation, you are redirected to the full **Scenario Configuration** editor.

## 3. Understand the Editor Layout

At the top-right area of the editor, you have key actions:

- **Generate with AI**: generate a full config from business context
- **Enhance Config**: improve the current config using AI
- **Simulate**: test the scenario in a chat modal (available after scenario has an ID)
- **Save Draft**: persist your current changes
- **Publish**: make this scenario active

Main tabs:

- General
- Rules
- Slots
- Actions
- Stages
- Prompts
- LLM Config
- Follow-ups
- JSON Editor

## 4. Fill the General Tab First

In **General**:

1. Set **Scenario Name** (required).
2. Add **Description**.
3. Update **Version** (example: `1.0`, `1.1`).

Tip: use version bumping each time you make meaningful logic updates.

## 5. Define Communication Rules

In **Rules**, configure the guardrails for message quality:

1. Toggle **One question per message**.
2. Toggle **Facts only (no speculation)**.
3. Toggle **Allow emoji**.
4. Set **Max sentences per message** (UI enforces practical bounds).
5. Set **Character Limits** for:
   - default
   - outbound
   - followup
6. Add **Forbidden phrases** (comma-separated), for example: `I think, probably, maybe`.

These settings help keep chatbot replies concise, compliant, and consistent.

## 6. Define Data You Want to Collect (Slots)

In **Slots**:

1. Click **Add Slot**.
2. Rename the slot key (example: `industry`, `role`, `budget`, `email`).
3. Choose slot type (`string`, `number`, `boolean`, `datetime`, `email`).
4. Mark required slots with **Req** when needed.

Slots are the structured fields your chatbot should collect during conversation.

## 7. Define Triggerable Actions

In **Actions**:

1. Click **Add Action**.
2. Set action tag (usually prefixed with `#`, for example `#handover`).
3. Choose action type:
   - `handover_to_human`
   - `send_products`
   - `book_meeting`
   - `stop_automation`
   - `custom`

Use actions for events that must trigger backend behavior.

## 8. Build the Conversation Flow in Stages

This is the most important tab.

### 8.1 Add and Configure Stages

1. Open **Stages** and click **Add Stage**.
2. For each stage, set:
   - **Stage ID** (example: `stage_1_qualification`)
   - **Priority** (higher priority evaluated first)
   - **Instruction** (high-level AI behavior in this stage)
   - **Entry Condition** (when the stage should run)
   - **Required Slots** (comma-separated)
   - **Fallback Phrases**

Entry condition supports types such as:

- `always`
- `first_message`
- `all` / `any` / `none` with `when` clauses

Example `when` clauses:

- `slots.industry is not null`
- `slots.email is null`
- `context.user_replied == true`

### 8.2 Add Tasks Inside Each Stage

1. Inside a stage, click **Add Task**.
2. For each task, set:
   - **Task ID**
   - **Priority**
   - **Instruction**
   - **Criteria**
   - **Tags** (comma-separated, example `#handover`)
   - **Approved Phrases**

Recommended pattern:

1. Stage instruction defines strategic behavior.
2. Task instruction defines exact tactical behavior.
3. Approved phrases provide compliant phrasing examples.

## 9. Configure Prompt Blocks

In **Prompts**:

1. Add prompt templates from the dropdown (`main`, `qualification`, `outbound`, etc.).
2. Add custom prompt keys when needed.
3. Write/adjust prompt text.
4. Use **Enhance** per prompt key to improve wording with AI.

Keep prompts specific, short, and aligned with your Rules tab constraints.

## 10. Configure LLM Behavior

In **LLM Config**:

1. Review **Model** (fixed/locked by platform).
2. Set **Temperature** for creativity vs consistency.
3. Set **Max Tokens**.
4. Optionally enable **KB Only Strict Mode** in Settings if responses must rely only on knowledge base content.

## 11. Add Follow-up Sequences

In **Follow-ups**:

1. Click **Add Sequence**.
2. Add follow-up items with:
   - **delay** (example: `60m`, `24h`)
   - **message**
   - optional tag strategy in your flow design

Use follow-ups to re-engage leads that go silent.

## 12. Use JSON Editor for Power Editing

In **JSON Editor**, you get two workflows:

1. **AI Config Editor**: write natural-language instructions (example: "Add a booking stage after qualification") and click **Apply**.
2. **Raw JSON Configuration**: direct JSON editing with validation and formatting.

Best practice:

1. Build visually first.
2. Fine-tune in JSON only for advanced edits.
3. Return to visual tabs to verify structure still matches expectations.

## 13. Save, Simulate, Publish

1. Click **Save Draft** frequently.
2. Click **Simulate** and run realistic chat turns.
3. Validate:
   - stage transitions
   - slot collection
   - action tags
   - response quality and constraints
4. Click **Publish** when behavior is ready.

Important: published scenarios are production-facing, so always simulate before publish.

## 14. Suggested Build Order (Fast and Safe)

Use this order for most projects:

1. General
2. Rules
3. Slots
4. Actions
5. Stages + Tasks
6. Prompts
7. LLM Config
8. Follow-ups
9. JSON Editor final pass
10. Save Draft -> Simulate -> Publish

## 15. Common Mistakes to Avoid

1. Empty scenario name (save will fail).
2. Stage IDs and task IDs with spaces (prefer underscore format).
3. Missing approved/fallback phrasing, causing weak first responses.
4. No simulation before publish.
5. Overly broad prompts that conflict with Rules.

---

If you want, the next guide can cover scenario design patterns by use case (lead qualification, booking, support triage, reactivation) with ready-to-copy templates.
