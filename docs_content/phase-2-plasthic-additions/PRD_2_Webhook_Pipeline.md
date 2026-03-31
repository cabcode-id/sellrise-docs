# PRD 2: Webhook & Data Pipeline

## Goal
Integrate Sellrise (System 1) with Phlastic Patient Service (System 2) to ensure seamless data flow while maintaining PII isolation.

## Key Features
- **Webhook Handler**: 
    - Receive data from Sellrise chatbot.
    - Map chatbot answers to patient profile fields.
    - Store mandatory consent records (timestamp + IP + version).
- **Upsert Logic**: 
    - Merge data if patient (email + workspace) already exists.
    - Create `patient_updated` event on merge.
- **Reference Mapping**: 
    - Store `external_patient_id` in Sellrise.
    - Store `sellrise_lead_id` in Phlastic.

## Success Metrics
- Lead data from chatbot successfully populates Phlastic DB.
- `consent_given=true` is verified before any storage.
- `external_patient_id` is correctly updated in Sellrise lead record.
