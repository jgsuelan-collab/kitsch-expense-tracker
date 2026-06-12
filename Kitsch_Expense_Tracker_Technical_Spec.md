# Kitsch Expense Tracker — Technical Product Specification

**Version:** 1.0 (MVP-ready)
**Author:** Prepared for KJ Cabatana, Kitsch
**Date:** 2026-06-11
**Status:** Draft for engineering kickoff

---

## 0. The one thing to read first (data reality)

The shared folder does **not** contain transaction- or invoice-level data. It contains **monthly, pre-aggregated vendor spend**: one row per `Month × Vendor × GL Account × Department = summed Amount`. This single fact reshapes the whole project:

| Spec requirement | Available in folder? | Consequence |
|---|---|---|
| Invoice number | **No** | Render "Not available" until NetSuite sync |
| Invoice date (day-level) | **No** — only month name | Period granularity is **monthly**, not daily |
| Transaction status | **No** | "Not available" until NetSuite |
| Memo / description | **No** (GL "Simplified" category is the closest) | "Not available" |
| Subsidiary / business unit | **No** | Filter ships disabled until NetSuite exposes it |
| Vendor, GL account, GL category, department, dept head, amount, month | **Yes** | These power the entire MVP |

**Therefore:** a "daily" NetSuite sync against a monthly base is a granularity mismatch. The recommended model is a **monthly grain for the historical base** and a **daily *delta* pull from NetSuite that is bucketed into the same monthly grain** (with optional transaction-level detail stored alongside once NetSuite access is confirmed). Every figure in this spec and the working dashboard ties exactly to the source workbook SUM cells: **2024 = \$218,220,488.75 · 2025 = \$341,375,464.09 · 2026 YTD (Jan–Apr) = \$126,898,940.50**.

---

## 1. Folder inspection summary (the base dataset)

**Location:** `Vendor Spend Analysis/` (Dropbox-synced). 12 workbooks + a `2025/` subfolder.

**Canonical source tabs** (the clean normalized data) are the `RAW DATA` / `… Full Year` / `… Vendor Spend` sheets. The per-department tabs (`SALES`, `G&A`, `MKTG`, `DESIGN`, `WAREHOUSE`, `AMAZON`, `PRODUCTION`) are **Excel pivot tables derived from RAW DATA** — do not import them; import the raw sheets.

**Chosen non-overlapping sources** (avoids double-counting, since the monthly 2026 files are cumulative YTD supersets):

| Year | File | Sheet | Rows |
|---|---|---|---|
| 2024 | `2025/2024 and 2025 Oct YTD Vendor Spend Analysis.xlsx` | `2024 Vendor Spend` | 4,117 |
| 2025 | `04 2026 Monthly Vendor Spend Analysis - YTD.xlsx` | `2025 Full Year` | 44,393 |
| 2026 | `04 2026 Monthly Vendor Spend Analysis - YTD.xlsx` | `2026 RAW DATA` | 21,155 |

**Total normalized: 69,665 line items, $686,494,893.34.**

**Fields present** (after normalization): `Month` (name → month number), `Vendor Name`, `GL Account` (e.g. `6290-20 - Warehouse, AMZ FBA Fees`), `GL Simplified` (category, e.g. `Warehouse`, `Freight Out`, `Samples R&D`), `NetSuite Dept` / `QBODept`, `Department`, `Dept Head`, `Amount`.

**Departments (canonical 7):** Sales, G&A, Marketing, Design, Warehouse, Amazon, Purchases - COGS.

**Supporting tabs** worth ingesting later: `NS Control` (monthly control totals per GL bucket — use for reconciliation), `NS NumName Lookup` (vendor number → vendor name crosswalk — use to map NetSuite vendor IDs to folder vendor names).

