# **Client AI Chatbot Setup Questionnaire**

Purpose:

Collect all information required to configure the AI conversation engine for your business.

The answers will be used to build:

- Knowledge Base
- Conversation Script
- Stage Logic
- Qualification Flow
- Actions & Integrations
- Follow-up Sequences

---

# **1. Company Information**

Basic business context.

**1.1 Company name**

**1.2 Website**

**1.3 Industry**

Examples:

- ecommerce
- real estate
- healthcare
- education
- consulting
- automotive
- SaaS

---

**1.4 Geographic markets**

Examples:

- Italy
- EU
- USA
- Global

---

**1.5 Languages the chatbot must support**

Examples:

- English
- Italian
- Spanish

---

# **2. Product / Service Information**

Information used to build the **knowledge base**.

---

**2.1 What products or services do you sell?**

Provide short descriptions.

Example:

```
Cargo bikes for families
Electric bikes
Bike accessories
Maintenance service
```

---

**2.2 What are the key benefits of your product?**

Examples:

```
saves time
eco-friendly transportation
safe for children
reduces commuting costs
```

---

**2.3 What problems does your product solve?**

Examples:

```
transporting children safely
urban mobility
reducing car dependency
saving time in traffic
```

---

**2.4 What makes your product unique compared to competitors?**

Examples:

```
lighter than competitors
stronger motor
custom configurations
expert consultation
```

---

**2.5 Pricing structure**

Provide one of the following:

- fixed price
- price range
- custom quote

Example:

```
€3,000–€6,000 depending on configuration
```

---

**2.6 Available packages or product categories**

Example:

```
Family bike
Cargo bike
Electric cargo bike
```

---

**2.7 Do you have product photos or product cards?**

If yes:

- provide links
- provide product segments

Example:

```
#product_request → send product cards
```

---

# **3. Ideal Customer Profile (ICP)**

Defines who the bot should talk to.

---

**3.1 Who is your ideal customer?**

Examples:

```
families with children
urban commuters
delivery companies
small businesses
```

---

**3.2 Customer demographics**

Examples:

```
age
location
family status
business type
```

---

**3.3 Customer motivations**

Examples:

```
saving time
sustainability
comfort
price
status
```

---

# **4. Lead Qualification Criteria**

Defines how the bot evaluates leads.

---

**4.1 What information must be collected from the lead?**

Examples:

```
city
budget
timeline
family size
business type
```

---

**4.2 Which questions are mandatory before presenting the product?**

Example:

```
How many children?
Age of children?
City?
Terrain type?
```

---

**4.3 When is a lead considered qualified?**

Example:

```
customer has children
customer interested in cargo bike
customer open to consultation
```

---

# **5. Sales Funnel Structure**

Defines the conversation stages.

---

**5.1 What is your typical sales process?**

Example:

```
1 lead inquiry
2 qualification
3 product presentation
4 consultation
5 purchase
```

---

**5.2 What is the main goal of the chatbot?**

Choose one:

```
lead qualification
product consultation
appointment booking
product sales
customer support
```

---

# **6. Consultation / Demo Process**

If the bot books calls.

---

**6.1 Do you offer consultations?**

Examples:

```
product demo
sales call
technical consultation
```

---

**6.2 How long is the consultation?**

Example:

```
15 minutes
30 minutes
```

---

**6.3 What calendar system do you use?**

Examples:

```
Google Calendar
Calendly
HubSpot
Custom CRM
```

---

**6.4 What information is required to book a meeting?**

Examples:

```
name
email
date/time
timezone
```

---

# **7. Objection Handling**

Used in **Stage 5 – refusal handling**.

---

**7.1 What are the most common objections?**

Examples:

```
too expensive
not enough time
just researching
need partner approval
```

---

**7.2 How should the bot respond to each objection?**

Provide short responses.

Example:

```
"It only takes 15 minutes and can save months of research."
```

---

# **8. Follow-up Strategy**

Defines the follow-up messages.

---

**8.1 How many follow-ups should be sent if the user does not respond?**

Examples:

```
3
4
```

---

**8.2 What value should be sent in follow-ups?**

Examples:

```
case studies
videos
guides
reviews
```

---

**8.3 When should the follow-ups stop?**

Examples:

```
after 4 attempts
after user refusal
after meeting booking
```

---

# **9. Actions & Integrations**

Defines backend actions.

---

**9.1 What actions should the bot trigger?**

Examples:

```
#product_request → send products
#book_demo → book meeting
#send_case → send case study
#stop_script → stop automation
#handover → transfer to human
```

---

**9.2 CRM Integration**

Which CRM is used?

Examples:

```
HubSpot
Salesforce
Zoho
Pipedrive
```

---

**9.3 Messaging platforms**

Where will the bot operate?

Examples:

```
WhatsApp
Telegram
Instagram
Website chat
Facebook Messenger
```

---

# **10. Content Resources**

Provide materials the bot can use.

Examples:

```
website pages
product catalogs
videos
customer reviews
case studies
PDF brochures
```

---

# **11. Brand Voice**

Defines the tone of the chatbot.

---

**11.1 Tone of voice**

Examples:

```
friendly
professional
luxury
playful
technical
```

---

**11.2 Message length preference**

Examples:

```
short messages
detailed explanations
```

---

# **12. Restrictions**

Define things the bot must NOT say.

Examples:

```
no guarantees
no discount promises
no legal advice
```

---

# **Final Deliverables**

