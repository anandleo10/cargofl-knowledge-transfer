# Configuration Guide

This document covers every configuration variable and file, what each controls, when to change it, and how to apply changes.

---

## Configuration Files Overview

| File | Location | Edit Via | Restart Required? |
|------|----------|---------|-------------------|
| `.env` | Root | Text editor | Yes — `docker compose down && up -d` |
| `data/maria_config.json` | data/ | UI Configure tab or text editor | No — read at runtime |
| `data/maria_subscriptions.json` | data/ | UI Configure tab or email command | No — read at runtime |
| `data/maria_allowed_senders.json` | data/ | UI Configure tab or email command | No — read at runtime |
| `table.yaml` | Root | Text editor | Yes + re-run `/api/setup/generate` |
| `frontend/.env.production` | frontend/ | Text editor | Yes — needs `npm run build` |

---

## `.env` — All Secrets & Config

### OpenAI

| Variable | Description | Example |
|----------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API key — required for all LLM calls | `sk-proj-...` |

### Database (AWS RDS MySQL)

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_HOSTNAME` | RDS endpoint hostname | `cargofl-puma-sync.c52ewqoqkh1q.ap-south-1.rds.amazonaws.com` |
| `DATABASE_NAME` | Database name | `cargofl` |
| `DATABASE_USERNAME` | DB user | `superset_readonly` |
| `DATABASE_PASSWORD` | DB password | `Ac8C085f2K1lLDF99` |
| `DATABASE_PORT` | Port | `3306` |
| `DATABASE_URL` | Full connection URL (auto-built or set manually) | `mysql+pymysql://${USER}:${PASS}@${HOST}:${PORT}/${DB}` |

> The `DATABASE_URL` variable is constructed from the individual parts. If you need to override (e.g. for PostgreSQL), set `DATABASE_URL` directly.

### Redis

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` (dev) / `redis://redis:6379/0` (Docker) |

> In Docker Compose, Redis is accessible as `redis:6379` (service name). Locally it's `localhost:6379`.

