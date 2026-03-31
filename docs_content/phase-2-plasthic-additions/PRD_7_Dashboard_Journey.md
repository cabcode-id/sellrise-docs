# PRD 7: Patient Dashboard & Journey Tracking

## Goal
Provide a personalized dashboard for patients to view their medical journey, upcoming services, and profile.

## Key Features
- **Dashboard**: Overview of service status, points balance, and next steps.
- **Journey Tracking**: Timeline of `journey_entries` (e.g., "Consultation Booked", "Photos Uploaded").
- **Profile Management**: View/edit personal info, avatar, and medical notes.

## Success Metrics
- Dashboard loads patient-specific data via Phlastic API.
- Journey timeline accurately reflects `patient_events`.
- Profile updates sync correctly to the Phlastic DB.
