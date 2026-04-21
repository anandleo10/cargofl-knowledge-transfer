# Data Flow & Workflows

This document covers the two primary end-to-end workflows in the system:

- **Workflow A** — User query → Analysis → Dashboard (Research Agent pipeline)
- **Workflow B** — Maria proactive cycle (scheduled jobs + email flow)
- **Workflow C** — Maria Q&A via email

---

## Workflow A: User Query → Analysis → Dashboard

### Trigger
A user types a natural-language question in the Analysis tab and clicks "Run Analysis".

### High-Level Steps

```
Browser (React)
  └─ POST /api/research/start
        └─ FastAPI creates request_id, launches background task
              └─ LangGraph StateGraph executes 8 nodes
                    └─ Result stored in active_sessions[request_id]
  └─ GET /api/research/status/{request_id}  (frontend polls every 2s)
        └─ Returns progress events + final result when complete
```

### Step-by-Step Detail

#### Step 0: Request Arrival
```python
POST /api/research/start
Body: {
  "query": "Show top 10 transporters by OTD% last 30 days",
  "data_source": "database",  # or "excel"
  "use_crews": false,         # override RA_USE_CREWS
  "time_filter": "last_30_days"
}
```
FastAPI:
1. Validates request with Pydantic
2. Generates `request_id` (UUID)
3. Creates initial state in `active_sessions[request_id]`
4. Launches `asyncio.create_task(run_research_workflow(...))`
5. Returns `{"request_id": "...", "status": "pending"}` immediately

#### Step 1: Guardrail Check
**Agent:** `GuardrailAgent`
**Purpose:** Safety check before any LLM or DB call
**Checks:**
- Does the query ask for PII (phone numbers, personal data)?
- Does it contain SQL injection patterns?
- Does it ask for DROP/DELETE/UPDATE operations?
- Is it about a forbidden table?

**Output:** `{safe: true}` → continue | `{safe: false, reason: "..."}` → abort with error

#### Step 2: Clarification (conditional)
**Agent:** `ClarificationAgent`
**Triggered when:** Query is ambiguous (e.g. "show me performance" — performance of what?)
**What it does:**
1. Asks GPT-4o: "Is this query clear enough to answer without clarification?"
2. If not, generates 1–3 clarifying questions
3. Frontend shows questions, user answers them
4. Answers are injected back into the state
5. Workflow resumes from Planning

**Skipped when:** Query is specific enough (most of the time).

#### Step 3: Planning
**Agent:** `QueryPlanningAgent`
**Input:** Query + clarifications + schema context (from `table.yaml` + `data_context.json`)
**What it does:**
1. Loads schema context (table names, columns, relationships, sample values)
2. Sends to GPT-4o with system prompt explaining the DB structure
3. GPT-4o returns a structured `ExecutionPlan`:
   ```json
   {
     "steps": [
       {
         "name": "transporter_otd",
         "sql": "SELECT truckername, COUNT(*) as total, ...",
         "operation_type": "aggregate",
         "expected_columns": ["truckername", "otd_pct", "total_deliveries"]
       }
     ]
   }
   ```
4. Plan is validated (no cycles, valid table names, no forbidden operations)

#### Step 4: Execution
**Agent:** `ExecutionAgent`
**Input:** `ExecutionPlan` + SQLAlchemy engine
**What it does:**
1. For each step in the plan:
   - Runs SQL via `_get_or_create_engine()` (thread-safe pool)
   - Wraps in `asyncio.to_thread()` to avoid blocking
   - Enforces `SQL_TIMEOUT_SECONDS=30` limit
   - Enforces `MAX_RESULT_ROWS=100000` limit
   - Returns result as Pandas DataFrame
2. Stores all DataFrames in `results[step_name] = df`
3. On SQL error: retries once with a corrected query (asks GPT-4o to fix)

**DB Connection:**
```
_get_or_create_engine() in research_graph.py
  → Creates SQLAlchemy engine (mysql+pymysql://)
  → Pool size: 5, max_overflow: 10
  → Keyed by DATABASE_URL (reused across requests)
```

#### Step 5: Visualization
**Agent:** `VisualizationAgent`
**Input:** DataFrames from execution
**What it does:**
1. Inspects each DataFrame's column types (numeric, categorical, datetime)
2. Auto-selects chart type:
   - 1 categorical + 1 numeric → bar chart
   - datetime + numeric → line chart
   - 2 numerics → scatter plot
   - Small cardinality (≤6) → pie chart
   - Large/complex → table
