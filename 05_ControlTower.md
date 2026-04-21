# Control Tower (CT) — Deep Dive

The Control Tower is the pre-built, live logistics dashboard. Unlike the Research Dashboard (where users craft custom queries), the CT offers five fixed analytical views into the `tlbooking` table — each answering a specific operational question.

---

## What Is the Control Tower?

The CT is a set of **5 parameterised views** that query the same source table (`tlbooking`) but shape the data differently for specific operational purposes:

| View | Name | "What does it answer?" |
|------|------|------------------------|
| v1 | Overview | How many shipments do we have in total, and what's the breakdown? |
| v2 | In-Transit Analysis | Which shipments are currently moving, which are delayed, and what's due today? |
| v3 | Delayed Deep-Dive | Why are shipments delayed? Who is responsible? |
| v4 | Future EDD | What deliveries are coming up this week? Day-by-day schedule. |
| v5 | Delivered Performance | How well are we doing on On-Time Delivery (OTD%)? |

---

## Database Source: `tlbooking`

The underlying table that all CT views query.

### Key Columns

| Column | Type | Description |
|--------|------|-------------|
| `bookingid` | VARCHAR | Unique booking/shipment ID |
| `bookingdate` | DATE | Date booking was created (primary time filter column) |
| `truckername` | VARCHAR | Transporter/LSP name (e.g., DHL, BlueDart) |
| `schdeliverydate` | DATE | Scheduled delivery date (EDD — Expected Delivery Date) |
| `actualdeliverydate` | DATE | Actual delivery date (NULL if not yet delivered) |
| `lsp_status` | VARCHAR | Delivery status: "Delivered", "In Transit", "Pending", etc. |
| `pod_lsp_status` | VARCHAR | POD (Proof of Delivery) upload status: "Y" or "N" |
| `branch` | VARCHAR | Source/origin branch (Delhi, Mumbai, Chennai, etc.) |
| `lrtype` | VARCHAR | Shipment type: "PTL" (Part Truck Load) or "FTL" (Full Truck Load) |
| `gross_total` | DECIMAL | Booking value (freight amount) |
| `packets` | INT | Number of packages/boxes in the shipment |
| `item_count` | INT | Number of items |
| `destination` | VARCHAR | Delivery destination |

### Derived Columns (Added by `_apply_ct_derived_columns`)

These columns do NOT exist in the DB — they are computed in Python on the fetched DataFrame:

| Column | Computed As | Used In |
|--------|------------|---------|
| `is_delivered` | `lsp_status == "Delivered"` | All views |
| `delivery_status` | More granular: "Delivered-OT" / "Delivered-Late" / "In Transit" / "Overdue" | v5 |
| `is_intransit` | `NOT is_delivered AND lrtype IN ('PTL','FTL')` | v2, v3 |
| `overdue_days` | `(today - schdeliverydate).days` (only for in-transit) | v2, v3 |
| `is_delayed_non_delivered` | `is_intransit AND overdue_days > 0` | v2 |
| `is_todays_edd` | `schdeliverydate == today` | v2, v4 |
| `is_future_edd` | `schdeliverydate > today` | v2, v4 |
| `edd_category` | "Today" / "Tomorrow" / "This Week" / "Next Week" / "Later" | v4 |
| `delay_hours` | `overdue_days * 24` (capped at 720h = 30 days) | Maria anomaly check |
| `delivery_month` | `actualdeliverydate.strftime("%Y-%m")` | v5 |
| `pod_delay_days` | `(actualdeliverydate - schdeliverydate).days` | v5 |
| `service_performance` | "On Time" / "Early" / "Late 1-3d" / "Late 4-7d" / "Late 7d+" | v5 |

> **Important:** The 30-day cap on `delay_hours` exists to exclude AIR TRANSPORT outliers that have extreme delay values (100+ days) and would skew Maria's anomaly check calculations.

---

## The View Cache

CT views query up to 573K rows. Running this on every page load would be too slow and put heavy load on RDS. The cache solves this.

### How It Works

```python
# In main_complete.py
view_cache: dict[str, pd.DataFrame] = {}          # in-memory cache
_ct_cache_locks: dict[str, asyncio.Lock] = {}     # one lock per view

async def _fetch_ct_df(view: str, force: bool = False) -> pd.DataFrame:
    if not force and view in view_cache:
        return view_cache[view]                   # cache hit

    async with _ct_cache_locks[view]:             # prevent duplicate DB hits
        if not force and view in view_cache:
            return view_cache[view]               # double-check after lock

        df = await asyncio.to_thread(_run_ct_sql, view)  # hits MySQL
        view_cache[view] = df                             # store in cache
        return df
```

### Cache Invalidation

The cache is cleared when:
1. **Manual Reload** — User clicks "Reload from DB" → `POST /api/dashboard/refresh-cache/{view}`
2. **Nightly CT Refresh** — Maria's `ct_refresh` job runs at 11 PM IST (calls `force=True` for all views)
3. **Server restart** — `view_cache` is in-memory, so a restart = cold cache (first request after restart hits DB)

### Why Session Eviction Matters

When a user saves a dashboard built on a CT view, their session (`active_sessions[run_id]`) stores the DataFrame from that moment. If the cache is refreshed but the session still holds the old DataFrame, "Reload from DB" wouldn't help.

Fix: When `refresh-cache` is called, any `active_sessions` entries whose `run_id` maps to a saved dashboard using that CT view are evicted. The next panel render triggers a fresh `_get_result_df()` lookup, which falls through to the freshly-loaded `view_cache`.

---

## View Details

### v1 — Overview

**Purpose:** Bird's-eye status of all shipments.

