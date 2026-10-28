# OutreachEngine — Production Deployment Guide

**Document Version:** 1.0.0-draft  
**Status:** **`[PROPOSED]`** (Deployment Specification)  
**Target Infrastructure:** Render.com (API & Worker), Vercel (SPA Frontend), PostgreSQL 16 (Render Managed)  
**Last Updated:** 2026-10-28  

> **Prerequisite Notice:**  
> Production deployment requires manual account creation, OAuth client credential approval from Google Cloud Console, and API keys. The steps below define the exact configuration needed for a verified zero-downtime deployment.

---

## 1. Architecture Deployment Topology

```text
[ Vercel Edge Network ] ──▶ Static React SPA (Watermelon UI)
         │
         │ API Calls (https://api.outreachengine.com)
         ▼
[ Render Web Service ] ──▶ FastAPI Application (Uvicorn ASGI)
         │
         ├─── PostgreSQL 16 (Render Managed Database)
         │           ▲
         │           │ Row Lease Polling (SKIP LOCKED)
[ Render Worker ] ───┘
(python -m app.worker.daemon)
```

---

## 2. Google Cloud Platform (GCP) Configuration

### 2.1 Project & OAuth Consent Setup
1. Log in to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new GCP Project: `outreachengine-prod`.
3. Enable Required APIs:
   - **Gmail API** (`gmail.googleapis.com`)
   - **Google Sheets API** (`sheets.googleapis.com`)
   - **Generative Language API** (`generativelanguage.googleapis.com`)
4. Configure **OAuth Consent Screen**:
   - User Type: **External**
   - App Name: `OutreachEngine`
   - User Support Email: your operational email
   - Developer Contact: your operational email
   - Scopes:
     - `https://www.googleapis.com/auth/gmail.send`
     - `https://www.googleapis.com/auth/gmail.readonly`
     - `https://www.googleapis.com/auth/userinfo.email`
     - `https://www.googleapis.com/auth/spreadsheets`
5. Create **OAuth 2.0 Client IDs**:
   - Application Type: **Web Application**
   - Name: `OutreachEngine Production Client`
   - Authorized Javascript Origins:
     - `https://outreachengine.vercel.app`
     - `https://api.outreachengine.com`
   - Authorized Redirect URIs:
     - `https://api.outreachengine.com/api/v1/auth/google/callback`

### 2.2 Google Gemini API Setup
1. Visit [Google AI Studio](https://aistudio.google.com/).
2. Generate an API Key under project `outreachengine-prod`.
3. Note the key for injection into Render environment variables.

---

## 3. Render.com Infrastructure Setup

We recommend defining infrastructure declaratively using Render's `render.yaml` Blueprint:

```yaml
services:
  # 1. FastAPI Web Service
  - type: web
    name: outreachengine-api
    env: python
    region: oregon
    plan: standard
    buildCommand: "pip install -r requirements.txt && alembic upgrade head"
    startCommand: "uvicorn app.main:app --host 0.0.0.0 --port $PORT --workers 2"
    healthCheckPath: /health/ready
    envVars:
      - key: ENVIRONMENT
        value: production
      - key: LOG_LEVEL
        value: INFO
      - key: DATABASE_URL
        fromDatabase:
          name: outreachengine-db
          property: connectionString
      - key: SECRET_KEY
        generateValue: true
      - key: FERNET_KEY
        sync: false # Injected manually via Render Dashboard
      - key: GOOGLE_CLIENT_ID
        sync: false
      - key: GOOGLE_CLIENT_SECRET
        sync: false
      - key: GOOGLE_REDIRECT_URI
        value: https://api.outreachengine.com/api/v1/auth/google/callback
      - key: GEMINI_API_KEY
        sync: false
      - key: CORS_ORIGINS
        value: '["https://outreachengine.vercel.app"]'

  # 2. Background Dispatch Worker
  - type: worker
    name: outreachengine-worker
    env: python
    region: oregon
    plan: standard
    buildCommand: "pip install -r requirements.txt"
    startCommand: "python -m app.worker.daemon"
    envVars:
      - key: ENVIRONMENT
        value: production
      - key: LOG_LEVEL
        value: INFO
      - key: DATABASE_URL
        fromDatabase:
          name: outreachengine-db
          property: connectionString
      - key: FERNET_KEY
        sync: false
      - key: GOOGLE_CLIENT_ID
        sync: false
      - key: GOOGLE_CLIENT_SECRET
        sync: false
      - key: GEMINI_API_KEY
        sync: false

databases:
  # 3. Managed PostgreSQL Database
  - name: outreachengine-db
    region: oregon
    plan: standard
    postgresMajorVersion: 16
```

---

## 4. Vercel Frontend Deployment

1. Connect your GitHub repository to Vercel.
2. Configure Project Settings:
   - **Framework Preset**: Vite
   - **Root Directory**: `frontend`
   - **Build Command**: `npm run build`
   - **Output Directory**: `dist`
3. Environment Variables:
   - `VITE_API_BASE_URL`: `https://api.outreachengine.com/api/v1`
4. Routing Rewrites (`frontend/vercel.json`):
   ```json
   {
     "rewrites": [
       { "source": "/(.*)", "destination": "/index.html" }
     ]
   }
   ```

---

## 5. Safe Enablement of Live Sending (Sanity & Dry Run)

To avoid accidental live blasts immediately upon deployment:

1. **Dry-Run Mode Configuration**:
   - Set `MOCK_GMAIL_DISPATCH=true` in Render environment variables during initial smoke testing.
   - The worker will execute all prompt generation, placeholder validation, and scheduling routines without transmitting actual emails.
2. **First Live Smoke Test**:
   - Once dry-run verification passes, set `MOCK_GMAIL_DISPATCH=false`.
   - Set `MAX_GLOBAL_DAILY_SENDS=5`.
   - Create a test contact with your own secondary email address.
   - Generate, review, approve, and observe the live dispatch, tracking pixel delivery, and reply detection.
   - Increase `MAX_GLOBAL_DAILY_SENDS` to desired production quota (e.g. 40).

---

## 6. Health Checks, Rollback, & Disaster Recovery

- **Liveness & Readiness**:
  - `GET /health`: Basic process ping.
  - `GET /health/ready`: Checks PostgreSQL database connection pool and worker lease freshness.
- **Rollback Procedure**:
  - If a deployment failure occurs, roll back the Git commit on Render.
  - If schema migrations need rollback: `alembic downgrade -1`.
- **Database Backups**:
  - Render Managed PostgreSQL provides automated point-in-time recovery (PITR) up to 7 days.
  - Nightly logical dumps can be automated via cron executing `pg_dump`.