### Data-quality findings & cleaning applied
1. **Column mislabel (2024 file):** 255 rows had a person's name ("Jordan Sandrini", "Kyle Morrison") in the `Department` column while the real department sat in `QBODept`. **Fix:** if `Department` ∉ canonical-7, fall back to the NetSuite/QBO dept column; push the leaked name into `Dept Head`. Result: clean 7-department set.
2. **Header drift across years:** 2024 lacks a `GL Simplified` column and uses `QBODept`; 2025/2026 use `NetSuite Dept` + `GL Simplified`. **Fix:** header-name detection rather than fixed column positions.
3. **Month-only dates:** no day component. **Fix:** synthesize `period = YYYY-MM` from the file's year + month name; flag day-level as unavailable.
4. **Vendor name inconsistency:** the same vendor can appear under slightly different spellings/number-prefixes across years (e.g. `Amazon`, `Amazon FBA Fees`, `Amazon Shipping`). **Recommendation:** build a vendor alias/crosswalk table (post-MVP) seeded from `NS NumName Lookup`.
5. **Mixed cost types:** "spend" includes COGS/inventory purchases (Amazon FBA Fees, Glory Will International, Soapcreek Mfg) — this is *vendor spend*, not opex only. Keep this explicit in the UI so finance leaders aren't misled.

---

## 2. Recommended implementation approach

**Recommendation: build it as a standalone web module (Next.js app) that is *also* iframe-embeddable, not as a raw React component drop-in.**

Reasoning:
- A **React component** library forces the host portal to share Kitsch's build toolchain, data-fetching, and auth — brittle and couples release cycles.
- A **standalone Next.js app** owns its own API, auth, and data layer. It can be embedded anywhere via `<iframe>`, linked to directly, or (later) exposed as a micro-frontend. This is the most portable and the least coupled to whatever the internal portal is built in.
- Ship a thin `<iframe>` embed (with `postMessage` for height/SSO handoff) for "drop into the existing dashboard," and keep the full app reachable at its own URL for finance power users.

So: **System of record + API + dashboard = one deployable Next.js service**; embedding is a presentation concern handled by an iframe wrapper.

---

## 3. Architecture (high level)

```
┌────────────────────────────────────────────────────────────────┐
│  Host portal / internal dashboard                                │
│   └── <iframe src="https://expenses.kitsch.internal/embed">      │
└───────────────────────────┬──────────────────────────────────────┘
                            │ HTTPS + SSO (postMessage handoff)
┌───────────────────────────▼──────────────────────────────────────┐
│  Next.js app  (Vercel / container)                                │
│   • React dashboard (App Router)                                  │
│   • API routes  /api/*                                            │
│   • Auth (NextAuth + RBAC middleware)                             │
└───────┬───────────────────────────────────────────┬───────────────┘
        │                                           │
┌───────▼────────┐                       ┌──────────▼─────────────┐
│  PostgreSQL    │                       │  Sync worker (cron)     │
│  • base import │◄──────reconcile───────│  • daily NetSuite delta │
│  • ns synced   │                       │  • bucket → monthly     │
│  • sync logs   │                       │  • dedupe + reconcile   │
└───────▲────────┘                       └──────────┬─────────────┘
        │ one-time                                  │ SuiteQL / REST
┌───────┴────────┐                       ┌──────────▼─────────────┐
│ Excel importer │                       │  NetSuite (TBA OAuth1)  │
│ (folder → DB)  │                       │  SuiteQL + saved search │
└────────────────┘                       └────────────────────────┘
```

---

## 4. Data model (PostgreSQL)

Design principle: **one immutable `expense_transactions` fact table** carrying a `source` discriminator, fed by both the folder importer and the NetSuite sync, plus dimension and audit tables. Monthly grain in MVP; a nullable transaction-detail extension supports NetSuite line-level data later.

