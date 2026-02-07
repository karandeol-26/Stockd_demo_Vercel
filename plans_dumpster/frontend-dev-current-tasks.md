# Frontend Developer — Current Tasks

## Project Status

- Module 0 (Repo + Setup): DONE (backend + frontend)
- Module 1 (Supabase Schema): DONE — 7 tables + 2 enums live in cloud
- Module 2 (Ingestion): Backend RPCs DONE — onboarding + daily ingest + bulk close all live
- Module 3 (Consumption Engine): DONE — daily close, reverse, snapshot RPCs all live
- Module 4 (Inventory Ops): Backend RPCs DONE — receive_inventory + count_inventory live
- Backend test suite: 28/28 tests passing, all RPCs verified bug-free
- Production data: 22,964 sales rows, 91 menu items, 358 days (Jan-Dec 2015) loaded

## What You Can Work On Now

### 1. Frontend Project Setup (Module 0 — Frontend Side)
### DONE WITH THIS

### 2. Landing Page / Onboarding Flow (PRIORITY)

The app uses `sales_line_items` as the **single source of truth**. On first launch, the user must upload their full sales history to establish the baseline. After that, daily uploads are incremental.

**Routing logic (on every app load):**
```js
const { data } = await supabase.rpc('get_onboarding_status');
if (!data.setup_complete) {
  // Show LANDING PAGE (onboarding)
} else {
  // Show MAIN DASHBOARD
}
```

**Landing Page steps:**
1. Welcome message: "Upload your sales history to get started"
2. File input for the history CSV (Toast ItemSelectionDetails format)
3. Parse with PapaParse, show preview: date range found, row count, sample items
4. On confirm: call `ingest_daily_sales` RPC with aggregated rows
   - For large files (~48K rows), batch into chunks of ~500 aggregated rows per call
5. Call `complete_onboarding_ingest()` — records the date range in app_config
6. Call `run_bulk_close()` — processes all historical dates (consumption engine)
7. Show progress/success: "365 dates processed, 32 menu items created"
8. Redirect to main dashboard

**Done when:** first-time user uploads history CSV, system initializes, dashboard loads.

### 3. Daily Upload Page (Module 2 — Post-Setup)

- Upload page (shown after setup is complete): file input + "Upload Today's Report"
- Use PapaParse (CDN or npm) to parse CSV in the browser
- Supported formats:
  - Toast `ItemSelectionDetails.csv`
  - Simplified daily file: `business_date, menu_item, qty, net_sales`
- Filter out voided items (`Void? = True`)
- Group by `(business_date, Menu Item)`, sum qty + net_sales
- Call `ingest_daily_sales` RPC with aggregated rows
- After ingest, call `run_daily_close` for the uploaded date(s)
- Show success/error feedback after upload
- Show count of rows processed and new menu items created
- **Done when:** uploading 1 day of Toast CSV creates rows and updates inventory

### 4. Inventory Snapshot Dashboard (Module 5)

- Call `get_inventory_snapshot()` RPC on page load
- Render table with columns:
  - Ingredient name
  - Unit
  - On hand (qty)
  - Reorder point
  - Avg daily usage (last 14 days)
  - Days of supply
  - Days to reorder
  - Status badge (OK / Reorder Soon / Critical)
- Color-coded status badges: green (ok), yellow (reorder_soon), red (critical)
- Sort by urgency (critical items at top — already sorted by RPC)
- Auto-refresh or manual refresh button
- **Done when:** dashboard renders current inventory and flags low items automatically

### 5. Inventory Ops Pages (Module 4)

Backend RPCs are live. Build two pages:

**Receive Inventory page:**
- Dropdown to select ingredient (fetch from `ingredients` table)
- Input: quantity received
- Input: optional note (e.g. "Sysco delivery")
- Submit calls `receive_inventory` RPC
- Show confirmation with updated qty on hand

**Count Inventory page:**
- Dropdown to select ingredient
- Shows current on-hand qty (fetch from `inventory_on_hand`)
- Input: actual counted qty
- Submit calls `count_inventory` RPC
- Show confirmation with delta and new qty

**Done when:** manager can receive a delivery and correct inventory via physical count.

---

## Backend RPCs Available (call via `supabase.rpc()`)

### `get_onboarding_status()`

Check if the app has been set up. Call on every app load to decide routing.

```js
const { data, error } = await supabase.rpc('get_onboarding_status');
// Returns: { setup_complete: false, history_uploaded: false,
//            history_start_date: null, history_end_date: null,
//            history_rows_ingested: 0, bulk_close_complete: false }
```

### `ingest_daily_sales(p_rows)`

Batch upserts sales data. Auto-creates missing menu items. Re-uploading the same date+item replaces the values (safe to re-run).

```js
const { data, error } = await supabase.rpc('ingest_daily_sales', {
  p_rows: [
    {
      business_date: '2015-01-01',
      menu_item_name: 'The Hawaiian Pizza (M)',
      category: 'Classic',
      qty: 10,
      net_sales: 132.50,
      source: 'toast'
    }
    // ... more rows
  ]
});
// Returns: { status: 'success', rows_processed: N, menu_items_created: N }
```

### `complete_onboarding_ingest()`

Call after the historical CSV has been fully ingested. Records the date range.

