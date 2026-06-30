# Stock & Forecasting — Portfolio Prototype

A sanitised demo of a stock-management dashboard I designed and built end-to-end for a previous employer. It replaced two ageing Excel workbooks with a single web app that surfaces what to order, what to transfer between locations, what's coming in on POs, and how the P&L is tracking — all from live ERP data and a handful of weekly file uploads.

**Live demo:** `[ADD GITHUB PAGES URL]`
**Repo:** `[ADD GITHUB REPO URL]`

> The demo runs entirely in the browser on synthetic data. The real version sits inside a Google Sheet and pulls live data from BigQuery + uploads + reference tables.

---

## At a glance

- **Replaced** two manually-maintained Excel workbooks shared across an ops team
- **Audience** ops + lab teams who weren't developers — every interaction had to be a click, not a formula
- **Stack** Google Sheets (as the persistent store), Apps Script (server + web app host), Connected Sheets to BigQuery (live ERP data), vanilla HTML + Chart.js (front-end). No external servers, no recurring costs.
- **Scope** ~3500 lines of Apps Script + ~3500 lines of front-end. Four views, twelve KPI tiles per view, drill-downs on every tile, five upload widgets, period-aware P&L, daily audit snapshots.
- **Outcome** a tool that's used daily by a non-developer team to make purchasing decisions, owned and maintained by the business not IT.

---

## Why this exists

The team I was building for had two problems:

1. **Two stale workbooks**. Stock data lived in `Lab Purchasing.xlsx` (HQ side) and `JEM Stock.xlsx` (fulfilment partner side). Both were manually updated, both drifted out of step with the ERP, and neither answered the question "what do I need to order today?" without manual VLOOKUPs.

2. **No IT support and no budget for a SaaS tool**. Whatever I built had to be free, sit on Google Workspace, and be maintainable by a non-developer after I left.

The shape that fell out of those constraints: one Google Sheet doing triple duty as data store, config UI, and orchestration layer, with an Apps Script web app on top serving the dashboard. Every refresh re-reads the Sheet, runs the calculations server-side, and ships a JSON bundle to the browser.

---

## What's in the demo

Open the live demo link above. Four tabs across the top:

### Purchasing
The day-to-day operational view. One row per item per location. KPI tiles count items by status (To order, Transfer, Low/Watch, OK, Open POs) and filter the table when clicked. Each row drills into an audit panel showing bin-level stock and the reorder calculation breakdown. Status logic is the heart of the tool — see [the reorder calculation](#the-reorder-calculation) below.

### Costs
Twelve KPI tiles arranged in two rows: stock state + commitments on top, current-month money + outputs underneath. Spend vs Budget uses a donut with arc-colour shifts (teal under 80%, amber 80–100%, coral over 100%). Click any bar on the Budget vs Actual chart to drill into the invoice lines that made up that month's spend. P&L tables sit underneath — period-aware, group-by-kit-type for kits, group-by-service-family for services, click a row for the underlying shipment lines.

### Kits
Greedy kit allocator: every kit with demand, sorted by priority, given first claim on shared components. Each row shows how many of that kit can actually be built after higher-priority kits have taken their share. JEM stock variance table flags items where the partner's physical shelf count and BC's ledger have drifted apart.

### Kit Dispatches
Trailing-window chart stacked by kit type, current-month pie, daily breakdown. Filters apply to both the chart and the CSV download.

---

## The reorder calculation

The interesting bit — and the one that took the longest to get right.

```
lead_time_weeks  = items.lead_time_days / 7
D_LT             = weekly_usage × lead_time_weeks
D_review         = weekly_usage × review_cycle_weeks
safety_stock     = ceil(2 × lead_time_weeks × weekly_usage)
target           = (D_LT + max(safety_stock, D_review)) × (1 + EDR_buffer) × (1 + stock_buffer)
```

Then for HQ items:

```
cover            = stores_stock + on_PO
if cover ≥ target  → recommended_order = 0
else               → recommended_order = max(0, target − stores_stock − on_PO)
```

The non-obvious design decision: **HQ uses stores stock for cover, not total stock**. The previous version used total (stores + lab + assembly), with the reasoning "lab will keep feeding stores naturally". That was backwards — in this business, stores feeds the lab (lab consumes from stores at the weekly_usage rate). When stores depletes, you need to order regardless of how much lab is currently holding, because lab can't refill stores. This bug let items with empty stores silently skip ordering for weeks; finding it required tracing through the audit panel on a single item with a colleague.

JEM (the fulfilment partner) uses total + on_PO because there's no internal stores/lab split at that location.

---

## The revenue cascade

For each posted shipment line, P&L revenue is computed via a three-strategy fallback:

| Step | Source | Used when |
|---|---|---|
| 1 | `ile_sales_amount` from the Item Ledger Entry | Line has been invoiced — authoritative figure from the ERP |
| 2 | Sales-line `line_amount × (qty_shipped / qty_ordered)` | Shipped but not yet invoiced — uses the open sales order line, captures line-level discounts |
| 3 | `qty × items.unit_price` | No matching sales order line — last-resort list-price approximation |

Same shape for COGS using `ile_cost → sales_line.unit_cost × qty → items.unit_cost × qty`.

The cascade keeps the recent-month P&L meaningful even before the ERP has invoiced everything — without it, today's P&L would look like £0 until invoicing catches up two weeks later.

---

## Architecture

```
BigQuery views          ─┐
(BC export, nightly)     ├→  Google Sheet (Connected Sheets tabs)
File uploads             │
(weekly / monthly xlsx,  │
 HubSpot CSV)            │       ┌──→ Web app (Apps Script doGet)
                         ├─→ Apps Script ─┤
Manual-edit Sheet tabs   │   (Code.gs)    └──→ Refresh + uploads via google.script.run
(BOMs, vendors, budget)  ┘
```

Three deliberate design choices worth calling out:

1. **Sheet as the persistent store.** No external database. The Sheet is the database — every tab is a table. This sounds primitive but it means the business owner can fix data herself in Sheets without my involvement, and there's no cloud bill.

2. **Apps Script as both backend and host.** One project, one deployment, one URL. The web app calls back into Apps Script via `google.script.run` for refresh + uploads + settings save. Trade-off: cold starts are slow (~30s on first hit of the day) but there's nothing else to maintain.

3. **Bundle-shaped responses.** Every page load gets a single JSON bundle with everything the front-end needs. No client-side fetching, no API contracts. Simpler to debug, but means a stale tab requires a full refresh (no incremental loading).

The full deployed version reads from 14 BigQuery views (item ledger, posted sales shipments, sales lines, posted purchase invoices, etc.) plus 5 upload-target tabs plus ~10 manual-edit reference tabs. The mock data in this demo is a synthetic stand-in for all of that.

---

## What I'd do differently

A few things I learned by the end that I'd start with on a similar project:

- **Pre-aggregate in BigQuery.** The single biggest performance issue is reading large detail tables (item ledger, posted shipments) into Apps Script and aggregating in-memory. Half a day of SQL writing summary views would have shaved 20 seconds off every refresh. I deferred this for handover because the trade-off was risk of breaking numbers vs the value of an extra 20s of speed.

- **Build the audit panel first.** I added the per-row audit panel late and it became the most-used debugging surface. Every "this number looks wrong" question gets answered by clicking the row and reading the audit. Should have been the first feature, not the last.

- **Treat the docs as a deliverable from day one.** The handover document was the most valuable single artefact at the end. Writing it earlier — even rough — would have forced clearer thinking about user-facing terminology vs internal terminology and saved several renames.

- **Status rules need a test harness.** The reorder logic has five status states with HQ/JEM variants. I caught the HQ stores-vs-total bug by chance, not by testing. A small JS test suite around `computeReorderSignals` would have caught it instantly.

---

## Sanitisation notes

This is a portfolio version. Everything that could identify the real business has been generalised:

- Real company name → "Demo Co"
- Real vendor names → "Vendor Alpha / Beta / Gamma..."
- Real kit codes → "KIT_AIR_01" etc
- Real customer names → "Acme Corp / Globex / Initech..."
- Real BigQuery project name → removed
- Hardcoded internal mappings → generic equivalents
- Apps Script + Sheet integration points → replaced by an in-browser mock that simulates the same RPC contract (see the `google.script.run` stub at the top of `index.html`)

The calculations, status rules, cascades, and overall architecture are exactly as deployed.

---

## Running locally

No build step. Just open `index.html` in a browser:

```
git clone [REPO URL]
cd stock-forecasting-prototype
open index.html
```

For GitHub Pages, push to `main` and enable Pages from the repo settings — pointed at the root.

---

## Files

| File | What it is |
|---|---|
| `index.html` | Self-contained single-file app. ~3400 lines: HTML structure + CSS + client JS + mock-data generator + Apps Script stub. |
| `README.md` | This file. |
| `screenshots/` | UI screenshots referenced from this README. |
