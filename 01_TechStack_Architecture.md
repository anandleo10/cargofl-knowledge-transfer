# Tech Stack & System Architecture

## Technology Stack

### Backend

| Layer | Technology | Version | Role |
|-------|-----------|---------|------|
| **Web Framework** | FastAPI | 0.109.0 | REST API, async request handling, startup/shutdown lifecycle |
| **ASGI Server** | Uvicorn | 0.41.0 | Production ASGI server (no `--reload` in prod) |
| **Workflow Engine** | LangGraph | 1.0.9 | Stateful multi-step AI workflow as a directed graph |
| **LLM Orchestration** | CrewAI | 1.14.2 | Optional multi-agent crews (enabled via `RA_USE_CREWS=1`) |
| **LLM Provider** | OpenAI GPT-4o | API | All natural language → SQL, summaries, dashboards, Q&A |
| **LLM Client** | openai SDK | 2.21.0 | API calls to OpenAI |
| **DB ORM** | SQLAlchemy | 2.0.25 | Thread-safe connection pool, SQL execution |
| **DB Driver** | PyMySQL | 1.1.0 | MySQL connector |
| **Data Processing** | Pandas | 2.2.0 | DataFrame operations on query results |
| **Scheduler** | APScheduler | 3.11.2 | AsyncIOScheduler — 6 recurring Maria jobs |
| **Vector DB** | Chroma | 1.1.1 | Schema/semantic RAG index |
| **Cache** | Redis | 5.0.1 | Session cache, optional rate limiting |
| **Config** | Pydantic Settings | 2.10.1 | Type-safe `.env` loading |
| **Email (out)** | smtplib (stdlib) | — | Gmail SMTP, HTML email construction |
| **Email (in)** | imaplib (stdlib) | — | Gmail IMAP polling for Q&A questions |
| **Charting (server)** | Plotly + Kaleido | 5.18.0 | Server-side chart image generation |
| **Excel** | openpyxl + xlsxwriter | — | Report export |
| **SQL Safety** | SQLGlot + RestrictedPython | — | Query validation, safe code execution |

### Frontend

| Layer | Technology | Version | Role |
|-------|-----------|---------|------|
| **UI Framework** | React | 18 | Component-based UI |
| **Language** | TypeScript | 5.x | Type safety |
| **Build Tool** | Vite | 7.3.1 | Fast dev server + production bundler |
| **Styling** | Tailwind CSS | 3.x | Utility-first CSS |
| **Components** | shadcn/ui | — | Pre-built accessible component library |
| **Charts** | Recharts | — | Declarative charts from backend JSON specs |
| **HTTP Client** | fetch (native) | — | Type-safe wrapper in `services/api.ts` |

### Infrastructure

| Layer | Technology | Role |
|-------|-----------|------|
| **Containerisation** | Docker + Docker Compose | Three-container stack |
| **Reverse Proxy** | Nginx (alpine) | Serves React build, proxies `/api/*` to backend |
| **Database** | MySQL on AWS RDS | `cargofl` database, `tlbooking` table (~573K rows) |
| **Cache Store** | Redis 7 (alpine) | In-process cache backend |
| **Hosting** | Linux VM | `ai-research.cargofl.com` |

---

## Why Each Technology Was Chosen

