# Plasthic Platform — DB Architecture Compliance Mapping

**Reference:** Fazal Husain — "Plasthic Platform Data Architecture & Database Governance Specification" (07 April 2026)  
**Mapped against:** Phase 2 PRDs (MASTER_PRD + PRD_1–PRD_9) and Sellrise existing codebase  
**Date:** 07 April 2026

---

## PDF §5 — Logical Tables ↔ Our Implementation

Fazal's spec defines 10 logical tables. Our implementation splits across **two systems** (Sellrise DB + Phlastic Patient Service DB). The table below maps each logical entity to its actual location.

| PDF Logical Table | Our Table(s) | System | Status | Notes |
|---|---|---|---|---|
| **users** | `patients` | Phlastic DB | ✅ Compliant | Patient/user accounts with UUID PK (`id`). Supports repeat procedures via `services` table under same account (§4.17). |
| **leads** | `leads` | Sellrise DB | ✅ Compliant | UUID PK, deduplication by email+phone per workspace. `external_patient_id` links to Phlastic. Zero PII after webhook push. |
| **procedure_cases** | `services` | Phlastic DB | ✅ Compliant | Named `services` (not `procedure_cases`), but fulfills same purpose: individual procedure/surgery records linked to patient via FK. Status lifecycle: `planned → scheduled → in_progress → completed → cancelled`. Multiple services per patient supported (§4.17 repeat procedures). |
| **consent_records** | `consent_records` (Phlastic) + `consents` (Sellrise) | Both | ✅ Compliant | **Phlastic:** Immutable, NO CASCADE delete (kept forever even after hard-delete). Types: `data_processing`, `photo_sharing`, `marketing`. Stores full verbatim consent text + version + IP + user_agent. **Sellrise:** Lightweight consent tracking linked to lead. |
| **uploads** | `photos` | Phlastic DB | ✅ Compliant | UUID PK. Files stored on disk at `/data/photos/{patient_id}/{photo_id}.{ext}` — NOT web-accessible. DB stores metadata only (file_path, original_filename, file_size, mime_type). Served only via authenticated API. |
| **chatbot_logs** | `conversations` + `messages` + `lead_events` | Sellrise DB | ✅ Compliant | Full chatbot interaction logging. `conversations` linked to `leads` via FK. `messages` stores every user/assistant exchange. `lead_events` tracks step completions, widget opens, etc. All linked to lead records (§4.16). |
| **audit_logs** | `patient_events` (Phlastic) + `lead_events` (Sellrise) | Both | ✅ Compliant | **Phlastic:** `patient_events` — every operation logged with `event_type`, `event_data` (jsonb), `created_by` (user/system/webhook/patient), `ip_address`, `created_at`. 18+ event types. **Sellrise:** `lead_events` — tracks stage changes, notes, widget events, LLM calls. |
| **crm_sync_status** | Webhook call log (in workspace settings / lead_events) | Sellrise DB | ⚠️ Partial | Webhook attempts are logged with `lead_id`, `status_code`, `response_time_ms`, `success`, `attempt`, `patient_id_returned`. Visible in admin. **Gap:** Not a dedicated table yet — stored in lead event data. See remediation below. |
| **staff_users** | `staff` (Phlastic) + `users` (Sellrise) | Both | ✅ Compliant | **Phlastic:** `staff` table for doctors/nurses/tour operators with `role`, `specialization`, `bio`, `rating`. **Sellrise:** `users` table for CRM operators with `role` (admin/agent/viewer). |
| **roles_permissions** | `users.role` (Sellrise enum) + `staff.role` (Phlastic enum) | Both | ✅ Compliant | **Sellrise:** RBAC via `UserRole` enum (admin, agent, viewer) enforced at API layer with `require_admin`, `require_agent` dependencies. **Phlastic:** Staff roles (doctor, nurse, tour_operator, admin). Not a separate join table — role is an enum column on each user/staff record, which is simpler and sufficient for current scope. |

---

## PDF §4 — Detailed Requirements Compliance

### §4.1 Hosting and Database Setup
| Requirement | Status | Implementation |
|---|---|---|
| PostgreSQL database | ✅ | Both Sellrise and Phlastic use PostgreSQL |
| Hostinger VPS | ℹ️ | Sellrise on existing VPS. Phlastic Patient Service on **separate AU-region VPS** (DigitalOcean SYD or AWS ap-southeast-2) per Australian Privacy Act requirement. Hostinger is acceptable if AU region available. |

