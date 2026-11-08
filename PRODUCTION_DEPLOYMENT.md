# Carevanta EMR - Production Deployment Guide

## Overview

Production deployment guide for Carevanta EMR V1.0 controlled clinic pilot.

**IMPORTANT:** Do not deploy to a public server without explicit approval. Do not expose real patient data during pilot.

---

## 1. System Requirements

### Backend
- Python 3.12+
- 512 MB RAM minimum (1 GB recommended)
- 2 GB disk space minimum
- PostgreSQL 16+ (production) or SQLite (development/pilot)

### Frontend
- Node.js 18+
- 256 MB RAM minimum
- 500 MB disk space minimum

### Infrastructure (Docker deployment)
- Docker 24+
- Docker Compose 2.20+
- 2 GB RAM minimum (4 GB recommended)

---

## 2. Environment Configuration

### Step 1: Copy production environment templates

```bash
cd D:\EMR\backend
cp .env.production.example .env

cd D:\EMR\frontend
cp .env.example .env
```

### Step 2: Edit backend `.env`

Replace all `<REQUIRED>` placeholders:

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

### Step 3: Edit frontend `.env`

```ini
VITE_API_URL=https://api.emr.yourclinic.com
```

**Security rules:**
- Never commit `.env` to version control (it is gitignored)
- Never print or expose SMTP credentials
- Never use placeholder SECRET_KEY in production
- Never allow wildcard `*` in production CORS_ORIGINS

---

## 3. Secret Generation

### Generate SECRET_KEY

```bash
openssl rand -hex 32
```

### Generate SMTP app password

Use your SMTP provider's app password feature (e.g., Gmail App Passwords).

### Store secrets securely

For Docker deployments, store SECRET_KEY in a secrets file:

```bash
mkdir -p ./secrets
echo -n "<your-secret-key>" > ./secrets/secret_key.txt
```

The `docker-compose.yml` references `${SECRET_KEY_FILE:-./secrets/secret_key.txt}`.

---

## 4. Database Setup

### Option A: PostgreSQL (Production - Required for multi-user)

#### Using Docker Compose

```bash
cd D:\EMR
docker-compose up -d postgres
```

Wait for postgres to be healthy, then run migrations:

```bash
cd D:\EMR\backend
alembic upgrade head
python seed.py
```

#### Manual PostgreSQL

```bash
# Install PostgreSQL 16+
# Create database and user
# Update DATABASE_URL in .env
cd D:\EMR\backend
alembic upgrade head
python seed.py
```

### Option B: SQLite (Single-clinic pilot only)

SQLite works for a single-clinic pilot with low concurrency. Foreign keys are enforced via SQLAlchemy event listeners.

```bash
cd D:\EMR\backend
alembic upgrade head
python seed.py
```

**Do not migrate or delete the existing database without a backup.**

---

## 5. Docker Setup

### Step 1: Create secrets directory

```bash
cd D:\EMR
mkdir -p secrets
echo -n "<your-secret-key>" > secrets/secret_key.txt
```

### Step 2: Create backend `.env` for Docker

Copy `.env.production.example` to `.env` and fill in all values.

### Step 3: Build and start

```bash
cd D:\EMR
docker-compose up -d --build
```

### Step 4: Verify health

```bash
curl http://localhost:8000/health
# Returns: {"status": "ok", "database": "ok"}

curl http://localhost/
# Returns: frontend HTML
```

### Docker Compose Services

| Service | Port | Purpose |
|---------|------|---------|
| postgres | 5432 | PostgreSQL database |
| api | 8000 | Backend FastAPI |
| frontend | 80 | Frontend (served by nginx) |

### Health Checks

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Backend + database status |
| `GET /` | Frontend availability |

---

## 6. Backend Startup

### Using Docker Compose

```bash
cd D:\EMR
docker-compose up -d api
```

### Manual Startup