### Application Settings

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_ENV` | `development` or `production` | `development` |
| `LOG_LEVEL` | Python logging level | `INFO` |
| `MAX_UPLOAD_SIZE_MB` | Max Excel file upload size | `100` |
| `QUERY_TIMEOUT_SECONDS` | Max seconds for full analysis workflow | `300` |
| `SECRET_KEY` | JWT signing key — change in production! | `your-secret-key-change-in-production` |
| `ALLOWED_ORIGINS` | CORS allowed origins (JSON array) | Includes localhost + cargofl.com |

### Storage Paths

| Variable | Description | Default |
|----------|-------------|---------|
| `UPLOAD_DIR` | Where uploaded Excel files are stored | `./data/uploads` |
| `OUTPUT_DIR` | Where generated reports/exports go | `./data/outputs` |
| `CHROMA_DB_DIR` | Chroma vector DB path | `./data/chroma_db` |

> All path variables are relative to the working directory. In Docker, the container starts from `/app`, so these resolve to `/app/data/...`. The `_PROJECT_ROOT` anchor in code overrides these for Maria-specific paths.

### Rate Limiting & Execution Limits

| Variable | Description | Default |
|----------|-------------|---------|
| `RATE_LIMIT_PER_MINUTE` | Max API requests per minute per IP | `10` |
| `MAX_RESULT_ROWS` | Max rows returned from any SQL query | `100000` |
| `MAX_RETRY_ATTEMPTS` | SQL retry attempts on failure | `3` |
| `SQL_TIMEOUT_SECONDS` | Per-query SQL timeout | `30` |

### RAG Settings

| Variable | Description | Default |
|----------|-------------|---------|
| `EMBEDDING_MODEL` | OpenAI embedding model for schema RAG | `text-embedding-3-small` |
| `CHUNK_SIZE` | RAG chunk size (characters) | `1000` |
| `CHUNK_OVERLAP` | RAG chunk overlap (characters) | `200` |
| `TOP_K_CHUNKS` | Top K chunks returned per RAG query | `5` |

### Visualization

| Variable | Description | Default |
|----------|-------------|---------|
| `MAX_CHARTS_PER_REPORT` | Max charts in auto-generated report | `3` |
| `CHART_WIDTH` | PNG chart width (pixels) | `800` |
| `CHART_HEIGHT` | PNG chart height (pixels) | `500` |

### CrewAI

| Variable | Description | Default |
|----------|-------------|---------|
| `RA_USE_CREWS` | Use CrewAI agents (`1`) or direct OpenAI (`0`) | `1` |
| `LLM_TEMPERATURE` | LLM sampling temperature (0 = deterministic) | `0.0` |
| `CREW_VERBOSE` | Log detailed CrewAI execution | `True` |
| `CREWAI_TRACING_ENABLED` | Enable CrewAI telemetry | `True` |

### Email (Outbound SMTP — Gmail)

| Variable | Description | Example |
|----------|-------------|---------|
| `SMTP_HOST` | SMTP server hostname | `smtp.gmail.com` |
| `SMTP_PORT` | SMTP port (TLS = 587, SSL = 465) | `587` |
| `SMTP_USER` | Gmail address to send from | `maria.puma@cargofl.com` |
| `SMTP_PASSWORD` | Gmail App Password (16-char) | `thxm uibc lprd eahs` |
| `EMAIL_FROM` | From address in email header | `maria.puma@cargofl.com` |
| `EMAIL_FROM_NAME` | Display name in From header | `CargoFL Reporting Copilot` |

### Email (Inbound IMAP — Gmail)

| Variable | Description | Example |
|----------|-------------|---------|
| `IMAP_HOST` | IMAP server | `imap.gmail.com` |
| `IMAP_PORT` | IMAP SSL port | `993` |
| `IMAP_USER` | Gmail address to poll | `maria.puma@cargofl.com` |
| `IMAP_PASSWORD` | Gmail App Password | `thxm uibc lprd eahs` |
| `IMAP_FOLDER` | Mailbox folder to watch | `INBOX` |
| `IMAP_POLL_INTERVAL_SECONDS` | How often to poll (seconds) | `300` (5 min) |

### Operations Team

| Variable | Description | Example |
|----------|-------------|---------|
| `OPS_EMAIL` | Operations team email | `malini.menon.ext@puma.com` |
| `OPS_ESCALATION_EMAIL` | Escalation email | `malini.menon.ext@puma.com` |
| `SENIOR_MANAGER_EMAIL` | Primary morning brief recipient | `malini.menon.ext@puma.com` |

### Maria Agent

| Variable | Description | Default |
|----------|-------------|---------|
| `MARIA_ENABLED` | Master switch — `true` to activate | `false` |
| `MARIA_DASHBOARD_URL` | URL shown in email links | `https://ai-research.cargofl.com` |
| `MARIA_ALLOWED_SENDERS` | Comma-separated Q&A whitelist | `anand@innoctive.com` |
| `MARIA_CC_EMAILS` | Static CC list for broadcasts | `anand@innoctive.com,deepesh@cargofl.com` |
| `MARIA_SUPPORT_EMAIL` | Ops alert TO address | `support@innoctive.com` |
| `MARIA_SUPPORT_CC` | Ops alert CC (comma-separated) | `mubin@innoctive.com,deepesh@innoctive.com` |
| `MARIA_ALERT_COOLDOWN_MINUTES` | Min minutes between same-type ops alerts | `30` |
| `MARIA_MORNING_BRIEF_HOUR` | Hour for morning brief (0–23 IST) | `8` |
| `MARIA_WEEKLY_DIGEST_WEEKDAY` | Day for weekly digest (0=Mon, 6=Sun) | `0` |
| `MARIA_ANOMALY_CHECK_INTERVAL_MINUTES` | Minutes between anomaly checks | `120` |
| `MARIA_OTD_ALERT_THRESHOLD` | OTD warning threshold (%) | `65.0` |
| `MARIA_CRITICAL_OTD_THRESHOLD` | OTD critical threshold (%) | `55.0` |
| `MARIA_DELAY_ALERT_HOURS` | Delay warning threshold (hours) | `72.0` |
| `MARIA_CRITICAL_DELAY_HOURS` | Delay critical threshold (hours) | `168.0` |
| `MARIA_OVERDUE_COUNT_THRESHOLD` | Overdue count warning threshold | `3500` |
| `MARIA_CRITICAL_OVERDUE_COUNT` | Overdue count critical threshold | `5000` |

---

## `data/maria_config.json` — Runtime Config

This file is read by the scheduler at startup and when config changes are made via the UI. Changes here take effect immediately for threshold-based checks; schedule changes require `MariaScheduler.reschedule()` (called automatically by the UI).

```json
{
  "schedule": {
    "morning_brief_hour": 8,
    "morning_brief_days": "mon-fri",
    "weekly_digest_weekday": 0,
    "anomaly_check_interval_minutes": 120,
    "alert_digest_hour": 17,
    "ct_refresh_enabled": true,
    "ct_refresh_hour": 23
  },
  "thresholds": {
    "otd_warning": 65.0,
    "otd_critical": 55.0,
    "delay_warning_hours": 72.0,
    "delay_critical_hours": 168.0,
    "overdue_count_warning": 3500,
    "overdue_count_critical": 5000
  },
  "digest_content": {
    "morning_brief": {
      "time_filter": {
        "preset": "last_7_days"
      }
    }
  }
}
```

