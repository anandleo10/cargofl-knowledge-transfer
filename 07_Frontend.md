# Frontend Architecture

## Overview

The frontend is a single-page React application (SPA) built with TypeScript and Vite. It communicates exclusively with the FastAPI backend at `/api/*`. There is no SSR — everything is client-side rendered after the initial HTML/JS load.

**Tech:**
- React 18 + TypeScript 5
- Vite 7 (build + dev server)
- Tailwind CSS (utility classes)
- shadcn/ui (accessible component library built on Radix UI)
- Recharts (declarative chart library)
- Native `fetch` API (type-safe wrapper in `api.ts`)

---

## Application Structure

```
App.tsx
  └── AppImpl.tsx          ← Main app logic
        ├── Tab: Analysis  ← DashboardBuilder + SavedDashboards
        ├── Tab: CT v1–v5  ← CT view panels (per view)
        ├── Tab: Activity  ← MariaActivityTab
        └── Tab: Configure ← MariaConfigTab
```

### Entry Points

**`main.tsx`**
```typescript
import ReactDOM from 'react-dom/client'
import App from './App'

ReactDOM.createRoot(document.getElementById('root')!).render(<App />)
```

**`App.tsx`** — thin wrapper, loads `AppImpl`

**`AppImpl.tsx`** — the real app:
- Holds top-level state (active tab, Maria status, alert counts)
- Renders the tab navigation bar
- Loads each tab component on demand
- Polls `GET /api/maria/status` every 30s to show alert badge counts

---

## The 4 Tabs

### Tab 1 — Analysis

**Components:** `DashboardBuilder` + `SavedDashboards`

**DashboardBuilder** (`components/dashboard/DashboardBuilder.tsx`)
This is the main query interface. User flow:

```
1. User types query in textarea
2. Selects data source (Database or Excel file)
3. Selects optional time filter
4. Clicks "Run Analysis"
   → POST /api/research/start
   → Starts polling GET /api/research/status/{request_id} every 2s
5. Progress steps shown in real-time as they arrive
6. On completion: renders DashboardViewer with results
7. Options: Save Dashboard / Download Report / Export Excel
```

**Progress Display:**
```typescript
// Each step shown as it arrives:
progressEvents.map(ev => (
  <div key={ev.step}>
    <span className="step-label">{ev.step}</span>
    <span className="step-message">{ev.message}</span>
    <span className="step-duration">{ev.duration_ms}ms</span>
  </div>
))
```

**Clarification flow:**
If the backend returns `status: "awaiting_clarification"`, a dialog pops up with the clarifying questions. User answers → `POST /api/research/clarify` → workflow resumes.

**SavedDashboards** (`components/dashboard/SavedDashboards.tsx`)
- Lists all saved dashboards: `GET /api/dashboard/list`
- Click to load: `GET /api/dashboard/{id}` → renders in DashboardViewer
- Delete: `DELETE /api/dashboard/{id}`

---

### Tab 2 — Control Tower

**Per-view components** (one set for each of v1–v5):

Each CT view tab renders a set of panels built from `POST /api/dashboard/create-control-tower-v{N}`.

**Filter Panel (sidebar):**
- Branch multi-select
- Transporter multi-select
- Date range picker / preset selector
- "Reload from DB" button → `POST /api/dashboard/refresh-cache/{view}`

**Panel Grid:**
- Grid of `DashboardPanel` components
- Each panel: chart + title + optional AI summary
- Summary generated on demand: `POST /api/dashboard/{id}/summarize`

---

### Tab 3 — Maria Activity Log

**Component:** `MariaActivityTab` (`components/maria/MariaActivityTab.tsx`)

Displays the Maria activity log — a real-time audit trail of everything Maria has done.

**What it shows:**
- List of activity events (morning_brief, anomaly_check, qa_answered, etc.)
- Each event: type badge, title, start time, duration, status (completed/failed)
- Expandable: shows individual steps taken within each event
- Expandable: shows metadata (recipients, finding count, etc.)

**Data source:** `GET /api/maria/activity` (polls every 30s while tab is open)

**Event types and their icons:**
| Type | Icon | Meaning |
|------|------|---------|
| `morning_brief` | ☀️ | Morning brief email sent |
| `weekly_digest` | 📊 | Weekly digest email sent |
| `anomaly_check` | 🔍 | KPI check — all clear or alerts found |
| `alert_digest` | 🔔 | EOD alert digest sent |
| `qa_answered` | 💬 | Email Q&A answered |
| `ct_refresh` | 🔄 | CT data refreshed from DB |
| `system_alert` | 🚨 | Critical system error alert sent |

---

### Tab 4 — Configure

**Component:** `MariaConfigTab` (`components/admin/MariaConfigTab.tsx`)

The admin panel for Maria configuration. All changes are persisted immediately via `PUT /api/maria/config`.

**Sections:**

**1. Email Schedule**
- Morning Brief: time picker (hour) + days toggle (Mon–Fri)
- Weekly Digest: day selector
- Anomaly Check: interval (minutes)
- EOD Alert Digest: time picker
- CT Data Refresh: toggle + hour picker

**2. Alert Thresholds**
- OTD Warning % (number input)
- OTD Critical % (number input)
- Delay Warning (hours)
- Delay Critical (hours)
- Overdue Count Warning
- Overdue Count Critical

**3. Recipients**
- Table of current TO/CC recipients
- Add recipient form (email + role: TO or CC)
- Remove button per row

**4. Allowed Q&A Senders**
- Table of whitelisted senders
- Add/remove form

**5. Manual Triggers**
- "Send Morning Brief Now" button → `POST /api/maria/brief`
- "Run Anomaly Check Now" button → `POST /api/maria/check`
- "Send Alert Digest Now" button → `POST /api/maria/alert-digest`
- "Send Demo Email" button → `POST /api/maria/demo-email`

