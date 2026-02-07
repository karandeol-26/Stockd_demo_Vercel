# Frontend Developer — Current Tasks

## Project Status (Updated Feb 7, 2026)

- **Module 0** (Repo + Setup): DONE
- **Module 1** (Supabase Schema): DONE — 8 tables, 2 enums live in cloud
- **Module 2** (Ingestion): Backend RPCs DONE — onboarding + daily ingest + bulk close all live
- **Module 3** (Consumption Engine): DONE — daily close, reverse, snapshot RPCs all live
- **Module 4** (Inventory Ops): Backend RPCs DONE — `receive_inventory` + `count_inventory` live
- **Module 6** (Forecasting v1): Backend RPCs DONE — `generate_forecast` + `get_forecast` live
- **BOM**: Fully seeded — 32 ingredients, 674 recipe links across all 91 menu items
- **Backend test suite**: 36/36 tests passing, all RPCs verified bug-free
- **Production data**: 22,964 sales rows, 91 menu items, 32 ingredients, 674 BOM entries, 11,280 consume txns, 620 item forecasts, 224 ingredient forecasts

### All Backend Modules COMPLETE — Frontend is the bottleneck now.

---

## What You Need to Build

### 1. Landing Page / Onboarding Flow (PRIORITY)

The app uses `sales_line_items` as the **single source of truth**. On first launch, the user must upload their full sales history. After that, daily uploads are incremental.

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
7. Show progress/success: "X dates processed, Y menu items created"
8. Redirect to main dashboard

**Done when:** first-time user uploads history CSV, system initializes, dashboard loads.

---

### 2. Daily Upload Page (Module 2 — Post-Setup)

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

---

### 3. Inventory Snapshot Dashboard (Module 5)

- Call `get_inventory_snapshot()` RPC on page load
- Render table with columns:
  - Ingredient name
  - Unit
  - On hand (qty)
  - Reorder point
  - Avg daily usage
  - Days of supply
  - Days to reorder
  - Status badge (OK / Reorder Soon / Critical)
- Color-coded status badges: green (ok), yellow (reorder_soon), red (critical)
- Sort by urgency (critical items at top — already sorted by RPC)
- Auto-refresh or manual refresh button
- **Done when:** dashboard renders current inventory and flags low items automatically

**Sample data you'll get back (real production data):**

| Ingredient | On Hand | Unit | Avg/Day | Supply | Status |
|---|---|---|---|---|---|
| Brie Cheese | 2.74 | oz | 3.39 | 0.8 days | critical |
| Alfredo Sauce | 16.01 | oz | 12.04 | 1.3 days | critical |
| Pizza Dough | 2,159.91 | oz | 1,710.86 | 1.3 days | reorder_soon |
| Mozzarella Cheese | 1,033.42 | oz | 855.43 | 1.2 days | critical |

---

### 4. Inventory Ops Pages (Module 4)

Backend RPCs are live and tested.

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

### 5. Forecast Page (Module 6) — NEW

Backend RPCs are live and tested. 620 item forecasts + 224 ingredient forecasts already generated.

**Forecast Dashboard:**
- On page load, call `get_forecast()` (defaults to next 7 days from most recent data)
- If you want to re-generate fresh forecasts first: call `generate_forecast()` then `get_forecast()`
- Render table with columns:
  - Forecast date
  - Ingredient name
  - Qty needed (forecasted)
  - Current on hand
  - Shortfall (negative = need to order)
  - Unit
