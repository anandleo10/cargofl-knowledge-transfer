# Maria Agent — Deep Dive

## What Is Maria?

Maria is the proactive intelligence layer of the CargoFL platform. While the Research Dashboard waits for a user to ask a question, Maria operates autonomously — monitoring KPIs around the clock, sending scheduled reports to the Senior Manager, and replying to email questions.

Maria lives in `src/agents/maria/` and is wired into the FastAPI server via startup/shutdown lifecycle events. She is activated when `MARIA_ENABLED=true` in `.env`.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          MariaBrain                                  │
│  (single instance, lives in server memory for server lifetime)       │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     MariaScheduler                          │    │
│  │  APScheduler.AsyncIOScheduler (timezone: Asia/Kolkata)      │    │
│  │  6 registered jobs ──────────────────────────────────────┐  │    │
│  │  1. morning_brief     (cron, 8 AM, Mon-Fri)              │  │    │
│  │  2. weekly_digest     (cron, 7 AM, Monday)               │  │    │
│  │  3. anomaly_check     (interval, every 2h)               │  │    │
│  │  4. alert_digest      (cron, 5 PM, Mon-Fri)              │  │    │
│  │  5. imap_poll         (interval, every 5min)             │  │    │
│  │  6. ct_refresh        (cron, 11 PM)                      │  │    │
│  └──────────────────────────────────────────────────────────┘  │    │
│                                                                  │    │
│  ┌──────────────────┐  ┌───────────────────┐  ┌─────────────┐  │    │
│  │  IMAPListener    │  │  MariaAnalyst     │  │  EmailSvc   │  │    │
│  │  polls Gmail     │  │  Planner→Exec→    │  │  SMTP send  │  │    │
│  │  inbox every 5m  │  │  Narrator (GPT4o) │  │  HTML tmpl  │  │    │
│  └──────────────────┘  └───────────────────┘  └─────────────┘  │    │
│                                                                  │    │
│  State:                                                          │    │
│  - _alert_state: dict[str, dict]   ← live KPI breaches          │    │
│  - _last_alert_fired: dict          ← cooldown timestamps        │    │
│  - SubscriptionStore               ← email recipients           │    │
│  - AllowedSendersStore             ← Q&A sender whitelist       │    │
└──────────────────────────────────────────────────────────────────┘
         │                │                │
         ▼                ▼                ▼
     MySQL RDS       Gmail SMTP        Gmail IMAP
  (via view_cache)  (outbound email)  (inbound Q&A)
