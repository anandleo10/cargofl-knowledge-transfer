# Deployment & Operations

## Infrastructure Overview

```
Production Server: ai-research.cargofl.com (Linux VM)

Docker Compose Stack:
  ┌─────────────────────────────────────────────────┐
  │  Host (Linux VM)                                │
  │                                                 │
  │  Port 80  → frontend container (nginx)          │
  │  Port 8000 → api container (uvicorn)            │
  │                                                 │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────┐ │
  │  │  frontend    │  │   api        │  │ redis │ │
  │  │  nginx:1.27  │  │  python:3.11 │  │ :6379 │ │
  │  │  :80         │  │  :8000       │  │       │ │
  │  └──────────────┘  └──────────────┘  └───────┘ │
  │                                                 │
  │  Volume: ./data → /app/data (api container)     │
  └─────────────────────────────────────────────────┘
```

---

## Docker Compose (`docker-compose.yml`)

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    env_file:
      - .env
    volumes:
      - ./data:/app/data       # Persistent storage
    depends_on:
      - redis
    restart: unless-stopped

  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
      args:
        VITE_API_BASE: http://ai-research.cargofl.com:8000
    ports:
      - "80:80"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped
```

**Important settings:**
- `restart: unless-stopped` — containers restart automatically after server reboot
- `env_file: .env` — all secrets injected at container start
- `./data:/app/data` — dashboards, configs, logs survive deployments

---

## Dockerfiles

### `Dockerfile` (API — Multi-Stage)

```dockerfile
# Stage 1: Builder — install Python dependencies
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime — slim production image
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /install /usr/local    # copy only installed packages
COPY src/ ./src/
COPY table.yaml .
COPY start.sh .
RUN chmod +x start.sh
RUN mkdir -p data/uploads data/outputs data/chroma_db
EXPOSE 8000
CMD ["./start.sh"]
```

The two-stage build keeps the final image lean — only runtime artifacts are included, not build tools.

### `start.sh` (API Entrypoint)

```bash
#!/bin/bash
uvicorn src.api.main_complete:app \
  --host 0.0.0.0 \
  --port 8000 \
  --workers 1
# NO --reload flag!
```

**Critical:** Do NOT add `--reload`. It spawns a second process, causing APScheduler to register duplicate jobs, resulting in double morning briefs, double anomaly checks, and double alert emails.

### `Dockerfile.frontend`

```dockerfile
# Stage 1: Build React app
FROM node:20-slim AS build
WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
ARG VITE_API_BASE=http://localhost:8000
ENV VITE_API_BASE=$VITE_API_BASE
RUN npm run build

# Stage 2: Serve with nginx
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---

## Deployment Procedures

### Initial Deployment

```bash
# 1. SSH into the server
ssh user@ai-research.cargofl.com

# 2. Clone the repository
git clone https://github.com/innoctive/cargofl-research-agent.git
cd cargofl-research-agent

# 3. Create .env file with all required variables
cp .env.example .env
nano .env   # fill in all values

# 4. Create data directory structure
mkdir -p data/uploads data/outputs data/chroma_db data/dashboards

# 5. Build and start containers
docker compose up -d --build

# 6. Check startup logs
docker logs research-agent-complete-api-1 --tail=50 -f

# 7. Verify health
curl http://localhost:8000/health
curl http://localhost:80/
```

### Routine Deployment (Code Update)

```bash
# 1. Pull latest code
cd /path/to/research-agent-complete
git pull origin main

# 2. Rebuild and restart (zero-downtime: stops old, starts new)
docker compose up -d --build

# 3. Verify
docker compose ps
curl http://localhost:8000/health
```

**Note:** `./data` is a mounted volume, so all saved dashboards, Maria config, and subscriptions survive the redeploy.

### Rebuild Only One Service

```bash
# Rebuild only the API (e.g., after Python code change)
docker compose up -d --build api

# Rebuild only the frontend (e.g., after React code change)
docker compose up -d --build frontend
```

---

## Monitoring & Logs

### Container Logs

```bash
# All containers (stream)
docker compose logs -f

# API only (last 100 lines)
docker logs research-agent-complete-api-1 --tail=100

# Frontend / nginx
docker logs research-agent-complete-frontend-1 --tail=50

# Redis
docker logs research-agent-complete-redis-1 --tail=20
```

### Health Check

```bash
# API health
curl http://localhost:8000/health
# → {"status":"healthy","version":"1.0.0","openai_configured":true}

# Maria status (all 6 jobs)
curl http://localhost:8000/api/maria/status

# Cache status
curl http://localhost:8000/api/ct/cache-status
```

### Application Logs

The API container logs to stdout, captured by Docker. Key log patterns:

| Log Pattern | Meaning |
|-------------|---------|
| `[Maria] Brain initialized` | Maria started successfully |
| `[Maria/scheduler] Started 6 jobs` | All jobs registered |
| `[Maria] Generating morning brief…` | Brief starting |
| `[Maria/imap] N total unseen message(s)` | IMAP poll result |
| `[Maria] Anomaly check passed` | All KPIs OK |
| `[Maria] New alert 'OTD_WARNING': 62.4` | Alert detected |
| `[Maria] IMAP error: …` | IMAP connection failed |
| `[Maria] send_system_alert: …` | Ops alert sent |
| `INFO: … 200 OK` | Successful HTTP request |
| `ERROR: …` | Application error |