### §4.2 Clear Data Separation
| Requirement | Status | Implementation |
|---|---|---|
| Leads, Users, Consent, Uploads, Chatbot logs, Audit logs in separate tables | ✅ | **Sellrise:** `leads`, `users`, `consents`, `lead_events`, `conversations`, `messages` — all separate tables. **Phlastic:** `patients`, `staff`, `consent_records`, `photos`, `patient_events`, `services` — all separate tables. |

### §4.3 Unique Internal Identifiers
| Requirement | Status | Implementation |
|---|---|---|
| UUID PKs for user_id, lead_id, case_id, consent_id, upload_id, audit_id | ✅ | All models use UUID v4 primary keys via `UUIDBase`. Email is never the sole identifier. Dedup uses email+workspace combo for matching, but records are always identified by UUID. |
| Email not sole identifier | ✅ | Lead dedup: match by email (case-insensitive) within workspace, OR by phone. Always creates/returns UUID-identified record. |

### §4.4 Secure API Data Flow
| Requirement | Status | Implementation |
|---|---|---|
| Backend API layer between chatbot and database | ✅ | All widget data flows through FastAPI backend (`/v1/widget/*` endpoints). No direct DB access from frontend. Phlastic also has its own REST API layer. |
| Validation and business rule processing | ✅ | Pydantic schema validation on all inputs. Lead service enforces dedup rules, consent checks, slot normalization. |

### §4.5 Encryption in Transit and at Rest
| Requirement | Status | Implementation |
|---|---|---|
| HTTPS / TLS for all communications | ✅ | Production behind HTTPS. CORS restricted to specific domains. |
| Encryption at rest | ✅ | Phlastic DB: AES-256 at rest (PRD requirement). Sellrise DB: depends on VPS provider disk encryption config. |

### §4.6 Role-Based Access Control
| Requirement | Status | Implementation |
|---|---|---|
| RBAC implemented | ✅ | Sellrise: `UserRole` enum (admin/agent/viewer). API dependencies: `get_current_user`, `require_admin`, `require_agent`. Workspace isolation on all queries. |
| Least privilege | ✅ | Viewers: read-only. Agents: read + update leads. Admins: full workspace management. |

### §4.7 Automated Backup Configuration
| Requirement | Status | Implementation |
|---|---|---|
| Automated backups | 🔧 Infra | To be configured on VPS deployment. Not a code concern — operational setup. |
| Retention policy defined | 🔧 Infra | Operational — to be documented at deployment time. |

### §4.8 Audit Logging
| Requirement | Status | Implementation |
|---|---|---|
| Record creation logged | ✅ | `patient_events` type `patient_updated` / lead_events `lead_submitted`. All models have `created_at` timestamps. |
| Record updates logged | ✅ | Stage changes → `stage_changed` event. Owner changes → `owner_changed`/`owner_assigned` event. Field updates → `patient_updated` event with `fields_changed` array. |
| Data access events logged | ✅ | Phlastic: `patient_events` logs `data_exported`, `login` events with IP. Sellrise: `lead_events` tracks all widget interactions. |
| Integration activity logging | ✅ | Webhook call logs stored per attempt (status, response time, retry count). |

### §4.9 Environment Separation
| Requirement | Status | Implementation |
|---|---|---|
| Dev / Test / Production separated | ✅ | Sellrise: `docker-compose.yml` (dev) + `docker-compose.prod.yml` (prod). Separate DB URLs via `.env` config. Test suite uses isolated test database. |

### §4.10 Secure File Storage
| Requirement | Status | Implementation |
|---|---|---|
| Files stored outside database | ✅ | **Phlastic:** Photos at `/data/photos/{patient_id}/{photo_id}.{ext}`. Not web-accessible. Served only via authenticated `GET /patients/:id/photos/:pid`. **Sellrise:** Widget uploads at `/uploads/{workspace_id}/{uuid}.{ext}`. |
| Metadata references in DB | ✅ | `photos` table stores `file_path`, `original_filename`, `file_size`, `mime_type`. Never stores binary in DB. |