| Technology | Reason |
|-----------|--------|
| **FastAPI** | Async-native, auto OpenAPI docs, Pydantic validation, fast startup |
| **LangGraph** | Handles complex multi-step AI pipelines with state, retries, and branching |
| **CrewAI** | Structured multi-agent orchestration with YAML-defined roles and tasks |
| **GPT-4o** | Best-in-class reasoning for SQL generation and logistics narrative |
| **MySQL/RDS** | Existing CargoFL production database — no migration needed |
| **APScheduler** | Lightweight async-compatible scheduler, no broker dependency |
| **Gmail IMAP** | Free, reliable — Maria's mailbox is an existing Gmail account |
| **Recharts** | Declarative, works well with backend-generated JSON specs |
| **Docker Compose** | Simple single-node deployment; all dependencies containerised |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET / USER                                    │
└─────────────────────────────────────┬───────────────────────────────────────────┘
                                      │ HTTPS
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          ai-research.cargofl.com                                │
│                                                                                 │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────────┐   │
│  │  FRONTEND CONTAINER         │    │  API CONTAINER                       │   │
│  │  nginx:1.27-alpine          │    │  Python 3.11 + Uvicorn               │   │
│  │  Port: 80                   │    │  Port: 8000                          │   │
│  │                             │    │                                      │   │
│  │  React + TypeScript (Vite)  │───▶│  FastAPI (main_complete.py)          │   │
│  │  Recharts, shadcn/ui        │    │  ├─ Research routes (/api/research)  │   │
│  │  Tailwind CSS               │    │  ├─ Dashboard routes (/api/dashboard)│   │
│  │  4 Tabs:                    │    │  ├─ CT routes (/api/control-tower)   │   │
│  │  • Analysis                 │    │  ├─ Maria routes (/api/maria)        │   │
│  │  • Control Tower            │    │  ├─ Setup routes (/api/setup)        │   │
│  │  • Activity                 │    │  └─ Health (/health)                 │   │
│  │  • Configure                │    │                                      │   │
│  └─────────────────────────────┘    │  LangGraph Workflow Engine           │   │
│                                     │  ├─ Guardrail Agent                  │   │
│  ┌─────────────────────────────┐    │  ├─ Clarification Agent              │   │
│  │  REDIS CONTAINER            │    │  ├─ Planning Agent                   │   │
│  │  redis:7-alpine             │◀───│  ├─ Execution Agent                  │   │
│  │  (internal port 6379)       │    │  ├─ Visualization Agent              │   │
│  │  • Session cache            │    │  ├─ Dashboard Design Agent           │   │
│  │  • Rate limiting            │    │  ├─ Summarizer Agent                 │   │
│  └─────────────────────────────┘    │  └─ Report Generator                 │   │
│                                     │                                      │   │
│                                     │  Maria Agent (APScheduler)           │   │
│                                     │  ├─ Morning Brief (cron 8AM)         │   │
│                                     │  ├─ Weekly Digest (cron Mon 7AM)     │   │
│                                     │  ├─ Anomaly Check (every 2h)         │   │
│                                     │  ├─ Alert Digest (cron 5PM)          │   │
│                                     │  ├─ IMAP Poll (every 5min)           │   │
│                                     │  └─ CT Refresh (cron 11PM)           │   │
│                                     └────────────────┬─────────────────────┘   │
│                                                      │                         │
└──────────────────────────────────────────────────────┼─────────────────────────┘
                                                       │
              ┌────────────────────────────────────────┼────────────────────────┐
              │                                        │                        │
              ▼                                        ▼                        ▼
┌─────────────────────┐              ┌─────────────────────┐     ┌────────────────────┐
│  AWS RDS MySQL      │              │  Gmail SMTP         │     │  OpenAI API        │
│  cargofl.tlbooking  │              │  maria.puma@        │     │  GPT-4o            │
│  ~573K rows         │              │  cargofl.com        │     │  LLM calls:        │
│                     │              │  (outbound email)   │     │  • SQL planning    │
│  Read via:          │              │                     │     │  • Summarization   │
│  SQLAlchemy pool    │              │  Gmail IMAP         │     │  • Q&A             │
│  (thread-safe)      │              │  (inbound Q&A)      │     │  • Dashboard design│
└─────────────────────┘              └─────────────────────┘     └────────────────────┘
```

---

## How the Components Interact

### Request Path (User Query)
```
Browser → Nginx (port 80) → FastAPI (port 8000) → LangGraph workflow
  → Guardrail → Planning (OpenAI) → Execution (MySQL) → Visualization
  → Dashboard Design (OpenAI) → Summarizer (OpenAI) → Response JSON
  → Browser (renders Recharts from JSON spec)
