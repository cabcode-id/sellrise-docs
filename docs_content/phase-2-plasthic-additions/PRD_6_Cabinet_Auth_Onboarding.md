# PRD 6: Patient Cabinet — Authentication & Onboarding

## Overview

| Field | Value |
|-------|-------|
| **Modules** | 6 (Contract Automation — BLOCKED) + 7 (Cabinet Auth) + 8 (Onboarding Wizard) |
| **System** | System 2: Phlastic Patient Service + Cabinet Frontend |
| **Owner** | Iga Narendra (CV Diantha) |
| **Estimate** | 6–8 days (Modules 7 + 8 combined; Module 6 blocked pending contract template) |
| **Dependencies** | Module 1 (patient DB with auth fields); Module 7 prerequisite for Module 8 |
| **Priority** | P1 — Core patient-facing experience |

---

## Module 6: Contract Automation

> ⚠️ **BLOCKED** — Waiting on Daria/Avi to provide:
> 1. Contract template (Word or PDF with placeholder fields)
> 2. Which data fields to insert into the contract
> 3. Signature format: e-sign (DocuSign/similar) or PDF download + wet signature
> 4. At which pipeline stage the contract is generated
> 5. Who initiates generation (doctor? admin? automatic?)

### What to Build (after template received)

**New endpoints on Phlastic Patient Service:**

```
POST   /api/v1/patients/:id/contracts       Generate contract PDF
GET    /api/v1/patients/:id/contracts/:cid  Download contract PDF
PATCH  /api/v1/patients/:id/contracts/:cid  Update status (sent/signed)
```

**Table: `contracts`**

| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| patient_id | UUID | FK → patients |
| template_version | string | Which template version was used |
| generated_pdf_path | string | File path to generated PDF |
| variables_used | jsonb | Snapshot of all variables inserted at generation time |
| status | enum | `generated` → `sent` → `viewed` → `signed` |
| sent_at | datetime | When emailed to patient |
| signed_at | datetime | When marked as signed |
| created_by | string | Who generated it |
| created_at | datetime | |

**Template variables (preliminary — confirm with Daria):**

| Variable | Source |
|----------|--------|
| `{{patient_name}}` | From patient record |
| `{{patient_email}}` | From patient record |
| `{{patient_phone}}` | From patient record |
| `{{procedure_name}}` | From `procedure_interest` array |
| `{{procedure_price}}` | TBD (needs pricing data) |
| `{{procedure_date}}` | TBD (needs scheduling) |
| `{{clinic_name}}` | `"Phlastic"` |
| `{{clinic_address}}` | From config |
| `{{doctor_name}}` | From config or assigned doctor |
| `{{today_date}}` | Generation date |

**Sellrise CRM changes:**
- "Generate Contract" button on lead detail page (visible only when `stage >= doctor_reviewed`)
- Clicking generates PDF → shows download link in CRM
- Contract status visible in timeline (`generated → sent → signed`)
- "Send to Patient" button → emails PDF to patient
- "Mark as Signed" → updates `contract.status=signed` + moves patient to `contract_signed` stage

**Module 6 Acceptance Tests** (after template received):

- [ ] "Generate Contract" button visible when `stage >= doctor_reviewed`
- [ ] Clicking generates PDF with patient data correctly inserted
- [ ] PDF downloadable from CRM
- [ ] "Send to Patient" sends PDF to patient email
- [ ] Event `contract_generated` created in `patient_events`
- [ ] Event `contract_sent` created when emailed
- [ ] "Mark as Signed" → `contract.status=signed` + `patient.stage=contract_signed` + event
- [ ] Contract PDF stored on Phlastic service (NOT in Sellrise)
- [ ] Multiple contracts possible per patient (e.g., revised contract)

---

## Module 7: Patient Cabinet Authentication

### Goal

Patient-facing authentication system. Patients receive an invite from staff, create their account, and log in to their personal cabinet.

**Tech stack:**

| Component | Choice |
|-----------|--------|
| Frontend | React / Next.js / any SPA framework |
| Served from | `https://cabinet.phlastic.com.au` |
| API calls | Phlastic REST API (`/auth/*` and `/cabinet/*` endpoints) |
| Auth tokens | JWT: access token (15 min) + refresh token (30 days, `httpOnly` cookie) |

---

### 7.1 — Invite Flow

```
1. Staff clicks "Send Cabinet Invite" in Sellrise CRM (Module 4.7)
2. CRM calls POST /auth/invite { patient_id }
3. Phlastic service:
   a. Generates unique invite_token (UUID, single-use, expires in 7 days)
   b. Saves invite_token + invite_sent_at on patient record
   c. Sends email to patient:
        Subject: "Welcome to Phlastic — Set Up Your Account"
        Body: Link to https://cabinet.phlastic.com.au/register?token={invite_token}
   d. Creates event: invite_sent
4. Returns 200 { "invite_sent": true, "expires_at": "2026-04-07T14:22:00Z" }
```