3. Builds Recharts-compatible JSON spec:
   ```json
   {
     "chart_type": "bar",
     "data_key": "truckername",
     "metrics": [{"key": "otd_pct", "color": "#4CAF50", "name": "OTD %"}],
     "data": [{"truckername": "DHL", "otd_pct": 82.3}, ...]
   }
   ```
4. If base64 image requested (for report), generates PNG via Plotly/Kaleido

#### Step 6: Dashboard Design
**Agent:** `DashboardDesignAgent`
**Input:** Chart specs + original query
**What it does:**
1. Groups charts into logical panels (e.g. "Transporter Performance", "Branch Breakdown")
2. Assigns grid positions (React responsive grid)
3. Sets panel titles and descriptions
4. Returns `DashboardConfig` (list of panels with chart specs)

#### Step 7: Summarization
**Agent:** `SummarizerAgent`
**Input:** Panel data + original query
**What it does:**
1. For each panel: sends chart data to GPT-4o → returns 1–3 sentence summary
2. Combines panel summaries → generates overall `key_findings` (3–5 bullet points)
3. Optionally generates Excel export

#### Step 8: Report Generation
**Agent:** `ReportGenerator`
**Input:** Full analysis result
**What it does:**
1. Renders HTML report from template
2. Saves to `data/outputs/{request_id}_report.html`
3. Available for download at `GET /api/research/download/{request_id}`

### State Object (Shared Across All Nodes)
```python
class ResearchState(TypedDict):
    request_id: str
    query: str
    data_source: str
    clarification_needed: bool
    clarification_qa: list[dict]
    guardrail_result: dict
    execution_plan: ExecutionPlan
    execution_results: dict[str, pd.DataFrame]
    chart_specs: list[dict]
    dashboard_config: DashboardConfig
    summaries: list[dict]
    key_findings: list[str]
    status: str  # pending | running | completed | error
    error: str
    progress_events: list[dict]
```

### Progress Events (Real-Time)
The frontend polls `GET /api/research/status/{request_id}` every 2 seconds. The response includes:
```json
{
  "status": "running",
  "progress_events": [
    {"step": "guardrail", "message": "Safety check passed", "duration_ms": 120},
    {"step": "planning", "message": "Generated 2-step execution plan", "duration_ms": 1840},
    {"step": "execution", "message": "Step 1/2: transporter_otd — 143 rows", "duration_ms": 890}
  ],
  "result": null
}
```
When `status == "completed"`, `result` contains the full `DashboardConfig`.

---

## Workflow B: Maria Proactive Cycle

Maria runs 6 scheduled jobs. Here's the full flow for each.

### Job 1: Morning Brief (Mon–Fri, 8:00 AM IST)

```
APScheduler triggers send_morning_brief()
  │
  ├─ Load digest config (data/maria_config.json → morning_brief.time_filter)
  │
  ├─ Fetch all 5 CT views in parallel (asyncio.gather)
  │   ├─ _fetch_and_summarize("v1") → {summary, key_findings, kpi_snapshot, df}
  │   ├─ _fetch_and_summarize("v2") → ...
  │   ├─ _fetch_and_summarize("v3") → ...
  │   ├─ _fetch_and_summarize("v4") → ...
  │   └─ _fetch_and_summarize("v5") → ...
  │       └─ Each: _fetch_ct_df(view) → view_cache hit or DB query
  │              MariaAnalyst summarizes the DataFrame using GPT-4o
  │
  ├─ Compute KPI snapshot: OTD%, overdue count, delay stats
  │
  ├─ Build active alerts list from self._alert_state
  │
  ├─ Build HTML using build_morning_brief_html(...)
  │
  └─ send_email(to=SENIOR_MANAGER_EMAIL + extra_to, cc=CC_EMAILS + subscribers)
      │
      └─ activity_store.complete("morning_brief")
```

### Job 2: Weekly Digest (Monday, 7:00 AM IST)

Similar to morning brief but covers the full previous week's performance trends, including week-over-week comparison if prior week data is available. Sent only on Mondays.

### Job 3: Anomaly Check (Every 2 Hours)