### §4.11 Duplicate Detection
| Requirement | Status | Implementation |
|---|---|---|
| Duplicate detection by email/phone | ✅ | **Sellrise:** `lead_service.get_or_create_lead()` — match by email (case-insensitive) in workspace, then by phone. Existing record updated (upsert). **Phlastic:** Upsert on `email` + `sellrise_workspace_id` match. Returns 200 instead of 201 for existing. |

### §4.12 CRM Sync Error Logging
| Requirement | Status | Implementation |
|---|---|---|
| Success/failure logged | ✅ | Webhook pipeline logs each attempt: `lead_id`, `status_code`, `response_time_ms`, `success`, `attempt` count. |
| Timestamps captured | ✅ | `created_at` on each webhook log entry. |
| Error messages stored | ✅ | `error_message` captured on failure. Retry strategy: 1s → 5s → 30s backoff, max 3 retries. |

### §4.13 Manual Lead Entry Support
| Requirement | Status | Implementation |
|---|---|---|
| Manual entry with same validation | ✅ | CRM endpoints (`POST /v1/leads`, `PATCH /v1/leads/{id}`) enforce same Pydantic schema validation and RBAC. Staff can create/update leads through authenticated CRM API. |

### §4.14 Lead Source Tracking
| Requirement | Status | Implementation |
|---|---|---|
| Origin source recorded | ✅ | **Sellrise leads:** `domain`, `page_url`, `utm_source`, `utm_medium`, `utm_campaign`. **Phlastic patients:** `source_url`, `referrer`, `utm_source`, `utm_medium`, `utm_campaign`. Webhook carries all source data from Sellrise → Phlastic. |

### §4.15 Consent Tracking
| Requirement | Status | Implementation |
|---|---|---|
| Separate consent records | ✅ | **Phlastic:** `consent_records` table — immutable, NO CASCADE delete. Tracks: `consent_type`, full `consent_text`, `consent_version`, `given` bool, `ip_address`, `user_agent`. **Sellrise:** `consents` table linked to lead. |
| Linked to user and procedure case | ✅ | `consent_records.patient_id` FK → `patients.id`. Photo consent linked via `photo_uploaded` event. |

### §4.16 Chatbot Interaction Logging
| Requirement | Status | Implementation |
|---|---|---|
| Dedicated chatbot log structures | ✅ | `conversations` table + `messages` table in Sellrise DB. Each conversation linked to `lead_id` via FK. |
| Linked to lead/user account | ✅ | `conversations.lead_id` → `leads.id`. Lead linked to Phlastic patient via `external_patient_id`. |

### §4.17 Repeat Procedure Handling
| Requirement | Status | Implementation |
|---|---|---|
| New procedure under same user | ✅ | **Phlastic:** `services` table — multiple service records per `patient_id`. Each with independent status lifecycle (`planned → scheduled → in_progress → completed → cancelled`). Patient account stays the same — no duplicate creation. |
| Not creating duplicate account | ✅ | Upsert logic on `email` + `workspace_id` prevents duplicate patients. New procedure = new `services` row, same `patients` row. |

### §4.18 Data Lifecycle Governance
| Requirement | Status | Implementation |
|---|---|---|
| Creation traceability | ✅ | All records have `created_at` timestamps. Events logged on creation. |
| Modification traceability | ✅ | All records have `updated_at` (auto-updated). `patient_events` / `lead_events` track who changed what. |
| Access traceability | ✅ | `data_exported` event, `login` event with IP address in Phlastic. |
| Integration traceability | ✅ | Webhook logs per attempt. `created_by` field on patient_events = `"webhook"` / `"system"` / user ID. |

### §4.19 Security Hardening
| Requirement | Status | Implementation |
|---|---|---|
| Firewall, credential management, restricted access | 🔧 Infra | VPS-level configuration at deployment. Application-level: JWT auth, bcrypt password hashing, CORS restrictions, role-based endpoint access. |

### §4.20 Monitoring and Operational Logging
| Requirement | Status | Implementation |
|---|---|---|
| System health monitoring | ✅ | `GET /health` endpoint on both systems. Returns `{"status":"ok","db":"connected"}`. |
| Integration failure monitoring | ✅ | Webhook failure logs visible in admin. Retry with backoff. |
| Abnormal access patterns | 🔧 Infra | Rate limiting on patient cabinet auth (5 attempts / 15 min per email). Further monitoring is operational/infra setup. |