```sql
-- DIMENSIONS
CREATE TABLE departments (
  id            SERIAL PRIMARY KEY,
  name          TEXT UNIQUE NOT NULL,           -- canonical 7
  dept_head     TEXT,
  netsuite_dept TEXT
);

CREATE TABLE vendors (
  id            SERIAL PRIMARY KEY,
  name          TEXT NOT NULL,
  netsuite_id   TEXT,                            -- from NS NumName Lookup
  canonical_id  INT REFERENCES vendors(id),      -- alias collapsing (post-MVP)
  UNIQUE (name)
);

CREATE TABLE gl_accounts (
  id            SERIAL PRIMARY KEY,
  code          TEXT,                            -- e.g. 6290-20
  full_name     TEXT NOT NULL,                   -- "6290-20 - Warehouse, AMZ FBA Fees"
  category      TEXT,                            -- GL Simplified
  UNIQUE (full_name)
);

-- AUDIT / PROVENANCE
CREATE TABLE source_files (
  id            SERIAL PRIMARY KEY,
  filename      TEXT NOT NULL,
  sheet_name    TEXT NOT NULL,
  file_hash     TEXT NOT NULL,                   -- sha256, dedupe re-imports
  row_count     INT,
  period_min    DATE,
  period_max    DATE,
  imported_at   TIMESTAMPTZ DEFAULT now(),
  imported_by   TEXT,
  UNIQUE (file_hash, sheet_name)
);

-- FACT
CREATE TYPE source_kind AS ENUM ('folder_import','netsuite_sync','reconciled');

CREATE TABLE expense_transactions (
  id             BIGSERIAL PRIMARY KEY,
  period         DATE NOT NULL,                  -- first day of month (monthly grain)
  vendor_id      INT  NOT NULL REFERENCES vendors(id),
  gl_account_id  INT  NOT NULL REFERENCES gl_accounts(id),
  department_id  INT  NOT NULL REFERENCES departments(id),
  amount         NUMERIC(16,2) NOT NULL,
  expense_type   TEXT NOT NULL DEFAULT 'OpEx',          -- 'COGS' (inventory) | 'OpEx'  (see §4a)
  source         source_kind NOT NULL,
  source_file_id INT REFERENCES source_files(id),       -- folder rows
  -- NetSuite transaction-level (NULL for folder rows = "Not available")
  ns_tran_id     TEXT,
  ns_tran_type   TEXT,
  invoice_number TEXT,
  invoice_date   DATE,
  tran_status    TEXT,
  memo           TEXT,
  subsidiary     TEXT,
  -- dedupe / reconciliation
  dedupe_key     TEXT NOT NULL,                  -- see §6
  superseded_by  BIGINT REFERENCES expense_transactions(id),
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE sync_logs (
  id            SERIAL PRIMARY KEY,
  run_at        TIMESTAMPTZ DEFAULT now(),
  source        TEXT,                            -- 'netsuite'
  window_from   DATE, window_to DATE,
  rows_pulled   INT, rows_inserted INT, rows_updated INT, rows_skipped INT,
  status        TEXT,                            -- success | partial | failed
  error_detail  TEXT
);

CREATE TABLE reconciliation_logs (
  id            SERIAL PRIMARY KEY,
  run_at        TIMESTAMPTZ DEFAULT now(),
  dedupe_key    TEXT,
  folder_txn_id BIGINT, netsuite_txn_id BIGINT,
  resolution    TEXT,                            -- kept_netsuite | kept_folder | merged | flagged
  delta_amount  NUMERIC(16,2),
  note          TEXT
);

-- RBAC
CREATE TABLE users (
  id     SERIAL PRIMARY KEY,
  email  TEXT UNIQUE NOT NULL,
  role   TEXT NOT NULL DEFAULT 'viewer'          -- admin | finance | dept_head | viewer
);
CREATE TABLE user_department_access (
  user_id       INT REFERENCES users(id),
  department_id INT REFERENCES departments(id),
  PRIMARY KEY (user_id, department_id)
);
```

### 4a. COGS vs Operating Expense (inventory separation)

Inventory purchases must **not** be mixed into department expense totals — doing so overstates every department and the headline. Each transaction carries `expense_type`:

- **`COGS`** = the `Purchases - COGS` department: Inventory Purchases / Purchase Clearing ($211.8M), Freight In ($9.1M), Freight In Duties & Taxes ($5.7M), Packaging ($2.1M), Labels, Production Supplies. **Total $237.9M.**
- **`OpEx`** = everything else (Marketing, Amazon, Warehouse, Sales, G&A, Design). **Total $448.6M.**

Classification rule (folder): `expense_type = 'COGS' if department = 'Purchases - COGS' else 'OpEx'`. For NetSuite, classify by the account's COGS account-type / the inventory-purchase GL range rather than department name, since NetSuite exposes account type directly.

**Product behavior:** the Overview / Departments / Vendors / GL views filter to `expense_type='OpEx'`. A dedicated **Inventory (COGS)** view filters to `expense_type='COGS'` and answers KJ's question directly — *how much we paid each vendor for inventory* — with per-vendor totals, % of inventory spend, monthly trend, cost-type breakdown (purchases/freight/duties/packaging), and vendor drilldown. Amazon FBA fees and Freight Out are **fulfillment, not inventory**, so they remain in OpEx (Amazon/Warehouse) unless Finance directs otherwise.

