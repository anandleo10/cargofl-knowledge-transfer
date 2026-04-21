# Glossary — Terms, Abbreviations & Domain Vocabulary

---

## Business / Domain Terms

| Term | Full Form | Meaning |
|------|-----------|---------|
| **OTD** | On-Time Delivery | The percentage of shipments delivered on or before the scheduled delivery date. Primary KPI. Formula: `(on-time deliveries / total deliveries) × 100` |
| **EDD** | Expected Delivery Date | Same as `schdeliverydate` — the date a shipment was scheduled to arrive. |
| **POD** | Proof of Delivery | Document confirming delivery. `pod_lsp_status = "Y"` means POD has been uploaded. |
| **LSP** | Logistics Service Provider | The transporter company handling the shipment (e.g., DHL, BlueDart). Also called "trucker". |
| **PTL** | Part Truck Load | A shipment that shares truck space with other consignments (smaller loads). |
| **FTL** | Full Truck Load | A shipment that occupies an entire truck. |
| **Overdue** | — | A shipment that is past its EDD and has not yet been delivered. `overdue_days = today - schdeliverydate` (for in-transit shipments). |
| **In-Transit** | — | A shipment that has been dispatched but not yet delivered. `is_intransit = NOT delivered AND lrtype IN (PTL, FTL)`. |
| **LR** | Lorry Receipt | The transport document issued when goods are handed to the transporter. |
| **Branch** | — | The origin or source office (e.g., Delhi, Mumbai, Chennai) from which the booking was made. |
| **Transporter** | — | The company physically moving the goods. Same as LSP. Stored in `truckername` column. |
| **Booking** | — | A single shipment entry in `tlbooking`. One row = one booking/consignment. |
| **Consignment** | — | Another word for a single shipment/booking. |
| **GTV** | Gross Total Value | The value of goods in the booking. Stored in `gross_total`. |

---

## System / Code Terms

| Term | Meaning | Where Used |
|------|---------|-----------|
| **view_cache** | Python dict in `main_complete.py` holding CT DataFrames keyed by view name ("v1"–"v5"). In-memory only, rebuilt on demand or nightly. | `main_complete.py` |
| **active_sessions** | `_BoundedSessionStore` dict holding live analysis state keyed by `run_id`. Cleared after session timeout. | `main_complete.py` |
| **_alert_state** | Python dict in `MariaBrain` holding current active KPI alerts. In-memory, rebuilt each anomaly check. | `maria_agent.py` |
| **MariaBrain** | The main Maria orchestration class. Single instance per server lifetime. Holds all Maria state and methods. | `src/agents/maria/maria_agent.py` |
| **MariaAnalyst** | The Q&A engine. Instantiated per-request. Runs Planner→Executor→Narrator pipeline against CT DataFrames. | `src/agents/maria/analyst.py` |
| **MariaScheduler** | Wrapper around APScheduler. Manages 6 recurring jobs. | `src/agents/maria/scheduler.py` |
| **IMAPListener** | Polls Gmail inbox for incoming Q&A questions. Runs every 5 minutes via APScheduler. | `src/agents/maria/imap_listener.py` |
| **ExecutionPlan** | Structured plan produced by `QueryPlanningAgent`: a list of SQL steps with expected columns. | `src/agents/planning_agent.py` |
| **ExecutionStep** | One step in an `ExecutionPlan`: `{name, sql, operation_type, expected_columns}`. | `src/agents/planning_agent.py` |
| **ResearchState** | LangGraph `TypedDict` passed between all workflow nodes. Accumulates the results of each step. | `src/orchestration/research_graph.py` |
| **DashboardConfig** | Frontend data structure: a list of panels with chart specs, positions, and titles. | Shared: Python + TypeScript |
| **ChartSpec** | Recharts-compatible JSON: `{chart_type, data_key, metrics, data}`. Produced by backend, rendered by Recharts. | `src/agents/visualization_agent.py` |
| **_PROJECT_ROOT** | `Path(__file__).resolve().parents[N]` anchor used to construct all data file paths, ensuring correctness regardless of working directory. | `maria_agent.py`, `main_complete.py` |
| **_ct_refresh_job** | Async callback function defined in `main_complete.py` and injected into `MariaScheduler`. Refreshes all CT views nightly. | `main_complete.py` |
| **DataSliceService** | Service that safely filters/aggregates CT DataFrames for panel rendering. Always copies before filtering. | `src/services/data_slice_service.py` |
| **SubscriptionStore** | Manages dynamic email recipient list (TO/CC). Persists to JSON. | `src/agents/maria/maria_agent.py` |
| **AllowedSendersStore** | Manages Q&A sender whitelist. Persists to JSON. | `src/agents/maria/maria_agent.py` |
| **_SystemAlertStore** | In-memory cooldown guard for ops error emails. Prevents duplicate ops alerts. | `src/agents/maria/maria_agent.py` |
| **activity_store** | Singleton `ActivityStore`. Records all Maria actions to `data/maria_activity.json`. | `src/agents/maria/activity_store.py` |
| **_get_or_create_engine** | Thread-safe SQLAlchemy engine factory. Keyed by DATABASE_URL. Reused across requests. | `src/orchestration/research_graph.py` |
| **run_id** | UUID identifying a specific analysis session. Used to look up results in `active_sessions` or `view_cache`. | Throughout backend |
| **request_id** | UUID assigned when a research request starts. Used by frontend to poll status. | `main_complete.py` |

