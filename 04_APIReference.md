# API Reference

**Base URL:** `http://ai-research.cargofl.com:8000` (production) / `http://localhost:8000` (local)

All endpoints return JSON. Errors return `{"detail": "..."}` with appropriate HTTP status codes.

---

## Health

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Returns `{"status":"healthy","version":"1.0.0","openai_configured":true}` |
| GET | `/` | Root — redirects to docs |

---

## Research & Analysis

### Start Analysis
```
POST /api/research/start

Request:
{
  "query": "string — natural language question",
  "data_source": "database" | "excel",
  "file_id": "string (optional) — Excel file ID if data_source=excel",
  "time_filter": "last_7_days" | "last_30_days" | "last_90_days" | "this_month" | "all_time",
  "use_crews": boolean (optional, overrides RA_USE_CREWS env var)
}

Response:
{
  "request_id": "uuid-string",
  "status": "pending"
}
```

### Poll Analysis Status
```
GET /api/research/status/{request_id}

Response (running):
{
  "status": "running",
  "progress_events": [
    {"step": "guardrail", "message": "...", "duration_ms": 120},
    ...
  ],
  "result": null
}

Response (completed):
{
  "status": "completed",
  "progress_events": [...],
  "result": {
    "dashboard_config": { ...panels... },
    "key_findings": ["finding 1", "finding 2"],
    "summary": "overall summary text",
    "run_id": "uuid"
  }
}

Response (error):
{
  "status": "error",
  "error": "error message",
  "progress_events": [...]
}
```

### Submit Clarification Answers
```
POST /api/research/clarify

Request:
{
  "request_id": "uuid",
  "answers": [
    {"question": "Which branch?", "answer": "Delhi"}
  ]
}

Response: {"status": "resumed"}
```

### Cancel Analysis
```
POST /api/research/cancel/{request_id}
Response: {"status": "cancelled"}
```

### Download Report
```
GET /api/research/download/{request_id}
Response: HTML file download (Content-Disposition: attachment)
```

### Run History
```
GET /api/research/history
Response: [
  {
    "id": "uuid",
    "query": "...",
    "status": "completed",
    "started_at": "ISO datetime",
    "finished_at": "ISO datetime",
    "row_count": 143
  },
  ...  (last 100 runs)
]
```

---

## Setup & Schema

### Check Setup Status
```
GET /api/setup/status
Response: {
  "schema_ready": true,
  "rag_index_ready": true,
  "tables_configured": 5,
  "last_profiled": "ISO datetime"
}
```

### List Configured Tables
```
GET /api/setup/tables
Response: [{"name": "tlbooking", "description": "...", "row_count": 573000}, ...]
```

### List DB Tables
```
GET /api/setup/db_tables
Response: ["tlbooking", "table2", ...]
```

### Get / Update Semantic Graph
```
GET  /api/setup/semantic_graph
PUT  /api/setup/semantic_graph
Body: { nodes: [...], edges: [...] }
```

### Get Columns for Table
```
GET /api/setup/table-columns/{table_name}
Response: [{"name": "bookingid", "type": "VARCHAR", "description": "..."}, ...]
```

### Refresh Schema Cache
```
POST /api/setup/refresh-schema-cache
Response: {"status": "refreshed", "tables": 5}
```

### Profile Database
```
POST /api/setup/profile
Body: {"tables": ["tlbooking"]}  (optional; profiles all if omitted)
Response: {"status": "profiling_started", "task_id": "..."}
```

### Initialize Setup
```
POST /api/setup/init
Body: {"database_url": "...", "profile": true}
Response: {"status": "initialized"}
```

### Get / Update Metadata
```
GET /api/setup/metadata
PUT /api/setup/metadata
Body: { ...metadata fields... }
```

### Generate Schema Artifacts
```
POST /api/setup/generate
Response: {"status": "generating", "task_id": "..."}
Note: Rebuilds data_context.json and Chroma RAG index. Run after table.yaml changes.
```

---

## Dashboard

### Design Dashboard from Query Results
```
POST /api/dashboard/design
Body: {
  "request_id": "uuid",
  "query": "natural language query",
  "panel_preferences": ["bar chart for transporter", "table for raw data"]
}
Response: { dashboard_config: DashboardConfig }
```

### Slice Data for Panel
```
POST /api/dashboard/data-slice
Body: {
  "run_id": "uuid",
  "panel_id": "string",
  "filters": {"branch": "Delhi"},
  "group_by": ["truckername"],
  "metrics": ["otd_pct", "total_count"],
  "sort": {"column": "otd_pct", "direction": "desc"}
}
Response: { data: [...rows], chart_spec: {...} }
```