**Indexing**
```sql
CREATE INDEX ix_etx_period      ON expense_transactions(period);
CREATE INDEX ix_etx_dept_period ON expense_transactions(department_id, period);
CREATE INDEX ix_etx_vendor      ON expense_transactions(vendor_id);
CREATE INDEX ix_etx_gl          ON expense_transactions(gl_account_id);
CREATE INDEX ix_etx_source      ON expense_transactions(source);
CREATE UNIQUE INDEX ux_etx_dedupe ON expense_transactions(dedupe_key) WHERE superseded_by IS NULL;
```
The partial unique index on `dedupe_key` is the structural guarantee against duplicates between folder and NetSuite (see §6).

---

## 5. Folder import logic

1. For each canonical (file, sheet), compute `sha256` → upsert `source_files`; **skip if hash already imported** (idempotent).
2. Header-detect the row containing `Month` + `Vendor Name`; map columns by name (handles 2024 vs 2025/26 drift).
3. Per data row: normalize month→`period` (YYYY-MM-01); upsert `vendors` / `gl_accounts` (parse leading GL code via regex `^[0-9]{3,5}(-[0-9]{1,3})?`) / `departments`; apply the §1 cleaning rules.
4. Compute `dedupe_key` (§6), set `source='folder_import'`, attach `source_file_id`. NetSuite-only fields stay `NULL` (surface as "Not available").
5. **Preserve originals for audit:** copy the raw `.xlsx` files into immutable object storage (S3/Box) keyed by `file_hash`; never mutate them. `source_files` is the provenance ledger.

The included Python `import_folder.py` already produces the verified normalized dataset (`expense_data.csv`/`.json`) — the Node importer mirrors that logic against Postgres.

---

## 6. NetSuite integration

**Latest date in base = 2026-04 (April).** Daily sync begins pulling transactions dated **on/after 2026-05-01**.

**Auth:** NetSuite **Token-Based Authentication (TBA), OAuth 1.0a** — Consumer Key/Secret + Token ID/Secret, scoped to an integration role with read-only access to transactions, vendors, accounts, departments. (OAuth 2.0 client-credentials is an alternative but TBA is the most broadly supported for server-to-server SuiteQL.) Secrets live in a secrets manager / env vars, never in code.

**Method:** **SuiteQL over the REST web services** (`POST /services/rest/query/v1/suiteql`) is the recommended primary — it lets us pull exactly the joined vendor/account/department/transaction shape we need and filter by `lastmodifieddate`. A **saved search exposed via RESTlet** is the fallback if SuiteQL access isn't granted on the integration role.

Representative SuiteQL (parameterized window):
```sql
SELECT t.id, t.tranid, t.trandate, t.status, t.memo,
       tl.subsidiary, v.entityid AS vendor, v.id AS vendor_id,
       a.acctnumber, a.fullname AS gl_account,
       d.name AS department,
       tl.foreignamount AS amount
FROM transaction t
JOIN transactionline tl ON tl.transaction = t.id
JOIN vendor v   ON v.id = t.entity
JOIN account a  ON a.id = tl.expenseaccount
LEFT JOIN department d ON d.id = tl.department
WHERE t.type IN ('VendBill','VendCred','Check','CardChrg')
  AND t.lastmodifieddate >= ?       -- incremental watermark
  AND t.trandate >= DATE '2026-05-01'
```

**Daily job does:**
1. Read watermark = max(`lastmodifieddate` seen) from `sync_logs`; pull only new/changed transactions since.
2. Map NetSuite vendor IDs → folder vendor names via `NS NumName Lookup`; upsert dimensions.
3. **Bucket each transaction into monthly grain** (`period = date_trunc('month', trandate)`) for the rollups, while *also* storing the transaction-level fields (`invoice_number`, `invoice_date`, `tran_status`, `memo`, `subsidiary`, `ns_tran_id`) so the vendor/GL drilldowns light up.
4. Upsert on `dedupe_key`; dedupe + reconcile (§ below).
5. Write a `sync_logs` row (counts, status, errors). **Missing/incomplete data handled gracefully:** NULL dept → `Unassigned`; NULL account → `Uncategorized`; a single bad row is logged and skipped, never aborts the run. Failed runs are retried with backoff and alert on 2 consecutive failures.

