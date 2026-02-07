# Restaurant Inventory & Forecasting System — Master Plan

> **Project**: UGAHacks 11 — Restaurant Inventory Management + Demand Forecasting
> **Stack**: Supabase (cloud), Vanilla JS + CSS (frontend), PostgreSQL RPCs + Edge Functions (backend)
> **Last Updated**: 2026-02-07

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Module Tracker](#module-tracker)
4. [Module 0 — Repo + Local Setup](#module-0--repo--local-setup)
5. [Module 1 — Supabase Schema](#module-1--supabase-schema)
6. [Module 2 — Ingestion (CSV Upload)](#module-2--ingestion-csv-upload)
7. [Module 3 — Consumption Engine](#module-3--consumption-engine)
8. [Module 4 — Inventory Ops (Receive + Count)](#module-4--inventory-ops-receive--count)
9. [Module 5 — Inventory Snapshot Dashboard](#module-5--inventory-snapshot-dashboard)
10. [Module 6 — Forecasting v1](#module-6--forecasting-v1)
11. [Module 7 — Admin (Menu + BOM Editor)](#module-7--admin-menu--bom-editor)
12. [Team Roles](#team-roles)
13. [Data Model Reference](#data-model-reference)
14. [RPC Reference](#rpc-reference)
15. [Test Data](#test-data)

---

## Project Overview

A web app that lets restaurant managers:
- On first setup, upload full POS sales history to establish the **single source of truth**
- After setup, upload daily POS reports that incrementally update the source of truth
- Automatically compute ingredient consumption from sales via recipes (BOM)
- Track real-time inventory (receive deliveries, count stock)
- Forecast ingredient needs for the next 7 days
- See a dashboard flagging low stock and reorder alerts

### Core Principle: Single Source of Truth

`sales_line_items` is the **single source of truth** for all system behavior.

```
FIRST LAUNCH (Onboarding):
  Landing Page -> Upload Full History CSV -> ingest_daily_sales (bulk)
       -> complete_onboarding_ingest -> run_bulk_close
       -> All historical consumption computed
       -> Inventory baseline established
       -> setup_complete = true -> Redirect to Dashboard

DAILY OPERATION (after setup):
  Dashboard -> Upload Today's Report -> ingest_daily_sales
       -> run_daily_close -> Inventory updated
       -> Dashboard refreshes with new snapshot
       -> Forecast recalculates

EVERYTHING DERIVES FROM sales_line_items:
  sales_line_items  --(BOM join)--> consumption (inventory_txns)
  sales_line_items  --(BOM join)--> inventory_on_hand
  sales_line_items  --(DOW avg)---> forecasting
  sales_line_items  --(aggregation)--> dashboard metrics
```

### Key Requirements

| ID | Requirement | Module | Priority |
|----|-------------|--------|----------|
| R0 | Landing page: first-time setup with historical CSV upload | M2 | Must |
| R1 | Upload Toast CSV and store itemized daily sales | M2 | Must |
| R2 | Define menu items, ingredients, and recipes (BOM) | M1, M7 | Must |
| R3 | Compute daily ingredient usage from sales + BOM | M3 | Must |
| R4 | Receive inventory deliveries and adjust stock | M4 | Must |
| R5 | Physical count corrections to inventory | M4 | Must |
| R6 | Dashboard: on-hand, avg usage, days of supply, alerts | M5 | Must |
| R7 | 7-day ingredient demand forecast (rolling avg by DOW) | M6 | Should |
| R8 | Admin UI to manage menu items, ingredients, recipes | M7 | Should |
| R9 | Auth: login required for all operations | M0 | Must |
| R10 | Idempotent daily close (safe to re-run) | M3 | Must |
| R11 | Reverse daily close for corrections | M3 | Should |
| R12 | Onboarding state tracking (setup_complete, history dates) | M2 | Must |
| R13 | Bulk close: process all historical dates in one call | M3 | Must |
| R14 | Multi-tenant support (future) | -- | Won't (v1) |

---

## Architecture

```
                        +-------------------------------+
                        |     FIRST LAUNCH?             |
                        |  get_onboarding_status()      |
                        +------+----------+-------------+
                               |          |
                          setup_complete   setup_complete
                          = false          = true
                               |          |
                               v          v
                        +-----------+  +------------------+
                        | LANDING   |  | MAIN DASHBOARD   |
                        | PAGE      |  |                  |
                        | Upload    |  | Inventory snap   |
                        | history   |  | Daily upload     |
                        | CSV       |  | Forecast         |
                        +-----------+  +------------------+
                               |          |
                               v          v
                        +-------------------------------+
                        |   Frontend: Vanilla JS + CSS  |
                        |   PapaParse, Supabase Auth    |
                        +-------------------------------+
                                       |
                            supabase-js client
                                       |
                                       v
                        +-------------------------------+
                        |      Supabase Cloud           |
                        |-------------------------------|
                        | PostgreSQL                    |
                        |   SINGLE SOURCE OF TRUTH:     |
                        |   sales_line_items            |
                        |      |                        |
                        |      +--> bom join --> consumption (inventory_txns)
                        |      +--> bom join --> inventory_on_hand
                        |      +--> DOW avg --> forecast
                        |      +--> agg    --> dashboard metrics
                        |                               |
                        |   Config: app_config          |
                        |   Auth: email/password        |
                        +-------------------------------+
```

---

## Module Tracker

| Module | Name | Owner | Status | Notes |
|--------|------|-------|--------|-------|
| **M0** | Repo + Local Setup | DevOps/Lead | DONE | Supabase linked, CLI working |
| **M1** | Supabase Schema | Backend | DONE | 6 tables, 2 enums, RLS, indexes |
| **M2** | Ingestion (CSV Upload) | Frontend + Backend | IN PROGRESS | Backend RPC done, frontend UI pending |
| **M3** | Consumption Engine | Backend | DONE | run_daily_close + reverse + snapshot RPCs |
| **M4** | Inventory Ops | Frontend + Backend | NOT STARTED | Receive + Count workflows |
| **M5** | Inventory Dashboard | Frontend | NOT STARTED | Backend RPC ready (get_inventory_snapshot) |
| **M6** | Forecasting v1 | Data/Backend | NOT STARTED | Rolling avg by DOW planned |
| **M7** | Admin (Menu + BOM) | Frontend | NOT STARTED | CRUD for menu, ingredients, BOM |

### Current Focus

```
DONE        DONE       DOING        DONE        TODO        TODO        TODO        TODO
 M0 -------> M1 -------> M2 -------> M3 -------> M5 -------> M4 -------> M6 -------> M7
Setup       Schema     Ingestion   Consumption  Dashboard   Inv Ops    Forecast    Admin
                       ^frontend    ^backend
```

---

## Module 0 — Repo + Local Setup

**Owner**: DevOps/Lead
**Status**: DONE

### Requirements
- [x] Supabase cloud project created and linked via CLI
- [x] `.envexample` with `SUPABASE_URL` and `SUPABASE_ANON_KEY`
- [x] `.gitignore` with `supabase/.temp`
- [x] Migration workflow: write SQL in `supabase/migrations/`, push with `supabase db push`
- [ ] Frontend served locally (Vite or live-server) — frontend dev
- [ ] Login page visible on clone+run — frontend dev

### Deliverables
- `supabase/` directory with config and migrations
- `.envexample`, `.gitignore`

### Done When
Teammate can clone, install, run, and see a login page.

---

## Module 1 — Supabase Schema

**Owner**: Backend
**Status**: DONE
**Migration**: `20260207000100_init_schema.sql`

### Requirements
- [x] `menu_items` table (id, name, category, active, created_at)
- [x] `ingredients` table (id, name, unit enum, reorder_point, lead_time_days, unit_cost)
- [x] `bom` table (menu_item_id, ingredient_id, qty_per_item; composite PK)
- [x] `sales_line_items` table (id, business_date, menu_item_id, qty, net_sales, source)
- [x] `inventory_on_hand` table (ingredient_id PK, qty_on_hand, updated_at)
- [x] `inventory_txns` table (id, ingredient_id, txn_type enum, qty_delta, created_at, note)
- [x] Enums: `unit_type` (g/oz/lb/each), `inventory_txn_type` (RECEIVE/COUNT/CONSUME)
- [x] RLS enabled on all tables, authenticated select/insert/update
- [x] FK constraints: bom->menu_items, bom->ingredients, sales->menu_items, inv->ingredients
- [x] Unique index on `(business_date, menu_item_id)` for daily aggregation
- [x] Indexes on `sales_line_items(business_date)`, `inventory_txns(ingredient_id, created_at)`
- [x] Nullable `org_id` columns reserved for future multi-tenant

### Done When
Can insert menu items, BOM, sales, inventory rows and query joins.

---

## Module 2 — Ingestion (CSV Upload) + Onboarding

**Owner**: Frontend (UI) + Backend (RPC)
**Status**: IN PROGRESS — backend RPCs done, frontend UI pending
**Migration**: `20260207000200_consumption_engine.sql` + `20260207000300_onboarding_and_bulk_close.sql`

### Requirements — Backend (DONE)
- [x] `ingest_daily_sales(p_rows jsonb)` RPC
- [x] Auto-creates missing menu_items from CSV data
- [x] Upserts by `(business_date, menu_item_id)` — safe to re-upload
- [x] Returns `{ status, rows_processed, menu_items_created }`
- [x] `app_config` table with onboarding state tracking
- [x] `get_onboarding_status()` RPC — frontend calls on load to route user
- [x] `complete_onboarding_ingest()` RPC — records history date range after bulk upload
- [x] `run_bulk_close()` RPC — processes all un-consumed dates at once (for setup)

### Requirements — Frontend: Landing Page / Onboarding (PENDING)
- [ ] On app load, call `get_onboarding_status()`
- [ ] If `setup_complete = false`, show **Landing Page** (onboarding flow)
- [ ] If `setup_complete = true`, show **Main Dashboard**
- [ ] Landing page steps:
  1. Welcome message + explanation
  2. File input: "Upload your sales history CSV"
  3. Parse with PapaParse, show preview (date range, row count)
  4. On confirm: call `ingest_daily_sales` (may need to batch for large files)
  5. Call `complete_onboarding_ingest()` to record date range
  6. Call `run_bulk_close()` to compute all historical consumption
  7. Show progress/success: "X dates processed, Y items created"
  8. Redirect to main dashboard

### Requirements — Frontend: Daily Upload (PENDING)
- [ ] Upload page (post-setup): file input + "Upload" button
- [ ] Parse CSV with PapaParse in the browser
- [ ] Support Toast `ItemSelectionDetails.csv` format
- [ ] Support simplified format: `business_date, menu_item, qty, net_sales`
- [ ] Filter out voided items (`Void? = True`)
- [ ] Group by `(business_date, Menu Item)`, sum qty + net_sales
- [ ] Call `ingest_daily_sales` RPC with aggregated rows
- [ ] Optionally call `run_daily_close` for the uploaded date(s)
- [ ] Show success/error feedback after upload
- [ ] Show count of rows processed and new menu items created

### Toast CSV Column Mapping

| Toast Column | Target Field | Transform |
|---|---|---|
| `Order Date` | `business_date` | Extract date from `MM/DD/YYYY HH:MM:SS` |
| `Menu Item` | `menu_item_name` | Direct |
| `Sales Category` | `category` | Direct |
| `Net Price` | `net_sales` | Sum per group |
| `Qty` | `qty` | Sum per group |
| `Void?` | (filter) | Skip if `True` |
| (hardcode) | `source` | `"toast"` |

### Done When
Uploading 1 day of Toast CSV creates correct rows in `sales_line_items`.

---

## Module 3 — Consumption Engine

**Owner**: Backend
**Status**: DONE
**Migration**: `20260207000200_consumption_engine.sql`

### Requirements
- [x] `run_daily_close(p_business_date)` RPC
- [x] Joins `sales_line_items` with `bom` for given date
- [x] Computes `ingredient_usage = SUM(qty_sold * qty_per_item)`
- [x] Writes `CONSUME` txns to `inventory_txns` (negative qty_delta)
- [x] Decrements `inventory_on_hand`
- [x] Idempotent: skips if already run for that date
- [x] Returns no_data status if no sales for the date
- [x] `reverse_daily_close(p_business_date)` RPC for corrections
- [x] `business_date` column added to `inventory_txns` for tracking
- [x] `get_inventory_snapshot()` RPC for dashboard
  - [x] Avg daily usage from last 14 days of CONSUME txns
  - [x] Days of supply = on_hand / avg_daily_usage
  - [x] Days to reorder = (on_hand - reorder_point) / avg_daily_usage
  - [x] Status badges: ok / reorder_soon / critical
  - [x] Sorted by urgency (critical first)

### Done When
Click "Run Daily Close" for a date with sales -> inventory decrements correctly.

---

## Module 4 — Inventory Ops (Receive + Count)

**Owner**: Frontend + Backend
**Status**: NOT STARTED

### Requirements — Backend
- [ ] `receive_inventory(p_ingredient_id, p_qty, p_note)` RPC
  - Insert `RECEIVE` txn with positive qty_delta
  - Increment `inventory_on_hand`
- [ ] `count_inventory(p_ingredient_id, p_actual_qty)` RPC
  - Compute delta = actual - current on_hand
  - Insert `COUNT` txn with delta
  - Set `inventory_on_hand` to actual

### Requirements — Frontend
- [ ] **Receive Inventory page**: select ingredient, enter qty, optional note, submit
- [ ] **Count Inventory page**: select ingredient, enter actual count, submit
- [ ] Confirmation feedback after each operation
- [ ] Show updated on-hand after submit

### Features
- Receive: adds stock when a delivery arrives
- Count: corrects stock after a physical count (NCR-style)
- Both create audit trail via `inventory_txns`

### Done When
Manager can receive a delivery and correct inventory; snapshot reflects changes.

---

## Module 5 — Inventory Snapshot Dashboard

**Owner**: Frontend
**Status**: NOT STARTED (backend RPC ready)

### Requirements
- [ ] Call `get_inventory_snapshot()` RPC on page load
- [ ] Render table with columns:
  - Ingredient name
  - Unit
  - On hand (qty)
  - Reorder point
  - Avg daily usage (last 14 days)
  - Days of supply
  - Days to reorder
  - Status badge (OK / Reorder Soon / Critical)
- [ ] Color-coded status badges: green (ok), yellow (reorder_soon), red (critical)
- [ ] Sort by urgency (critical items at top — already sorted by RPC)
- [ ] Auto-refresh or manual refresh button
- [ ] Responsive layout

### Done When
Dashboard renders current inventory and flags low items automatically.

---

## Module 6 — Forecasting v1

**Owner**: Data/Backend
**Status**: NOT STARTED

### Requirements — Backend
- [ ] `forecast_items` table: (id, forecast_date, menu_item_id, qty)
- [ ] `forecast_ingredients` table: (id, forecast_date, ingredient_id, qty)
- [ ] `generate_forecast(p_days_ahead integer default 7)` RPC or Edge Function
  - Daily item demand = rolling average by day-of-week (last 4-6 weeks)
  - Convert to ingredient forecast via BOM
  - Store results in forecast tables
- [ ] `get_forecast(p_days_ahead integer default 7)` RPC
  - Returns next N days of ingredient needs

### Requirements — Frontend
- [ ] Forecast page showing next 7 days of ingredient needs
- [ ] Table: date, ingredient, forecasted qty, current on_hand, shortfall
- [ ] Visual indicator for shortfalls

### Algorithm (v1 — simple)
```
For each menu_item:
  For each day_of_week (Mon-Sun):
    avg_qty = AVG(qty) from sales_line_items
              WHERE EXTRACT(DOW FROM business_date) = target_dow
              AND business_date >= current_date - 42  (6 weeks)

For next 7 days:
  For each menu_item:
    forecast_qty = avg_qty for that DOW
  For each ingredient:
    forecast_ingredient_qty = SUM(forecast_qty * bom.qty_per_item)
```

### Done When
Can show "next 7 days ingredient needs" based on historical sales patterns.

---

## Module 7 — Admin (Menu + BOM Editor)

**Owner**: Frontend
**Status**: NOT STARTED

### Requirements
- [ ] **Menu Items page**: list, add, edit, deactivate menu items
- [ ] **Ingredients page**: list, add, edit ingredients (name, unit, reorder_point, lead_time, cost)
- [ ] **BOM Builder page**: select menu item, add/edit/remove ingredient rows with qty_per_item
- [ ] Validation: prevent duplicate BOM entries, require positive quantities
- [ ] Confirmation dialogs for destructive actions

### Done When
Non-technical person can set up a new restaurant's menu, ingredients, and recipes.

---

## Team Roles

| Role | Modules | Current Focus |
|------|---------|---------------|
| **Backend (you)** | M0, M1, M2 (RPC), M3, M4 (RPC), M6 | M4 backend RPCs next |
| **Frontend Dev** | M0 (UI), M2 (UI), M4 (UI), M5, M7 | M0 setup + M2 CSV upload |
| **Data/Forecast** | M6 | Blocked until M2+M3 have real data flowing |
| **DevOps/Lead** | M0, code review, merge | Ongoing |

---

## Data Model Reference

### Tables

| Table | PK | Key Columns | Notes |
|-------|-----|-------------|-------|
| `menu_items` | `id` (uuid) | name, category, active | Auto-created by ingest RPC |
| `ingredients` | `id` (uuid) | name, unit, reorder_point, lead_time_days, unit_cost | |
| `bom` | `(menu_item_id, ingredient_id)` | qty_per_item | Recipe/bill of materials |
| **`sales_line_items`** | **`id` (uuid)** | **business_date, menu_item_id, qty, net_sales** | **SINGLE SOURCE OF TRUTH** |
| `inventory_on_hand` | `ingredient_id` | qty_on_hand, updated_at | Derived from txns |
| `inventory_txns` | `id` (uuid) | ingredient_id, txn_type, qty_delta, business_date | Audit trail |
| `app_config` | `key` (text) | value (jsonb), updated_at | Onboarding state |

### Enums

| Enum | Values |
|------|--------|
| `unit_type` | g, oz, lb, each |
| `inventory_txn_type` | RECEIVE, COUNT, CONSUME |

---

## RPC Reference

| Function | Input | Returns | Module | Status |
|----------|-------|---------|--------|--------|
| `get_onboarding_status()` | (none) | `{ setup_complete, history_uploaded, ... }` | M2 | LIVE |
| `ingest_daily_sales(p_rows)` | jsonb array of sales rows | `{ status, rows_processed, menu_items_created }` | M2 | LIVE |
| `complete_onboarding_ingest()` | (none) | `{ status, start_date, end_date, rows }` | M2 | LIVE |
| `run_daily_close(p_business_date)` | date | `{ status, consume_txns_created, ingredients_updated }` | M3 | LIVE |
| `run_bulk_close()` | (none) | `{ status, dates_processed, total_consume_txns }` | M3 | LIVE |
| `reverse_daily_close(p_business_date)` | date | `{ status, txns_reversed }` | M3 | LIVE |
| `get_inventory_snapshot()` | (none) | jsonb array of ingredient snapshots | M5 | LIVE |
| `receive_inventory(...)` | ingredient_id, qty, note | TBD | M4 | NOT BUILT |
| `count_inventory(...)` | ingredient_id, actual_qty | TBD | M4 | NOT BUILT |
| `generate_forecast(...)` | days_ahead | TBD | M6 | NOT BUILT |
| `get_forecast(...)` | days_ahead | TBD | M6 | NOT BUILT |

---

## Test Data

| File | Description | Rows | Date Range |
|------|-------------|------|------------|
| `Test data/Toast_ItemSelectionDetails_from_Pizza.csv` | Item-level sales (Tony's Pizza Atlanta) | ~48K | 01/01/2015 - 12/31/2015 |
| `Test data/Toast_OrderDetails_DAILY_INJECT_latestDay.csv` | Order-level summary (last day) | 74 | 12/31/2015 |

---

## Migrations Applied

| Timestamp | File | Contents |
|-----------|------|----------|
| `20260207000100` | `init_schema.sql` | Tables, enums, indexes, RLS policies |
| `20260207000200` | `consumption_engine.sql` | business_date column, ingest/close/reverse/snapshot RPCs |
| `20260207000300` | `onboarding_and_bulk_close.sql` | app_config table, onboarding RPCs, bulk close |
