# Sellrise Integration Testing Guide — Backend ↔ Frontend

Panduan lengkap untuk mentest seluruh fungsionalitas Sellrise secara end-to-end. Semua test dilakukan dengan **backend** (`localhost:8000`) dan **frontend** (`localhost:3000`) berjalan bersamaan.

---

## Prasyarat

```powershell
# 1. Start PostgreSQL via Docker
cd d:\pukul6\sellrise\Sellrise-BackEnd
docker compose up -d db

# 2. Run migrations
.venv\Scripts\alembic upgrade head

# 3. Seed data (admin user + workspace + domain + dummy leads)
.venv\Scripts\python scripts\seed_all.py

# 4. Start backend
.venv\Scripts\uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 5. Start frontend (terminal terpisah)
cd d:\pukul6\sellrise\Sellrise-Front-End
npm run dev
```

> [!TIP]
> Seeder [seed_all.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/scripts/seed_all.py) akan membuat admin user, workspace, domain, leads, KB articles, FAQs, dan scenario dummy. Catat **email/password** admin yang di-print di terminal.

---

## Bagian A: Backend Unit/Integration Tests (Pytest)

Jalankan semua existing backend tests sekaligus:

```powershell
cd d:\pukul6\sellrise\Sellrise-BackEnd
.venv\Scripts\pytest tests\ -v
```

Existing test files:

| File | Cakupan |
|---|---|
| [test_rbac.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_rbac.py) | RBAC enforcement (admin/agent/viewer permissions) |
| [test_workspaces_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_workspaces_api.py) | Workspace CRUD + isolation |
| [test_users_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_users_api.py) | User management (invite/role/deactivate) |
| [test_domains_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_domains_api.py) | Domain CRUD + branding |
| [test_notification_settings.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_notification_settings.py) | Hot lead notification config |
| [test_widget.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_widget.py) | Widget session, lead submit, KB search |

> [!IMPORTANT]
> Test database terpisah di port **5432** (bukan 5433). Pastikan PostgreSQL Docker juga expose port 5432, atau buat database `sellrise_test_db` di instance yang sama.

---

## Bagian B: Manual Integration Test — Step by Step

### 🔐 Module 1: Authentication (US-2.1, 2.4)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `http://localhost:3000/login` | Halaman login muncul | |
| 2 | Login dengan email/password admin dari seeder | Redirect ke `/dashboard`, nama user muncul | |
| 3 | Refresh halaman | Tetap logged in (token refresh bekerja) | |
| 4 | Klik Logout | Redirect ke `/login`, token dihapus | |
| 5 | Akses `/dashboard` tanpa login | Redirect otomatis ke `/login` | |

**Verifikasi API:** Buka DevTools → Network tab, pastikan:
- `POST /v1/auth/login` → 200, response berisi `access_token`
- `GET /v1/auth/me` → 200, return user info
- `POST /v1/auth/logout` → 200

---

### 🏢 Module 2: Workspace & Settings (US-1.1, 9.1)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka halaman `/settings` | Workspace settings muncul | |
| 2 | Update workspace name | Name berubah, API call `PATCH /v1/workspaces/{id}` → 200 | |
| 3 | Cek notification settings (hot lead recipients) | Form muncul, bisa save | |

---

### 👥 Module 3: Team Management (US-2.2, 2.3, 9.2)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/settings` → tab Team | Daftar users (minimal admin) muncul | |
| 2 | Invite user baru (email, role: Agent) | `POST /v1/users` → 201, user muncul di list | |
| 3 | Invite user dengan email yang sama | Error 409 (duplicate) | |
| 4 | Ubah role user ke Viewer | `PATCH /v1/users/{id}` → 200, role berubah | |
| 5 | Login sebagai Viewer, coba akses fitur admin-only | Akses ditolak (403) | |

---

### 🌐 Module 4: Domain Management (US-1.3)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/widget-settings` | Halaman domain/widget settings muncul | |
| 2 | Tambah domain baru (e.g. `testdomain.com`) | `POST /v1/domains` → 201, domain muncul di list | |
| 3 | Edit branding (name, color, logo URL) | `PATCH /v1/domains/{id}` → 200, branding terupdate | |
| 4 | Delete domain | Domain hilang dari list | |

---

### 🤖 Module 5: Scenarios (US-4.1, 4.2, 4.3)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/scenarios` | List scenarios muncul (dari seed data) | |
| 2 | Create scenario baru | `POST /v1/scenarios` → 201, muncul di list | |
| 3 | Buka `/scenario-config` dan edit scenario | Config editor muncul, bisa edit nodes/steps | |
| 4 | Publish scenario | `POST /v1/scenarios/{id}/publish` → 200, status → published | |
| 5 | Coba delete published scenario | Error 400/409 (tidak bisa delete yang published) | |
| 6 | Test **Scenario Simulator** (jika ada di UI) | `POST /v1/scenarios/{id}/simulate` → 200, bot reply muncul | |

---

### 📋 Module 6: Lead Management (US-5.1 – 5.8)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/leads` | List leads muncul (dari seed data) | |
| 2 | Buka `/pipeline` | Kanban board muncul dengan leads di setiap stage | |
| 3 | Drag lead ke stage lain di kanban | `PATCH /v1/leads/{id}` → 200, stage berubah | |
| 4 | Klik lead → detail view | Lead detail muncul (info, events, notes) | |
| 5 | Tambah note di lead detail | `POST /v1/leads/{id}/notes` → 201, note muncul | |
| 6 | Search lead by name | Filter bekerja, hasil sesuai | |
| 7 | Filter by stage (e.g. `hot`) | Hanya leads dengan stage tersebut muncul | |
| 8 | Assign lead ke user | `PATCH /v1/leads/{id}` dengan `user_id`, status berubah | |

