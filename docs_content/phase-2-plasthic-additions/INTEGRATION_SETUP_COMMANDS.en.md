# Sellrise ↔ PlasthicBE — Integration Setup Commands (English)

> **Context:** SellriseBE runs on port **8000**, PlasthicBE runs on port **8001** (localhost).

---

## PHASE 1 — Setup PlasthicBE

### 1.1 Ensure PlasthicBE `.env` is correct

```bash
# Check the .env contents
cat /path/to/PlasthicBE/.env

# Critical values that must exist:
# SECRET_KEY — MUST be replaced from the default "change-me-to-a-long-random-secret"
# DATABASE_URL — ensure it points to the correct DB
# ALLOWED_ORIGINS — add Sellrise origins (frontend and BE)
```

**Minimal example `.env` for production:**

```env
APP_NAME=Phlastic Patient Service
APP_ENV=production
API_V1_PREFIX=/api/v1
SECRET_KEY=<REPLACE_WITH_RANDOM_STRING_MIN_32_CHAR>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
REFRESH_TOKEN_EXPIRE_DAYS=30
DATABASE_URL=postgresql+asyncpg://postgres:<PASSWORD>@localhost:5432/phlasticbe
ALLOWED_ORIGINS=["http://localhost:8000","http://localhost:5173","http://localhost:3000"]
PATIENT_MEDIA_ROOT=./storage/patient-photos
MAX_PHOTO_SIZE_BYTES=10485760
RESEND_API_KEY=
RESEND_FROM_EMAIL=noreply@example.com
```

> ⚠️ **Important:** `SECRET_KEY` is used to sign JWTs. Tokens generated in the following steps must be created on the PlasthicBE server using the same `SECRET_KEY` found in `.env`.

---

### 1.2 Install dependencies & run migrations

```bash
cd /path/to/PlasthicBE

# Create a virtual environment if not present
python3 -m venv .venv

# Install dependencies
.venv/bin/pip install -e .

# Create storage directory for photos
mkdir -p storage/patient-photos

# Run all DB migrations
.venv/bin/alembic upgrade head
```

---

### 1.3 Generate Service JWT (TOKEN SELLRISE → PLASTHIC)

> This token is used by Sellrise to call PlasthicBE. It must be generated on the PlasthicBE server because it uses the `SECRET_KEY` from `.env`.

```bash
cd /path/to/PlasthicBE

python3 scripts/generate_service_token.py
```

**Copy the token output** — you'll need it in the next phase.

Example output:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzZWxscmlzZS13ZWJob29r...
```

> The token is valid for **10 years**. Store it securely — it is a system credential.

---

### 1.4 Verify the token can call PlasthicBE

```bash
# Replace <TOKEN> with the output from step 1.3
TOKEN="<OUTPUT_FROM_GENERATE_SERVICE_TOKEN>"

curl -s -X POST "http://localhost:8001/api/v1/patients" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "sellrise_lead_id": "token-verify-test",
    "sellrise_workspace_id": "test-workspace",
    "name": "Token Verify",
    "email": "token-verify@test.com",
    "consent_given": true,
    "consent_text_version": "v1",
    "consent_timestamp": "2026-01-01T00:00:00Z"
  }'
```

**Expected response:** HTTP 201 with patient data.
If you get 403 → token is invalid or `SECRET_KEY` in `.env` doesn't match the one used to generate the token.

```bash
# Remove the test patient afterwards
psql -U postgres -d phlasticbe -c "DELETE FROM patients WHERE sellrise_lead_id = 'token-verify-test';"
```

---

## PHASE 2 — Insert Settings into SellriseBE DB

> For each workspace to integrate, run the SQL below.
> If you prefer to set via API, use `PATCH /v1/workspaces/{workspace_id}/settings` for workspace settings and `PUT /v1/workspaces/{workspace_id}/pipeline-stages` for stage list. There is no special `POST` for these integration settings.

### 2.1 Find the workspace to integrate

```bash
psql -U postgres -d sellrise -c "SELECT id, name, slug FROM workspaces ORDER BY created_at;"
```

Example output:
```
                  id                  |            name            |           slug
--------------------------------------+----------------------------+---------------------------
 e108dc53-ee57-4bbb-ad8d-05eb0cd3550f | Default Workspace          | default
 fda65c4c-82c6-4a0d-9505-30bd47aa2362 | Plasthic Admin's Workspace | plasthic-admin-s-workspace
 75beb404-f23c-458b-8868-aeccd5efd00a | Widget Test Workspace      | widget-test-ws
