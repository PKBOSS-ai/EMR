# Carevanta EMR - Production Deployment Guide

## Overview

Production deployment guide for Carevanta EMR V1.0 controlled clinic pilot.

**IMPORTANT:** Do not deploy to a public server without explicit approval. Do not expose real patient data during pilot.

## Architecture

- **Backend:** Python + FastAPI (port 8000)
- **Frontend:** React + TypeScript + Vite (port 5173)
- **Database:** SQLite (development) / PostgreSQL (production required)
- **Authentication:** JWT access tokens (30 min) + refresh tokens (7 days)
- **Authorization:** RBAC with clinic-level data isolation

## Prerequisites

- Python 3.12+
- Node.js 18+
- PostgreSQL 16+ (production required)
- Docker & Docker Compose (optional, for containerized deployment)

## Step 1: Backend Production Configuration

### 1.1 Create Production Environment File

```bash
cd backend
cp .env.production.example .env
```

Edit `.env` with production values. **All `<REQUIRED>` placeholders must be replaced.**

```ini
ENVIRONMENT=production
SECRET_KEY=<generate with: openssl rand -hex 32>
DATABASE_URL=postgresql+psycopg2://emr:strong-password@localhost:5432/emr
CORS_ORIGINS=https://emr.yourclinic.com
FRONTEND_URL=https://emr.yourclinic.com
SMTP_HOST=smtp.yourprovider.com
SMTP_PORT=587
SMTP_USER=your-smtp-user
SMTP_PASS=your-smtp-password
SMTP_FROM=noreply@yourclinic.com
SMTP_USE_TLS=true
```

### 1.2 Generate a Secure Secret Key

```bash
openssl rand -hex 32
```

**Never use the default placeholder. The application will refuse to start with a weak or placeholder SECRET_KEY.**

### 1.3 Install Dependencies

```bash
cd backend
python -m venv .venv
.venv\Scripts\activate  # Windows
# or
source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt
```

### 1.4 Run Database Migrations

```bash
cd backend
alembic upgrade head
```

### 1.5 Seed the Database

```bash
cd backend
python seed.py
```

This creates:
- Clinic: "Carevanta Clinic"
- Admin user: admin@example.com / AdminPass123!
- Roles: Clinic Admin, Doctor, Receptionist, Nurse
- Permissions: All system permissions

**Change the admin password after first login.**

## Step 2: Frontend Production Configuration

### 2.1 Create Production Environment File

```bash
cd frontend
cp .env.example .env
```

Edit `.env`:

```ini
VITE_API_URL=https://api.emr.yourclinic.com
```

### 2.2 Install Dependencies and Build

```bash
cd frontend
npm install
npm run build
```

The production build outputs to `dist/`.

## Step 3: Database Setup

### Option A: SQLite (Single-server, small pilot only)

SQLite works for a single-clinic pilot with low concurrency. Foreign keys are enforced via SQLAlchemy event listeners.

Backup the database regularly:

```bash
cd backend
python backup_db.py emr.db backups
```

### Option B: PostgreSQL (Required for multi-user production)

```bash
docker run --name emr-postgres -e POSTGRES_USER=emr -e POSTGRES_PASSWORD=strong-password -e POSTGRES_DB=emr -p 5432:5432 -v pgdata:/var/lib/postgresql/data -d postgres:16
```

Then run migrations and seed as described above.

## Step 4: Running the Application

### Development

Backend:

```bash
cd backend
uvicorn app.main:app --reload
```

Frontend:

```bash
cd frontend
npm run dev
```

### Production (using Docker Compose)

```bash
docker-compose up -d
```

### Production (manual)

Backend:

```bash
cd backend
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Frontend (serve `dist/` with nginx or similar):

```bash
# Copy dist/ to your web server root
```

## Step 5: Security Checklist

- [ ] `SECRET_KEY` is a strong, randomly generated value (min 32 chars)
- [ ] `ENVIRONMENT=production`
- [ ] `CORS_ORIGINS` only includes your production domain(s), no wildcards
- [ ] SMTP credentials are configured (not left blank)
- [ ] `.env` is NOT committed to version control (gitignored)
- [ ] Admin password changed after first login
- [ ] HTTPS configured at the reverse proxy level
- [ ] Database backups scheduled and tested

## Step 6: Database Backups

Automated backup script is available at `backend/backup_db.py`:

```bash
# Create a backup
python backup_db.py emr.db backups

# Schedule with cron (daily at 2 AM)
0 2 * * * /path/to/python /path/to/backend/backup_db.py /path/to/emr.db /path/to/backups
```

Backups are rotated (keeps last 30 by default). Each backup is verified with `PRAGMA integrity_check`.

## Step 7: Health Checks

```bash
curl https://api.emr.yourclinic.com/health
# Returns: {"status": "ok"}
```

## Rollback Instructions

1. Stop the application
2. Restore the database from the most recent backup:

   ```bash
   cp backups/emr.db.YYYYMMDD-HHMMSS.bak emr.db
   ```

3. Redeploy the previous version of the application
4. Verify health endpoint

## Default Accounts

| Role | Email | Password |
|------|-------|----------|
| Clinic Admin | admin@example.com | AdminPass123! |

**Change all default passwords before pilot.**

## Security Requirements

- Never commit `.env` to version control
- Never print or expose SMTP credentials
- Never use placeholder SECRET_KEY in production
- Never allow wildcard `*` in production CORS_ORIGINS
- Do not migrate or delete the production database without a backup
- Do not deploy publicly without approval

## Support

Report issues at: https://github.com/Kilo-Org/kilocode/issues
