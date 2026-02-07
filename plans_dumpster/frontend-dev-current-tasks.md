# Frontend Developer — Current Tasks

## Project Status

- Module 0 (Repo + Setup): DONE (backend/DB side)
- Module 1 (Supabase Schema): DONE — all 6 core tables live in cloud
- Module 3 (Consumption Engine): In progress (backend)

## What You Can Work On Now

### 1. Frontend Project Setup (Module 0 — Frontend Side)

- Set up a simple static frontend (Vite or plain live-server)
- Create the base folder structure:
  - `frontend/index.html`
  - `frontend/css/` — global styles
  - `frontend/js/` — vanilla JS modules
- Add Supabase JS client (`@supabase/supabase-js` via CDN or npm)
- Create a basic login/auth page using Supabase Auth
- Wire up `.env` values (`SUPABASE_URL`, `SUPABASE_ANON_KEY`)
- **Done when:** teammate can clone, run, and see a working login page

### 2. CSV Upload UI (Module 2 — Frontend Part)

- Build a simple upload page: file input + "Upload" button
- Use PapaParse (CDN or npm) to parse CSV in the browser
- Supported formats:
  - Toast `ItemSelectionDetails.csv`
  - Simplified daily file: `business_date, menu_item, qty, net_sales`
- After parsing, normalize rows and insert into `sales_line_items` table via Supabase client
- Match `menu_item` by name against the `menu_items` table
- Upsert by `(business_date, menu_item_id)` for daily aggregation
- Show success/error feedback after upload
- **Done when:** uploading 1 day of CSV creates rows in `sales_line_items`

### 3. Inventory Snapshot Dashboard Shell (Module 5 — Start UI)

- Build the dashboard layout (table/grid view) with these columns:
  - Ingredient name
  - On hand (qty)
  - Reorder point
  - Avg daily usage (placeholder — backend will provide this)
  - Days of supply (placeholder)
  - Days to reorder (placeholder)
  - Status badge (OK / Reorder Soon / Critical)
- Fetch data from `inventory_on_hand` joined with `ingredients`
- Style with status badges (green/yellow/red)
- **Done when:** dashboard renders current inventory with status indicators

## Database Tables Available (for reference)

| Table | Key Columns |
|---|---|
| `menu_items` | id, name, category, active |
| `ingredients` | id, name, unit, reorder_point, lead_time_days, unit_cost |
| `bom` | menu_item_id, ingredient_id, qty_per_item |
| `sales_line_items` | id, business_date, menu_item_id, qty, net_sales, source |
| `inventory_on_hand` | ingredient_id, qty_on_hand, updated_at |
| `inventory_txns` | id, ingredient_id, txn_type, qty_delta, created_at, note |

## Priority Order

1. Frontend setup + login page (unblocks everything else)
2. CSV upload UI (enables data ingestion testing end-to-end)
3. Dashboard shell (can be built in parallel, will light up once backend consumption engine is ready)

## Notes

- All tables have RLS enabled — requests must use an authenticated Supabase session
- Use vanilla JS and CSS only (no React/Vue/etc.)
- Supabase JS client handles auth tokens automatically after login
