# PRD 1: Phlastic Patient Service (Core Engine)

## Goal
Build a standalone, AU-compliant microservice to manage patient PII and medical data, ensuring complete isolation from the Sellrise CRM.

## Key Features
- **Database Schema**: Implement `patients`, `patient_events`, `photos`, and `consent_records` tables.
- **Compliance Layer**: 
    - Encryption at rest (AES-256).
    - TLS 1.2+ for all API traffic.
    - Audit logging for every read/write operation.
- **API Foundation**: 
    - RESTful CRUD endpoints for patient management.
    - API Key authentication (min 32 chars, one per workspace).

## Success Metrics
- Service health check (`/health`) returns `{"status":"ok","db":"connected"}`.
- Audit logs correctly record all patient data interactions.
- Data is successfully encrypted in the database.
