# Diagram: System Architecture

## Full System Overview (Mermaid)

```mermaid
graph TB
    subgraph USER["Users"]
        OM[Operations Manager<br/>Browser]
        SM[Senior Manager<br/>Email Client]
    end

    subgraph DOCKER["Docker Compose Stack — ai-research.cargofl.com"]
        subgraph FE["Frontend Container (Port 80)"]
            NGINX[nginx:1.27-alpine]
            REACT[React 18 + TypeScript<br/>Vite Build · Recharts · shadcn/ui]
        end

        subgraph API["API Container (Port 8000)"]
            FASTAPI[FastAPI + Uvicorn<br/>main_complete.py]
            LG[LangGraph<br/>StateGraph Workflow]
            MARIA[MariaBrain<br/>APScheduler 6 Jobs]
            ANALYST[MariaAnalyst<br/>Planner→Executor→Narrator]
        end

        REDIS[(Redis<br/>:6379)]
    end

    subgraph EXTERNAL["External Services"]
        RDS[(AWS RDS MySQL<br/>cargofl.tlbooking<br/>~573K rows)]
        OPENAI[OpenAI GPT-4o<br/>API]
        GMAIL_OUT[Gmail SMTP<br/>maria.puma@cargofl.com<br/>Outbound]
        GMAIL_IN[Gmail IMAP<br/>maria.puma@cargofl.com<br/>Inbound]
    end

    OM -->|HTTPS Port 80| NGINX
    NGINX -->|Serve React SPA| REACT
    REACT -->|POST /api/research/start| FASTAPI
    REACT -->|GET /api/ct/...| FASTAPI
    REACT -->|PUT /api/maria/config| FASTAPI

    FASTAPI --> LG
    FASTAPI --> MARIA
    FASTAPI --> REDIS

    LG -->|SQL Queries| RDS
    LG -->|Planning/Summarize| OPENAI

    MARIA -->|CT Data Queries| RDS
    MARIA -->|Q&A Narrative| OPENAI
    MARIA -->|Send Email| GMAIL_OUT
    MARIA -->|Poll Inbox| GMAIL_IN
    MARIA --> ANALYST

    ANALYST -->|CT DataFrame Q&A| RDS
    ANALYST -->|Narrate Answer| OPENAI

    SM -->|Receives briefs/alerts| GMAIL_OUT
    SM -->|Replies with questions| GMAIL_IN

    style DOCKER fill:#f0f4ff,stroke:#4a6cf7
    style EXTERNAL fill:#fff8f0,stroke:#f7a44a
    style USER fill:#f0fff4,stroke:#4af77a
```

---

## Simplified Component Interaction

```
┌──────────────────────────────────────────────────────────────────────────┐
│  BROWSER (Operations Manager)                                            │
│  4 Tabs: Analysis | Control Tower | Activity | Configure                 │
└──────────────────┬───────────────────────────────────────────────────────┘
                   │ HTTP (port 80 → nginx → port 8000 → FastAPI)
                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  FASTAPI  (main_complete.py — 6500+ lines)                               │
│                                                                          │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────┐  │
│  │  Research        │  │  Control Tower    │  │  Maria                  │  │
│  │  /api/research/* │  │  /api/ct/*        │  │  /api/maria/*           │  │
│  │  /api/dashboard/*│  │  view_cache       │  │  APScheduler            │  │
│  │                  │  │  5 CT views       │  │  6 scheduled jobs       │  │
│  │  LangGraph       │  │  Derived columns  │  │  MariaBrain             │  │
│  │  8-node workflow │  │                   │  │  IMAPListener           │  │
│  └────────┬─────────┘  └────────┬──────────┘  └───────────┬────────────┘  │
│           │                     │                          │              │
└───────────┼─────────────────────┼──────────────────────────┼──────────────┘
            │                     │                          │
     ┌──────▼──────┐        ┌─────▼──────┐         ┌────────▼────────┐
     │  OpenAI      │        │  MySQL RDS  │         │  Gmail          │
     │  GPT-4o      │        │  tlbooking  │         │  SMTP + IMAP    │
     └─────────────┘        └─────────────┘         └─────────────────┘
```

---

## Container Networking

```
Internet
   │
   │ :80
   ▼
┌──────────────────────────────────────────────────────────┐
│  Docker Network: research-agent-complete_default         │
│                                                          │
│  ┌──────────────┐        ┌────────────────┐             │
│  │  frontend    │        │  api           │             │
│  │  :80 (nginx) │───────▶│  :8000 (uvicorn)│            │
│  └──────────────┘        └───────┬────────┘             │
│                                  │                       │
│                          ┌───────▼────────┐              │
│                          │  redis         │              │
│                          │  :6379         │              │
│                          └────────────────┘              │
└──────────────────────────────────────────────────────────┘
   │
   │ EXTERNAL (outbound from api container)
   ├── RDS MySQL: cargofl-puma-sync.c52ewqoqkh1q.ap-south-1.rds.amazonaws.com:3306
   ├── OpenAI API: api.openai.com:443
   └── Gmail: smtp.gmail.com:587 / imap.gmail.com:993
```