---

## Alert Types

| Alert Type | Metric | Tier |
|------------|--------|------|
| `OTD_WARNING` | OTD% below warning threshold (65%) | warning |
| `OTD_CRITICAL` | OTD% below critical threshold (55%) | critical |
| `DELAY_ALERT` | Avg delay above warning threshold (72h) | warning |
| `DELAY_CRITICAL` | Avg delay above critical threshold (168h) | critical |
| `OVERDUE_WARNING` | Overdue count above warning threshold (3500) | warning |
| `OVERDUE_CRITICAL` | Overdue count above critical threshold (5000) | critical |

---

## Technology Abbreviations

| Abbrev. | Full Name | Role |
|---------|-----------|------|
| **LLM** | Large Language Model | GPT-4o — used for SQL planning, narrative generation, Q&A |
| **RAG** | Retrieval-Augmented Generation | Enhances LLM prompts with relevant schema/document chunks |
| **SMTP** | Simple Mail Transfer Protocol | Used for sending emails (Gmail, port 587 TLS) |
| **IMAP** | Internet Message Access Protocol | Used for reading emails (Gmail, port 993 SSL) |
| **SPA** | Single Page Application | The React frontend — one HTML file, all routing in JS |
| **ORM** | Object-Relational Mapper | SQLAlchemy — Python layer over MySQL |
| **ASGI** | Async Server Gateway Interface | The protocol Uvicorn/FastAPI use for async HTTP |
| **IST** | Indian Standard Time | UTC+5:30. All Maria schedules use IST timezone. |
| **CT** | Control Tower | The pre-built 5-view logistics dashboard |
| **KPI** | Key Performance Indicator | Metrics Maria monitors: OTD%, overdue count, avg delay |
| **RFC 2822** | Email threading standard | Defines `In-Reply-To` and `References` headers for email thread grouping |

---

## Email Subject Conventions (Maria)

| Email Type | Subject Pattern |
|-----------|----------------|
| Morning Brief | `Maria \| CargoFL Operations Intelligence \| {date}` |
| Weekly Digest | `Maria \| Weekly Performance Digest \| {date}` |
| EOD Alert Digest | `Maria \| EOD Alert Summary \| {date}` |
| Q&A Reply | `Re: {original subject}` |
| System Alert (ops) | `URGENT \| Maria \| {ERROR_TYPE} — {source}` |

---

## File Naming Conventions

| Pattern | Meaning |
|---------|---------|
| `data/dashboards/{uuid}.json` | Saved dashboard config |
| `data/uploads/{uuid}_{filename}.xlsx` | Uploaded Excel file |
| `data/outputs/{request_id}_report.html` | Generated HTML report |
| `data/outputs/{request_id}_export.xlsx` | Generated Excel export |
| `src/agents/prompts/{agent}/agents.yaml` | CrewAI agent role definition |
| `src/agents/prompts/{agent}/tasks.yaml` | CrewAI task definition |

---

## Config Quick Reference

| Want to… | Edit… |
|---------|-------|
| Change morning brief time | `data/maria_config.json` → `schedule.morning_brief_hour` |
| Add email recipient | Configure tab → Recipients, or `data/maria_subscriptions.json` |
| Add Q&A sender | Configure tab → Allowed Senders, or `data/maria_allowed_senders.json` |
| Change alert thresholds | Configure tab → Alert Thresholds, or `data/maria_config.json` → `thresholds.*` |
| Disable nightly CT refresh | Configure tab → CT Data Refresh toggle, or `data/maria_config.json` → `ct_refresh_enabled: false` |
| Change DB credentials | `.env` → `DATABASE_*` vars + `docker compose restart api` |
| Change OpenAI key | `.env` → `OPENAI_API_KEY` + `docker compose restart api` |
| Rotate Gmail password | `.env` → `SMTP_PASSWORD` + `IMAP_PASSWORD` + `docker compose restart api` |
| Add new DB table to schema | `table.yaml` → add entry + `POST /api/setup/generate` |
