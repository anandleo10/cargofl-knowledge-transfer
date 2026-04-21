# Folder Structure — Annotated Reference

## Root Level

```
research-agent-complete/
│
├── .env                        ← ALL secrets and runtime config. Never commit.
├── .gitignore
├── docker-compose.yml          ← Defines 3 services: api, frontend, redis
├── Dockerfile                  ← Multi-stage build for Python API container
├── Dockerfile.frontend         ← Node build + nginx for React container
├── requirements.txt            ← Python dependencies (pinned versions)
├── start.sh                    ← Entrypoint: uvicorn (no --reload in prod!)
├── table.yaml                  ← Schema definitions — SOURCE OF TRUTH for DB tables
│
├── data/                       ← All runtime data (mounted as Docker volume)
├── src/                        ← All Python source code
├── frontend/                   ← All React/TypeScript source code
├── scripts/                    ← Utility and diagnostic scripts
├── logs/                       ← Runtime log files
├── examples/                   ← Example usage scripts
└── KnowledgeTransfer/          ← This documentation folder
```

---

## data/ — Runtime Data (Docker Volume)

This folder is mounted as a volume in Docker (`./data:/app/data`). Everything here persists across deployments unless explicitly deleted.

```
data/
├── dashboards/                 ← Saved dashboard configs
│   └── {uuid}.json             ← One file per saved dashboard (panel layout + query)
│
├── uploads/                    ← Excel files uploaded by users
│   └── {uuid}_{filename}.xlsx
│
├── outputs/                    ← Generated exports
│   ├── {request_id}_report.html
│   ├── {request_id}_export.xlsx
│   └── data_context.json       ← Schema context for LLM (auto-generated)
│
├── chroma_db/                  ← Chroma vector store (schema/semantic RAG index)
│   └── (Chroma internal files)
│
├── maria_config.json           ← Maria schedule + thresholds (editable via UI)
├── maria_subscriptions.json    ← Dynamic TO/CC recipients list
├── maria_allowed_senders.json  ← Whitelist of email senders for Q&A
├── maria_activity.json         ← Audit log: all Maria actions
├── maria_last_alert_fired.json ← Last fired timestamp per alert type (cooldown guard)
└── run_history.json            ← Last 100 analysis run records
```

**When you'd touch this folder:**
- Restore a backup of dashboards → copy `dashboards/*.json`
- Clear the schema index → delete `chroma_db/` and re-run `POST /api/setup/generate`
- Manually add a recipient → edit `maria_subscriptions.json`
- Reset alert cooldowns → delete `maria_last_alert_fired.json`

---

## src/ — Python Source

### src/api/

```
src/api/
├── main_complete.py            ← THE main file. 6500+ lines. All FastAPI routes,
│                                  startup/shutdown, CT views, view_cache, session store,
│                                  Maria wiring, global exception handler
└── main.py                     ← Legacy version (kept for reference, not used in prod)
```

`main_complete.py` is the heart of the backend. It contains:
- All `@app.get / @app.post / @app.put / @app.delete` route handlers
- `startup_event()` — initialises DB connection, starts MariaScheduler, registers CT refresh job
- `shutdown_event()` — gracefully stops MariaScheduler
- `view_cache` dict — in-memory CT DataFrame cache
- `active_sessions` (`_BoundedSessionStore`) — active analysis session state
- `_fetch_ct_df(view, force)` — async CT data fetcher with locking
- `_apply_ct_derived_columns(df, view)` — adds derived flags to CT DataFrames
- `_get_result_df(run_id, view)` — 4-step lookup: active_sessions → view_cache → SQLite → live DB
- `_MARIA_CONFIG_PATH / _MARIA_ALERT_STATE_PATH` — Path constants for data files
- Global `@app.exception_handler(Exception)` — fires `send_system_alert` for critical errors

---

### src/agents/

