# Phlastic Phase 2 — Sprint Planning

**Project:** Phlastic Patient Service + Cabinet (Scope 2)  
**Client:** Phlastic (Avi)  
**Contractor:** CV Diantha (Iga Narendra)  
**Ordered by:** Sellrise Limited (Ivan, Director)  
**Total effort:** ~33–42 days (across 4–5 sprints)  
**Sprint length:** 5 working days (1 week)

---

## Dependency Map

```
Module 1 (Patient Service — DB + API)
    ├── Module 2 (Webhook Pipeline)
    │       └── Module 3 (Photo Upload in Chatbot)
    │               └── Module 4 (CRM Customization)
    ├── Module 5 (Email Validation)        [parallel with Module 2]
    ├── Module 6 (Contract Automation)     [BLOCKED — needs template from Daria]
    └── Module 7 (Cabinet Auth)            [parallel with Module 4]
            └── Module 8 (Onboarding Wizard)
                    └── Module 9 (Dashboard)
                            ├── Module 10 (Services & Journey)
                            ├── Module 11 (Staff, Reviews, Rewards)
                            └── Module 12 (Community)
```

---

## Sprint 1 — Backend Foundation

**Duration:** Days 1–5  
**Theme:** Core infrastructure — Patient Service API, DB, compliance endpoints, email validation  
**Deliverable:** Fully functional Phlastic Patient Service running on AU VPS, ready to receive data

### Pre-Sprint Blockers (Must Be Resolved Before Day 1)

| Blocker | Owner | Action |
|---------|-------|--------|
| VPS access credentials OR DigitalOcean API token | Ivan | Provide before Day 1 |
| Domain DNS setup: `patients.phlastic.com.au` | Ivan | Configure before Day 1 |
| Email service credentials (Resend / SendGrid / SES) | Ivan | Provide before Day 1 |

---

### Sprint 1 Backlog

#### TASK-1.1 — VPS Setup & Infrastructure
**PRD:** Module 1 (Infrastructure section)  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] Provision VPS (2 vCPU, 4 GB RAM, 50 GB SSD) in AU region (Sydney preferred)
- [ ] Install Ubuntu 22.04 LTS
- [ ] Configure SSH (key-based auth only — no password login)
- [ ] Install runtime (Node.js / Python / Go) and database (PostgreSQL / MySQL)
- [ ] Configure firewall: port 443 open, port 80 → redirect to 443
- [ ] Set up SSL with Let's Encrypt + certbot auto-renew
- [ ] Configure daily automated DB backup (30-day retention)
- [ ] Verify VPS region: `curl ifconfig.me` confirms AU IP

**Acceptance:**  
- [ ] VPS responds on `https://patients.phlastic.com.au/health`  
- [ ] `{"status":"ok","db":"connected"}` returned

---

#### TASK-1.2 — Database Schema
**PRD:** PRD 1 (Database Schema section)  
**Estimate:** 1 day  
**Owner:** Iga

Create all tables:

- [ ] `patients` — all fields including auth fields (`password_hash`, `invite_token`, `onboarding_step`, etc.)
- [ ] `patient_events` — full event_type enum
- [ ] `photos` — with storage path pattern
- [ ] `consent_records` — with NO CASCADE on `patient_id` FK
- [ ] `services` — for procedure tracking
- [ ] `staff` — for doctor/nurse profiles
- [ ] `reviews` — with approval workflow fields
- [ ] `rewards` — for points transactions
- [ ] `journey_entries` — for patient timeline
- [ ] `community_groups` — for community feature
- [ ] `community_posts` — for community posts
- [ ] `contracts` — for contract automation (future module)
- [ ] Enable database encryption at rest (AES-256)
- [ ] Run migrations successfully

---

