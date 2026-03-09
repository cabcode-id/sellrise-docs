# MVP 1 Status Review

## Kesimpulan Utama

Status saat ini terlihat baru menutup **fondasi MVP 1**, tetapi **belum memenuhi MVP 1 end-to-end sesuai dokumen**.

### Yang sudah ada
- Login dasar
- Leads list / pipeline dasar
- Knowledge Base CRUD dasar
- Scenario CRUD dasar
- Analytics summary dasar
- Widget endpoints dasar

### Area besar yang masih belum tercapai
- Workspace, users, domain management
- Widget flow MVP end-to-end
- Scenario engine deterministic + versioned
- Mini-CRM lengkap
- Analytics funnel MVP
- Notification + settings
- Automated tests minimum MVP

---

# 1. Core Platform & Admin Belum Lengkap

Docs MVP Phase 1 meminta:
- workspace
- users
- domain registration
- settings admin
- isolation tenant

Referensi:
- FEATURE_BREAKDOWN.md:93-229
- AGILE_PRD.md:45-154

## Yang belum tercapai

### Workspace management API belum ada

Docs meminta endpoint berikut:

POST /v1/workspaces  
GET /v1/workspaces  
PATCH /v1/workspaces  
DELETE /v1/workspaces  

Router aktif di backend (main.py) saat ini hanya:
- auth
- widget
- leads
- scenarios
- kb
- analytics
- health

### Users / Team management API belum ada

Docs meminta endpoint berikut:

POST /v1/users  
GET /v1/users  
PATCH /v1/users  
DELETE /v1/users  

Tetapi **tidak ada router users di v1**.

### API Domains belum ada

Frontend sudah memanggil endpoint berikut pada domainService.js:

GET /v1/domains  
POST /v1/domains  
DELETE /v1/domains  

Tetapi backend **tidak memiliki router domains.py**.

### Workspace settings masih statis

Halaman Settings.jsx masih **hardcoded UI** dan belum terhubung ke backend.

### Dampak

Admin belum bisa:
- mengelola tenant/workspace
- mengelola tim
- mengelola domain allowed list
- mengelola workspace settings

---

# 2. Auth MVP Baru Sebagian

Sudah ada:
- login
- refresh
- logout
- me

Frontend juga sudah memiliki protected routes.

## Yang belum tercapai

### Signup masih dummy

SignUp.jsx hanya melakukan redirect ke dashboard, belum:
- create workspace
- create user

### Forgot Password masih dummy

ForgotPassword.jsx hanya menggunakan alert().

### Invite / create user belum ada

Backend belum memiliki endpoint users.

### Email unik masih global

Model user.py menggunakan email UNIQUE global, padahal docs meminta unik **per workspace**.

### Dampak

Flow onboarding berikut belum bisa dipakai:

admin → invite team → team login

---

# 3. Widget MVP Belum Memenuhi Kontrak Dokumen

Endpoint utama:

POST /v1/widget/session

Docs meminta request berisi:
- domain
- page_url
- referrer
- utm
- timezone

Response harus mengembalikan:
- session_id
- branding
- scenario_version_id
- published scenario
- booking links

Implementasi sekarang hanya mengembalikan:
- session_id
- workspace_id
- scenario
- branding

### Validasi domain belum sesuai

Docs meminta error:
403 domain_not_allowed

Implementasi sekarang return:
404

### Fallback widget belum ada

Docs meminta:
- fallback saat session gagal
- fallback lead endpoint

Backend belum memiliki endpoint fallback.

### Embed snippet belum realistis

UI pada Domains.jsx dan WidgetPreview.jsx masih mock / hardcoded.

Belum ada **widget bundle production** yang benar-benar bisa dipasang di website client.

### Dampak

Flow berikut belum siap production:

install widget → website load widget → create session → run scenario → lead masuk CRM

---

# 4. Event Logging & Transcript Belum Konsisten

Docs menyebut event minimum berikut:

- widget_opened
- chat_started
- step_completed
- lead_submitted
- booking_link_shown
- booking_link_clicked

Implementasi saat ini belum memiliki sebagian event tersebut.

Submit lead masih menggunakan event:
contact_submitted

Padahal docs meminta:
lead_submitted

Bug tambahan:

lead_event.py mewajibkan:
lead_id NOT NULL

Namun widget_event mencoba menyimpan event dengan:
lead_id = None

### Dampak

Transcript dan funnel event **belum reliable**.

---

# 5. Scenario Engine Belum Sesuai MVP

Sudah ada:
- CRUD scenario dasar
- UI scenario dasar

Yang belum ada:
- schema validation
- immutable scenario version
- deterministic engine runner
- scenario simulator
- visual bot builder

Scenario saat ini hanya disimpan sebagai **JSON blob**.

---

# 6. Mini-CRM Belum Lengkap

Inbox masih mock.

Lead detail belum lengkap:
- owner
- transcript
- notes
- tags
- actions assign owner/stage

Tags belum ada di model maupun API.

Search dan filter belum lengkap:
- owner
- date range
- procedure

Event logging untuk CRM actions juga belum lengkap.

Lead scoring belum memiliki event:
score_assigned

---

# 7. Knowledge Base Masih Dasar

Sudah ada:
- CRUD KB
- UI KnowledgeBase
- public search

Yang belum ada:
- language field
- updated_by
- metadata / tags
- ranking search lebih baik
- fallback KB → contact flow

---

# 8. Analytics MVP Belum Sesuai

Sudah ada:
- analytics summary endpoint
- dashboard dasar

Yang belum ada:
- funnel endpoint
- date range filter backend
- funnel metrics lengkap
- CSV export via API
- source analytics (referrer, utm, page_url)

---

# 9. Notifications & Settings

Sudah ada:
- stub email hot lead

Yang belum ada:
- configurable recipients
- notification_sent / notification_failed events
- CRM link valid
- queue / worker email
- workspace settings persistence

---

# 10. Testing Minimum MVP

Kondisi sekarang:

File test berikut masih kosong:
- test_widget.py
- test_leads.py
- test_scenarios.py

Hanya test_rbac.py yang relatif ada, tetapi masih memiliki beberapa `xfail`.

Tests yang diminta docs:
- scenario validation
- deterministic engine
- lead dedup
- KB search fallback
- widget smoke flow

---

# Prioritas Gap Paling Kritis

1. Backend Admin APIs
   - workspaces
   - users/team
   - domains
   - settings

2. Widget Flow End-to-End

3. Scenario Engine MVP

4. Mini-CRM Completion

5. Analytics Funnel

6. Minimum MVP Tests

---

# Penilaian Akhir

Secara objektif:

**MVP 1 belum selesai.**

Status saat ini lebih tepat disebut:

backend foundation + CRUD dasar

Belum:
- admin workflow MVP
- widget-to-CRM flow production ready
- acceptance criteria MVP terpenuhi

Masih ada **gap signifikan untuk menyebut sistem ini MVP 1 selesai**.