---

### 📚 Module 7: Knowledge Base (US-6.1 – 6.4)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/knowledge-base` | List articles muncul | |
| 2 | Create article baru | `POST /v1/kb/articles` → 201, article muncul | |
| 3 | Edit article | `PATCH /v1/kb/articles/{id}` → 200 | |
| 4 | Delete article | Article hilang dari list | |
| 5 | Search article (ketik keyword) | Hasil pencarian muncul sesuai keyword | |

---

### 📊 Module 8: Analytics (US-7.1, 7.2)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka `/analytics` | Dashboard analytics muncul | |
| 2 | Cek summary metrics | `GET /v1/analytics/summary` → 200, data muncul | |
| 3 | Cek funnel chart | `GET /v1/analytics/funnel` → 200, chart muncul | |
| 4 | Export CSV | File CSV ter-download | |

---

### 🔌 Module 9: Widget Flow (US-3.1 – 3.4, 4.4)

Test ini menggunakan file [test_widget.html](file:///d:/pukul6/sellrise/Sellrise-BackEnd/test_widget.html) yang sudah ada di backend:

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Buka [d:\pukul6\sellrise\Sellrise-BackEnd\test_widget.html](file:///d:/pukul6/sellrise/Sellrise-BackEnd/test_widget.html) di browser | Test page muncul | |
| 2 | Cek console: "Session established" | `POST /v1/widget/session` → 200, session_id muncul | |
| 3 | Badge "✅ Connected to Backend" muncul | Handshake berhasil | |

**Widget API test via curl/Postman:**

```powershell
# Session handshake
curl -X POST http://localhost:8000/v1/widget/session -H "Content-Type: application/json" -d "{\"domain\": \"sellrise.com\", \"page_url\": \"http://test.com\"}"

# Submit lead via widget
curl -X POST http://localhost:8000/v1/widget/lead -H "Content-Type: application/json" -d "{\"workspace_id\": \"YOUR_WS_ID\", \"name\": \"Test Lead\", \"email\": \"test@example.com\", \"phone\": \"08123456789\", \"consent_given\": true}"

# Search KB via widget
curl "http://localhost:8000/v1/widget/kb/search?q=pricing&workspace_id=YOUR_WS_ID"

# Log event
curl -X POST http://localhost:8000/v1/widget/event -H "Content-Type: application/json" -d "{\"workspace_id\": \"YOUR_WS_ID\", \"event_type\": \"step_completed\", \"step_name\": \"welcome\", \"data\": {}}"
```

---

### 📨 Module 10: Hot Lead Notification (US-8.1, 8.2)

| # | Langkah | Expected Result | ✅ |
|---|---|---|---|
| 1 | Pastikan SMTP config di [.env](file:///d:/pukul6/sellrise/Sellrise-BackEnd/.env) benar | `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD` terisi | |
| 2 | Set notification recipients di Settings | Recipients tersimpan | |
| 3 | Update lead score ke `hot` | `PATCH /v1/leads/{id}` dengan `score: "hot"` | |
| 4 | Cek email recipient | Email notifikasi diterima (cek juga spam folder) | |
| 5 | Cek lead events | Event `notification_sent` muncul di lead detail | |

---

### 📅 Module 11: Booking Token (US-10.1)

Test via curl/Postman (fitur ini mungkin belum ada di FE):

```powershell
# Create booking token
curl -X POST http://localhost:8000/v1/booking/tokens -H "Authorization: Bearer YOUR_TOKEN" -H "Content-Type: application/json" -d "{\"lead_id\": \"LEAD_ID\"}"

# Get token info
curl http://localhost:8000/v1/booking/tokens/TOKEN_VALUE -H "Authorization: Bearer YOUR_TOKEN"

# Consume token (single-use)
curl -X POST http://localhost:8000/v1/booking/tokens/TOKEN_VALUE/consume -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Bagian C: Postman Collection

Import collection yang sudah ada untuk test API secara terstruktur:

1. Buka Postman → Import → File
2. Pilih [d:\pukul6\sellrise\Sellrise-BackEnd\Sellrise_API_Collection.postman_collection.json](file:///d:/pukul6/sellrise/Sellrise-BackEnd/Sellrise_API_Collection.postman_collection.json)
3. Set variables:
   - `base_url` = `http://localhost:8000`
   - `workspace_id` = (dari seeder output)
4. Jalankan **Login** request terlebih dahulu (auto-set `access_token`)
5. Jalankan request lainnya secara sequential

---

## Checklist Ringkasan

| Module | Method | Status |
|---|---|---|
| Auth (Login/Logout/Refresh) | Manual FE + Pytest | ⬜ |
| Workspace CRUD | Manual FE + Pytest | ⬜ |
| Team Management (RBAC) | Manual FE + Pytest | ⬜ |
| Domain Management | Manual FE + Pytest | ⬜ |
| Scenarios (CRUD + Publish + Simulate) | Manual FE + Postman | ⬜ |
| Leads (List/Detail/Pipeline/Notes/Assign/Score) | Manual FE | ⬜ |
| Knowledge Base (Article/FAQ CRUD + Search) | Manual FE | ⬜ |
| Analytics (Summary/Funnel/Export) | Manual FE | ⬜ |
| Widget (Session/Lead/Event/KB Search) | curl + [test_widget.html](file:///d:/pukul6/sellrise/Sellrise-BackEnd/test_widget.html) | ⬜ |
| Hot Lead Notification | Manual + curl | ⬜ |
| Booking Token | curl/Postman | ⬜ |