#### TASK-1.3 — Core Patient API Endpoints
**PRD:** PRD 1 (API Specification section)  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] `POST /patients` — create patient with consent validation + upsert logic
- [ ] `GET /patients` — list with pagination + filters (stage, email_status, procedure_interest, date range, owner, search, sort)
- [ ] `GET /patients/:id` — single patient + photos metadata + events
- [ ] `PUT /patients/:id` — update patient fields
- [ ] `PATCH /patients/:id/stage` — stage change with event creation
- [ ] `POST /patients/:id/notes` — add note (creates event)
- [ ] `GET /health` — health check endpoint
- [ ] API Key authentication middleware (`X-API-Key` header)
- [ ] CORS configuration (only Sellrise + Cabinet domains)
- [ ] Consistent error response format across all endpoints

---

#### TASK-1.4 — Photo Upload & Retrieval
**PRD:** PRD 1 (Photos section) + PRD 3  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] `POST /patients/:id/photos` — multipart upload with validation (10 MB limit, MIME type check)
- [ ] `GET /patients/:id/photos/:pid` — binary download (auth required)
- [ ] `DELETE /patients/:id/photos/:pid` — delete photo + file
- [ ] File stored in non-web-accessible directory (`/data/photos/{patient_id}/{uuid}.ext`)
- [ ] Creates `photo_uploaded` event + `photo_sharing` consent record on upload

---

#### TASK-1.5 — Compliance Endpoints & Audit Log
**PRD:** PRD 5 (Compliance section)  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] `DELETE /patients/:id` — soft delete (anonymize PII, set `is_deleted=true`)
- [ ] `POST /patients/:id/hard-delete` — permanent deletion (files + DB records, keep `consent_records`)
- [ ] `GET /patients/:id/export` — full JSON export (patient + photos base64 + events + consents)
- [ ] `GET /patients/export/csv` — bulk CSV export with filters
- [ ] `GET /audit-log?patient_id=:id` — access log for specific patient
- [ ] Every read/write operation creates audit event in `patient_events`

---

#### TASK-1.6 — Email Validation (Async)
**PRD:** PRD 5 (Email Validation section)  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] Background async job triggered after `POST /patients` (non-blocking)
- [ ] Format check (regex validation)
- [ ] MX record check (DNS lookup)
- [ ] Update `email_status`: `valid` / `invalid` / `unknown`
- [ ] Re-trigger on email update via `PUT /patients/:id`
- [ ] DNS timeout handled gracefully (`unknown` status, no server error)

---

### Sprint 1 Definition of Done

- [ ] All Module 1 acceptance tests pass (see PRD 1)
- [ ] All Module 5 email validation acceptance tests pass
- [ ] Service accessible at `https://patients.phlastic.com.au/api/v1`
- [ ] VPS confirmed in AU region
- [ ] Database encrypted at rest
- [ ] All API traffic over HTTPS
- [ ] CORS correctly restricts to Sellrise + Cabinet domains only

---

## Sprint 2 — Webhook Pipeline, CRM Integration & Photo Chatbot Step

**Duration:** Days 6–10  
**Theme:** Sellrise ↔ Phlastic integration — webhook, CRM customization, chatbot photo upload  
**Deliverable:** End-to-end data flow from chatbot → Phlastic DB, staff can view unified patient data in CRM

---

### Sprint 2 Backlog

#### TASK-2.1 — Webhook Pipeline (Sellrise codebase)
**PRD:** PRD 2  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] Add `patient_service` workspace settings (enabled, base_url, api_key, profile_mapping)
- [ ] Add `external_patient_id` column to Sellrise `leads` table
- [ ] Implement `lead_submitted` event hook → webhook logic
- [ ] Build payload from lead data + chatbot answers (using profile_mapping)
- [ ] `POST` to Phlastic `{base_url}/patients`
- [ ] On success: save returned `patient_id` as `external_patient_id` on lead
- [ ] On failure: exponential backoff retry (1s → 5s → 30s, max 3 retries)
- [ ] Webhook call log stored and visible in Sellrise admin
- [ ] Workspace without `patient_service` config → no webhook fired (backward compatible)