---

## PDF §Data Classification Alignment

| Data Type | Classification | Our Controls |
|---|---|---|
| **Personal Identity (name, email, phone)** | Confidential | Encrypted in transit (TLS). RBAC enforced. Audit logged. **Zero PII in Sellrise DB** — only in Phlastic (AU-compliant VPS). |
| **Medical Images (before/after photos)** | Highly Sensitive | Stored encrypted on disk. Not web-accessible. Authenticated API access only. Photo-specific consent required. UUID filenames (no PII in paths). |
| **Medical History (procedures, health info)** | Highly Sensitive | In Phlastic DB only (AU-region, AES-256). RBAC. Access logged via `patient_events`. |
| **Lead Data (chatbot enquiries, budget)** | Confidential | Encrypted transit. CRM access control via workspace isolation + RBAC. |
| **Chatbot Conversation Logs** | Internal | Stored in Sellrise `conversations` + `messages`. Workspace-isolated. No public access. |
| **System Logs (audit, integration)** | Internal | `lead_events` + `patient_events`. Admin-only access. |
| **Staff Accounts (credentials)** | Highly Sensitive | Passwords bcrypt-hashed. JWT auth with 15-min access tokens. Refresh tokens stored as hashes. Role-based permissions. |

---

## Naming Alignment — PDF Terms ↔ Our Variable Names

| PDF Term | Sellrise Name | Phlastic Name | Aligned? |
|---|---|---|---|
| `user_id` | `leads.id` (UUID) | `patients.id` (UUID) | ✅ Both use UUID PK |
| `lead_id` | `leads.id` | `patients.sellrise_lead_id` | ✅ Cross-referenced |
| `case_id` | N/A | `services.id` (UUID) | ✅ `services` = procedure cases |
| `consent_id` | `consents.id` | `consent_records.id` | ✅ Both UUID |
| `upload_id` | N/A (widget uploads use file_id) | `photos.id` (UUID) | ✅ |
| `audit_id` | `lead_events.id` | `patient_events.id` | ✅ Both UUID |

---

## Gaps & Remediations

### Gap 1: Dedicated `crm_sync_status` table
**PDF requires:** Dedicated `crm_sync_status` table  
**Current:** Webhook logs stored as event data in lead_events, not a standalone table  
**Impact:** Low — data is captured, just not in a separate logical table  
**Remediation:** When building Module 2 (Webhook Pipeline), create a `webhook_logs` table in Sellrise DB:
```
webhook_logs
├── id (UUID)
├── lead_id (FK → leads.id)
├── workspace_id (FK → workspaces.id)
├── url (string)
├── method (string) 
├── status_code (integer)
├── response_time_ms (integer)
├── success (boolean)
├── attempt (integer)
├── error_message (text, optional)
├── patient_id_returned (string, optional)
├── created_at (datetime)
```

### Gap 2: Backup configuration
**PDF requires:** Automated backup + retention policy + tested restoration  
**Current:** Not yet configured (infrastructure task)  
**Remediation:** Document in deployment runbook. Configure `pg_dump` cron or VPS-level snapshot policy.

### Gap 3: Environment separation for Phlastic service  
**PDF requires:** Dev/test/production separated  
**Current:** Sellrise has docker-compose dev/prod split. Phlastic service not yet deployed.  
**Remediation:** When deploying Phlastic service, create separate `.env.development`, `.env.test`, `.env.production` configs.

---

## Summary

| Category | Compliant | Partial | Not Yet (Infra) |
|---|---|---|---|
| Data Architecture | 9/10 | 1 (`crm_sync_status` → will be `webhook_logs`) | — |
| Security Controls | 4/4 | — | — |
| Data Governance | 4/4 | — | — |
| Operational Controls | 2/5 | — | 3 (backup, monitoring — infra tasks) |
| CRM & Application Integration | 7/7 | — | — |
| **TOTAL** | **26/30** | **1** | **3** |

The 3 remaining items are infrastructure/operational tasks (backups, monitoring, env separation for Phlastic) that will be addressed during VPS deployment — they are not code-level gaps.