```js
const { data, error } = await supabase.rpc('complete_onboarding_ingest');
// Returns: { status: 'success', start_date: '2015-01-01', end_date: '2015-12-31', rows: 1234 }
```

### `run_bulk_close()`

Processes all historical dates that haven't been consumed yet. Call after `complete_onboarding_ingest`. This establishes the full consumption baseline.

```js
const { data, error } = await supabase.rpc('run_bulk_close');
// Returns: { status: 'success', dates_processed: 365, total_consume_txns: 4500 }
```

### `run_daily_close(p_business_date)`

Runs the consumption engine for a single date (post-setup daily use). Idempotent (skips if already run).

```js
const { data, error } = await supabase.rpc('run_daily_close', {
  p_business_date: '2015-01-01'
});
// Returns: { status: 'success', consume_txns_created: N, ingredients_updated: N }
```

### `reverse_daily_close(p_business_date)`

Reverses a daily close (deletes CONSUME txns, restores inventory). For corrections.

```js
const { data, error } = await supabase.rpc('reverse_daily_close', {
  p_business_date: '2015-01-01'
});
// Returns: { status: 'success', txns_reversed: N }
```

### `get_inventory_snapshot()`

Returns all ingredients with current stock, avg daily usage (last 14 days), days of supply, days to reorder, and status badge (ok / reorder_soon / critical). Already sorted by urgency.

```js
const { data, error } = await supabase.rpc('get_inventory_snapshot');
// Returns array: [{ ingredient_id, name, unit, qty_on_hand, reorder_point,
//   avg_daily_usage, days_of_supply, days_to_reorder, status, ... }]
```

### `receive_inventory(p_ingredient_id, p_qty, p_note)`

Receives a delivery. Adds qty to inventory, creates a RECEIVE audit txn. Validates qty > 0. Auto-creates inventory row if none exists.

```js
const { data, error } = await supabase.rpc('receive_inventory', {
  p_ingredient_id: '...',
  p_qty: 25,
  p_note: 'Weekly delivery from Sysco'  // optional, can be null
});
// Returns: { status: 'success', ingredient_id, qty_received: 25, new_qty_on_hand: 75 }
// Error:   { status: 'error', message: 'qty must be greater than 0' }
```

### `count_inventory(p_ingredient_id, p_actual_qty)`

Physical count correction. Sets inventory to actual counted qty, computes and records the delta. Validates actual_qty >= 0.

```js
const { data, error } = await supabase.rpc('count_inventory', {
  p_ingredient_id: '...',
  p_actual_qty: 85
});
// Returns: { status: 'success', ingredient_id, previous_qty: 100,
//            actual_qty: 85, delta: -15, new_qty_on_hand: 85 }
// Error:   { status: 'error', message: 'actual_qty must be >= 0' }
```

---

## Toast CSV Column Mapping

When parsing `Toast_ItemSelectionDetails.csv`, map these columns:

| Toast CSV Column | Maps To | Notes |
|---|---|---|
| `Order Date` | `business_date` | Extract date only from `MM/DD/YYYY HH:MM:SS` |
| `Menu Item` | `menu_item_name` | e.g. "The Hawaiian Pizza (M)" |
| `Sales Category` | `category` | e.g. "Classic", "Veggie", "Chicken" |
| `Net Price` | `net_sales` | Per-item net price |
| `Qty` | `qty` | Usually 1.0 per line |
| `Void?` | (filter) | Skip rows where `Void? = True` |
| (hardcode) | `source` | Use `"toast"` |

### Frontend CSV Processing Steps

1. Parse CSV with PapaParse
2. Filter out rows where `Void?` is `"True"`
3. Extract date from `Order Date` (take the date part: `MM/DD/YYYY`)
4. Group rows by `(business_date, Menu Item)`:
   - Sum `Qty` for total qty
   - Sum `Net Price` for total net_sales
   - Take first `Sales Category` as category
5. Call `supabase.rpc('ingest_daily_sales', { p_rows: aggregatedRows })`
6. Show success/error to user

---

## Database Tables Available (for reference)

| Table | Key Columns |
|---|---|
| `menu_items` | id, name, category, active |
| `ingredients` | id, name, unit, reorder_point, lead_time_days, unit_cost |
| `bom` | menu_item_id, ingredient_id, qty_per_item |
| **`sales_line_items`** | **id, business_date, menu_item_id, qty, net_sales, source** (SINGLE SOURCE OF TRUTH) |
| `inventory_on_hand` | ingredient_id, qty_on_hand, updated_at |
| `inventory_txns` | id, ingredient_id, txn_type, qty_delta, business_date, created_at, note |
| `app_config` | key (PK), value (jsonb), updated_at |

---

## Priority Order

1. ~~Frontend setup + login page~~ DONE
2. **Landing page / onboarding flow** (first-time setup experience) -- DO THIS NEXT
3. Daily upload page (post-setup incremental uploads)
4. Dashboard (backend RPC ready — just call `get_inventory_snapshot()`)
5. Inventory ops pages: Receive + Count (backend RPCs ready)

## Notes

- All tables have RLS enabled — requests must use an authenticated Supabase session
- Use vanilla JS and CSS only (no React/Vue/etc.)
- Supabase JS client handles auth tokens automatically after login
- The RPC functions use `SECURITY DEFINER` so they bypass RLS internally — the frontend just needs a valid auth session
- `sales_line_items` is the single source of truth — everything else derives from it