Based on the answers, we will generate:

1. Knowledge Base
2. Conversation Script
3. Qualification Logic
4. Meeting Booking Flow
5. Objection Handling
6. Follow-up Sequences
7. AI Prompt Configuration
8. Integration Actions

---

If you want, I can also build **one extremely powerful thing for you next**:

A **“Chatbot Builder Template”** that takes this questionnaire and automatically converts it into:

- script_config.yaml
- knowledge_base.json
- stage_tasks.json

So you can deploy a **new AI chatbot for a client in ~30 minutes** instead of rebuilding everything each time.

---

# **Chatbot Builder Template**

**Purpose:** Automatically convert client questionnaire answers into configuration files required by the AI Conversation Engine.

This builder allows rapid deployment of new bots by generating the following configuration files:

```
knowledge_base.json
script_config.yaml
followups.json
bot_settings.json
client_config.json
```

The generated configuration will be consumed by the Conversation Engine.

---

# **1. Client Input Configuration**

The builder receives a single configuration file.

File:

```
client_config.json
```

Example:

```
{
  "company": {
    "name": "Donno Bikes",
    "website": "https://donnobikes.com",
    "industry": "mobility",
    "markets": ["Italy"],
    "languages": ["Italian"]
  },

  "products": [
    {
      "name": "Cargo Bike",
      "description": "Family cargo bike designed to transport children safely",
      "benefits": [
        "safe for children",
        "eco-friendly transport",
        "efficient urban mobility"
      ]
    }
  ],

  "ideal_customer": {
    "segments": [
      "families with children",
      "urban commuters"
    ]
  },

  "qualification_slots": [
    "children_count",
    "children_age",
    "city",
    "terrain"
  ],

  "sales_goal": "consultation_booking",

  "consultation": {
    "type": "video consultation",
    "duration_minutes": 15,
    "calendar": "google_calendar"
  },

  "followups": {
    "max_attempts": 4
  },

  "actions": [
    "#product_request",
    "#New_meeting_time",
    "#stop_script"
  ]
}
```

---

# **2. Knowledge Base Generation**

The builder generates a knowledge base file used by the LLM.

File:

```
knowledge_base.json
```

Example:

```
{
  "company": {
    "name": "Donno Bikes",
    "website": "https://donnobikes.com"
  },

  "products": [
    {
      "name": "Cargo Bike",
      "description": "Bike designed for transporting children safely",
      "benefits": [
        "safe family transportation",
        "eco-friendly mobility",
        "efficient city commuting"
      ]
    }
  ],

  "consultation": {
    "format": "video",
    "duration_minutes": 15,
    "calendar": "google_calendar"
  }
}
```

This file is injected into:

```
LLM_INPUT → knowledge_base
```

---

# **3. Script Configuration Generation**

The builder generates the conversation script used by the router.

File:

```
script_config.yaml
```

Example:

```
stages:

  - id: stage_1_start
    tasks:

      - id: greeting
        criteria: conversation_started
        instruction: greet the user and ask their goal
        approved_phrases:
          - "Hi! How can I help you today?"
          - "Hello! What are you looking for today?"

  - id: stage_2_qualification
    tasks:

      - id: collect_children_info
        criteria: children_unknown
        instruction: ask how many children will use the bike and their ages
        approved_phrases:
          - "How many children will ride in the bike and how old are they?"

  - id: stage_4_consultation
    tasks:

      - id: offer_consultation
        criteria: user_qualified
        instruction: offer video consultation
        approved_phrases:
          - "We offer a short 15-minute video consultation to help you choose the right bike."

      - id: collect_time
        criteria: user_accepts
        instruction: ask for a convenient time
        tags:
          - "#New_meeting_time"
```

---

# **4. Follow-up Configuration**

The builder generates follow-up messages used when the user becomes inactive.

File:

```
followups.json
```

Example:

```
{
  "stage_1_3_followups": [
    "Just checking if you're still there 🙂",
    "Did you have a chance to review my previous message?",
    "We can also schedule a quick 15-minute consultation if that’s easier."
  ],

  "stage_4_followups": [
    "If you'd like, we can still schedule a quick consultation.",
    "Here are some resources you might find helpful."
  ]
}
```

---

# **5. Bot Settings**

Defines system rules and messaging behavior.

File:

```
bot_settings.json
```

Example:

```
{
  "rules": {
    "max_questions": 1,
    "max_sentences": 2,
    "no_hallucination": true
  },

  "tone": "friendly professional",

  "languages": ["Italian"]
}
```

---

# **6. Builder Workflow**

The chatbot builder follows this workflow:

### **Step 1**

Client completes the questionnaire.

```
CLIENT_AI_BOT_SETUP_QUESTIONNAIRE
```

---

### **Step 2**

Operator converts responses into:

```
client_config.json
```

---

### **Step 3**

Builder generates configuration files:

```
knowledge_base.json
script_config.yaml
followups.json
bot_settings.json
```

---

### **Step 4**

Backend loads configuration:

```
load_bot_config()
```

---

### **Step 5**

Conversation Engine starts processing messages.

```
Router → Prompt Builder → LLM → Validator → Actions → State
```

---

# **7. Final Architecture**

```
Client Questionnaire
        ↓
client_config.json
        ↓
Chatbot Builder
        ↓
knowledge_base.json
script_config.yaml
followups.json
bot_settings.json
        ↓
AI Conversation Engine
        ↓
Router → LLM → Actions → State
```