---

### 7.2 — Create Account Page

**URL:** `/register?token={invite_token}`

**Layout:** Split-screen
- Left: Full-bleed Phlastic branding photo
- Right: Registration form (562 px wide)

**UI:**
```
Welcome to the Aesthetic

We are glad to welcome you to our platform.
Please create your password to access your personal cabinet.

[Password]          — min 8 chars, show/hide toggle
[Confirm Password]  — must match

[Create Account]    — disabled until valid

Already have an account? [Sign In]
```

**Logic:**
```
1. Page loads → validate invite_token via GET /auth/validate-token?token=xxx
2. If invalid/expired → error: "This invite link has expired. Please contact Phlastic."
3. If valid → show form, pre-fill patient name from API response
4. On submit:
   a. POST /auth/register { invite_token, password }
   b. Backend: hash password (bcrypt), save to patient.password_hash
   c. Backend: set account_created=true
   d. Backend: invalidate invite_token (single-use — clear after use)
   e. Backend: create event: account_created { method: "invite_link" }
   f. Backend: return JWT access + refresh tokens
   g. Frontend: auto-login, redirect to onboarding wizard
```

**Password requirements:**
- Minimum 8 characters
- At least 1 letter + 1 number
- Validated client-side AND server-side

---

### 7.3 — Sign In Page

**URL:** `/login`

**Layout:** Split-screen (matches Figma)

**UI:**
```
Welcome back

Please enter your details

[Email]
[Password]      — show/hide toggle

[Sign In]

[Forgot password?]
```

**Logic:**
```
1. POST /auth/login { email, password }
2. Success → JWT tokens returned
   - If onboarding_completed=true  → redirect to /dashboard
   - If onboarding_completed=false → redirect to onboarding wizard
3. Wrong credentials → "Invalid email or password" (don't reveal which is wrong)
4. Account not created → "Account not found. Check your email for an invite link."
5. Rate limiting: max 5 attempts per email per 15 minutes
   - 6th attempt → blocked with message: "Too many attempts. Try again in 15 minutes."
```

---

### 7.4 — Forgot Password

**URL:** `/forgot-password`

```
1. User enters email
2. POST /auth/forgot-password { email }
3. If email exists + account_created=true → send password reset email:
   - Token expires in 1 hour
   - Reset link: https://cabinet.phlastic.com.au/reset-password?token={reset_token}
4. Always show: "If this email exists, we sent a reset link." (don't reveal if email exists)

Reset page (/reset-password?token=xxx):
1. Validate token
2. Show new password + confirm password form
3. POST /auth/reset-password { token, password }
4. Success → redirect to login
```

---

### 7.5 — JWT Token Management

| Token | Expiry | Storage |
|-------|--------|---------|
| Access token | 15 minutes | Memory / localStorage |
| Refresh token | 30 days | `httpOnly` cookie |

**Auto-refresh logic:**
```
1. On any 401 response → try refresh (POST /auth/refresh)
2. If refresh succeeds → retry original request with new access token
3. If refresh fails (token expired) → redirect to /login
4. POST /auth/logout → invalidate refresh token server-side
```

---

### Module 7 Acceptance Tests