- Color-code shortfall: red if positive (means we'll run out), green if negative (we have enough)
- Group by date, or group by ingredient — your call on best UX
- **Done when:** user can see "next 7 days ingredient needs" with shortfall alerts

---

### 6. Admin Pages (Module 7) — Lower Priority

CRUD pages for managing the restaurant setup:

**Menu Items page:** list, add, edit, deactivate menu items
**Ingredients page:** list, add, edit ingredients (name, unit, reorder_point, lead_time, cost)
**BOM Builder page:** select menu item, add/edit/remove ingredient rows with qty_per_item

**Done when:** non-technical person can set up a new restaurant's menu and recipes.

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
// Returns: { status: 'success', start_date: '2015-01-01', end_date: '2015-12-31', rows: 22964 }
```

### `run_bulk_close()`

Processes all historical dates that haven't been consumed yet. Call after `complete_onboarding_ingest`.

```js
const { data, error } = await supabase.rpc('run_bulk_close');
// Returns: { status: 'success', dates_processed: 358, total_consume_txns: 11280 }
```

### `run_daily_close(p_business_date)`

Runs the consumption engine for a single date (post-setup daily use). Idempotent (skips if already run).

```js
const { data, error } = await supabase.rpc('run_daily_close', {
  p_business_date: '2015-12-31'
});
// Returns: { status: 'success', consume_txns_created: N, ingredients_updated: N }
// Or:      { status: 'skipped', reason: 'already_processed' }
// Or:      { status: 'no_data' }
```

### `reverse_daily_close(p_business_date)`

Reverses a daily close (deletes CONSUME txns, restores inventory). For corrections.

```js
const { data, error } = await supabase.rpc('reverse_daily_close', {
  p_business_date: '2015-12-31'
});
// Returns: { status: 'success', txns_reversed: N }
```

### `get_inventory_snapshot()`

Returns all ingredients with current stock, avg daily usage, days of supply, days to reorder, and status. Auto-detects the data window (works with both historical and real-time data). Already sorted by urgency.

```js
const { data, error } = await supabase.rpc('get_inventory_snapshot');
// Returns array:
// [
//   {
//     ingredient_id: 'uuid',
//     name: 'Mozzarella Cheese',
//     unit: 'oz',
//     reorder_point: 150,
//     lead_time_days: 2,
//     unit_cost: 0.22,
//     qty_on_hand: 1033.42,
//     avg_daily_usage: 855.43,
//     days_of_supply: 1.2,
//     days_to_reorder: 1.0,
//     status: 'critical'       // 'ok' | 'reorder_soon' | 'critical' | 'unknown'
//   },
//   ...
// ]
```

### `receive_inventory(p_ingredient_id, p_qty, p_note)`

Receives a delivery. Adds qty to inventory, creates a RECEIVE audit txn. Validates qty > 0. Auto-creates inventory row if none exists.

```js
const { data, error } = await supabase.rpc('receive_inventory', {
  p_ingredient_id: 'uuid-here',
  p_qty: 25,
  p_note: 'Weekly delivery from Sysco'  // optional, can be null
});
// Returns: { status: 'success', ingredient_id: '...', qty_received: 25, new_qty_on_hand: 75 }
// Error:   { status: 'error', message: 'qty must be greater than 0' }
```

### `count_inventory(p_ingredient_id, p_actual_qty)`

Physical count correction. Sets inventory to actual counted qty, computes and records the delta. Validates actual_qty >= 0.

```js
const { data, error } = await supabase.rpc('count_inventory', {
  p_ingredient_id: 'uuid-here',
  p_actual_qty: 85
});
// Returns: { status: 'success', ingredient_id: '...', previous_qty: 100,
//            actual_qty: 85, delta: -15, new_qty_on_hand: 85 }
// Error:   { status: 'error', message: 'actual_qty must be >= 0' }
```

### `generate_forecast(p_days_ahead, p_reference_date)`

Generates item-level and ingredient-level forecasts using rolling day-of-week averages from the last 6 weeks. Idempotent (re-running replaces previous forecast). Converts item forecasts to ingredient needs via BOM.

```js
const { data, error } = await supabase.rpc('generate_forecast', {
  p_days_ahead: 7,                 // default 7
  p_reference_date: '2016-01-01'   // default CURRENT_DATE
});
// Returns: { status: 'success', item_forecasts: 620, ingredient_forecasts: 224 }
```

### `get_forecast(p_reference_date)`

Returns 7-day ingredient forecast with current stock and shortfall. Call after `generate_forecast`.

```js
const { data, error } = await supabase.rpc('get_forecast', {
  p_reference_date: '2016-01-01'   // default CURRENT_DATE
});
// Returns array:
// [
//   {
//     forecast_date: '2016-01-01',
//     ingredient_id: 'uuid',
//     name: 'Mozzarella Cheese',
//     unit: 'oz',
//     qty_needed: 855.43,
//     qty_on_hand: 1033.42,
//     shortfall: -178.0         // positive = we'll run out, negative = we have enough
//   },
//   ...
// ]
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

## Database Tables (for reference)

| Table | Key Columns | Rows (current) |
|---|---|---|
| `menu_items` | id, name, category, active | 91 |
| `ingredients` | id, name, unit, reorder_point, lead_time_days, unit_cost | 32 |
| `bom` | menu_item_id, ingredient_id, qty_per_item | 674 |
| **`sales_line_items`** | **id, business_date, menu_item_id, qty, net_sales, source** | **22,964** |
| `inventory_on_hand` | ingredient_id, qty_on_hand, updated_at | 32 |
| `inventory_txns` | id, ingredient_id, txn_type, qty_delta, business_date, created_at, note | 11,280 |
| `app_config` | key (PK), value (jsonb), updated_at | 2 |
| `forecast_items` | id, forecast_date, menu_item_id, qty | 620 |
| `forecast_ingredients` | id, forecast_date, ingredient_id, qty | 224 |

---

## Fetching Dropdown Data

For pages that need ingredient lists (Receive, Count, Admin):

```js
// Get all ingredients for dropdowns
const { data: ingredients } = await supabase
  .from('ingredients')
  .select('id, name, unit')
  .order('name');

// Get current on-hand for a specific ingredient
const { data: onHand } = await supabase
  .from('inventory_on_hand')
  .select('qty_on_hand')
  .eq('ingredient_id', selectedIngredientId)
  .single();

// Get all menu items for admin pages
const { data: menuItems } = await supabase
  .from('menu_items')
  .select('id, name, category, active')
  .order('name');
```

---

## Supabase Client Setup

```js
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  'YOUR_SUPABASE_URL',       // from .env or config
  'YOUR_SUPABASE_ANON_KEY'   // from .env or config — use ANON key, not service key
);
```

**Important:** The frontend uses the **anon key** (not the service key). All tables have RLS — the user must be logged in. The RPC functions use `SECURITY DEFINER` so they bypass RLS internally; the frontend just needs a valid auth session.

---

## Priority Order

1. **Landing page / onboarding flow** (first-time setup experience)
2. **Dashboard** (call `get_inventory_snapshot()` — all data is live)
3. **Daily upload page** (post-setup incremental uploads)
4. **Inventory ops pages**: Receive + Count
5. **Forecast page** (call `generate_forecast()` then `get_forecast()`)
6. Admin pages: Menu + Ingredients + BOM editor (lower priority)

---

## Notes

- All tables have RLS enabled — requests must use an authenticated Supabase session
- Use vanilla JS and CSS only (no React/Vue/etc.)
- Supabase JS client handles auth tokens automatically after login
- The RPC functions use `SECURITY DEFINER` so they bypass RLS internally
- `sales_line_items` is the single source of truth — everything else derives from it
- All credentials go in `.env` (gitignored) — see `.envexample` for template
- The snapshot RPC auto-detects the data window, so it works with both our historical 2015 data and future real-time data
