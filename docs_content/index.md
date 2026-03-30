# Sellrise Documentation

Welcome to the Sellrise platform documentation. Use the navigation to browse through the different sections.

[Read the Chatbot Scenario Build Tutorial](.CHATBOT_SCENARIO_BUILDING_TUTORIAL)

## Sections

| Section | Description |
|---------|-------------|
| **Product Requirements** | PRDs, feature breakdowns, and MVP status |
| **AI Conversation Engine** | Technical specs, architecture, and flow diagrams |
| **Scenario System** | Configuration guides, step types, and templates |
| **CRM** | Lead management PRDs (creation, inbox, pipeline, scoring, etc.) |
| **Implementation** | Event logging and implementation audit details |
| **Testing** | Integration testing guides (English & Indonesian) |
| **New Features** | Upcoming features like the AI Scenario Generator |


Addendum: Scope of Work Addition

In addition to chatbot development, the first party offers, and both parties agree, to expand the scope of work beyond intelligent chatbot development. Both parties agree to add six new scope items: database creation, user profiling, chatbot integration with the second party's system, Customer Relationship Management setup (hereinafter referred to as CRM), user contract automation, and email validation. Any scope not explicitly mentioned in these six points will be discussed separately and must be agreed by both parties.

1) Database Creation
- The first party agrees to support the second party in enhancing the second party's website backend system.
- Support is defined as adding database capability, reconfiguring the existing architecture, or creating a separate backend system with a standard visitor-based specification in a microservice manner.
- The first party has the right to advise the second party on backend configuration and technical specification, and the final specification must be agreed by both parties.
- The first party has the right to coordinate backend and database deployment with the second party, and deployment may be executed by one party or both parties based on future agreement.
- The first party has the right to begin development using the most efficient query-based database configuration.
- The second party retains full ownership responsibility for the database and may delegate maintenance to the first party, but may not delegate legal or social responsibility.

2) User Profiling
- The first party agrees to support the second party in user profiling feature development.
- User profiling is defined as integration between the first party's chatbot and the second party's backend and database to enable smarter data collection, preference gathering, and future customization using conversational analytics and artificial intelligence.
- The second party agrees to provide and share frontend design and business flow for the user profiling feature with the first party.
- The first party agrees to develop the user profiling feature based on the second party's design and business flow. Any changes to design or business flow must be informed and agreed by both parties.
- The first party has the right to share development workload with the second party. For example, the first party may focus on backend and data management, while the second party develops frontend design and web application implementation.

3) Chatbot Integration
- To maximize the value and capability of points 1 and 2, both parties agree to integrate their services, including data sharing and related capabilities through Application Programming Interface (API) or webhook triggers (hereinafter referred to as integration).
- The second party has full responsibility for the integration.
- The second party may delegate technical maintenance to the first party but may not delegate legal or social responsibility for this integration.
- The first party agrees to support the integration process and testing using standard security protocols, including personal API keys that can only be generated when the second party logs in to the first party's platform using valid credentials.
- The first party has the right to initiate integration using a standardized payload in JavaScript Object Notation (JSON), including:
        - API security key in request header
        - Sellrise user ID
        - Sellrise user name
        - Lead ID
        - Complete lead data

4) CRM Setup
- The first party agrees to develop a CRM system within the first party's user platform, owned by the first party.
- The second party agrees to receive CRM customizations for the second party's business needs.
- The first party agrees to support the second party in connecting points 1, 2, and 3 into CRM workflows.
- The first party has the right to coordinate CRM development with the second party.
- The first party has the right to begin development using the most efficient query-based database configuration.
- The second party retains full ownership responsibility for lead data and may delegate maintenance to the first party, but may not delegate legal or social responsibility.

5) User Contract Automation
- The first party agrees to support the second party in developing user contract automation features for faster agreement handling and document processing.
- User contract automation includes digital contract generation, agreement workflow setup, and integration with contract signing through PandaDoc based on the second party's business flow.
- The second party agrees to provide contract templates, approval flow, legal wording, and required signatory data format to the first party.
- The first party agrees to implement the automation workflow based on approved templates and process, while legal content changes remain the second party's responsibility.
- The first party has the right to coordinate technical implementation and integration with related systems, including CRM and user profiling data where needed.
- The second party has full responsibility for contract content ownership, legal validity, and regulatory compliance, and may delegate technical maintenance to the first party.

6) Email Validation and Identification for Chatbot
- The first party agrees to develop and integrate email validation and identification capabilities within chatbot interactions.
- Email validation includes regex-based email format validation followed by deliverability checking using a third-party email verification service to reduce invalid addresses and minimize email bounce risk.
- The second party agrees to define validation rules, business conditions, and follow-up flow after email validation success or failure.
- The first party agrees to connect validated email data into relevant systems, including backend database records, chatbot profile data, and CRM lead records.
- The first party has the right to begin implementation using secure and efficient methods for email data handling while maintaining compatibility with existing integration points.
- The second party has full responsibility for user consent, legal usage, and communication policy for collected email data, and may delegate technical maintenance to the first party.