- [ ] Invite email sent with correct link when staff clicks "Send Cabinet Invite"
- [ ] `/register` page loads with valid token, shows patient name pre-filled
- [ ] `/register` page shows error with invalid or expired token
- [ ] Password validation: min 8 chars, at least 1 letter + 1 number (client + server)
- [ ] Password mismatch shows validation error
- [ ] Successful registration → auto-login → redirect to onboarding wizard
- [ ] Invite token is single-use (second use → error message)
- [ ] Sign in with correct credentials → redirect to dashboard (or onboarding if not completed)
- [ ] Sign in with wrong credentials → generic error message (not revealing which field is wrong)
- [ ] Rate limiting: 6th attempt within 15 min → blocked with clear message
- [ ] Forgot password sends reset email (response doesn't reveal if email exists)
- [ ] Password reset works with valid token
- [ ] Password reset token expires after 1 hour
- [ ] JWT access token expires after 15 min, refresh automatically works
- [ ] Logout invalidates refresh token (subsequent refresh → redirect to login)
- [ ] All auth pages are mobile responsive
- [ ] Split-screen layout matches Figma design

---

## Module 8: Onboarding Wizard

### Goal

Step-by-step data collection wizard shown automatically on first login. Collects patient info not captured during chatbot interaction.

**Trigger:** Shown automatically after registration, OR on subsequent logins when `onboarding_completed = false`.

### Layout

Modal overlay:
- Desktop: 1144 px wide
- Mobile: Full-screen
- Left sidebar: Step numbers with progress indicator (✅ completed, 🔵 current, ⚫ future)
- Right content area: Step-specific form
- Top right: X button to close (saves progress, resumes on next login)

---

### Step 1 — Welcome

```
Welcome to Phlastic!

We'd love to learn more about you to provide the best possible experience.
This will only take a few minutes.

[Let's Start]
```

No data collected. Introduction only.

---

### Step 2 — Personal Details

**Fields:**
- Full name (first + last)
- Date of birth (`DD / MM / YYYY`)
- Gender (dropdown: Male / Female / Other / Prefer not to say)
- Phone (`+61 ...`)

**Saves to:** `patients.name`, `patients.date_of_birth`, `patients.gender`, `patients.phone`

---

### Step 3 — Location & Contact

**Fields:**
- Country (searchable dropdown)
- City (text input)
- Preferred contact method (radio: Email / Phone / WhatsApp)

**Saves to:** `patients.location` (composed from country + city)

---

### Step 4 — Medical Interest

**Fields:**
- Procedures interested in (checkbox grid):
  - Rhinoplasty, Facelift, Breast Augmentation, Liposuction, Tummy Tuck, Eyelid Surgery, Hair Transplant, Botox/Fillers, Other (free text)
- Approximate budget (dropdown: Under $5k / $5–10k / $10–20k / $20–50k / $50k+)
- Timeline (dropdown: ASAP / 1–3 months / 3–6 months / 6–12 months / Just exploring)
- Previous surgeries or medical conditions (textarea — optional)

**Saves to:** `patients.procedure_interest`, `patients.budget_range`, `patients.timeframe`, `patients.medical_notes`

**Pre-fill:** Chatbot data pre-fills overlapping fields if already collected.

---

### Step 5 — Photo Upload

```
Upload photos for your consultation

Our doctor will review these before your appointment.
Photos help us give you the most accurate assessment.

[Drag & drop or click to upload]

Recommended photos:
• Front face        → [Upload slot]
• Side profile      → [Upload slot]
• 45° angle         → [Upload slot]
• Area of interest  → [Upload slot]

☐ I consent to sharing these photos with the Phlastic medical team
  for consultation purposes only.

[Upload]    [Skip for now]
```

**Saves to:** `POST /patients/:id/photos` for each file (same API as chatbot upload, but `uploaded_via=cabinet`)

**Photo consent required** before upload is active. "Skip for now" always available.

---

### Step 6 — Preferences

**Fields:**
- How did you hear about Phlastic? (dropdown: Google / Social Media / Friend/Family / Doctor referral / Other)
- Receive updates about new services/offers? (Yes / No) → creates `marketing` consent record
- Preferred appointment times (checkboxes: Morning 8–12 / Afternoon 12–5 / Evening 5–8 / Weekends)

**Saves to:** `patients.qualification_data` JSON (or additional patient fields)

---

### Step 7 — Confirmation

```
Thank you, {name}! 🎉

Your profile is set up. Here's what happens next:

1. Our team will review your information
2. A specialist will contact you to schedule a consultation
3. You can track everything in your dashboard

[Go to Dashboard]
```

**On completion:**
- Set `onboarding_completed = true`, `onboarding_step = 7`
- Create event: `onboarding_completed { steps_completed: 7, time_spent_seconds: N }`
- Award **50 reward points** (creates entry in `rewards` table + `reward_earned` event)
- Redirect to `/dashboard`

---

### Progress Saving

- Each step saves data immediately on "Next" click (not only at wizard completion)
- API call: `POST /cabinet/onboarding/{step_number}` with step-specific data
- If user closes wizard mid-way → `onboarding_step` records last completed step number
- On next login → wizard resumes from last incomplete step
- Data from chatbot pre-fills overlapping fields (no duplicate entry for existing fields)

---

### Module 8 Acceptance Tests

- [ ] Wizard shows automatically after first login / registration
- [ ] Wizard shows on subsequent logins if `onboarding_completed=false`
- [ ] Step progress indicator shows correct state for each step (completed/current/future)
- [ ] Each step saves data to patient record via API on "Next"
- [ ] Closing wizard mid-way saves progress (`onboarding_step` updated)
- [ ] Re-opening wizard resumes from last incomplete step (not from beginning)
- [ ] Pre-fill from chatbot data works (procedure interest, budget, timeframe already filled)
- [ ] Photo upload in step 5 works (same flow as chatbot, but `uploaded_via=cabinet`)
- [ ] Photo consent required before upload button is active
- [ ] "Skip for now" works on photo step — proceeds to step 6
- [ ] Step 7 completion sets `onboarding_completed=true` and `onboarding_step=7`
- [ ] 50 reward points awarded on completion (visible in `/rewards`)
- [ ] After completion, redirect to `/dashboard` and wizard does NOT show again
- [ ] Full wizard is mobile responsive (full-screen on mobile)
- [ ] Sidebar step navigation visually matches Figma (numbered steps with progress indicators)
- [ ] Marketing consent (step 6 Yes/No) creates correct `consent_record` with `type=marketing`
