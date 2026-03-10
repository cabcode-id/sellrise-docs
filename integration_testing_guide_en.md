# Sellrise Integration Testing Guide — Backend ↔ Frontend

A comprehensive guide for end-to-end testing of Sellrise functionality. All tests are performed with the **backend** (`localhost:8000`) and **frontend** (`localhost:5173`) running concurrently.

---

## Prerequisites

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

# 5. Start frontend (separate terminal)
cd d:\pukul6\sellrise\Sellrise-Front-End
npm run dev
```

> [!TIP]
> The [seed_all.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/scripts/seed_all.py) seeder will create an admin user, workspace, domain, leads, KB articles, FAQs, and dummy scenarios. Note the **email/password** for the admin printed in the terminal.

---

## Part A: Backend Unit/Integration Tests (Pytest)

Run all existing backend tests at once:

```powershell
cd d:\pukul6\sellrise\Sellrise-BackEnd
.venv\Scripts\pytest tests\ -v
```

Existing test files:

| File | Coverage |
|---|---|
| [test_rbac.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_rbac.py) | RBAC enforcement (admin/agent/viewer permissions) |
| [test_workspaces_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_workspaces_api.py) | Workspace CRUD + isolation |
| [test_users_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_users_api.py) | User management (invite/role/deactivate) |
| [test_domains_api.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_domains_api.py) | Domain CRUD + branding |
| [test_notification_settings.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_notification_settings.py) | Hot lead notification config |
| [test_widget.py](file:///d:/pukul6/sellrise/Sellrise-BackEnd/tests/test_widget.py) | Widget session, lead submit, KB search |

> [!IMPORTANT]
> The test database is separate and runs on port **5432** (not 5433). Ensure PostgreSQL Docker also exposes port 5432, or create a `sellrise_test_db` in the same instance.

---

## Part B: Manual Integration Test — Step by Step

### 🔐 Module 1: Authentication (US-2.1, 2.4)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `http://localhost:5173/login` | Login page appears | |
| 2 | Login with seeder admin email/password | Redirect to `/dashboard`, user's name appears | |
| 3 | Refresh page | Stay logged in (token refresh working) | |
| 4 | Click Logout | Redirect to `/login`, token cleared | |
| 5 | Access `/dashboard` without living | Automatic redirect to `/login` | |

**API Verification:** Open DevTools → Network tab, ensure:
- `POST /v1/auth/login` → 200, response contains `access_token`
- `GET /v1/auth/me` → 200, returns user info
- `POST /v1/auth/logout` → 200

---

### 🏢 Module 2: Workspace & Settings (US-1.1, 9.1)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/settings` page | Workspace settings appear | |
| 2 | Update workspace name | Name changes, API call `PATCH /v1/workspaces/{id}` → 200 | |
| 3 | Check notification settings (hot lead recipients) | Form appears, can save | |

---

### 👥 Module 3: Team Management (US-2.2, 2.3, 9.2)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/settings` → Team tab | Users list (at least admin) appears | |
| 2 | Invite new user (email, role: Agent) | `POST /v1/users` → 201, user appears in list | |
| 3 | Invite user with same email | Error 409 (duplicate) | |
| 4 | Change user role to Viewer | `PATCH /v1/users/{id}` → 200, role updated | |
| 5 | Login as Viewer, try admin-only feature | Access denied (403) | |

---

### 🌐 Module 4: Domain Management (US-1.3)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/widget-settings` | Domain/widget settings page appears | |
| 2 | Add new domain (e.g. `testdomain.com`) | `POST /v1/domains` → 201, domain appears in list | |
| 3 | Edit branding (name, color, logo URL) | `PATCH /v1/domains/{id}` → 200, branding updated | |
| 4 | Delete domain | Domain disappears from list | |

---

### 🤖 Module 5: Scenarios (US-4.1, 4.2, 4.3)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/scenarios` | Scenarios list appears (from seed data) | |
| 2 | Create new scenario | `POST /v1/scenarios` → 201, appears in list | |
| 3 | Open `/scenario-config` and edit scenario | Config editor appears, can edit nodes/steps | |
| 4 | Publish scenario | `POST /v1/scenarios/{id}/publish` → 200, status → published | |
| 5 | Try to delete published scenario | Error 400/409 (cannot delete published scenario) | |
| 6 | Test **Scenario Simulator** (if in UI) | `POST /v1/scenarios/{id}/simulate` → 200, bot reply appears | |