**Dedupe key & reconciliation**
- Folder rows are aggregates, so their key is the aggregate identity: `dedupe_key = md5(period || vendor || gl_account || department || 'folder')`.
- NetSuite transaction rows carry `ns_tran_id`, so their natural key is `md5('ns:' || ns_tran_id || ':' || line_no)`. They roll **up** into the same monthly buckets for charts but are stored at line grain.
- **Overlap rule:** NetSuite is authoritative for any period **≥ 2026-05**. For the historical window (≤ 2026-04) the **folder import is authoritative** and NetSuite is *not* pulled, so the two sources never write the same period — duplication is structurally impossible for the base window.
- **If a period is ever pulled from both** (e.g. a backfill): for that `period × vendor × gl × dept`, compare NetSuite rolled-up total vs folder amount. If |Δ| ≤ \$1 → keep NetSuite, mark folder `superseded_by`, log `kept_netsuite`. If |Δ| > threshold → keep both, mark `reconciled`, and **flag** in `reconciliation_logs` for finance review. Never silently overwrite.

**Should NetSuite become the system of record?** Yes — eventually, for any period it fully covers. Recommended path: folder import is SoR for 2024–Apr 2026 (frozen, audit-preserved); NetSuite is SoR from May 2026 forward. Optionally backfill NetSuite over the historical window later and, only after a reconciliation pass confirms parity within tolerance, flip those periods to NetSuite-authoritative — keeping the folder rows as `superseded` (not deleted) for audit.

---

## 7. API design (Next.js route handlers)

All endpoints are RBAC-filtered server-side to the caller's authorized departments (§9). Money returned as integer cents or string to avoid float drift.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/meta` | periods, departments, sources, date range |
| GET | `/api/summary?from&to&dept&gl&source` | totals, per-dept rollup, MoM/YoY |
| GET | `/api/trends?dimension=dept|vendor|gl&...` | monthly series for charts |
| GET | `/api/departments/:id?from&to` | dept detail + vendor breakdown |
| GET | `/api/vendors?q=&dept=` | vendor search/list |
| GET | `/api/vendors/:id?from&to` | vendor drilldown + line items |
| GET | `/api/gl/:id?from&to` | GL drilldown |
| GET | `/api/transactions?...&format=csv` | filtered line items / export |
| POST | `/api/sync/run` (admin) | trigger NetSuite sync |
| GET | `/api/sync/logs` (admin) | sync + reconciliation history |

Every transaction payload includes a `source` field (`folder_import` / `netsuite_sync` / `reconciled`) so the UI can badge provenance, and NetSuite-only fields return `null` → the UI prints "Not available."

---

## 8. Frontend component structure

```
web/
  app/
    layout.tsx
    page.tsx                  → Overview
    departments/[id]/page.tsx
    vendors/[id]/page.tsx
    gl/[id]/page.tsx
    embed/page.tsx            → iframe-optimized shell
  components/
    FilterBar.tsx             date range, dept, vendor, GL, status, source, subsidiary
    KpiCard.tsx
    SpendTrendChart.tsx       (Recharts)
    DepartmentShareChart.tsx
    TopVendorsChart.tsx
    DataTable.tsx             sortable, with SourceBadge column
    SourceBadge.tsx           "Shared Folder Import" / "NetSuite Sync"
    NotAvailable.tsx          consistent "Not available" pill
    ExportButton.tsx
  lib/ api.ts, format.ts, rbac.ts
```

The shipped `kitsch_expense_dashboard.html` is a single-file, dependency-light realization of this exact structure (vanilla JS + Chart.js) running on the real data — use it as the visual/behavioral reference when porting to React/Recharts.

---

## 9. Permissions & access

- **AuthN:** NextAuth with Kitsch SSO (Google Workspace / Okta SAML). For the iframe embed, pass a short-lived signed token via `postMessage` from the host portal, exchanged for a session.
- **AuthZ (RBAC):** roles `admin`, `finance` (all departments), `dept_head` (own department[s] via `user_department_access`), `viewer`. **Enforced in the API layer** (row filter on `department_id`), never only in the UI. Dept heads literally cannot query other departments' data.
- Audit: log every export and sync trigger with user + filter context.

---

## 10. Tech stack & trade-offs

