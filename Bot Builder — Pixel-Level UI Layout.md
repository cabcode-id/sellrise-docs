## **Bot Builder — Pixel-Level UI Layout Specification**

This section specifies the **exact interface structure** for the Bot Builder. The goal is to make the **Stage → Task → Action architecture** easy to understand and modify while keeping configuration power accessible but not overwhelming.

The layout follows patterns used by products like Intercom, HubSpot, and ManyChat.

---

# **1. Bot Builder Main Layout**

The interface is divided into **four primary zones**.

```
┌───────────────────────────────────────────────────────────────┐
│ Top Toolbar                                                   │
├───────────────┬───────────────────────────────┬───────────────┤
│ Stage Panel   │ Conversation Flow Canvas      │ Properties    │
│ (Left)        │ (Center)                      │ Panel (Right) │
├───────────────┴───────────────────────────────┴───────────────┤
│ Bottom Testing Console                                        │
└───────────────────────────────────────────────────────────────┘
```

---

# **2. Top Toolbar**

Height: **64px**

### **Purpose**

Global controls for bot editing and deployment.

### **Elements**

Left side:

- Bot name
- Bot status indicator (Draft / Live)
- Save status

Center:

- Version selector
- Environment selector (Draft / Production)

Right side:

Buttons:

- **Test Bot**
- **Preview**
- **Publish**
- **More options (⋯)**

Dropdown actions:

```
Duplicate bot
Export config
Rollback version
Delete bot
```

---

# **3. Stage Panel (Left Sidebar)**

Width: **280px**

Displays the hierarchical structure of the bot.

### **Structure**

```
Bot
 ├ Stage 1 – Greeting
 │   ├ Task 1
 │   ├ Task 2
 │
 ├ Stage 2 – Qualification
 │   ├ Task 1
 │   ├ Task 2
 │
 ├ Stage 3 – Product Presentation
 │
 ├ Stage 4 – Booking
 │
 └ Stage 5 – Refusal Handling
```

### **Actions**

Users can:

- add stage
- rename stage
- reorder stages
- duplicate stage
- delete stage

Stage options menu:

```
Edit
Duplicate
Add task
Delete
```

---

# **4. Conversation Flow Canvas (Center)**

Width: **flexible**

This area visualizes the **conversation flow**.

### **Visual Flow Example**

```
Start
  │
  ▼
Stage 1: Greeting
  │
  ▼
Stage 2: Qualification
  │
  ▼
Stage 3: Product Presentation
  │
  ▼
Stage 4: Meeting Booking
```

Stages appear as **vertical nodes** connected by arrows.

Each stage contains its tasks.

---

## **Stage Node Design**

```
┌───────────────────────────────┐
│ Stage: Qualification          │
│ Entry condition: children_unknown │
├───────────────────────────────┤
│ Task 1                        │
│ Task 2                        │
│ Task 3                        │
├───────────────────────────────┤
│ + Add Task                    │
└───────────────────────────────┘
```

---

## **Task Card UI**

```
┌────────────────────────────────┐
│ Ask children age               │
│ Condition: children_unknown    │
├────────────────────────────────┤
│ Instruction:                   │
│ Ask how old the children are   │
├────────────────────────────────┤
│ Approved phrases:              │
│ • How old are your children?   │
│ • What ages are the kids?      │
├────────────────────────────────┤
│ Actions:                       │
│ none                           │
└────────────────────────────────┘
```

Users can drag tasks within a stage.

---

# **5. Properties Panel (Right)**

Width: **360px**

This panel displays configuration for the selected element.

---

## **Stage Configuration**

Fields:

```
Stage name
Stage description
Entry condition
Fallback behavior
Follow-up rules
```

Example:

```
Entry condition:
children_age_unknown
```

---

## **Task Configuration**

Fields:

```
Task name
Task criteria
Instruction
Approved phrases
Action tags
```

Example configuration:

```
Task name:
Ask Children Age

Criteria:
children_age_unknown

Instruction:
Ask the user how old their children are.

Approved phrases:
• How old are your children?

Tags:
none
```

---

# **6. Action Configuration**

Actions represent backend integrations.

Examples:

```
Send product cards
Book meeting
Send case study
Stop script
Transfer to human
```

Mapped to backend tags:

```
#product_request
#New_meeting_time
#send_case
#stop_script
#handover
```

Users choose actions from a dropdown.

---

# **7. Bottom Testing Console**

Height: **260px**

Purpose: simulate conversations with the bot.

### **Layout**

```
-----------------------------------------
| Chat Simulator                        |
-----------------------------------------
| User:                                 |
| Bot:                                  |
-----------------------------------------
| Input message field                   |
-----------------------------------------
```

Features:

- simulate user message
- show stage/task triggered
- display executed actions
- debug output

Example debug panel:

```
Stage triggered: Qualification
Task triggered: Ask children age
Action: none
```

---

# **8. Conversation Inbox Wireframe**

This interface is similar to the inbox used by platforms like Drift and Intercom.

---

## **Layout**

```
┌──────────────┬──────────────────────────────┬──────────────┐
│ Conversations│ Conversation Timeline         │ Lead Profile │
│ List         │                               │ Panel        │
└──────────────┴──────────────────────────────┴──────────────┘
```

---

## **Conversation List**

Columns:

```
User
Channel
Last message
Status
Time
```

Filters:

```
Active
Waiting
Leads
Escalated
Closed
```

---

## **Conversation Timeline**

Displays chat history.

```
User message
Bot response
Action executed
Timestamp
```

Example:

```
User: I need a cargo bike
Bot: How many children will use it?
```

---

## **Lead Profile Panel**

Displays captured user information.

Fields:

```
Name
Email
Phone
City
Children count
Lead score
```

Actions:

```
Add note
Assign agent
Export to CRM
```

---

# **9. Analytics Dashboard Layout**

---

## **KPI Section**

```
Total Conversations
Qualified Leads
Booked Meetings
Conversion Rate
```

---

## **Conversation Funnel**

```
Visitors
 ↓
Conversations
 ↓
Qualified Leads
 ↓
Meetings
 ↓
Sales
```

---

## **Stage Drop-off Chart**

Shows where conversations fail.

Example:

```
Greeting stage: 5%
Qualification stage: 35%
Product stage: 18%
Booking stage: 12%
```

---

# **10. Core UX Design Goals**

### **1. Clarity**

Users must understand conversation logic instantly.

---

### **2. Low technical barrier**

Business users must be able to configure bots without code.

---

### **3. Visual automation**

Conversation logic should feel like **building a workflow**.

---

### **4. AI assistance**

Users should be able to click:

```
Generate bot with AI
```

and automatically produce:

- stages
- tasks
- responses

---

# **Next logical product step**

To fully define the product, the next thing to design is:

1. **Complete Information Architecture Diagram**
2. **Bot Builder Interaction Model**
3. **Multi-Bot Workspace UX**
4. **Template Marketplace for Bots**

Those four elements transform the platform from **a tool into a scalable SaaS product**.