---

### 📋 Module 6: Lead Management (US-5.1 – 5.8)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/leads` | Leads list appears (from seed data) | |
| 2 | Open `/pipeline` | Kanban board appears with leads in stages | |
| 3 | Drag lead to another stage | `PATCH /v1/leads/{id}` → 200, stage update | |
| 4 | Click lead → detail view | Lead detail appears (info, events, notes) | |
| 5 | Add note in lead detail | `POST /v1/leads/{id}/notes` → 201, note appears | |
| 6 | Search lead by name | Filter works, results match | |
| 7 | Filter by stage (e.g. `hot`) | Only leads with that stage appear | |
| 8 | Assign lead to user | `PATCH /v1/leads/{id}` with `user_id`, status updated | |

---

### 📚 Module 7: Knowledge Base (US-6.1 – 6.4)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/knowledge-base` | Articles list appears | |
| 2 | Create new article | `POST /v1/kb/articles` → 201, article appears | |
| 3 | Edit article | `PATCH /v1/kb/articles/{id}` → 200 | |
| 4 | Delete article | Article removed from list | |
| 5 | Search article (typing keyword) | Search results appear based on keyword | |

---

### 📊 Module 8: Analytics (US-7.1, 7.2)

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open `/analytics` | Analytics dashboard appears | |
| 2 | Check summary metrics | `GET /v1/analytics/summary` → 200, data appears | |
| 3 | Check funnel chart | `GET /v1/analytics/funnel` → 200, chart appears | |
| 4 | Export CSV | CSV file downloads | |

---

### 🔌 Module 9: Widget Flow (US-3.1 – 3.4, 4.4)

This test uses the existing [test_widget.html](file:///d:/pukul6/sellrise/Sellrise-BackEnd/test_widget.html) file in the backend:

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Open [d:\pukul6\sellrise\Sellrise-BackEnd\test_widget.html](file:///d:/pukul6/sellrise/Sellrise-BackEnd/test_widget.html) in browser | Test page appears | |
| 2 | Check console: "Session established" | `POST /v1/widget/session` → 200, session_id appears | |
| 3 | Badge "✅ Connected to Backend" appears | Handshake successful | |

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

| # | Step | Expected Result | ✅ |
|---|---|---|---|
| 1 | Ensure SMTP config in [.env](file:///d:/pukul6/sellrise/Sellrise-BackEnd/.env) is correct | `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD` filled | |
| 2 | Set notification recipients in Settings | Recipients saved | |
| 3 | Update lead score to `hot` | `PATCH /v1/leads/{id}` with `score: "hot"` | |
| 4 | Check recipient email | Notification email received (check spam folder) | |
| 5 | Check lead events | `notification_sent` event appears in lead detail | |

---

### 📅 Module 11: Booking Token (US-10.1)

Test via curl/Postman (feature may not be in FE yet):

```powershell
# Create booking token
curl -X POST http://localhost:8000/v1/booking/tokens -H "Authorization: Bearer YOUR_TOKEN" -H "Content-Type: application/json" -d "{\"lead_id\": \"LEAD_ID\"}"

# Get token info
curl http://localhost:8000/v1/booking/tokens/TOKEN_VALUE -H "Authorization: Bearer YOUR_TOKEN"

# Consume token (single-use)
curl -X POST http://localhost:8000/v1/booking/tokens/TOKEN_VALUE/consume -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Part C: Postman Collection

Import the existing collection for structured API testing:

1. Open Postman → Import → File
2. Select [d:\pukul6\sellrise\Sellrise-BackEnd\Sellrise_API_Collection.postman_collection.json](file:///d:/pukul6/sellrise/Sellrise-BackEnd/Sellrise_API_Collection.postman_collection.json)
3. Set variables:
   - `base_url` = `http://localhost:8000`
   - `workspace_id` = (from seeder output)
4. Run the **Login** request first (auto-sets `access_token`)
5. Run other requests sequentially

---

## Summary Checklist

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