```
src/agents/
│
├── clarification_agent.py      ← Asks clarifying questions when a query is ambiguous
│                                  Input: raw query + schema context
│                                  Output: list of clarification Q&A pairs or None
│
├── planning_agent.py           ← Converts natural language → ExecutionPlan
│                                  Input: query + schema + clarifications
│                                  Output: ExecutionPlan (list of ExecutionStep objects)
│                                  Each step: sql, operation_type, expected_columns
│
├── execution_agent.py          ← Executes steps in an ExecutionPlan
│                                  Input: ExecutionPlan + DB engine
│                                  Output: dict of DataFrames keyed by step name
│                                  Handles: timeout, row limits, error recovery
│
├── guardrail_agent.py          ← Safety checks before any query runs
│                                  Blocks: PII queries, DROP/DELETE/UPDATE SQL,
│                                          queries about forbidden tables
│                                  Output: {safe: bool, reason: str}
│
├── dashboard_design_agent.py   ← Designs panel layout from query results
│                                  Input: DataFrames + original query
│                                  Output: DashboardConfig (panels with chart specs)
│
├── visualization_agent.py      ← Creates Recharts JSON specs and optional images
│                                  Input: DataFrame + panel config
│                                  Output: chart_spec JSON (rendered by Recharts)
│
├── summarizer_agent.py         ← Generates AI summaries of panels and dashboards
│                                  Input: panel data + original query
│                                  Output: [{panel_id, summary}] + {summary, key_findings}
│
├── report_generator.py         ← Renders HTML report from completed analysis
│                                  Input: full ResearchResponse
│                                  Output: HTML string saved to outputs/
│
├── nl_config_agent.py          ← Natural language schema editor
│                                  e.g. "rename column X to Y" → updates table.yaml
│
├── query_optimizer_agent.py    ← Optimizes a generated SQL query for performance
│
│
├── maria/                      ← Maria proactive agent (see 06_MariaAgent.md)
│   ├── maria_agent.py          ← MariaBrain — orchestrator class
│   ├── analyst.py              ← MariaAnalyst — planner/executor/narrator for Q&A
│   ├── scheduler.py            ← MariaScheduler — 6 APScheduler jobs
│   ├── imap_listener.py        ← IMAPListener — Gmail inbox polling
│   ├── email_service.py        ← send_email() + HTML template builders
│   ├── activity_store.py       ← ActivityStore — audit log persistence
│   └── __init__.py
│
└── prompts/                    ← CrewAI YAML definitions (used when RA_USE_CREWS=1)
    ├── clarification/
    │   ├── agents.yaml         ← Role definition for clarification agent
    │   └── tasks.yaml          ← Task definition
    ├── planning/
    ├── guardrail/
    ├── dashboard_design/
    ├── report_generator/
    ├── summarizer/
    ├── query_optimizer/
    └── nl_config/
```

---

### src/orchestration/

```
src/orchestration/
└── research_graph.py           ← LangGraph StateGraph definition
                                   - create_research_workflow() → CompiledGraph
                                   - _get_or_create_engine() → thread-safe SQLAlchemy engine
                                   - ResearchState TypedDict (shared across all nodes)
                                   - 8 graph nodes: guardrail, clarify, plan, execute,
                                     visualize, design, summarize, report
                                   - Conditional edges (e.g. skip clarify if not needed)
```

---

### src/services/

```
src/services/
├── dashboard_state_service.py  ← Save/load/list/delete dashboard JSON files
│                                  Persists to data/dashboards/{uuid}.json
│
├── data_slice_service.py       ← Safely filter a CT DataFrame for a dashboard panel
│                                  Always copies df before filtering (session-safe)
│                                  compute_slice(df, filters, group_by, sort) → DataFrame
│
├── run_history.py              ← RunHistory singleton — last 100 runs
│                                  RunRecord(id, query, status, started_at, finished_at)
│                                  Persisted to data/run_history.json
│
├── schema_rag_service.py       ← ChromaDB-backed schema search
│                                  Enables "which table has column X?" queries
│
├── semantic_rag_service.py     ← Semantic query enhancement via RAG
│
├── semantic_graph_service.py   ← Table relationship graph (JSON, editable via UI)
│
├── profiling_service.py        ← Data profiling: infer column types, value ranges
│
├── setup_profiler_service.py   ← Handles /api/setup/profile requests
│
├── setup_metadata_service.py   ← Manages schema metadata
│
└── schema_artifacts_builder.py ← Builds data_context.json and RAG index from schema
```

---

### src/handlers/

```
src/handlers/
├── schema_analyzer.py          ← DatabaseHandler: inspects MySQL schema, columns, FKs
│                                  ExcelHandler: inspects uploaded Excel files
└── document_rag.py             ← RAG over uploaded documents (PDFs, Word, etc.)
```