```
APScheduler triggers check_anomalies()
  │
  ├─ Fetch v5 DataFrame (OTD calculation)
  │   └─ _fetch_ct_df("v5") → view_cache or DB
  │
  ├─ Fetch v2 DataFrame (in-transit / overdue)
  │   └─ _fetch_ct_df("v2") → view_cache or DB
  │
  ├─ _compute_anomalies(df_v5, df_v2) → list of {type, metric_value, threshold, tier, details}
  │   ├─ OTD check: if otd_pct < MARIA_OTD_ALERT_THRESHOLD → OTD_WARNING
  │   │             if otd_pct < MARIA_CRITICAL_OTD_THRESHOLD → OTD_CRITICAL
  │   ├─ Delay check: mean delay > MARIA_DELAY_ALERT_HOURS → DELAY_ALERT
  │   │               mean delay > MARIA_CRITICAL_DELAY_HOURS → DELAY_CRITICAL
  │   │               (capped at 30 days per shipment to exclude bad data)
  │   └─ Overdue count: total_overdue > MARIA_OVERDUE_COUNT_THRESHOLD → OVERDUE_WARNING
  │                     total_overdue > MARIA_CRITICAL_OVERDUE_COUNT → OVERDUE_CRITICAL
  │
  ├─ Update self._alert_state:
  │   ├─ New anomaly: add to state with first_detected timestamp
  │   ├─ Existing anomaly: update last_value, previous_value, tier
  │   └─ Resolved anomaly: remove from state
  │
  └─ Log results to activity_store (no email sent — digest handles that)
```

### Job 4: EOD Alert Digest (Mon–Fri, 5:00 PM IST)

```
APScheduler triggers send_alert_digest()
  │
  ├─ Check self._alert_state
  │   ├─ If empty → send "All Clear" digest (brief OK message)
  │   └─ If has alerts → build full alert digest
  │
  ├─ _build_alert_display_list(alert_state.values(), now)
  │   ├─ Normalise last_value → metric_value
  │   ├─ Calculate duration_h (hours since first_detected)
  │   └─ Add trend symbol (▲ worsening / ▼ improving / → stable)
  │
  ├─ build_alert_digest_html(alerts, date_str) → HTML
  │
  └─ send_email(to=SENIOR_MANAGER_EMAIL, cc=CC_EMAILS)
```

### Job 5: IMAP Poll (Every 5 Minutes)

```
APScheduler triggers IMAPListener.poll_inbox()
  │
  ├─ asyncio.to_thread(_fetch_unseen_from_allowed)
  │   └─ imap.login → imap.select("INBOX") → imap.search(UNSEEN)
  │       ├─ For each unseen message:
  │       │   ├─ imap.fetch(RFC822)
  │       │   ├─ Parse From header
  │       │   ├─ Check if sender in AllowedSendersStore (whitelist)
  │       │   ├─ _extract_plain_text(msg) → strip HTML, get text body
  │       │   ├─ _remove_reply_quotes(text) → strip "On ... wrote:" etc.
  │       │   └─ Mark as SEEN
  │       └─ Return [{from, text, message_id, references, subject}]
  │
  └─ For each message:
      └─ MariaBrain.answer_question(question, reply_to, message_id, references, subject)
          └─ (See Workflow C below)
```

### Job 6: CT Data Refresh (11:00 PM IST)

```
APScheduler triggers _ct_refresh_job() (injected from main_complete.py)
  │
  ├─ For each view in [v1, v2, v3, v4, v5]:
  │   ├─ Check ct_refresh_enabled flag (maria_config.json) — skip view if disabled
  │   ├─ _fetch_ct_df(view, force=True) → hits DB, updates view_cache[view]
  │   ├─ Evict stale active_sessions entries for CT dashboards using this view
  │   └─ activity_store log: "CT Data Refresh: view={v}, {row_count} rows loaded"
  │
  └─ Total refresh of all 5 views completes overnight
     Next day's morning brief uses fresh data from view_cache
```

---

## Workflow C: Maria Email Q&A

### Trigger
A user (on the allowed senders list) sends an email to `maria.puma@cargofl.com`.