```

---

## File-by-File Reference

### `maria_agent.py` — MariaBrain (1600+ lines)

The orchestrator class. Everything Maria does flows through here.

#### Key Classes

**`MariaBrain`**
The main class. Instantiated once in `startup_event()` in `main_complete.py`.

| Method | Purpose | Called By |
|--------|---------|-----------|
| `send_morning_brief()` | Daily operational summary email | Scheduler (8 AM) |
| `send_weekly_digest()` | Weekly performance trend email | Scheduler (Mon 7 AM) |
| `check_anomalies()` | KPI breach detection | Scheduler (every 2h) |
| `send_alert_digest()` | EOD summary of active alerts | Scheduler (5 PM) |
| `answer_question()` | Process incoming email Q&A | IMAPListener |
| `send_system_alert()` | URGENT ops alert for critical errors | Exception handlers |
| `_notify_if_critical()` | Wrapper: classify error + notify | Scheduler except blocks |
| `_compute_anomalies()` | Calculate KPI breaches from CT DataFrames | `check_anomalies()` |
| `_build_alert_display_list()` | Enrich alerts with duration and trend | Email builders |
| `_answer_from_alert_state()` | Format active alerts as Q&A reply | `answer_question()` |
| `_fetch_and_summarize()` | Fetch a CT view + get LLM narrative | Brief/digest methods |
| `_trend_symbol()` | ▲ / ▼ / → based on last vs prev value | Alert display |

**`SubscriptionStore`**
Manages the dynamic email recipient list. Persists to `data/maria_subscriptions.json`.
- `get_cc()` — returns static (`.env`) + dynamic CC list
- `get_extra_to()` — additional TO recipients (beyond SENIOR_MANAGER_EMAIL)
- `add(email, role)` / `remove(email)`
- `detect_command(text)` — parses "add X to CC" type commands from emails

**`AllowedSendersStore`**
Manages the Q&A sender whitelist. Persists to `data/maria_allowed_senders.json`.
- `get_all()` — returns `SENIOR_MANAGER_EMAIL` + `MARIA_ALLOWED_SENDERS` + dynamic list
- `add(email)` / `remove(email)`
- `detect_command(text)` — parses "add X to senders" type commands

**`_SystemAlertStore`**
In-memory cooldown guard for ops alert emails. Prevents same error from spamming every minute.
- `should_alert(error_type)` — returns True if cooldown period has passed
- `mark_sent(error_type)` — records current timestamp for error type
- Cooldown: `MARIA_ALERT_COOLDOWN_MINUTES` (default 30 min)

#### Alert State (`_alert_state`)

```python
# Structure in MariaBrain._alert_state:
{
    "OTD_WARNING": {
        "type": "OTD_WARNING",
        "tier": "warning",           # "warning" or "critical"
        "last_value": 62.4,          # current metric value
        "previous_value": 63.1,      # value at previous check
        "threshold": 65.0,           # breach threshold
        "first_detected": datetime,  # when this alert first appeared
        "details": "OTD dropped to 62.4% (threshold: 65.0%)"
    },
    "DELAY_ALERT": {
        "type": "DELAY_ALERT",
        "tier": "warning",
        "last_value": 87.3,          # avg delay in hours
        "previous_value": 82.1,
        "threshold": 72.0,
        "first_detected": datetime,
        "details": "Average delay 87.3h (threshold: 72.0h)"
    }
}
```

Alert types:
- `OTD_WARNING` — OTD below warning threshold (65%)
- `OTD_CRITICAL` — OTD below critical threshold (55%)
- `DELAY_ALERT` — average delay above warning threshold (72h)
- `DELAY_CRITICAL` — average delay above critical threshold (168h)
- `OVERDUE_WARNING` — overdue count above warning threshold (3500)
- `OVERDUE_CRITICAL` — overdue count above critical threshold (5000)

---

### `analyst.py` — MariaAnalyst

The analyst handles on-demand Q&A questions. It uses a three-step pipeline against the pre-cached CT DataFrames.

#### Pipeline

```
Question → PLANNER → EXECUTOR → NARRATOR → Answer
```

**Step 1: PLANNER (GPT-4o)**
```
Input: question + CT view descriptions
Prompt: "Given these 5 CT views, which view should I use to answer this question?
         What filters and groupings should I apply?"

Output (structured JSON):
{
  "view": "v5",                          # which CT view to use
  "filters": {"date_range": "last_7_days"},
  "group_by": ["truckername"],
  "metrics": ["otd_pct", "total_count"],
  "clarification_needed": false,         # if true → "outside scope" response
  "clarification_question": ""
}
```

**Step 2: EXECUTOR**
```python
# Fetches the planned CT view
df = await _fetch_ct_df(planned_view)

# Applies filters from plan
df = df[df["bookingdate"] >= cutoff]

# Computes metrics
result = df.groupby(group_by)[metrics].agg(...)
```

**Step 3: NARRATOR (GPT-4o)**
```
Input: original question + computed DataFrame (top 20 rows as text)
Prompt: "Write a concise operational summary answering this question based on this data."

