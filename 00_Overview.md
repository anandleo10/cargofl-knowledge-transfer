# CargoFL Research Agent — Product Overview

## What Is This?

The **CargoFL Research Agent** is an AI-powered logistics intelligence platform built for the CargoFL / PUMA operations team. It allows operations managers and analysts to ask natural-language questions about shipment data, get instant answers in charts and summaries, and receive proactive daily briefings — all without writing a single line of SQL.

The system combines a **Research Dashboard** (query-on-demand) with **Maria**, a proactive AI agent that monitors KPIs around the clock, sends daily briefs, weekly digests, EOD alert summaries, and answers email questions.

---

## The Problem It Solves

| Before | After |
|--------|-------|
| Analyst writes SQL queries manually | Natural language → SQL → Chart automatically |
| Manager waits for weekly reports | Daily 8 AM morning brief via email |
| Delays spotted after the fact | Anomaly check every 2 hours, alert if OTD drops or delays spike |
| Questions go to analysts as ad-hoc requests | Email any question to Maria; get a reply in minutes |
| Dashboard setup takes days | Control Tower views pre-built, filterable in seconds |

---

## Who Uses It

| Role | How They Use It |
|------|----------------|
| **Senior Manager (Malini)** | Receives morning briefs, weekly digests, EOD alerts; replies to Maria via email for ad-hoc questions |
| **Operations Analysts** | Run queries via Research Dashboard; build and save dashboards; use Control Tower for shipment tracking |
| **IT / Support** | Deploy via Docker, manage `.env` config, respond to URGENT system alert emails |
| **CargoFL Dev Team** | Maintain and extend the codebase |

---

## Capability Map

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        CargoFL Research Agent                                │
│                                                                              │
│  ┌────────────────────────────┐    ┌──────────────────────────────────────┐  │
│  │   RESEARCH DASHBOARD       │    │   MARIA — PROACTIVE AI AGENT         │  │
│  │                            │    │                                      │  │
│  │  • Natural language query  │    │  • Daily morning brief (8 AM)        │  │
│  │  • Auto SQL generation     │    │  • Weekly performance digest (Mon)   │  │
│  │  • Interactive charts      │    │  • EOD alert digest (5 PM)           │  │
│  │  • Save dashboards         │    │  • Anomaly check every 2 hrs         │  │
│  │  • Export Excel / PDF      │    │  • Email Q&A (reply in thread)       │  │
│  │  • AI panel summaries      │    │  • Nightly CT data refresh (11 PM)   │  │
│  └────────────────────────────┘    └──────────────────────────────────────┘  │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │   CONTROL TOWER (5 Pre-Built Dashboard Views)                        │    │
│  │                                                                      │    │
│  │  v1: Overview  v2: In-Transit  v3: Delayed  v4: Future EDD  v5: OTD │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## The 4 Main Tabs (UI Walkthrough)

### Tab 1 — Analysis (Research Dashboard)
The main query interface. A user types a question like:
> "Show me the top 10 transporters by OTD percentage for the last 30 days"

The system:
1. Checks for safety (guardrail)
2. Clarifies if ambiguous
3. Plans the SQL query
4. Executes against MySQL
5. Builds charts
6. Returns a dashboard with findings

Users can save the result as a named dashboard or export it to Excel.

### Tab 2 — Control Tower
Five pre-built live views into shipment data, refreshed from DB on demand or nightly:
- **v1 Overview** — total shipments, breakdown by branch/transporter/status
- **v2 In-Transit** — undelivered shipments, today's EDD, delayed non-delivered
- **v3 Delayed Deep-Dive** — root cause analysis, responsible transporter, delay hours
- **v4 Future EDD** — day-by-day upcoming delivery schedule
- **v5 Delivered Performance** — OTD%, early/on-time/late splits by month/branch

Each view has interactive filters (date range, branch, transporter, etc.) and can be exported to Excel.

### Tab 3 — Maria Activity Log
A live audit trail of everything Maria has done: briefs sent, alerts fired, anomalies detected, Q&A answered, system errors. Each entry shows steps taken, timing, and outcome.

### Tab 4 — Configure (Maria Settings)
Manage Maria without redeploying:
- Adjust schedule times (morning brief hour, digest weekday, CT refresh hour)
- Tune alert thresholds (OTD warning/critical, delay hours, overdue count)
- Add/remove email recipients (TO and CC)
- Manage allowed Q&A senders
- Toggle features on/off

---

## Key Numbers (Live Baseline, April 2026)

| Metric | Typical Value |
|--------|--------------|
| Total bookings in DB | ~573,000 rows |
| OTD % (on-time delivery) | ~70% |
| Overdue shipments | ~2,790 |
| Morning brief sends | Mon–Fri at 8 AM IST |
| Anomaly check cadence | Every 2 hours |
| Email Q&A turnaround | Under 2 minutes |

---

## Technology at a Glance

```
Backend:   Python 3.11 · FastAPI · LangGraph · CrewAI (optional) · OpenAI GPT-4o
Database:  MySQL on AWS RDS (cargofl.tlbooking, 573K rows)
Cache:     Redis + in-memory view_cache (per CT view)
Email:     Gmail SMTP (outbound) + Gmail IMAP (inbound Q&A)
Frontend:  React 18 · TypeScript · Vite · Recharts · shadcn/ui
Scheduler: APScheduler (AsyncIOScheduler, Asia/Kolkata timezone)
Deploy:    Docker Compose — api (port 8000) + frontend (nginx port 80) + redis
```

---

## Where to Go Next

| If you want to understand… | Read… |
|---------------------------|-------|
| How all the pieces fit together | `01_TechStack_Architecture.md` |
| The codebase layout | `02_FolderStructure.md` |
| How a query actually works end-to-end | `03_DataFlow_Workflows.md` |
| Every API endpoint | `04_APIReference.md` |
| The Control Tower views | `05_ControlTower.md` |
| Maria in depth | `06_MariaAgent.md` |
| The React frontend | `07_Frontend.md` |
| How to deploy or restart | `08_DeploymentOps.md` |
| Every config variable | `09_Configuration.md` |
| Visual system diagrams | `10_Diagrams/` |
| Domain & code terms | `GLOSSARY.md` |