---

## Key Components Reference

### DashboardViewer (`components/dashboard/DashboardViewer.tsx`)

Renders a complete dashboard: a grid of `DashboardPanel` components.

```typescript
interface DashboardConfig {
  panels: Panel[]
  title: string
  run_id: string
}

interface Panel {
  id: string
  title: string
  chart_spec: ChartSpec
  summary?: string
  position: { x: number, y: number, w: number, h: number }
}
```

Uses a responsive grid layout (React-grid-layout or CSS Grid) to arrange panels.

### DashboardPanel (`components/dashboard/DashboardPanel.tsx`)

Single panel component:
- Renders title
- Renders `ChartRenderer` with chart_spec
- Shows AI summary if available
- "Generate Insight" button → `POST /api/dashboard/insight`
- "Download Data" button → exports panel data as CSV

### ChartRenderer (`components/dashboard/ChartRenderer.tsx`)

Routes a chart_spec to the correct Recharts component:

```typescript
function ChartRenderer({ spec }: { spec: ChartSpec }) {
  switch (spec.chart_type) {
    case "bar":       return <BarChartView spec={spec} />
    case "line":      return <LineChartView spec={spec} />
    case "pie":       return <PieChartView spec={spec} />
    case "scatter":   return <ScatterChartView spec={spec} />
    case "table":     return <DataTableView spec={spec} />
    case "heatmap":   return <HeatmapView spec={spec} />
    case "grouped_bar": return <GroupedBarView spec={spec} />
    default:          return <DataTableView spec={spec} />
  }
}
```

**Chart Spec format (from backend):**
```typescript
interface ChartSpec {
  chart_type: "bar" | "line" | "pie" | "scatter" | "table" | "heatmap" | "grouped_bar"
  data: Record<string, any>[]           // array of data rows
  data_key: string                       // x-axis / category key
  metrics: Array<{
    key: string                          // y-axis data key
    name: string                         // display label
    color: string                        // hex color
  }>
  title: string
  x_label?: string
  y_label?: string
}
```

### MariaChatWidget (`components/maria/MariaChatWidget.tsx`)

Inline chat interface for direct Q&A with Maria (without going through email).

```
User types question → POST /api/maria/chat
                    → Returns {task_id}
                    → Polls GET /api/maria/chat/{task_id} every 2s
                    → Shows answer + key_findings when complete
```

---

## API Client (`services/api.ts`)

ALL backend calls go through this file. No ad-hoc `fetch()` calls elsewhere.

```typescript
// Example exports:
export async function startResearch(req: ResearchRequest): Promise<{request_id: string}>
export async function getResearchStatus(id: string): Promise<ResearchResponse>
export async function createControlTower(view: CTView, filters: CTFilters): Promise<DashboardConfig>
export async function refreshCTCache(view: CTView): Promise<CacheRefreshResult>
export async function getMariaConfig(): Promise<MariaConfig>
export async function updateMariaConfig(config: Partial<MariaConfig>): Promise<MariaConfig>
export async function getMariaActivity(params: ActivityQueryParams): Promise<ActivityResponse>
export async function triggerMariaBrief(): Promise<{status: string}>
// ... ~40 more functions
```

**Base URL resolution:**
```typescript
const API_BASE = import.meta.env.VITE_API_BASE || "http://localhost:8000"
```
- Dev: reads from `frontend/.env` → `http://localhost:8000`
- Production build: reads from `frontend/.env.production` → `http://ai-research.cargofl.com:8000`

---

## Build & Dev

### Development
```bash
cd frontend
npm install
npm run dev
# → Vite dev server at http://localhost:5173
# → API proxied to http://localhost:8000 via vite.config.ts
```

### Production Build
```bash
npm run build
# → TypeScript check (tsc -b)
# → Vite build → dist/
# → dist/ served by nginx in container
```

### Nginx Config (Dockerfile.frontend)
```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;

  # Serve React app
  location / {
    try_files $uri $uri/ /index.html;   # SPA routing
  }

  # Proxy API to backend container
  location /api/ {
    proxy_pass http://api:8000;         # "api" = Docker service name
  }
}
```

---

## State Management

The frontend uses **local component state** (React hooks) — no Redux or Zustand. Global state is minimal:

| State | Lives In | Purpose |
|-------|---------|---------|
| Active tab | `AppImpl.tsx` | Which tab is shown |
| Maria status | `AppImpl.tsx` | Alert badge count, job next-run times |
| Current analysis | `DashboardBuilder.tsx` | Active request_id, progress events, result |
| Saved dashboards list | `SavedDashboards.tsx` | List of all saved dashboards |
| CT filters | Per-CT-view component | Current filter selections |
| Config form | `MariaConfigTab.tsx` | Local form state before save |

This means each tab is relatively independent. Switching tabs does not lose state — components are mounted/unmounted but state is preserved via React's reconciliation while they remain in the tree.

---

## Adding a New Feature

### New chart type
1. Add to `ChartSpec.chart_type` union in `api.ts`
2. Add case to `ChartRenderer.tsx`
3. Create new Recharts component

### New Maria config field
1. Add to `MariaConfig` interface in `api.ts`
2. Add input control in `MariaConfigTab.tsx`
3. Add to backend `settings.py` and `_load_maria_config()` defaults

### New CT view tab
1. Add `"v6"` to CT view type in `api.ts`
2. Create new tab in the CT section of `AppImpl.tsx`
3. Add `createControlTowerV6()` to `api.ts`
4. Ensure backend has `POST /api/dashboard/create-control-tower-v6`