Output:
{
  "answer": "Over the last 7 days, OTD performance across transporters shows...
             DHL leads with 89.3% OTD, while BlueDart at 71.2% is below average.",
  "key_findings": [
    "DHL leads with 89.3% OTD",
    "BlueDart below average at 71.2%",
    ...
  ]
}
```

**Alert Intercept (before PLANNER)**

Before going to MariaAnalyst, `answer_question()` checks if the question is about Maria's own alerts. If so, it answers directly from `_alert_state` — no DB query, no LLM needed:

```python
_ALERT_QUESTION_PATTERNS = re.compile(
    r"\b(alert|warning|critical|breach|threshold|daily report|morning brief|
         you mentioned|in your report|those alerts|the alerts)\b",
    re.IGNORECASE
)

if self._ALERT_QUESTION_PATTERNS.search(question):
    summary = self._answer_from_alert_state()
    # → direct reply without MariaAnalyst
```

---

### `scheduler.py` — MariaScheduler

Wraps APScheduler's `AsyncIOScheduler`. All 6 jobs are registered in `start()`.

```python
class MariaScheduler:
    def __init__(
        self,
        brain: MariaBrain,
        ct_refresh_callback: Optional[Callable] = None  # injected from main_complete.py
    ):
        self._scheduler = AsyncIOScheduler(timezone="Asia/Kolkata")
        self._brain = brain
        self._ct_refresh_callback = ct_refresh_callback

    def start(self):
        cfg = _load_ct_refresh_config()   # reads maria_config.json

        # 1. Morning Brief — Mon-Fri at 8 AM
        self._scheduler.add_job(
            self._brain.send_morning_brief,
            CronTrigger(hour=settings.MARIA_MORNING_BRIEF_HOUR, day_of_week="mon-fri"),
            id="maria_morning_brief"
        )

        # 2. Weekly Digest — Monday at 7 AM
        self._scheduler.add_job(
            self._brain.send_weekly_digest,
            CronTrigger(hour=7, day_of_week="mon"),
            id="maria_weekly_digest"
        )

        # 3. Anomaly Check — every 2 hours
        self._scheduler.add_job(
            self._brain.check_anomalies,
            IntervalTrigger(minutes=settings.MARIA_ANOMALY_CHECK_INTERVAL_MINUTES),
            id="maria_anomaly_check"
        )

        # 4. Alert Digest — Mon-Fri at 5 PM
        self._scheduler.add_job(
            self._brain.send_alert_digest,
            CronTrigger(hour=17, day_of_week="mon-fri"),
            id="maria_alert_digest"
        )

        # 5. IMAP Poll — every 5 minutes
        self._scheduler.add_job(
            self._imap_listener.poll_inbox,
            IntervalTrigger(seconds=settings.IMAP_POLL_INTERVAL_SECONDS),
            id="maria_imap_poll"
        )

        # 6. CT Refresh — nightly at configured hour
        if self._ct_refresh_callback and cfg["ct_refresh_enabled"]:
            self._scheduler.add_job(
                self._ct_refresh_callback,
                CronTrigger(hour=cfg["ct_refresh_hour"]),
                id="maria_ct_refresh"
            )

        self._scheduler.start()
```

**Reschedule (from UI config changes):**
When the user updates Maria's config via `PUT /api/maria/config`, `MariaScheduler.reschedule()` is called. It pauses, removes, and re-adds affected jobs without restarting the server.

---

### `imap_listener.py` — IMAPListener

Polls Gmail inbox every 5 minutes for incoming questions.

```python
async def poll_inbox(self) -> None:
    # Runs _fetch_unseen_from_allowed in a thread pool (blocking I/O)
    messages = await asyncio.to_thread(_fetch_unseen_from_allowed)

    for msg in messages:
        await self._brain.answer_question(
            question=msg["text"],
            reply_to_address=msg["from"],
            message_id=msg["message_id"],
            references=msg["references"],
            original_subject=msg["subject"],
        )