```

### Maria Proactive Path
```
APScheduler (server memory)
  → MariaBrain methods
    → _fetch_ct_df() (MySQL via view_cache)
    → MariaAnalyst (OpenAI GPT-4o for narrative)
    → send_email() (Gmail SMTP)
    → activity_store (audit log to data/maria_activity.json)
```

### Maria Q&A Path
```
Gmail inbox (IMAP poll every 5 min)
  → imap_listener._fetch_unseen_from_allowed()
  → answer_question()
    → IF alert keyword → _answer_from_alert_state() directly
    → ELSE → MariaAnalyst.answer() (Planner→Executor→Narrator over CT DataFrames)
    → send_email() reply-in-thread (In-Reply-To + References headers)
```

---

## Key Architectural Decisions

### 1. Single Python Process, No Worker Queue
All LangGraph workflows and Maria jobs run in the same Uvicorn process using Python `asyncio`. Heavy DB queries run in `asyncio.to_thread()` to avoid blocking the event loop. This simplifies deployment (no Celery/RabbitMQ) but means one server process handles everything.

### 2. No `--reload` in Production
`start.sh` runs `uvicorn` without `--reload`. This is critical — `--reload` spawns a second process, causing APScheduler to register duplicate jobs, leading to double emails and double alert fires.

### 3. LangGraph for Workflow, Not a Simple Chain
LangGraph's StateGraph allows conditional branching (e.g., skip clarification if query is clear, retry execution if SQL fails) and persistent state across steps. This is more reliable than a linear prompt chain.

### 4. CrewAI Optional (`RA_USE_CREWS=1`)
When `RA_USE_CREWS=1`, each agent (planner, executor, etc.) runs as a CrewAI crew with YAML-defined agent roles and tasks. When off, they use direct OpenAI SDK calls. The interface is identical — only the internal orchestration differs.

### 5. view_cache for Control Tower Performance
CT views query up to 573K rows. To avoid hitting MySQL on every page load, query results are cached in a Python dict (`view_cache["v1"...]`). The cache is invalidated by:
- Manual "Reload from DB" button (frontend → `POST /api/dashboard/refresh-cache/{view}`)
- Nightly CT refresh job at 11 PM (Maria scheduler)

### 6. `_PROJECT_ROOT` for Path Safety
All data file paths (`data/maria_config.json`, `data/dashboards/`, etc.) use `Path(__file__).resolve().parents[N]` as an anchor. This ensures paths work regardless of the working directory from which uvicorn is started.

### 7. Alert State in Memory (`_alert_state`)
Maria's active alert state (OTD breach, delay spike, overdue count) lives in `MariaBrain._alert_state` (a Python dict in server memory). It is rebuilt on each `check_anomalies()` run. It is NOT persisted to disk because anomaly checks run every 2 hours, so state is quickly refreshed after a restart.

---

## Data Storage Summary

| Data | Where Stored | Format | Persists Restart? |
|------|-------------|--------|-------------------|
| Shipment data | MySQL RDS (`tlbooking`) | Relational | Yes |
| CT view DataFrames | `view_cache` dict | Python in-memory | No (rebuilt on demand) |
| Analysis results | `active_sessions` dict | Python in-memory | No |
| Run history | `data/run_history.json` | JSON | Yes |
| Saved dashboards | `data/dashboards/*.json` | JSON | Yes |
| Maria config | `data/maria_config.json` | JSON | Yes |
| Maria subscriptions | `data/maria_subscriptions.json` | JSON | Yes |
| Allowed senders | `data/maria_allowed_senders.json` | JSON | Yes |
| Activity log | `data/maria_activity.json` | JSON | Yes |
| Active alerts | `MariaBrain._alert_state` | Python dict | No |
| Last alert fired | `data/maria_last_alert_fired.json` | JSON | Yes |
| Schema RAG index | `data/chroma_db/` | Chroma vector store | Yes |
