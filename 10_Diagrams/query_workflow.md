# Diagram: Query Workflow (LangGraph Pipeline)

## End-to-End Query Flow

```mermaid
sequenceDiagram
    actor User as User (Browser)
    participant FE as React Frontend
    participant API as FastAPI
    participant GR as GuardrailAgent
    participant CL as ClarificationAgent
    participant PL as PlanningAgent (GPT-4o)
    participant EX as ExecutionAgent
    participant DB as MySQL RDS
    participant VZ as VisualizationAgent
    participant DD as DashboardDesignAgent
    participant SU as SummarizerAgent (GPT-4o)

    User->>FE: Types query + clicks Run
    FE->>API: POST /api/research/start
    API->>API: Create request_id, launch async task
    API-->>FE: {request_id, status: "pending"}

    loop Poll every 2s
        FE->>API: GET /api/research/status/{id}
        API-->>FE: {status: "running", progress_events: [...]}
    end

    Note over API,SU: LangGraph StateGraph executes nodes

    API->>GR: Node 1: Guardrail check
    GR->>GR: Check PII, SQL injection, forbidden tables
    alt Unsafe query
        GR-->>API: {safe: false, reason: "..."}
        API-->>FE: {status: "error", error: "..."}
    else Safe
        GR-->>API: {safe: true}
    end

    API->>CL: Node 2: Clarification (if ambiguous)
    CL->>CL: GPT-4o: "Is this query clear?"
    alt Clarification needed
        CL-->>FE: {status: "awaiting_clarification", questions: [...]}
        User->>FE: Provides answers
        FE->>API: POST /api/research/clarify
        API->>CL: Resume with answers
    end

    API->>PL: Node 3: Planning
    PL->>PL: Load schema context (table.yaml + data_context.json)
    PL->>PL: GPT-4o: NL → ExecutionPlan (SQL steps)
    PL-->>API: ExecutionPlan [{sql, operation_type, expected_columns}]

    API->>EX: Node 4: Execution
    EX->>DB: Execute SQL queries (via SQLAlchemy, asyncio.to_thread)
    DB-->>EX: Result sets (up to 100K rows)
    EX->>EX: Convert to Pandas DataFrames
    EX-->>API: {step_name: DataFrame, ...}

    API->>VZ: Node 5: Visualization
    VZ->>VZ: Inspect column types (numeric, categorical, datetime)
    VZ->>VZ: Auto-select chart type (bar/line/pie/scatter/table)
    VZ->>VZ: Build Recharts JSON spec
    VZ-->>API: [ChartSpec, ...]

    API->>DD: Node 6: Dashboard Design (GPT-4o)
    DD->>DD: Group charts into panels, assign grid positions
    DD-->>API: DashboardConfig {panels: [...]}

    API->>SU: Node 7: Summarization (GPT-4o)
    SU->>SU: Summarize each panel (1-3 sentences)
    SU->>SU: Generate overall key_findings
    SU-->>API: {panel_summaries: [...], key_findings: [...]}

    API-->>FE: {status: "completed", result: DashboardConfig}
    FE->>User: Renders charts + findings
    User->>FE: Clicks "Save Dashboard"
    FE->>API: POST /api/dashboard/save
    API-->>FE: {dashboard_id: "uuid"}
```

---

## LangGraph State Machine

```
                    ┌─────────┐
                    │  START  │
                    └────┬────┘
                         │
                         ▼
               ┌─────────────────┐
               │   GUARDRAIL     │
               │ Safety check     │
               └────────┬────────┘
                        │
              ┌─────────┴──────────┐
              │ safe?              │
         No ──┤                    ├── Yes
              │                    │
              ▼                    ▼
           ┌─────┐        ┌────────────────┐
           │ERROR│        │  CLARIFICATION │
           └─────┘        │ (conditional)   │
                          └────────┬───────┘
                                   │
                      ┌────────────┴────────────┐
                      │ needs clarification?      │
                 Yes ─┤                           ├─ No / answered
                      │                           │
                      ▼                           ▼
              ┌──────────────┐         ┌──────────────────┐
              │ WAIT FOR     │         │    PLANNING       │
              │ USER INPUT   │         │  NL → SQL plan    │
              └──────┬───────┘         └────────┬─────────┘
                     │                          │
                     │ (answers provided)        ▼
                     └─────────────────▶ ┌──────────────────┐
                                         │    EXECUTION      │
                                         │  SQL → DataFrames │
                                         └────────┬─────────┘
                                                  │
                                         ┌────────┴─────────┐
                                         │ SQL error?        │
                                    Yes ─┤                    ├─ No
                                         │                    │
                                         ▼                    ▼
                                    ┌────────┐     ┌──────────────────┐
                                    │ RETRY  │     │  VISUALIZATION   │
                                    │ (once) │     │  Chart specs      │
                                    └────────┘     └────────┬─────────┘
                                                            │
                                                   ┌────────▼─────────┐
                                                   │  DASHBOARD DESIGN │
                                                   │  Panel layout     │
                                                   └────────┬─────────┘
                                                            │
                                                   ┌────────▼─────────┐
                                                   │  SUMMARIZATION    │
                                                   │  AI findings       │
                                                   └────────┬─────────┘
                                                            │
                                                   ┌────────▼─────────┐
                                                   │  REPORT GEN       │
                                                   │  HTML report       │
                                                   └────────┬─────────┘
                                                            │
                                                   ┌────────▼─────────┐
                                                   │     COMPLETE      │
                                                   └──────────────────┘
```

---

## Data Transformation at Each Step

```
User Query (string)
    │
    ▼ [Planning Agent + GPT-4o]
ExecutionPlan:
  steps = [
    {name: "transporter_otd", sql: "SELECT ...", expected_cols: [...]},
    {name: "branch_breakdown", sql: "SELECT ...", expected_cols: [...]}
  ]
    │
    ▼ [Execution Agent + MySQL]
Results:
  {
    "transporter_otd": pd.DataFrame(143 rows × 4 cols),
    "branch_breakdown": pd.DataFrame(12 rows × 3 cols)
  }
    │
    ▼ [Visualization Agent]
ChartSpecs:
  [
    {chart_type: "bar", data_key: "truckername", metrics: [{key: "otd_pct"}], data: [...]},
    {chart_type: "pie", data_key: "branch", metrics: [{key: "count"}], data: [...]}
  ]
    │
    ▼ [Dashboard Design Agent + GPT-4o]
DashboardConfig:
  panels: [
    {id: "p1", title: "Transporter OTD Performance", chart_spec: {...}, position: {x:0,y:0,w:8,h:4}},
    {id: "p2", title: "Branch Distribution", chart_spec: {...}, position: {x:8,y:0,w:4,h:4}}
  ]
    │
    ▼ [Summarizer Agent + GPT-4o]
Final Output:
  {
    dashboard_config: DashboardConfig,
    panel_summaries: [{panel_id: "p1", summary: "DHL leads with 89% OTD..."}],
    key_findings: ["DHL leads at 89%", "Chennai branch underperforming at 61%"],
    summary: "Overall OTD of 73.4% with DHL as top performer..."
  }
```