**Key panels:**
- Total shipment count
- Status breakdown (Delivered / In Transit / Pending / Overdue)
- Branch-wise distribution
- Transporter-wise volume
- Monthly trend of bookings

**SQL Base:**
```sql
SELECT
  bookingid, bookingdate, truckername, branch,
  schdeliverydate, actualdeliverydate,
  lsp_status, pod_lsp_status, lrtype,
  gross_total, packets, item_count, destination
FROM tlbooking
WHERE bookingdate BETWEEN :start_date AND :end_date
```

---

### v2 — In-Transit Analysis

**Purpose:** Current state of all moving shipments.

**Key panels:**
- In-transit count by transporter
- Today's EDD (shipments due today)
- Delayed non-delivered (past EDD, still in transit)
- Overdue heatmap by transporter × branch
- EDD category distribution (Today / This Week / Later)
- Pieces and packages counts (heatmap levels)

**Derived column highlights:**
- `is_todays_edd` — flags shipments with EDD = today (urgency indicator)
- `edd_category` — groups future deliveries into time buckets for planning

---

### v3 — Delayed Deep-Dive

**Purpose:** Root cause analysis of delays.

**Key panels:**
- Top delayed transporters
- Delay reason breakdown (if available in `delayed_reason` column)
- Branch-wise delay distribution
- Delay duration histogram (buckets: 1–7d, 8–14d, 15–30d, 30d+)
- Transporter × branch delay heatmap

**SQL Note:** Uses `_CT_V3_SQL` (different from base — includes delay-specific joins if applicable).

---

### v4 — Future EDD

**Purpose:** Delivery schedule planning — what's coming this week.

**Key panels:**
- Day-by-day delivery schedule (bar chart: date × expected deliveries)
- Transporter-wise upcoming deliveries
- Branch-wise upcoming load
- High-value shipments due this week

---

### v5 — Delivered Performance

**Purpose:** OTD% measurement — the primary KPI Maria monitors.

**Key panels:**
- Overall OTD% (on-time delivery percentage)
- OTD trend by month
- OTD by transporter
- OTD by branch
- Service performance breakdown (On Time / Early / Late 1-3d / Late 4-7d / Late 7d+)

**OTD Calculation:**
```python
# A shipment is "on time" if:
# actualdeliverydate <= schdeliverydate
on_time = (df["actualdeliverydate"] <= df["schdeliverydate"]) & df["is_delivered"]
otd_pct = on_time.sum() / df["is_delivered"].sum() * 100
```

---

## Nightly CT Data Refresh (Job 6)

Maria's nightly refresh job ensures the CT cache is always fresh for the next business day.

```python
# Configured in data/maria_config.json:
{
  "schedule": {
    "ct_refresh_enabled": true,
    "ct_refresh_hour": 23        # 11 PM IST
  }
}
```

**What it does:**
1. Runs at 11 PM IST (configurable via `ct_refresh_hour`)
2. For each of v1, v2, v3, v4, v5:
   - Re-checks `ct_refresh_enabled` flag (supports mid-night toggling without restart)
   - Calls `_fetch_ct_df(view, force=True)` → fresh DB query
   - Logs row count to Activity tab
   - If DB error → calls `_notify_if_critical()` → system alert email
3. Total time: typically 30–90 seconds depending on DB load

**Why at 11 PM?**
- Low traffic time (ops team not actively using the dashboard)
- Fresh data is ready for the 8 AM morning brief
- Avoids midday DB load during peak usage

---

## Filters

Users can apply filters to any CT view. Filters are applied **after** the view cache is loaded (in Python, not SQL), so:
- Filtering is instant (no DB roundtrip)
- The full unfiltered dataset is always cached

### Available Filters (all views)

| Filter | Type | Example |
|--------|------|---------|
| `branch` | String / multi-select | `"Delhi"` or `["Delhi", "Mumbai"]` |
| `transporter` (truckername) | String / multi-select | `"DHL"` |
| `date_range` | Preset | `"last_30_days"`, `"this_month"`, `"last_7_days"` |
| `start_date` / `end_date` | ISO date strings | `"2025-01-01"` |
| `lrtype` | `"PTL"` or `"FTL"` | |
| `status` | Delivery status | `"In Transit"` |

### How Filters Are Applied

```python
# DataSliceService.compute_slice()
def compute_slice(df, filters, group_by, metrics, sort):
    df = df.copy()                          # never mutate shared cache

    if filters.get("branch"):
        df = df[df["branch"].isin(filters["branch"])]

    if filters.get("date_range") == "last_30_days":
        cutoff = pd.Timestamp.now() - pd.Timedelta(days=30)
        df = df[df["bookingdate"] >= cutoff]

    # ... more filters ...

    return df.groupby(group_by)[metrics].sum().reset_index()
```

---

## Adding a New CT View

To add a new CT view (e.g., `v6 — Delay Remarks`):

1. **Define SQL** in `main_complete.py`:
   ```python
   _CT_V6_SQL = "SELECT ... FROM tlbooking ..."
   ```

2. **Add derived columns** in `_apply_ct_derived_columns(df, view)`:
   ```python
   elif view == "v6":
       df["delay_reason"] = ...
   ```

3. **Add cache lock** in startup:
   ```python
   _ct_cache_locks["v6"] = asyncio.Lock()
   ```

4. **Add route** in `main_complete.py`:
   ```python
   @app.post("/api/dashboard/create-control-tower-v6")
   async def create_control_tower_v6(req: CTRequest): ...
   ```

5. **Include in refresh job** — add `"v6"` to the views list in `_ct_refresh_job`

6. **Add frontend tab** — add v6 to the CT view selector in `AppImpl.tsx`

7. **Add to Maria** — add `"v6"` to `_get_all_ct_views()` if Maria should include it in briefs