```

---

### 2.2 Create SQL file for a single workspace

> Replace `WORKSPACE_ID` and `SERVICE_JWT` below.

Save to a file named `insert_integration_settings.sql`:

```sql
-- ============================================================
-- Sellrise ↔ PlasthicBE Integration Settings
-- Run: psql -U postgres -d sellrise -f insert_integration_settings.sql
-- ============================================================

-- Replace these two variables:
-- WORKSPACE_ID   : workspace id from the query above
-- SERVICE_JWT    : token from scripts/generate_service_token.py output

UPDATE workspaces
SET settings = '{
  "patient_service": {
    "enabled": true,
    "base_url": "http://localhost:8001/api/v1",
    "auth_token": "<SERVICE_JWT>"
  },
  "webhooks": [
    {
      "name": "phlastic",
      "url": "http://localhost:8001/api/v1/patients",
      "enabled": true,
      "events": ["lead_submitted"],
      "auth_type": "bearer",
      "auth_token": "<SERVICE_JWT>",
      "response_ref_path": "id",
      "payload_mapping": {
        "sellrise_lead_id":        "lead.id",
        "sellrise_workspace_id":   "lead.workspace_id",
        "name":                    "lead.name",
        "email":                   "lead.email",
        "phone":                   "lead.phone",
        "procedure_interest":      "lead.custom_fields.procedure",
        "budget_range":            "lead.custom_fields.budget_range",
        "timeframe":               "lead.custom_fields.timeframe",
        "location":                "lead.country",
        "qualification_score":     "lead.score_numeric",
        "qualification_data":      "lead.custom_fields",
        "consent_given":           "lead.consent_given",
        "consent_timestamp":       "lead.consent_date",
        "source_url":              "lead.page_url",
        "utm_source":              "lead.utm_source",
        "utm_medium":              "lead.utm_medium",
        "utm_campaign":            "lead.utm_campaign"
      },
      "payload_transforms": {
        "procedure_interest": "wrap_list"
      },
      "static_fields": {
        "consent_text_version": "phlastic_v1",
        "consent_text": "Patient consented to data processing via Sellrise chatbot widget"
      }
    }
  ]
}'::json
WHERE id = '<WORKSPACE_ID>';
```

> In Phlastic, `procedure_interest` is a `list[str]`, but Sellrise widget slots are safest stored as a string in `custom_fields.procedure`. The webhook will wrap it into an array using `payload_transforms.wrap_list`.

---

### 2.2A Set integration settings via REST API (recommended)

If SellriseBE is running and you have an admin user for the workspace, use the following request.

```bash
SELLRISE_URL="http://localhost:8000"
WORKSPACE_ID="<WORKSPACE_ID>"
SERVICE_JWT="<SERVICE_JWT>"
ADMIN_EMAIL="<ADMIN_EMAIL>"
ADMIN_PASSWORD="<ADMIN_PASSWORD>"

LOGIN_RES=$(curl -s -X POST "$SELLRISE_URL/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$ADMIN_EMAIL\",\"password\":\"$ADMIN_PASSWORD\"}")

ADMIN_TOKEN=$(echo "$LOGIN_RES" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -X PATCH "$SELLRISE_URL/v1/workspaces/$WORKSPACE_ID/settings" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"patient_service\": {
      \"enabled\": true,
      \"base_url\": \"http://localhost:8001/api/v1\",
      \"auth_token\": \"$SERVICE_JWT\"
    },
    \"webhooks\": [
      {
        \"name\": \"phlastic\",
        \"url\": \"http://localhost:8001/api/v1/patients\",
        \"enabled\": true,
        \"events\": [\"lead_submitted\"],
        \"auth_type\": \"bearer\",
        \"auth_token\": \"$SERVICE_JWT\",
        \"response_ref_path\": \"id\",
        \"payload_mapping\": {
          \"sellrise_lead_id\": \"lead.id\",
          \"sellrise_workspace_id\": \"lead.workspace_id\",
          \"name\": \"lead.name\",
          \"email\": \"lead.email\",
          \"phone\": \"lead.phone\",
          \"procedure_interest\": \"lead.custom_fields.procedure\",
          \"budget_range\": \"lead.custom_fields.budget_range\",
          \"timeframe\": \"lead.custom_fields.timeframe\",
          \"location\": \"lead.country\",
          \"qualification_score\": \"lead.score_numeric\",
          \"qualification_data\": \"lead.custom_fields\",
          \"consent_given\": \"lead.consent_given\",
          \"consent_timestamp\": \"lead.consent_date\",
          \"source_url\": \"lead.page_url\",
          \"utm_source\": \"lead.utm_source\",
          \"utm_medium\": \"lead.utm_medium\",
          \"utm_campaign\": \"lead.utm_campaign\"
        },
        \"payload_transforms\": {
          \"procedure_interest\": \"wrap_list\"
        },
        \"static_fields\": {
          \"consent_text_version\": \"phlastic_v1\",
          \"consent_text\": \"Patient consented to data processing via Sellrise chatbot widget\"
        }
      }
    ]
  }" | python3 -m json.tool