---

#### TASK-2.2 — Photo Upload Chatbot Step (Sellrise codebase)
**PRD:** PRD 3  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] Add `photo_upload` step type to scenario builder
- [ ] Widget UI: file picker + photo type dropdown + consent checkbox + progress bar
- [ ] Upload directly from browser to Phlastic API (not via Sellrise backend)
- [ ] Validation: 5 files max, 10 MB/file, JPEG/PNG/HEIC only
- [ ] Consent checkbox must be checked before upload is active
- [ ] "Skip" option when `required: false`
- [ ] Scenario builder warning if `photo_upload` placed before lead submission step
- [ ] Fallback: queue photos in memory if patient not yet created, upload after webhook

---

#### TASK-2.3 — CRM Unified View (Sellrise codebase)
**PRD:** PRD 4  
**Estimate:** 2 days  
**Owner:** Iga

**Lead Detail Page (7 blocks):**
- [ ] Contact Info block (Phlastic API data + email validation badge)
- [ ] Medical Interest block (procedures as chips, qualification score visual bar)
- [ ] Photos block (gallery grid via Phlastic API, full-size modal on click)
- [ ] Chat Transcript block (existing — no changes)
- [ ] Timeline block (merged events from Sellrise + Phlastic, sorted desc)
- [ ] Notes & Tags block (write note → `POST /patients/:id/notes`, tags → `PUT /patients/:id`)
- [ ] Stage & Owner block (stage change syncs to both systems; rollback on failure)

**Pipeline Views:**
- [ ] Inbox view (stage=new, sorted newest first)
- [ ] Kanban view (drag & drop stages, syncs to both systems)
- [ ] List view (all columns sortable, filters panel, pagination)

**Bulk Actions:**
- [ ] Change stage for selected leads
- [ ] Assign owner for selected leads
- [ ] Export CSV for selected leads

**Other:**
- [ ] "Send Cabinet Invite" button (visible when: has external_patient_id + account_created=false + email_status != invalid)
- [ ] 30-second response cache for Phlastic API calls
- [ ] Graceful degradation when Phlastic API is unavailable

---

### Sprint 2 Definition of Done

- [ ] Chatbot lead submission auto-creates patient in Phlastic DB
- [ ] `external_patient_id` correctly populated on Sellrise lead after webhook
- [ ] Photo upload from chatbot goes directly to Phlastic (verified via Network tab)
- [ ] CRM shows unified patient view (Sellrise + Phlastic data)
- [ ] Stage changes sync between both systems
- [ ] All Module 2 acceptance tests pass
- [ ] All Module 3 acceptance tests pass
- [ ] All Module 4 acceptance tests pass

---

## Sprint 3 — Patient Cabinet: Auth & Onboarding

**Duration:** Days 11–15  
**Theme:** Patient-facing authentication and first-run experience  
**Deliverable:** Patients can receive invite, create account, complete onboarding wizard

### Pre-Sprint Setup

- [ ] Domain `cabinet.phlastic.com.au` configured and SSL active
- [ ] Frontend project scaffolded (React / Next.js)
- [ ] Email service configured (Resend / SendGrid / SES) and tested

---

### Sprint 3 Backlog

#### TASK-3.1 — Cabinet Auth Backend Endpoints
**PRD:** PRD 6 (Module 7 — Auth section)  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] `POST /auth/invite` — generate invite_token (UUID, 7-day expiry), send invite email
- [ ] `GET /auth/validate-token` — validate invite or reset token
- [ ] `POST /auth/register` — hash password (bcrypt), set `account_created=true`, invalidate token, return JWT pair
- [ ] `POST /auth/login` — verify credentials, return JWT; redirect logic based on `onboarding_completed`
- [ ] `POST /auth/refresh` — refresh access token via httpOnly cookie
- [ ] `POST /auth/logout` — invalidate refresh token
- [ ] `POST /auth/forgot-password` — generate reset token (1-hour expiry), send reset email
- [ ] `POST /auth/reset-password` — validate token, update password hash
- [ ] Rate limiting: max 5 login attempts per email per 15 minutes
- [ ] JWT: access token 15 min, refresh token 30 days (httpOnly cookie)