```

**Security:** Only processes messages from `AllowedSendersStore.get_all()`. All other messages are silently skipped (but marked as SEEN so they don't pile up).

**Reply Threading:** Uses RFC 2822 `In-Reply-To` and `References` headers so replies land in the same email thread in Gmail/Outlook.

---

### `email_service.py` — EmailService

**`send_email(to, subject, html_body, text_body, cc, in_reply_to, references)`**

SMTP-based email sender. Key behaviours:
- Aborts if `to` list is empty (guard added to prevent silent failures)
- Deduplicates CC against TO list (prevents double delivery)
- Falls back to plain text if HTML fails to render
- Returns `True` on success, `False` on SMTP error

**HTML Template Builders:**

| Function | Produces | Used For |
|----------|----------|---------|
| `build_morning_brief_html(date, kpis, alerts, view_summaries)` | Morning brief email | `send_morning_brief()` |
| `build_weekly_digest_html(date, kpis, trends, view_summaries)` | Weekly digest email | `send_weekly_digest()` |
| `build_alert_digest_html(date, alerts)` | EOD alert digest | `send_alert_digest()` |
| `build_qa_reply_html(question, summary, key_findings)` | Q&A reply | `answer_question()` |
| `build_system_ready_html(config_summary)` | System startup notification | `startup_event()` |

---

### `activity_store.py` — ActivityStore

Records every action Maria takes. Persists to `data/maria_activity.json` (last 500 events kept).

```python
# Usage pattern:
ev = activity_store.start("morning_brief", "Morning Brief")   # create event
ev.step("Loading CT views", "Fetching 5 views")               # add step
ev.step("Analysis complete", "OTD: 68.3%")
ev.complete(
    summary="Brief sent to 3 recipients",
    metadata={"recipients": 3, "alerts": 1}
)

# On failure:
ev.fail("SMTP connection refused", summary="Brief failed")
```

Each event is visible in the Maria Activity Tab on the frontend.

---

## Alert Threshold System

### Thresholds and Their Defaults

| Alert Type | Metric | Warning Threshold | Critical Threshold | `.env` Variable |
|------------|--------|------------------|--------------------|-----------------|
| OTD | OTD % | < 65% | < 55% | `MARIA_OTD_ALERT_THRESHOLD` / `MARIA_CRITICAL_OTD_THRESHOLD` |
| Delay | Avg delay hours | > 72h | > 168h (7 days) | `MARIA_DELAY_ALERT_HOURS` / `MARIA_CRITICAL_DELAY_HOURS` |
| Overdue | Count of overdue shipments | > 3500 | > 5000 | `MARIA_OVERDUE_COUNT_THRESHOLD` / `MARIA_CRITICAL_OVERDUE_COUNT` |

**Why these values?** Calibrated to the live baseline (April 2026): OTD ~70%, overdue ~2790. Warning = roughly 7% below normal. Critical = severe degradation requiring immediate action.

### Alert Lifecycle

```
check_anomalies() runs every 2 hours
    │
    ├─ Compute current metrics from v5 + v2 DataFrames
    │
    ├─ For each metric:
    │   ├─ Below critical threshold → create OTD_CRITICAL (or upgrade from WARNING)
    │   ├─ Below warning threshold → create OTD_WARNING
    │   └─ Within normal range → no alert
    │
    ├─ Update _alert_state:
    │   ├─ New alert → record first_detected, initial value
    │   ├─ Existing alert → update last_value, previous_value
    │   └─ Resolved alert → remove from state
    │
    └─ No email sent from anomaly_check itself
       → Alerts accumulate in _alert_state during the day
       → send_alert_digest() at 5 PM sends the EOD summary
       → Morning brief at 8 AM includes any overnight alerts