```
User sends email: "What is the OTD% for last week by transporter?"
  │
  │ (within 5 minutes)
  ▼
IMAP Poll Job (Job 5 above) fetches the message
  │
  ▼
MariaBrain.answer_question(question, reply_to, message_id, references, subject)
  │
  ├─ Build threading headers:
  │   In-Reply-To: <original_message_id>
  │   References: <prior_references> <original_message_id>
  │   Subject: Re: <original_subject>   ← keeps thread in mail client
  │
  ├─ Check 1: Is this a sender management command?
  │   e.g. "add john@example.com to senders" → AllowedSendersStore.add()
  │   → reply immediately with confirmation
  │
  ├─ Check 2: Is this a subscription command?
  │   e.g. "add sarah@example.com to CC" → SubscriptionStore.add()
  │   → reply immediately with confirmation
  │
  ├─ Check 3: Does question match _ALERT_QUESTION_PATTERNS?
  │   Keywords: alert, warning, critical, breach, threshold, morning brief,
  │             in your report, you mentioned, daily report, those alerts, etc.
  │   → YES: call _answer_from_alert_state()
  │          Format: list of active alerts with value, threshold, duration, trend
  │          → reply immediately WITHOUT hitting MariaAnalyst
  │
  └─ Check 4: Route to MariaAnalyst for CT data questions
      │
      ├─ MariaAnalyst.answer(question)
      │   ├─ PLANNER (GPT-4o):
      │   │   Input: question + CT view descriptions
      │   │   Output: {view, filters, group_by, metrics, clarification_needed}
      │   │
      │   ├─ EXECUTOR:
      │   │   _fetch_ct_df(planned_view) → DataFrame
      │   │   Apply filters + group_by from plan
      │   │   Calculate requested metrics (OTD%, counts, percentages)
      │   │
      │   └─ NARRATOR (GPT-4o):
      │       Input: question + computed DataFrame (top N rows)
      │       Output: {answer: "narrative text", key_findings: [...]}
      │
      └─ send_email(
             to=reply_to_address,
             subject="Re: <original_subject>",
             in_reply_to=message_id,     ← threads in Gmail/Outlook
             references=reply_references
         )
```

---

## Control Tower Data Flow

How CT data flows from DB to browser panel:

```
Browser requests CT view v2 (In-Transit)
  │
  ▼
POST /api/dashboard/create-control-tower-v2
  {filters: {branch: "Delhi", date_range: "last_30_days"}}
  │
  ▼
_fetch_ct_df("v2", force=False)
  ├─ Acquire asyncio lock for "v2" (prevent duplicate DB hits)
  ├─ If view_cache["v2"] exists AND not stale → return cached DataFrame
  └─ Else → execute _CT_BASE_SQL against MySQL
              → store in view_cache["v2"]
              → release lock
  │
  ▼
_apply_ct_derived_columns(df, "v2")
  Adds:
  - is_delivered: lsp_status == "Delivered"
  - is_intransit: not is_delivered AND lrtype in ["PTL", "FTL"]
  - is_delayed_non_delivered: is_intransit AND overdue_days > 0
  - is_todays_edd: schdeliverydate == today
  - is_future_edd: schdeliverydate > today
  - edd_category: "Today" / "Tomorrow" / "This Week" / "Later"
  │
  ▼
DataSliceService.compute_slice(df, filters, group_by, metrics)
  ├─ df_copy = df.copy()     ← ALWAYS copy — never mutate shared cache
  ├─ Apply date filters
  ├─ Apply column filters (branch, transporter, status)
  ├─ Group by requested dimensions
  └─ Return aggregated DataFrame
  │
  ▼
Build panel chart specs (JSON)
  │
  ▼
Return DashboardConfig → Frontend renders Recharts
```

---

## Error Handling Flow

```
Any exception in workflow
  │
  ├─ LangGraph node: caught by try/except in node function
  │   → progress_event: {step: "...", status: "error", message: "..."}
  │   → ResearchState.status = "error"
  │   → ResearchState.error = str(exc)
  │
  ├─ Maria scheduled job: caught by _notify_if_critical(exc, source, tb_str)
  │   → _classify_error(exc) → (type, should_alert, severity)
  │   │   OPENAI_QUOTA_EXCEEDED → alert ops team immediately
  │   │   OPENAI_AUTH_FAILED    → alert ops team immediately
  │   │   DB_CONNECTION_FAILED  → alert ops team immediately
  │   │   Other                 → log, no alert
  │   │
  │   └─ MariaBrain.send_system_alert(exc, source, tb_str)
  │       → check _SystemAlertStore cooldown (30 min per error type)
  │       → send URGENT email to MARIA_SUPPORT_EMAIL
  │       → CC: MARIA_SUPPORT_CC
  │       → Subject: "URGENT | Maria | {error_type} — {source}"
  │
  └─ HTTP endpoint: caught by global @app.exception_handler(Exception)
      → If HTTPException → return JSONResponse with status_code
      → If actionable error → asyncio.create_task(send_system_alert(...))
      → Return 500 JSONResponse to client
```