```

If the workspace uses custom pipeline stages, set them with the pipeline stages endpoint:

```bash
curl -s -X PUT "$SELLRISE_URL/v1/workspaces/$WORKSPACE_ID/pipeline-stages" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "stages": [
      "new",
      "qualified",
      "consultation_booked",
      "photos_received",
      "doctor_reviewed",
      "contract_signed",
      "procedure_done",
      "lost"
    ]
  }' | python3 -m json.tool
```

Verify the settings:

```bash
curl -s "$SELLRISE_URL/v1/workspaces/$WORKSPACE_ID/settings" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool

curl -s "$SELLRISE_URL/v1/workspaces/$WORKSPACE_ID/pipeline-stages" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

---

### 2.2B Correct scenario / slot convention

For Phlastic integration, the widget scenario should save answers into `custom_fields` using these keys:

```json
{
  "procedure": "DHI hair transplant",
  "budget_range": "8000$",
  "timeframe": "may 2",
  "goal": "hair restoration"
}
```

Do not rely on `custom_fields.procedure_interest` as the primary source if the widget still sends a string. In Phlastic, `procedure_interest` is an array; conversion is handled in the webhook layer via `payload_transforms.wrap_list`.

---

### 2.3 Quick one-liner to populate multiple workspaces

```bash
# Set variables first
WORKSPACE_ID="75beb404-f23c-458b-8868-aeccd5efd00a"
SERVICE_JWT="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Substitute and run
sed -e "s|<WORKSPACE_ID>|$WORKSPACE_ID|g" \
    -e "s|<SERVICE_JWT>|$SERVICE_JWT|g" \
    insert_integration_settings.sql \
  | psql -U postgres -d sellrise
```

---

### 2.4 Verify settings were saved

```bash
WORKSPACE_ID="<WORKSPACE_ID>"

psql -U postgres -d sellrise -t -A \
  -c "SELECT settings FROM workspaces WHERE id = '$WORKSPACE_ID';" \
  | python3 -m json.tool
```

Ensure the output includes:
- `webhooks` array containing `"url": "http://localhost:8001/api/v1/patients"`
- `auth_token` containing the correct JWT
- `payload_mapping` fully populated

---

## PHASE 3 — Start Both Services

### 3.1 Start PlasthicBE (port 8001)

```bash
cd /path/to/PlasthicBE

# Development (with reload)
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

# Production
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8001 --workers 2
```

### 3.2 Start SellriseBE (port 8000)

```bash
cd /path/to/SellriseBE

# Development (with reload)
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Production
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
```

### 3.3 Health checks for both services

```bash
curl -s http://localhost:8000/health      # SellriseBE
curl -s http://localhost:8001/api/v1/health  # PlasthicBE
```

Expected responses:
```json
{"status":"ok","database":"ok"}
{"status":"ok","db":"connected"}
```

---

## PHASE 4 — End-to-End Synchronization Test

### 4.1 Submit a lead via the widget (trigger webhook to PlasthicBE)

```bash
WORKSPACE_ID="<WORKSPACE_ID>"

curl -s -X POST "http://localhost:8000/v1/widget/lead" \
  -H "Content-Type: application/json" \
  -d "{
    \"workspace_id\": \"$WORKSPACE_ID\",
    \"name\": \"Test Patient Integration\",
    \"email\": \"test-integration-$(date +%s)@example.com\",
    \"phone\": \"+62812345678\",
    \"country\": \"Indonesia\",
    \"consent_given\": true,
    \"consent_timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
    \"custom_fields\": {
      \"procedure\": \"Rhinoplasty\",
      \"budget_range\": \"10-15 juta\",
      \"timeframe\": \"3 bulan\",
      \"goal\": \"Memperbaiki bentuk hidung\"
    },
    \"page_url\": \"http://localhost:5173/rhinoplasty\",
    \"utm_source\": \"test\"
  }"
```

**Response:** `{"lead_id":"<UUID>","message":"Lead received"}`
Copy the returned `lead_id`.

---

### 4.2 Wait for the webhook background task to complete (3–5 seconds)

```bash
sleep 5
```

---

### 4.3 Check the lead in SellriseBE — verify `external_identities` was populated

```bash
LEAD_ID="<LEAD_ID_FROM_STEP_4.1>"

psql -U postgres -d sellrise -t -A \
  -c "SELECT id, name, email, score, external_identities FROM leads WHERE id = '$LEAD_ID';"
```

