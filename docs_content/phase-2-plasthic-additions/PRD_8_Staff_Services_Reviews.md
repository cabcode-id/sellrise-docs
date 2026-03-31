# PRD 8: Staff, Services, & Reviews

## Goal
Manage medical staff, track patient procedures, and collect patient feedback.

## Key Features
- **Staff Management**: CRUD for doctors, nurses, and tour operators.
- **Service Tracking**: Track procedure status (`planned`, `scheduled`, `completed`).
- **Review System**: Patient-submitted reviews (1-5 stars) with approval workflow.

## Success Metrics
- Staff profiles are visible to patients.
- Service status updates trigger `patient_events`.
- Reviews are submitted, moderated, and displayed if `is_public=true`.