| Layer | Choice | Why | Trade-off |
|---|---|---|---|
| Frontend | **Next.js (App Router) + React + TypeScript** | One deployable, SSR, iframe-friendly, owns its API | Heavier than a pure SPA |
| Charts | **Recharts** (prod) / Chart.js (the shipped HTML) | React-native, declarative | Very custom viz may need D3 |
| Backend | **Node + Next route handlers** (+ a separate worker for sync) | Single language, simplest ops | CPU-bound jobs belong in the worker, not routes |
| DB | **PostgreSQL** | Relational fits the star schema; window funcs for trends; partial unique index for dedupe | Needs a managed host |
| Migrations | **Drizzle or Prisma** | Typed schema, versioned | — |
| Sync schedule | **cron worker** (Vercel Cron / container cron / Cloud Scheduler) | Simple daily delta | Long syncs need a queue (post-MVP) |
| Secrets | Secrets manager / env | NetSuite TBA keys never in repo | — |
| Importer | **Python (openpyxl/pandas)** one-off + Node parity | Python already verified against source totals | Two languages; keep importer isolated |

---

## 11. Deployment plan

- **Env:** managed Postgres (RDS/Neon/Supabase), Next.js on Vercel or a container (ECS/Cloud Run), object storage for original workbooks.
- **Pipeline:** PR → CI (lint/test/migrate-check) → preview deploy → prod. Migrations run on deploy.
- **Secrets:** NetSuite TBA + DB creds in the platform secret store.
- **Observability:** structured logs, `sync_logs`/`reconciliation_logs` surfaced in an admin page, alert on failed sync.
- **Backups:** daily Postgres snapshot; original xlsx immutable in storage.

---

## 12. Error-handling strategy

- Importer: per-row try/catch, bad rows → quarantine table + count; whole-file hash guard prevents re-import.
- Sync: idempotent upserts; single-row failures logged & skipped; run-level failure retried w/ backoff; watermark only advances on success.
- API: typed errors, never leak NetSuite/DB internals; empty result = explicit empty state, not a 500.
- UI: missing fields → "Not available"; never fabricate or zero-fill silently.

---

## 13. Security considerations

- NetSuite least-privilege integration role (read-only, specific record types).
- TBA secrets in secret manager; rotate tokens; restrict outbound to NetSuite domain.
- RBAC enforced server-side; row-level department filtering.
- Audit log for exports/syncs; PII is minimal (vendor names, internal emails) but treat exports as sensitive financial data.
- iframe: set `Content-Security-Policy frame-ancestors` to Kitsch domains only; signed token handoff.

---

## 14. Phased roadmap

- **Phase 0 — Data foundation (done here):** folder inspected, normalized, verified to source totals; working dashboard on real data.
- **Phase 1 — MVP (see §15):** Postgres + importer + read API + React dashboard + daily NetSuite *delta into monthly grain* + basic RBAC + source tracking.
- **Phase 2 — Transaction depth:** confirm SuiteQL/saved-search access; store line-level NetSuite detail; light up invoice #/date/status/memo drilldowns; subsidiary + status + category filters.
- **Phase 3 — Reconciliation & SoR flip:** backfill + reconcile historical periods; optional NetSuite-authoritative flip with audit-preserved folder rows; reconciliation review UI.
- **Phase 4 — Production polish:** vendor alias crosswalk, budget vs actual (the folder already hints at budgets), anomaly flagging, scheduled email digests, full SSO/RBAC, embed hardening.

---

## 15. MVP scope (build first)

1. **Folder import** → Postgres (the verified 69,665-row, $686.5M dataset).
2. **Daily NetSuite sync** for periods ≥ 2026-05, bucketed to monthly grain, dedupe + log.
3. **Department-level summary** (totals, share, MoM).
4. **Expense growth chart** (monthly trend; YoY where 2024–2026 overlap exists).
5. **Vendor spend breakdown by department** (% of dept, top vendors).
6. **Vendor drilldown** to line-item detail (invoice fields show "Not available" until Phase 2).
7. **GL drilldown** where data exists.
8. **Basic filters** (date range, department, vendor, GL category, data source).
9. **Source tracking** (`folder_import` vs `netsuite_sync`) badged on every record.
10. **Basic RBAC** (finance = all; dept_head = own department).
11. **Inventory (COGS) view** — vendor-level inventory spend, separated from OpEx (`expense_type` split).

Explicitly **out of MVP / dependent on NetSuite config**: invoice #/date/status/memo, subsidiary & invoice-status filters, day-level dates, vendor alias de-duplication, budget-vs-actual.

---

*All dollar figures in this document and the companion dashboard are computed directly from the shared workbooks and tie exactly to their embedded control totals.*