---

#### TASK-3.2 — Cabinet Auth Frontend
**PRD:** PRD 6 (Module 7 — UI sections)  
**Estimate:** 1 day  
**Owner:** Iga

- [ ] `/register?token=xxx` — Create Account page (split-screen, Figma layout)
- [ ] `/login` — Sign In page (split-screen, Figma layout)
- [ ] `/forgot-password` — Forgot Password page
- [ ] `/reset-password?token=xxx` — Reset Password page
- [ ] Password validation: min 8 chars, 1 letter + 1 number (client + server)
- [ ] Auto-login after registration → redirect to onboarding
- [ ] JWT auto-refresh on 401 responses
- [ ] All auth pages mobile responsive

---

#### TASK-3.3 — Onboarding Wizard Backend
**PRD:** PRD 6 (Module 8 section)  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] `GET /cabinet/onboarding` — get current onboarding state + step
- [ ] `POST /cabinet/onboarding/:step` — save step data (step-by-step persistence)
- [ ] `POST /cabinet/onboarding/complete` — set `onboarding_completed=true`, `onboarding_step=7`, create event, award 50 points

---

#### TASK-3.4 — Onboarding Wizard Frontend
**PRD:** PRD 6 (Module 8 section)  
**Estimate:** 2 days  
**Owner:** Iga

- [ ] Modal overlay (1144 px desktop / full-screen mobile)
- [ ] Left sidebar: step number progress indicator (✅ completed / 🔵 current / ⚫ future)
- [ ] Step 1: Welcome screen
- [ ] Step 2: Personal details (name, DOB, gender, phone)
- [ ] Step 3: Location & contact preference
- [ ] Step 4: Medical interest (procedure checkboxes, budget, timeline, medical notes)
- [ ] Step 5: Photo upload (drag & drop, consent checkbox, Skip option)
- [ ] Step 6: Preferences (how heard, marketing consent, appointment times)
- [ ] Step 7: Confirmation + "Go to Dashboard"
- [ ] Pre-fill chatbot data where fields overlap
- [ ] Close (X) button saves progress → resume on next login
- [ ] Marketing consent → creates `consent_record` (type: `marketing`)
- [ ] Mobile responsive (full-screen on mobile)

---

### Sprint 3 Definition of Done

- [ ] All Module 7 acceptance tests pass
- [ ] All Module 8 acceptance tests pass
- [ ] Invite flow end-to-end working (staff sends invite → patient receives email → creates account → onboarding starts)
- [ ] Onboarding wizard saves progress at each step
- [ ] 50 reward points awarded on completion

---

## Sprint 4 — Patient Cabinet: Dashboard, Services & Journey

**Duration:** Days 16–20  
**Theme:** Core patient cabinet features — dashboard, services, journey timeline  
**Deliverable:** Patients can view their full status, services, and journey after login

---

### Sprint 4 Backlog

#### TASK-4.1 — Cabinet Profile Backend
**PRD:** PRD 7 (Module 9 section)  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] `GET /cabinet/me` — current patient profile (all fields)
- [ ] `PUT /cabinet/me` — update own profile
- [ ] `POST /cabinet/me/avatar` — upload avatar photo
- [ ] `GET /cabinet/services` — list patient's services
- [ ] `GET /cabinet/services/:id` — service detail
- [ ] `GET /cabinet/journey` — patient journey timeline entries

---