```

### System Alert Escalation (Ops Team)

Separate from business alerts. Fires when the pipeline itself breaks:

```
Error occurs in scheduler job / HTTP handler
    │
    ├─ _classify_error(exc) → (error_type, should_alert, severity)
    │   OPENAI_QUOTA_EXCEEDED → classify as OPENAI_QUOTA_EXCEEDED, alert=True
    │   OPENAI_AUTH_FAILED    → classify as OPENAI_AUTH_FAILED, alert=True
    │   DB_CONNECTION_FAILED  → classify as DB_CONNECTION_FAILED, alert=True
    │   Other                 → alert=False (log only)
    │
    ├─ _SystemAlertStore.should_alert(error_type) → check 30-min cooldown
    │
    └─ send_system_alert(exc, source, tb_str)
        Subject: "URGENT | Maria | OPENAI_QUOTA_EXCEEDED — morning_brief"
        To: MARIA_SUPPORT_EMAIL (support@innoctive.com)
        CC: MARIA_SUPPORT_CC (mubin@innoctive.com, deepesh@innoctive.com)
        Body: Error type + action hint + full traceback
```

---

## Configuration Reference

### `.env` Variables (Maria-Specific)

| Variable | Default | Description |
|----------|---------|-------------|
| `MARIA_ENABLED` | `false` | Master switch — must be `true` |
| `MARIA_DASHBOARD_URL` | — | URL shown in email links |
| `SENIOR_MANAGER_EMAIL` | — | Primary TO for all business emails |
| `MARIA_ALLOWED_SENDERS` | — | Comma-separated Q&A whitelist (besides SENIOR_MANAGER_EMAIL) |
| `MARIA_MORNING_BRIEF_HOUR` | `8` | Hour (0–23) for morning brief |
| `MARIA_WEEKLY_DIGEST_WEEKDAY` | `0` | 0=Mon, 1=Tue, ... 6=Sun |
| `MARIA_ANOMALY_CHECK_INTERVAL_MINUTES` | `120` | Minutes between anomaly checks |
| `MARIA_CC_EMAILS` | — | Static CC list (comma-separated) |
| `MARIA_SUPPORT_EMAIL` | — | Ops alert TO (URGENT emails) |
| `MARIA_SUPPORT_CC` | — | Ops alert CC (comma-separated) |
| `MARIA_ALERT_COOLDOWN_MINUTES` | `30` | Min minutes between same error type alerts |
| `MARIA_OTD_ALERT_THRESHOLD` | `65.0` | OTD warning threshold % |
| `MARIA_CRITICAL_OTD_THRESHOLD` | `55.0` | OTD critical threshold % |
| `MARIA_DELAY_ALERT_HOURS` | `72.0` | Delay warning threshold hours |
| `MARIA_CRITICAL_DELAY_HOURS` | `168.0` | Delay critical threshold hours |
| `MARIA_OVERDUE_COUNT_THRESHOLD` | `3500` | Overdue count warning |
| `MARIA_CRITICAL_OVERDUE_COUNT` | `5000` | Overdue count critical |
| `IMAP_POLL_INTERVAL_SECONDS` | `300` | Seconds between inbox polls |

### `data/maria_config.json` (Runtime, Editable via UI)

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
      "time_filter": {"preset": "last_7_days"}
    }
  }
}
```

---

## Common Operations

### Add a New Allowed Q&A Sender
Option A (UI): Configure tab → Allowed Senders → Add
Option B (Email): Send email to Maria saying "add john@example.com to senders"
Option C (API): `POST /api/maria/senders/add` with `{"email": "..."}`

### Add a New Email Recipient (CC)
Option A (UI): Configure tab → Recipients → Add CC
Option B (Email): Send "add sarah@example.com to CC"
Option C (API): `POST /api/maria/subscriptions/add` with `{"email": "...", "role": "cc"}`

### Temporarily Disable Maria
`.env` → set `MARIA_ENABLED=false` → restart server
Or via `PUT /api/maria/config` → set schedule hours to off-hours temporarily

### Reset Alert Cooldowns
Delete `data/maria_last_alert_fired.json` → next error will send alert immediately

### Silence a Specific Alert Type
Raise the threshold in `data/maria_config.json` → no restart needed
e.g. change `overdue_count_warning` from 3500 to 4500 to suppress current overdue alert