---

### src/models/

```
src/models/
├── data_context.py             ← DataContext Pydantic model (schema + stats snapshot)
└── setup_metadata.py           ← Setup metadata Pydantic models
```

---

### src/config/

```
src/config/
├── settings.py                 ← Pydantic BaseSettings — reads .env
│                                  All app config (DB, SMTP, IMAP, OpenAI, Maria thresholds)
│                                  Single import: from src.config.settings import settings
└── settings_rds.py             ← RDS-specific overrides
```

---

## frontend/ — React Source

```
frontend/
├── package.json                ← Dependencies: React 18, TypeScript, Vite, Recharts, shadcn
├── vite.config.ts              ← Vite build config (proxy /api → localhost:8000 in dev)
├── tsconfig.json               ← TypeScript config
├── tailwind.config.ts          ← Tailwind config
├── .env                        ← Dev env (VITE_API_BASE=http://localhost:8000)
├── .env.production             ← Prod env (VITE_API_BASE=http://ai-research.cargofl.com:8000)
│
└── src/
    ├── main.tsx                ← ReactDOM.createRoot() — app entry point
    ├── App.tsx                 ← Root component (loads AppImpl)
    ├── AppImpl.tsx             ← Main app: tab routing, global state, top-level layout
    │
    ├── services/
    │   └── api.ts              ← ALL API calls in one file. Type-safe fetch wrapper.
    │                              Exports functions: startResearch(), getStatus(),
    │                              createControlTower(), updateMariaConfig(), etc.
    │
    ├── lib/
    │   └── utils.ts            ← Utility helpers (cn() for Tailwind class merging, etc.)
    │
    └── components/
        ├── dashboard/
        │   ├── DashboardBuilder.tsx    ← Query input, submit, progress display
        │   ├── DashboardViewer.tsx     ← Renders a saved dashboard (panels grid)
        │   ├── DashboardPanel.tsx      ← Single panel: chart + title + summary
        │   ├── ChartRenderer.tsx       ← Routes chart_spec to correct Recharts component
        │   ├── DashboardInsight.tsx    ← AI insight generator for a panel
        │   ├── SavedDashboards.tsx     ← List + load saved dashboards
        │   └── index.ts               ← Re-exports
        │
        ├── maria/
        │   ├── MariaChatWidget.tsx     ← Chat Q&A interface (POST /api/maria/chat)
        │   └── MariaActivityTab.tsx    ← Activity log viewer (GET /api/maria/activity)
        │
        ├── admin/
        │   └── MariaConfigTab.tsx      ← Schedule, thresholds, recipients config editor
        │
        └── ui/                         ← shadcn/ui primitives
            ├── badge.tsx
            ├── button.tsx
            ├── card.tsx
            ├── dialog.tsx
            ├── input.tsx
            ├── select.tsx
            ├── switch.tsx
            ├── tabs.tsx
            └── tooltip.tsx
```

---

## scripts/ — Utilities

```
scripts/
├── build_data_dictionary.py    ← Generates human-readable data dictionary from DB
├── build_inferred_fk_graph.py  ← Infers foreign key relationships from column names
├── build_schema_rag_index.py   ← Builds Chroma RAG index from schema
├── data_profiler.py            ← Profiles DB tables (row counts, column stats)
└── windows/                    ← Windows-specific helper scripts
```

These are run once during initial setup, or re-run when the DB schema changes.

---

## Key Files Quick Reference

| Task | File |
|------|------|
| Add a new API endpoint | `src/api/main_complete.py` |
| Change Maria schedule | `data/maria_config.json` or `src/agents/maria/scheduler.py` |
| Add a new alert type | `src/agents/maria/maria_agent.py` → `_compute_anomalies()` |
| Add a new email template | `src/agents/maria/email_service.py` |
| Add a new CT view | `src/api/main_complete.py` → `_CT_BASE_SQL` + new route |
| Change DB schema context | `table.yaml` + re-run `/api/setup/generate` |
| Add a new React tab | `frontend/src/AppImpl.tsx` |
| Add a new chart type | `frontend/src/components/dashboard/ChartRenderer.tsx` |
| Change alert thresholds | `.env` or `data/maria_config.json` |
| Add a new config variable | `src/config/settings.py` + `.env` |