---

## Common Operations

### Restart the API Only

```bash
docker compose restart api
```

### Full Stack Restart

```bash
docker compose down
docker compose up -d
```

### View Maria Activity from CLI

```bash
curl http://localhost:8000/api/maria/activity | python -m json.tool | head -100
```

### Manually Trigger Morning Brief

```bash
curl -X POST http://localhost:8000/api/maria/brief
```

### Force Refresh CT Cache (All Views)

```bash
for view in v1 v2 v3 v4 v5; do
  curl -X POST "http://localhost:8000/api/dashboard/refresh-cache/$view"
  echo "Refreshed $view"
done
```

### Backup Data

```bash
# Backup the entire data directory
tar -czf backup_$(date +%Y%m%d).tar.gz ./data/
```

### Restore Data

```bash
tar -xzf backup_20260420.tar.gz
docker compose restart api
```

---

## Environment Variables — Required for Startup

These MUST be set before the server starts:

| Variable | Required | Notes |
|----------|----------|-------|
| `OPENAI_API_KEY` | Yes | sk-proj-... |
| `DATABASE_URL` | Yes | mysql+pymysql://... |
| `DATABASE_HOSTNAME` | Yes | RDS endpoint |
| `DATABASE_USERNAME` | Yes | DB user |
| `DATABASE_PASSWORD` | Yes | DB password |
| `DATABASE_PORT` | Yes | 3306 for MySQL |
| `DATABASE_NAME` | Yes | cargofl |
| `SMTP_HOST` | Yes (for Maria) | smtp.gmail.com |
| `SMTP_PORT` | Yes (for Maria) | 587 |
| `SMTP_USER` | Yes (for Maria) | maria.puma@cargofl.com |
| `SMTP_PASSWORD` | Yes (for Maria) | Gmail App Password |
| `IMAP_USER` | Yes (for Maria) | maria.puma@cargofl.com |
| `IMAP_PASSWORD` | Yes (for Maria) | Gmail App Password |
| `SENIOR_MANAGER_EMAIL` | Yes (for Maria) | Primary recipient |
| `MARIA_ENABLED` | Yes (for Maria) | Set to `true` |
| `SECRET_KEY` | Yes (production) | Random secret for auth |

**Gmail App Password Setup:**
The Gmail account (`maria.puma@cargofl.com`) must have:
1. 2-Factor Authentication enabled
2. An App Password generated at Google Account → Security → App Passwords
3. The App Password (format: `xxxx xxxx xxxx xxxx`) used for both SMTP and IMAP

---

## Known Operational Gotchas

### 1. No `--reload` Flag
**Problem:** `--reload` causes duplicate Maria jobs (double emails, double alerts).
**Solution:** `start.sh` never includes `--reload`. Only use `--reload` locally for development, never in production or Docker.

### 2. Cold Cache After Restart
**Problem:** After a server restart, `view_cache` is empty. The first CT request hits MySQL (can be slow for 573K rows).
**Solution:** Normal — expected behaviour. CT views self-populate on first use. The nightly refresh job at 11 PM ensures the cache is warm each morning.

### 3. `.env` Not Picked Up
**Problem:** Changes to `.env` don't take effect without restarting containers.
**Solution:** `docker compose down && docker compose up -d` after `.env` changes.

### 4. `data/` Directory Not Created
**Problem:** If `data/` subdirectories don't exist, Maria fails to write config/activity files.
**Solution:** Ensure `data/uploads`, `data/outputs`, `data/chroma_db`, `data/dashboards` exist before first start.

### 5. Maria Not Starting
**Symptoms:** No morning brief received, `/api/maria/status` shows 0 jobs
**Check:**
```bash
docker logs research-agent-complete-api-1 | grep -i "maria\|scheduler"
```
**Common causes:**
- `MARIA_ENABLED=false` in `.env`
- IMAP/SMTP credentials wrong (Maria still starts but logs errors)
- Exception in `startup_event()` — check full startup log

### 6. Alert Emails Not Sending
**Check:**
1. `GET /api/maria/ready` — checks SMTP connectivity
2. `POST /api/maria/demo-email` — sends a test email
3. Check `data/maria_last_alert_fired.json` — cooldown may be active
4. Check alert thresholds in `data/maria_config.json` — may need calibration

### 7. Q&A Emails Not Getting Replies
**Check:**
1. Sender must be in `AllowedSendersStore` (check `data/maria_allowed_senders.json`)
2. `GET /api/maria/status` — verify `imap_poll` job is registered
3. Check IMAP password — Gmail App Passwords can expire
4. Check logs for `[Maria/imap]` entries during the 5-minute poll window

---

## Scaling Considerations

The current architecture is single-node (one Docker Compose stack). For scaling:

| Concern | Current | If Needed |
|---------|---------|-----------|
| Concurrent users | ~5–10 (async, fine) | Add `--workers N` to uvicorn |
| Heavy DB queries | Single connection pool | Increase pool size in `_get_or_create_engine()` |
| Maria jobs | In-process scheduler | Move to Celery + Redis for multi-worker |
| Storage | Local `./data` volume | Move to S3 or shared NFS |
| Multiple regions | Single server | Would need state synchronisation |

For the current use case (one ops team, ~573K rows, 6 scheduled jobs), single-node is appropriate and simpler to operate.