Expected record includes:
```
<uuid>|Test Patient Integration|...|warm|{"phlastic": "<patient-uuid>"}
```

---

### 4.4 Check patient data in PlasthicBE — ensure data synced

```bash
LEAD_ID="<LEAD_ID_FROM_STEP_4.1>"

psql -U postgres -d phlasticbe -t -A \
  -c "SELECT id, name, email, procedure_interest, qualification_score, qualification_data FROM patients WHERE sellrise_lead_id = '$LEAD_ID';"
```

---

### 4.5 Check webhook logs in SellriseBE

```bash
LEAD_ID="<LEAD_ID_FROM_STEP_4.1>"

psql -U postgres -d sellrise \
  -c "SELECT webhook_name, status_code, success, external_ref_returned, response_time_ms, attempt FROM webhook_logs WHERE lead_id = '$LEAD_ID';"
```

**Should show:** `success = t`, `status_code = 201`, `external_ref_returned = <patient-uuid>`

---

### 4.6 Check webhook logs via REST API

```bash
# Login to get admin token
LOGIN_RES=$(curl -s -X POST "http://localhost:8000/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"<ADMIN_EMAIL>","password":"<ADMIN_PASSWORD>"}')

ADMIN_TOKEN=$(echo $LOGIN_RES | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Retrieve recent webhook logs
curl -s "http://localhost:8000/v1/webhooks/logs?limit=10" \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

---

## PHASE 5 — Troubleshooting

### Webhook failures: check error logs

```bash
psql -U postgres -d sellrise \
  -c "SELECT webhook_name, status_code, error_message, response_body FROM webhook_logs WHERE success = false ORDER BY created_at DESC LIMIT 10;"
```

### Invalid token (403 from PlasthicBE)

```bash
# Check SECRET_KEY in PlasthicBE .env
grep SECRET_KEY /path/to/PlasthicBE/.env

# Re-generate the token with the correct SECRET_KEY
cd /path/to/PlasthicBE && python3 scripts/generate_service_token.py

# Update workspace settings with the new token (repeat Phase 2)
```

### Consent_timestamp NULL (lead missing consent_date)

If a lead lacks `consent_date`, the webhook will send `consent_timestamp: null` to PlasthicBE. PlasthicBE requires this field — either set a `static_fields.consent_timestamp` in the webhook settings or ensure the widget always includes a `consent_timestamp` when submitting leads.

Example `static_fields` fallback:

```json
"static_fields": {
  "consent_text_version": "phlastic_v1",
  "consent_text": "Patient consented to data processing via Sellrise chatbot widget",
  "consent_timestamp": "2026-01-01T00:00:00Z"
}
```

### Workspace not receiving webhooks (no logs)

```bash
# Ensure workspace settings include webhooks
WORKSPACE_ID="<WORKSPACE_ID>"
psql -U postgres -d sellrise -t -A \
  -c "SELECT settings->>'webhooks' FROM workspaces WHERE id = '$WORKSPACE_ID';"
```

If the result is `null` or `[]` → reapply Phase 2 settings.

### Phlastic returns 422 for `procedure_interest`

This commonly means the webhook mapping is reading the wrong key from `lead.custom_fields`.

Quick checklist:

```bash
# 1. Check workspace mapping
WORKSPACE_ID="<WORKSPACE_ID>"
psql -U postgres -d sellrise -t -A \
  -c "SELECT settings->'webhooks'->0->'payload_mapping'->>'procedure_interest' FROM workspaces WHERE id = '$WORKSPACE_ID';"

# 2. Check the actual lead data
LEAD_ID="<LEAD_ID>"
psql -U postgres -d sellrise -t -A \
  -c "SELECT custom_fields FROM leads WHERE id = '$LEAD_ID';"
```

Recommended mapping value:

```
lead.custom_fields.procedure
```

If the lead stores `{"procedure": "DHI hair transplant"}`, the webhook will convert it to `"procedure_interest": ["DHI hair transplant"]` before POSTing to Plasthic.

---

## Deployment Checklist (Summary)

```
[ ] 1. SECRET_KEY in PlasthicBE .env is not the default value
[ ] 2. PlasthicBE DATABASE_URL points to the correct DB
[ ] 3. alembic upgrade head has been run on PlasthicBE
[ ] 4. Service JWT has been generated from PlasthicBE (scripts/generate_service_token.py)
[ ] 5. SellriseBE workspace settings updated with token and correct URL
[ ] 6. PlasthicBE running on port 8001 (health check: 200 OK)
[ ] 7. SellriseBE running on port 8000 (health check: 200 OK)
[ ] 8. End-to-end test: submit lead → verify external_identities → verify patient in phlasticbe
[ ] 9. Webhook logs show success = true and status_code = 201
```
