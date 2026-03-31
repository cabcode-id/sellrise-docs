# PRD 6: Patient Cabinet Authentication & Onboarding

## Goal
Implement secure patient access to the Phlastic Cabinet and a guided 7-step onboarding wizard.

## Key Features
- **Auth Flow**: Invite-based account creation, password hashing, and auto-login.
- **Onboarding Wizard**: 7-step data collection process.
- **State Management**: Track `onboarding_completed` and `onboarding_step` in `patients` table.

## Success Metrics
- Patients can successfully create accounts via invite links.
- Onboarding progress is saved at each step.
- Wizard completion triggers `onboarding_completed` event in `patient_events`.