### Save Dashboard
```
POST /api/dashboard/save
Body: {
  "name": "Transporter OTD October",
  "dashboard_config": { ...panels... },
  "run_id": "uuid (optional)"
}
Response: {"dashboard_id": "uuid", "name": "...", "saved_at": "ISO datetime"}
```

### List Saved Dashboards
```
GET /api/dashboard/list
Response: [
  {"id": "uuid", "name": "...", "created_at": "...", "panel_count": 3},
  ...
]
```

### Load / Delete Dashboard
```
GET    /api/dashboard/{dashboard_id}   → Returns full DashboardConfig
DELETE /api/dashboard/{dashboard_id}   → Response: {"deleted": true}
```

### Export Dashboard
```
POST /api/dashboard/{dashboard_id}/export
Body: {"format": "excel" | "html"}
Response: File download
```

### Export Raw Data
```
POST /api/dashboard/{dashboard_id}/export-raw
Response: CSV file download
```

### Summarize Dashboard
```
POST /api/dashboard/{dashboard_id}/summarize
Body: {"query": "original query for context"}
Response: {
  "panel_summaries": [{"panel_id": "...", "summary": "..."}],
  "overall_summary": "...",
  "key_findings": ["..."]
}
```

### Chat Edit Dashboard
```
POST /api/dashboard/chat-edit
Body: {
  "dashboard_id": "uuid",
  "message": "Add a pie chart of branch distribution"
}
Response: { updated_dashboard_config: DashboardConfig }
```

### Generate AI Insight
```
POST /api/dashboard/insight
Body: {"panel_id": "...", "panel_data": {...}, "query": "..."}
Response: {"insight": "narrative text"}
```

### Apply Time Filter
```
POST /api/dashboard/time-filter
Body: {
  "dashboard_id": "uuid",
  "time_filter": "last_30_days"
}
Response: { updated DashboardConfig with filtered data }
```

### NL Config Edit
```
POST /api/dashboard/nl-config
Body: {"instruction": "rename column truckername to Transporter"}
Response: {"status": "applied", "changes": [...]}
```

---

## Control Tower

### Create CT Views

All CT view endpoints follow the same pattern:

```
POST /api/dashboard/create-control-tower        ← v1: Overview
POST /api/dashboard/create-control-tower-v2     ← v2: In-Transit
POST /api/dashboard/create-control-tower-v3     ← v3: Delayed Deep-Dive
POST /api/dashboard/create-control-tower-v4     ← v4: Future EDD
POST /api/dashboard/create-control-tower-v5     ← v5: Delivered Performance

Request body (all views):
{
  "filters": {
    "branch": "Delhi",                          ← optional
    "transporter": "DHL",                       ← optional
    "date_range": "last_30_days",               ← optional
    "start_date": "2025-01-01",                 ← optional
    "end_date": "2025-03-31"                    ← optional
  },
  "view_config": {
    "group_by": "branch",                       ← optional override
    "top_n": 10                                 ← optional
  }
}

Response: DashboardConfig with CT-specific panels
```

### Refresh CT Cache
```
POST /api/dashboard/refresh-cache/{view}
Path param: view = "v1" | "v2" | "v3" | "v4" | "v5"

Response: {
  "status": "refreshed",
  "view": "v2",
  "row_count": 12847,
  "cache_updated_at": "ISO datetime"
}

Side effects:
- Evicts stale active_sessions entries for CT dashboards using this view
- Forces next request to re-query MySQL
```

### CT Cache Status
```
GET /api/ct/cache-status
Response: {
  "v1": {"cached": true, "row_count": 573420, "cached_at": "..."},
  "v2": {"cached": true, "row_count": 12847, "cached_at": "..."},
  ...
}
```

### Get Filter Options
```
GET /api/control-tower/{view}/filter-options
Path param: view = "v1" | "v2" | "v3" | "v4" | "v5"

Response: {
  "branches": ["Delhi", "Mumbai", "Chennai", ...],
  "transporters": ["DHL", "BlueDart", ...],
  "statuses": ["Delivered", "In Transit", ...]
}
```

### Get CT Column Definitions
```
GET /api/control-tower/columns
Response: {
  "v1": [{"name": "truckername", "label": "Transporter", "type": "string"}, ...],
  "v2": [...],
  ...
}
```

### Export Full CT Data
```
POST /api/control-tower/export-full
Body: {"view": "v2", "filters": {...}}
Response: Excel file download (all rows, all columns)
```

### Delay Remarks
```
POST /api/control-tower/delay-remarks/upload
Body: multipart/form-data with Excel file
Response: {"status": "uploaded", "rows": 450}

GET /api/control-tower/delay-remarks/status
Response: {"last_upload": "ISO datetime", "row_count": 450}
```