```bash
cd D:\EMR\backend
python -m venv .venv
.venv\Scripts\activate  # Windows
# source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

### Production ASGI Configuration

- Server: uvicorn with 4 workers
- Host: 0.0.0.0
- Port: 8000
- No `--reload` flag (production)
- Graceful shutdown via FastAPI lifespan handler

---

## 7. Frontend Startup

### Using Docker Compose

```bash
cd D:\EMR
docker-compose up -d frontend
```

### Manual Startup

```bash
cd D:\EMR\frontend
npm install
npm run build
# Serve dist/ with nginx or similar
```

### Development Mode

```bash
cd D:\EMR\frontend
npm run dev
```

---

## 8. HTTPS Requirements

HTTPS must be configured at the reverse proxy level (nginx, ALB, Cloudflare, etc.).

### Reverse Proxy Configuration (nginx example)

```nginx
server {
    listen 443 ssl;
    server_name emr.yourclinic.com;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

**Do not expose the application publicly without HTTPS.**

---

## 9. Backup Procedures

### SQLite Backups

```bash
# Manual backup
cd D:\EMR\backend
python backup_db.py emr.db backups

# Automated daily backup (cron example)
0 2 * * * /path/to/python /path/to/backend/backup_db.py /path/to/emr.db /path/to/backups
```

### PostgreSQL Backups

```bash
# Manual backup
cd D:\EMR\backend
DATABASE_URL=postgresql+psycopg2://user:pass@host:5432/db \
  python backup_db.py backup-postgres backups

# Docker Compose backup
docker-compose exec api python backup_db.py backup-postgres /app/backups
```

### Backup Retention

- Default: keeps last 30 backups
- Oldest backups are deleted automatically when limit is reached
- Configure `max_backups` parameter as needed

### Backup Verification

- SQLite: `PRAGMA integrity_check` on every backup
- PostgreSQL: verifies backup file is non-empty

---

## 10. Recovery Procedures

### SQLite Restore

```bash
# Stop the application
# Restore from backup
cd D:\EMR\backend
python backup_db.py restore-sqlite backups/emr.db.YYYYMMDD-HHMMSS.bak emr.db
# Start the application
```

### PostgreSQL Restore

```bash
# Stop the application
# Restore from backup
cd D:\EMR\backend
DATABASE_URL=postgresql+psycopg2://user:pass@host:5432/db \
  python backup_db.py restore-postgres backups/emr.db.YYYYMMDD-HHMMSS.sql
# Start the application
```

### Docker Compose Restore

```bash
# Copy backup into the api container
docker cp backups/emr.db.YYYYMMDD-HHMMSS.bak emr-api:/app/
docker-compose exec api python backup_db.py restore-sqlite /app/emr.db.YYYYMMDD-HHMMSS.bak /app/emr.db
```

**Never delete existing backups.**

---

## 11. Rollback Instructions

1. Stop the application:
   ```bash
   docker-compose down
   ```

2. Restore the database from the most recent backup:
   ```bash
   # SQLite
   cp backups/emr.db.YYYYMMDD-HHMMSS.bak emr.db

   # PostgreSQL
   DATABASE_URL=... python backup_db.py restore-postgres backups/emr.db.YYYYMMDD-HHMMSS.sql
   ```

3. Redeploy the previous version of the application

4. Verify health endpoint:
   ```bash
   curl http://localhost:8000/health
   ```

---

## 12. Troubleshooting

| Issue | Solution |
|-------|----------|
| Backend fails to start | Check SECRET_KEY is 32+ chars and not a placeholder |
| CORS errors | Ensure CORS_ORIGINS includes your frontend domain, no wildcards |
| Database connection fails | Verify DATABASE_URL and that PostgreSQL is running |
| Health check returns degraded | Database connectivity issue; check DATABASE_URL and credentials |
| Frontend shows blank page | Verify VITE_API_URL points to correct backend |
| SMTP emails not sent | Verify SMTP_HOST, SMTP_USER, SMTP_PASS, SMTP_USE_TLS |
| Container health check failing | Check container logs: `docker-compose logs api` |
| Backup fails | Verify database file path and write permissions to backup dir |

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
docker-compose logs -f frontend
docker-compose logs -f postgres
```

### Common Commands

```bash
# Stop all services
docker-compose down

# Restart a service
docker-compose restart api

# View running containers
docker-compose ps

# Execute command in container
docker-compose exec api bash
```

---

## Security Requirements

- Never commit `.env` to version control
- Never print or expose SMTP credentials
- Never expose SECRET_KEY in logs or error messages
- Never allow wildcard `*` in production CORS_ORIGINS
- Do not migrate or delete the production database without a backup
- Do not deploy publicly without approval
- Configure HTTPS at the reverse proxy level before public exposure
- Change default admin password after first login