#### TASK-4.2 — Dashboard Frontend
**PRD:** PRD 7 (Module 9 section)  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] Layout: sidebar (300 px, #F0F6FF) + main content area (1025 px)
- [ ] Header: logo + patient avatar/name dropdown + chatbot icon
- [ ] Sidebar: 7 navigation items (Dashboard, Personal Info, Services, Journey, Rewards, Reviews, Community)
- [ ] Section 1: Status overview (welcome message, pipeline stage bar, points balance, next appointment)
- [ ] Section 2: Recent activity (last 5 `patient_events`, relative timestamps, "View all" → journey)
- [ ] Section 3: Quick actions (Upload Photos, Chat with Us, Leave a Review)
- [ ] Profile dropdown: Profile / Settings / Logout
- [ ] Chatbot icon opens Sellrise widget overlay
- [ ] Mobile: sidebar collapses to hamburger / bottom tab bar
- [ ] Matches Figma layout

---

#### TASK-4.3 — Services & Journey Frontend
**PRD:** PRD 7 (Module 10 section)  
**Estimate:** 1.5 days  
**Owner:** Iga

**Services page (`/services`):**
- [ ] List of patient's services with status, date, doctor name, price
- [ ] Status color coding (planned=gray, scheduled=blue, in_progress=orange, completed=green, cancelled=red)
- [ ] Empty state: "No services yet"

**Service detail (`/services/:id`):**
- [ ] Full service info (name, description, dates, price, notes)
- [ ] Doctor card (photo, name, specialization)
- [ ] Related photos
- [ ] Related journey entries

**Journey page (`/journey`):**
- [ ] Vertical timeline with year grouping headers
- [ ] Dot color coding (completed=green, upcoming=blue, planned=gray, cancelled=red)
- [ ] Each entry: date, title, description, status badge, staff name
- [ ] Empty state: "Your journey hasn't started yet"
- [ ] Both pages mobile responsive

---

#### TASK-4.4 — Personal Profile Page (`/profile`)
**PRD:** PRD 7  
**Estimate:** 0.5 day  
**Owner:** Iga

- [ ] Display all patient info (name, DOB, gender, phone, location, procedure interest, budget, timeframe)
- [ ] Edit mode: update fields via `PUT /cabinet/me`
- [ ] Avatar upload via `POST /cabinet/me/avatar`
- [ ] Display current `onboarding_step` and completion status

---

### Sprint 4 Definition of Done

- [ ] All Module 9 acceptance tests pass
- [ ] All Module 10 acceptance tests pass
- [ ] Dashboard loads with correct patient data
- [ ] Services and Journey pages display data from API
- [ ] Mobile responsive across all pages

---

## Sprint 5 — Staff Profiles, Reviews, Rewards & Community

**Duration:** Days 21–27  
**Theme:** Engagement features — staff profiles, review system, reward history, community  
**Deliverable:** Full patient cabinet feature set complete; integration testing + polish

---

### Sprint 5 Backlog

#### TASK-5.1 — Staff, Reviews, Rewards Backend
**PRD:** PRD 8 + PRD 9 (Rewards section)  
**Estimate:** 1.5 days  
**Owner:** Iga

- [ ] `GET /cabinet/staff` — list active staff
- [ ] `GET /cabinet/staff/:id` — staff profile with approved public reviews
- [ ] `GET /cabinet/reviews` — patient's own reviews
- [ ] `POST /cabinet/reviews` — submit review (status=pending, award 25 points)
- [ ] `PUT /cabinet/reviews/:id` — edit own pending review
- [ ] `GET /cabinet/rewards` — current balance
- [ ] `GET /cabinet/rewards/history` — paginated transactions
- [ ] Auto-award points: review (+25), service completed (+200)
- [ ] Staff `rating` auto-calculation on review approval

---

#### TASK-5.2 — Staff Profiles & Reviews Frontend
**PRD:** PRD 8  
**Estimate:** 1.5 days  
**Owner:** Iga

**Staff profile (`/staff/:id`):**
- [ ] Photo, name, role, bio, star rating, review count
- [ ] List of approved + public reviews with author "First L." display

**My Reviews (`/reviews`):**
- [ ] List of patient's own reviews with status badges
- [ ] Write review form (staff dropdown, service dropdown, star rating, text, public checkbox)
- [ ] Edit pending reviews

**Rewards (`/rewards`):**
- [ ] Balance card ("🏆 Your Balance — 150 points")
- [ ] "How to earn points" section
- [ ] Transaction history (paginated, newest first)

---

#### TASK-5.3 — Community Backend
**PRD:** PRD 9 (Community section)  
**Estimate:** 1 day  
**Owner:** Iga

- [ ] `GET /cabinet/community/groups` — list groups
- [ ] `GET /cabinet/community/groups/:id` — group detail + paginated posts
- [ ] `POST /cabinet/community/groups/:id/join` — join group
- [ ] `POST /cabinet/community/groups/:id/leave` — leave group
- [ ] `GET /cabinet/community/posts` — feed from joined groups
- [ ] `POST /cabinet/community/posts` — create post (text + optional image)
- [ ] `GET /cabinet/community/users/:id` — public profile (limited fields)
- [ ] `POST /cabinet/community/posts/:id/like` — toggle like
- [ ] `POST /cabinet/community/posts/:id/report` — report post (status → reported)
- [ ] Moderation queue accessible for admin/staff

---

#### TASK-5.4 — Community Frontend
**PRD:** PRD 9 (Community section)  
**Estimate:** 1.5 days  
**Owner:** Iga

**Groups list (`/community`):**
- [ ] Grid of groups (cover image, name, member count, Join/Joined button)
- [ ] Create group modal (2-step)

**Group detail (`/community/groups/:id`):**
- [ ] Cover image + member count + Leave button
- [ ] Post composer (text + image attach)
- [ ] Posts feed (newest first, paginated)
- [ ] Like button (toggle), Report button per post
- [ ] Author shown as "First L." format

**Public user profile (`/community/users/:id`):**
- [ ] First name + last initial, member since, groups, post count
- [ ] No PII or medical data

---

#### TASK-5.5 — Integration Testing & Polish
**PRD:** All modules  
**Estimate:** 2 days  
**Owner:** Iga

- [ ] Full end-to-end flow test: chatbot → webhook → Phlastic → CRM → invite → cabinet → onboarding → dashboard
- [ ] Cross-browser testing (Chrome, Safari, Firefox, mobile browsers)
- [ ] Load testing: patient list endpoint with large dataset
- [ ] Security review: API key rotation, JWT expiry, consent gate
- [ ] Fix identified bugs from integration testing
- [ ] Polish: UI/UX consistency pass, error states, empty states
- [ ] Final acceptance test run for all modules

---

### Sprint 5 Definition of Done

- [ ] All Module 11 acceptance tests pass
- [ ] All Module 12 acceptance tests pass
- [ ] All reward point automations working correctly
- [ ] Community moderation queue functional for admin
- [ ] All modules integrated and tested end-to-end
- [ ] All acceptance tests from PRD 1–9 verified and passing

---

## Contract Automation (Module 6) — Blocked Sprint

> **Status: BLOCKED** pending contract template from Daria/Avi.

Once template is received, add to next available sprint:

**Estimate:** 2–3 days after template received

### TASK-6.1 — Contract Automation Backend (after template received)

- [ ] Receive and review contract template from Daria
- [ ] `POST /patients/:id/contracts` — generate PDF with patient data
- [ ] `GET /patients/:id/contracts/:cid` — download PDF
- [ ] `PATCH /patients/:id/contracts/:cid` — update status (sent / signed)
- [ ] `contracts` table created and migrated
- [ ] Email PDF to patient

### TASK-6.2 — Contract CRM UI

- [ ] "Generate Contract" button (visible when `stage >= doctor_reviewed`)
- [ ] Download link + "Send to Patient" button
- [ ] "Mark as Signed" button → `stage → contract_signed`
- [ ] Contract status in timeline

---

## Summary: Sprint Plan at a Glance

| Sprint | Theme | Modules | Days | Key Deliverable |
|--------|-------|---------|------|-----------------|
| **Sprint 1** | Backend Foundation | 1, 5 | 1–5 | Patient Service API + DB on AU VPS |
| **Sprint 2** | Integration | 2, 3, 4 | 6–10 | Webhook pipeline + CRM + chatbot photos |
| **Sprint 3** | Cabinet Auth & Onboarding | 7, 8 | 11–15 | Invite flow + account creation + onboarding wizard |
| **Sprint 4** | Cabinet Dashboard & Journey | 9, 10 | 16–20 | Dashboard + services + journey timeline |
| **Sprint 5** | Engagement + Testing | 11, 12 | 21–27 | Reviews + rewards + community + integration tests |
| **Blocked** | Contracts | 6 | TBD | PDF generation (blocked on template) |

**Total estimate: 27 days core + 2–3 days buffer + 2–3 days contracts = 31–33 working days**

---

## Pre-Sprint Requirements by Owner

### Ivan (Sellrise / Client) — Must provide before Sprint 1:

- [ ] VPS access credentials OR DigitalOcean API token (to provision VPS)
- [ ] Domain DNS: point `patients.phlastic.com.au` and `cabinet.phlastic.com.au` to new VPS IP
- [ ] Email service credentials (Resend / SendGrid / AWS SES)
- [ ] Confirm AU region requirement (Sydney preferred)

### Daria / Avi (Phlastic) — Must provide before Module 6:

- [ ] Contract template (Word / PDF with placeholder fields)
- [ ] Confirmation of data fields to insert into contract
- [ ] Signature format decision (e-sign vs. PDF + wet signature)
- [ ] Pipeline stage that triggers contract generation
- [ ] Who initiates contract (doctor / admin / automatic)

### Daria / Avi (Phlastic) — Must provide before Sprint 5 (Community):

- [ ] List of community groups to pre-create at launch
- [ ] Moderation rules and community guidelines
- [ ] Branding assets: logo, color palette, cover images for groups

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| VPS access delayed (Ivan) | Medium | High — blocks Sprint 1 | Request access at contract signing |
| Contract template not received | High | Medium — blocks Module 6 only | Module 6 is non-blocking for other modules; implement late |
| Figma designs not provided | Medium | Medium | Build with material UI defaults; update after designs received |
| Phlastic API performance issues in CRM | Low | Medium | 30-second caching already in spec; add circuit breaker if needed |
| DNS propagation delays | Low | Low | Pre-configure DNS before Sprint 1 starts |
| AU email service deliverability issues | Low | Medium | Test email delivery in Sprint 3; have backup service ready |

---

## Communication Plan

| Channel | Frequency | Content |
|---------|-----------|---------|
| Telegram | Daily (weekdays) | Progress update: done today / planned tomorrow / blockers |
| Telegram | Immediate | Blockers, decisions needed, questions |
| Git (Phlastic repo) | Per task completion | Phlastic Patient Service backend + Cabinet frontend code |
| Git (Sellrise repo) | Per task completion | Sellrise CRM changes + webhook + chatbot step |
| Sprint demos | End of each sprint | Demo deliverables, Ivan reviews acceptance tests |

---

## Payment Milestones

| Milestone | Amount | Trigger |
|-----------|--------|---------|
| Sprint 1 complete (Module 1 accepted) | — | Included in $500 upfront |
| All modules accepted by Ivan | **$500** | Final payment on full acceptance |

**Acceptance process:** Iga declares module delivery → Ivan tests within 48 hours using acceptance tests in each PRD → Issues reported → Iga fixes → Final sign-off.