---

## File Management

### Upload Excel File
```
POST /api/upload/file
Body: multipart/form-data
  - file: Excel (.xlsx, .xls, .csv)
  - name: "optional display name"

Response: {
  "file_id": "uuid",
  "name": "filename.xlsx",
  "columns": ["col1", "col2", ...],
  "row_count": 5000,
  "uploaded_at": "ISO datetime"
}
```

### List Files
```
GET /api/files
Response: [{"file_id": "...", "name": "...", "row_count": ..., "uploaded_at": "..."}, ...]
```

### Delete File
```
DELETE /api/files/{file_id}
Response: {"deleted": true}
```

---

## Maria Agent

### Chat Q&A (Async)
```
POST /api/maria/chat
Body: {"question": "What is the OTD% for Delhi branch last week?"}
Response: {"task_id": "uuid", "status": "processing"}

GET /api/maria/chat/{task_id}
Response (completed): {
  "status": "completed",
  "answer": "narrative text answer",
  "key_findings": ["finding 1", "..."],
  "view_used": "v5"
}
```

### Trigger Manual Sends
```
POST /api/maria/brief        ← Trigger morning brief immediately
POST /api/maria/digest       ← Trigger weekly digest immediately
POST /api/maria/check        ← Trigger anomaly check immediately
POST /api/maria/alert-digest ← Trigger EOD alert digest immediately

Response (all): {"status": "triggered", "task_id": "uuid"}
```

### Maria Status & Readiness
```
GET /api/maria/ready
Response: {
  "ready": true,
  "checks": {
    "smtp": true,
    "imap": true,
    "db": true,
    "openai": true
  }
}

GET /api/maria/status
Response: {
  "enabled": true,
  "jobs": [
    {"job": "morning_brief", "id": "...", "next_run_utc": "ISO datetime"},
    {"job": "weekly_digest", "id": "...", "next_run_utc": "..."},
    {"job": "anomaly_check", "id": "...", "next_run_utc": "..."},
    {"job": "alert_digest", "id": "...", "next_run_utc": "..."},
    {"job": "imap_poll", "id": "...", "next_run_utc": "..."},
    {"job": "ct_refresh", "id": "...", "next_run_utc": "..."}
  ]
}
```

### Config
```
GET /api/maria/config
Response: full MariaConfig object

PUT /api/maria/config
Body: partial or full MariaConfig
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
  }
}
Response: {"status": "updated", "config": {...}}
```

### Subscriptions (Recipients)
```
GET  /api/maria/subscriptions
Response: {"to": ["email@..."], "cc": ["email@..."]}

POST /api/maria/subscriptions/add
Body: {"email": "newperson@example.com", "role": "cc"}

POST /api/maria/subscriptions/remove
Body: {"email": "person@example.com"}
```

### Allowed Senders (Q&A Whitelist)
```
GET  /api/maria/senders
Response: {"senders": ["malini@puma.com", "anand@innoctive.com"]}

POST /api/maria/senders/add
Body: {"email": "newperson@example.com"}

POST /api/maria/senders/remove
Body: {"email": "person@example.com"}
```

### Active Alerts
```
GET /api/maria/alerts/active
Response: {
  "alerts": [
    {
      "type": "OTD_WARNING",
      "tier": "warning",
      "last_value": 62.4,
      "threshold": 65.0,
      "first_detected": "ISO datetime",
      "duration_h": 14.3,
      "details": "OTD dropped to 62.4% (threshold: 65.0%)"
    }
  ],
  "count": 1
}
```

### Activity Log
```
GET /api/maria/activity
Query params: limit=50 (default 50), offset=0, type=morning_brief (optional filter)

Response: {
  "events": [
    {
      "id": "uuid",
      "type": "morning_brief",
      "title": "Morning Brief",
      "started_at": "ISO datetime",
      "completed_at": "ISO datetime",
      "status": "completed",
      "summary": "Morning brief sent to 3 recipients",
      "steps": [...],
      "metadata": {...}
    }
  ],
  "total": 142
}

GET /api/maria/activity/stats
Response: {
  "total_events": 142,
  "by_type": {"morning_brief": 22, "anomaly_check": 84, ...},
  "last_24h": 12,
  "error_count": 1
}
```

### Demo Email
```
POST /api/maria/demo-email
Body: {"type": "morning_brief" | "weekly_digest" | "alert_digest", "to": "email@example.com"}
Response: {"status": "sent"}
```

---

## Debug

```
GET /api/debug/cache-status
Response: {
  "active_sessions_count": 3,
  "view_cache_keys": ["v1", "v2", "v5"],
  "redis_connected": true
}
```
