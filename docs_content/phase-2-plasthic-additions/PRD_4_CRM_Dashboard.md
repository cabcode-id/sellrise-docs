# PRD 4: CRM Unified View & Staff Dashboard

## Goal
Provide a unified view of patient data within the Sellrise CRM while keeping PII in the Phlastic service.

## Key Features
- **Unified View**: Fetch data from Phlastic API when staff views lead details in Sellrise.
- **Pipeline Management**: Sync pipeline stages (`new` → `lost`) between Sellrise and Phlastic.
- **Staff Actions**: 
    - Add notes (`/patients/:id/notes`).
    - Update patient info directly from CRM.

## Success Metrics
- Sellrise CRM displays Sellrise data + Phlastic patient data seamlessly.
- Pipeline stage changes are reflected in both systems.
- Staff can add notes and update info without latency issues.