### When to Edit Directly

- To temporarily disable CT refresh: set `"ct_refresh_enabled": false`
- To change morning brief time without UI: change `"morning_brief_hour"` and restart server
- To silence an alert: raise its threshold above the current metric value

---

## `data/maria_subscriptions.json` — Email Recipients

```json
{
  "to": [],
  "cc": []
}
```

- `to` — additional TO recipients (beyond `SENIOR_MANAGER_EMAIL`)
- `cc` — additional CC recipients (beyond `MARIA_CC_EMAILS` in `.env`)

**Adding via email:** Send email to Maria saying "add john@example.com to CC"
**Adding via UI:** Configure tab → Recipients → Add
**Adding via API:** `POST /api/maria/subscriptions/add`

---

## `data/maria_allowed_senders.json` — Q&A Whitelist

```json
["anand@innoctive.com"]
```

A JSON array of email addresses. Only emails from these addresses (plus `SENIOR_MANAGER_EMAIL`) trigger Q&A responses. All other emails are silently skipped.

> `SENIOR_MANAGER_EMAIL` and `MARIA_ALLOWED_SENDERS` from `.env` are always included, regardless of this file. This file stores dynamically-added senders only.

---

## `table.yaml` — Schema Definitions

The source of truth for what tables exist, what each column means, and how they relate. Used to build:
- `data_context.json` (LLM prompt context)
- Chroma RAG index (schema search)
- Data profiling reference

**After editing `table.yaml`:** Run `POST /api/setup/generate` to rebuild `data_context.json` and the RAG index.

**Structure:**
```yaml
tables:
  - name: tlbooking
    description: "Main logistics booking table..."
    primary_key: bookingid
    time_column: bookingdate
    columns:
      - name: bookingid
        type: VARCHAR
        description: "Unique booking identifier"
        example: "BK202401001"
      - name: truckername
        type: VARCHAR
        description: "Transporter/LSP company name"
        example: "DHL Express"
      # ... more columns
    indexes:
      - columns: [bookingdate, branch]
        description: "Primary date+branch filter index"
    bundles:
      - name: delivery_performance
        columns: [schdeliverydate, actualdeliverydate, lsp_status, pod_lsp_status]
        description: "Columns related to delivery timing and status"
```

---

## `frontend/.env.production`

Controls the API URL in the production frontend build.

```
VITE_API_BASE=http://ai-research.cargofl.com:8000
```

**How it's applied:**
- This file is read by Vite at build time (inside `Dockerfile.frontend`)
- It becomes a build argument: `docker build --build-arg VITE_API_BASE=...`
- The compiled JavaScript contains this URL hardcoded

**If the API moves to a new URL:**
1. Update `VITE_API_BASE` in `frontend/.env.production`
2. Rebuild frontend: `docker compose up -d --build frontend`

---

## Changing Configuration

### Change Morning Brief Time
**Option A (No restart):** Configure tab → Email Schedule → Morning Brief Hour
**Option B (Direct):** Edit `data/maria_config.json` → `schedule.morning_brief_hour` → no restart, takes effect at next scheduler run
**Option C (.env):** Change `MARIA_MORNING_BRIEF_HOUR` → requires `docker compose restart api`

### Change Alert Thresholds
**Option A (Immediate):** Configure tab → Alert Thresholds
**Option B (Direct):** Edit `data/maria_config.json` → `thresholds.*` → takes effect on next anomaly check

### Add/Change OpenAI Key
1. Edit `.env` → `OPENAI_API_KEY=sk-proj-...`
2. `docker compose restart api`
3. Verify: `curl http://localhost:8000/health` → `"openai_configured": true`

### Change DB Password
1. Edit `.env` → `DATABASE_PASSWORD=newpassword` and `DATABASE_URL`
2. `docker compose restart api`
3. Verify: `curl http://localhost:8000/api/setup/status`

### Rotate Gmail App Password
1. Go to Google Account → Security → App Passwords
2. Delete old password, generate new one
3. Update `.env` → `SMTP_PASSWORD` and `IMAP_PASSWORD`
4. `docker compose restart api`
5. Verify: `POST http://localhost:8000/api/maria/brief` → test send

---

## Security Checklist (Production)

- [ ] `SECRET_KEY` changed from default to a random 32+ character string
- [ ] `.env` not committed to git (check `.gitignore`)
- [ ] `DATABASE_USERNAME` uses read-only credentials (`superset_readonly`)
- [ ] SMTP/IMAP using App Password (not Gmail account password)
- [ ] `MARIA_ENABLED=true` only on the intended production server
- [ ] `ALLOWED_ORIGINS` does not include `*`
- [ ] `APP_ENV=production`
- [ ] `data/` directory has restricted permissions (`chmod 700 data/`)
