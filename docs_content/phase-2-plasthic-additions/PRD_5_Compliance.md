# PRD 5: Compliance & Data Lifecycle

## Goal
Ensure full compliance with the Australian Privacy Act 1988 (APP) through robust data lifecycle management.

## Key Features
- **Data Export**: Endpoint to export all patient data as JSON.
- **Hard-Delete**: Permanently remove all patient data + photos.
- **Soft-Delete**: Anonymize PII in Sellrise, keep `lead_id` + "deleted" status.
- **Audit Logging**: Log every read/write operation for each patient.

## Success Metrics
- Hard-delete successfully wipes all data from DB and disk.
- Soft-delete successfully anonymizes PII in Sellrise.
- Audit log is accessible for specific patient IDs.
