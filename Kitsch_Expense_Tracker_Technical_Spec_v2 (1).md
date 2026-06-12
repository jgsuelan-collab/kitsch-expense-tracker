# Kitsch Expense Tracker — Technical Spec **v2.0** (Action-Oriented Expense Management)

**Supersedes/extends** `Kitsch_Expense_Tracker_Technical_Spec.md` (v1). v1's data inspection, folder import, base architecture, RBAC, and deployment still apply — this document adds department pages, left-side navigation, revenue integration, a benchmark + recommendation engine, SPLY comparisons, influencer handling, the expanded data model, and the revised MVP.

**Date:** 2026-06-11 · Prepared for KJ Cabatana

---

## 0. What changed and the dependency that gates it

v2 reframes the product from a **reporting dashboard** into an **action-oriented expense-management tool** that tells each department head *what to investigate and where to save*. The companion `kitsch_expense_dashboard.html` already implements this on real folder data: left sidebar, per-department pages, headline action cards, a rules-based recommendation engine, SPLY bar charts, vendor/cost-type share + growth, and a dedicated Influencers view.

**The gating dependency (read first):** every "expense as % of revenue" benchmark — Amazon FBA % of Amazon revenue, Facebook ads % of DTC revenue, influencer % of DTC revenue — requires **revenue split by channel**, which the folder does not contain and which NetSuite does not expose as a single native field. Channel revenue almost always lives behind **NetSuite Classes, custom segments, location, or saved searches**, and must be mapped. Until that mapping is confirmed and synced, those benchmarks render **"Revenue not connected"** in the UI. Everything else (SPLY growth, vendor concentration, cost-type spikes, MoM anomalies) ships now because it needs expenses only.

What the folder already supports, verified: SPLY for 2026 (Jan–Apr) vs 2025; "Influencers" exists as an explicit cost type ($5.1M, with broader creator spend ~$26M scattered and needing a mapping rule); Facebook Ads is a $60M cost type; 7 departments; cost types = the GL "Simplified" categories.

---

## 1. Page inventory (each is its own route)

| Route | Page | Focus |
|---|---|---|
| `/` | Company Overview | OpEx total, SPLY, department share, largest growth driver, company recs |
| `/dept/:id` | Department page | One department only: headline cards, recs, SPLY, vendor & cost-type share + growth, drilldown |
| `/dept/marketing` | Marketing page | Department page + Facebook-ads benchmark card + Influencers entry |
| `/marketing/influencers` | Influencer dashboard | Influencer spend, SPLY, by vendor/creator, by GL, by month, recs |
| `/vendor/:id` | Vendor detail | Vendor spend, SPLY, by dept, by GL, line items |
| `/cost/:id` | Cost type detail | Cost-type spend, SPLY, top vendors, line items |
| `/gl/:id` | GL account detail | GL spend, SPLY, vendors, line items |
| `/invoice/:id` | Invoice drilldown | **NetSuite-dependent** — "Not available" until transaction sync |
| `/inventory` | Inventory (COGS) | Vendor inventory spend, separate from OpEx |

Each **department page is scoped to that department only** — its vendors, cost types, trends, benchmarks, and savings opportunities.

---

## 2. Left-side filter navigation

A persistent **left sidebar** holds page navigation + filters. **Department is the primary filter** (the page nav itself). Once a department is active, the sidebar shows department-scoped filters dynamically:

- Period from / to · Comparison (SPLY / prior month) · Cost type (that dept's) · GL account (that dept's) · Vendor search (that dept's) · Invoice status *(disabled — NetSuite)* · Subsidiary/BU *(disabled — NetSuite)* · Data source.

Each department's main page shows **each vendor's share of that department's total** (share table + doughnut). The sidebar's vendor/cost-type/GL lists are computed from the selected department's rows, not the global set.

---

## 3. Department-level analytics & warning indicators

Every department page computes and displays:

- Total expense; **vs SPLY** (same months CY vs PY); MoM movement; YoY movement.
- **Expense as % of relevant revenue** *(stub until NetSuite revenue — see §4)*.
- Vendor share of department total; cost-type share of department total.
- Largest drivers of expense growth (by absolute Δ vs SPLY, vendor and cost type).
- Suggested savings areas (recommendation engine, §9).

Warning logic is not just display — the engine evaluates whether spend "appears high, is growing unusually, or should be reviewed."

---

## 4. Revenue data from NetSuite (design + field availability)

Revenue is required for benchmarks. Design:

**Pull** (daily, alongside expenses) via **SuiteQL** over transaction/transactionline, filtered to income/sales transaction types and `trandate`, joined to the channel dimension. Store in `revenue_transactions` at monthly grain (`period`, `channel_id`, `amount`, plus `ns_tran_id` detail), and roll into `vendor_department_monthly_metrics` denominators.

**Channel mapping — be explicit about what's native vs derived:**

| Channel | Likely NetSuite source | Availability |
|---|---|---|
| Amazon revenue | Class/segment = "Amazon", or a dedicated subsidiary/location | **Mapping required** (Class or saved search) |
| Shopify / Kitsch DTC | Class/segment = "DTC/Shopify"; or integration-tagged sales orders | **Mapping required** |
| Wholesale | Customer category / class = "Wholesale" | **Mapping required, if tracked** |
| Total revenue by month | `SUM(income)` by period | **Native (SuiteQL)** |
| Revenue by channel | Depends on the Class/segment scheme above | **Derived via mapping** |

The app must **clearly label** each revenue field as native, mapped, saved-search-derived, or unavailable. Until mapping is confirmed, total revenue may sync (native) while channel splits stay "Not available," and channel benchmarks stay inactive.

**Benchmark denominators:** Amazon dept → Amazon revenue; Marketing/Facebook → Shopify/DTC revenue; influencers → DTC revenue; others → total revenue (configurable per `benchmark_rules`).

---

## 5. Benchmark engine (examples)

`benchmark_rules` defines flexible, per-department formulas. Each rule: numerator (expense filter: dept/vendor/cost-type/GL), denominator (revenue channel), threshold type (vs SPLY ratio, or absolute %), and message template.

- **Amazon FBA % of Amazon revenue** — flag if ratio > prior-year ratio or > threshold → *"Look into Amazon FBA costs. FBA as a % of Amazon revenue is higher than last year."*
- **Facebook ads % of Shopify/DTC revenue** — flag if increasing vs SPLY → *"Review Facebook ads efficiency. FB ads as a % of DTC revenue increased vs last year."*

Rules are data-driven so new ones add without code. `benchmark_results` stores each evaluation (see §12).

---

## 6. Same-period-last-year (SPLY) comparison

The dashboard's primary comparison is **CY vs PY on overlapping months** (e.g., Apr 2026 vs Apr 2025; 2026 YTD vs 2025 YTD). Implemented as grouped **bar charts** (PY vs CY by month) on every scope: company, department, vendor, cost type, GL. SPLY % change appears as a colored pill on every share/growth table. (Verified: Marketing 2026 YTD = +47.8% vs SPLY.)

---

## 7. Vendor & cost-type growth analysis

Each department page shows four ranked tables: **top vendors by CY spend**, **top vendors by YoY growth (Δ vs PY)**, **top cost types by CY spend**, **top cost types by YoY growth**. Rows exceeding growth/materiality thresholds are flagged and feed the recommendation engine ("unusual increases / should be reviewed"). Click any row → drill to that vendor/cost-type detail.

---

## 8. Influencer expenses

- Influencers are a **cost type / category**, grouped from the explicit "Influencers" GL-Simplified value.
- A dedicated **Influencers dashboard under Marketing** shows: total influencer spend, vs SPLY, % of relevant revenue *(stub)*, by vendor/creator, by GL account, by month, and a review recommendation if spend rises materially vs SPLY.
- **Honesty:** campaign- and creator-level detail is **not available** in folder data, and broader creator/UGC spend (TikTok, agencies, content production) sits outside the "Influencers" category. The UI states this and offers a future **influencer mapping rule** (vendor/GL → "Influencer") to attribute it fully once NetSuite/classification confirms.

---

## 9. Recommendation engine (rules-based MVP, AI-ready)

A deterministic engine evaluates each department's rows and emits recommendation cards. Shipped rules:

1. **Vendor concentration** — top vendor share ≥ 30% → review contract terms. *(e.g., Facebook = 55% of Marketing.)*
2. **Cost-type YoY growth** — cost type up ≥ 25% vs SPLY and ≥ $50k → investigate driving invoices. *(e.g., Facebook Ads +60%, +$6.6M.)*
3. **Vendor YoY growth** — vendor up ≥ 40% vs SPLY and material → confirm scope/rate.
4. **MoM spike** — latest month ≥ 1.5× trailing-3-month avg → check largest line items.
5. **Revenue-ratio benchmark** *(stub)* — emits a "connect NetSuite revenue" card naming the dept-specific benchmark.

Thresholds live in `recommendation_rules` (data-driven). Each generated recommendation is written to `generated_recommendations` after every sync. **AI summaries are a later layer** that consumes the same structured triggers to write narrative headlines.

---

## 10. Department-head guidance (every recommendation carries)

`issue` · `why it matters` · `triggering metric` · `entity (vendor/cost-type/GL)` · `suggested action` · `drilldown link` to the invoices/transactions behind it. The goal is to point the lead at *where money can be saved*, not just report numbers.

---

## 11. UI/UX (implemented in the companion dashboard)

Left-side filters · department landing pages · vendor-share & cost-type-share cards · SPLY comparison bar charts · benchmark cards · recommendation cards · savings-opportunity headlines · Marketing dashboard · Influencer dashboard · drilldown tables (vendor / cost type / GL / invoice). Each department page **opens with four headline cards**: *Main expense to review · Largest driver of growth · Top savings opportunity · Growing faster than revenue?* Executive-friendly, action-first, on Kitsch's brand palette.

---

## 12. Data model additions (v2)

```sql
CREATE TABLE revenue_channels (
  id SERIAL PRIMARY KEY, name TEXT UNIQUE,            -- Amazon, Shopify/DTC, Wholesale, Total
  ns_mapping TEXT,                                    -- class/segment/saved-search ref
  availability TEXT                                   -- native | mapped | saved_search | unavailable
);
CREATE TABLE revenue_transactions (
  id BIGSERIAL PRIMARY KEY, period DATE NOT NULL,
  channel_id INT REFERENCES revenue_channels(id),
  amount NUMERIC(16,2) NOT NULL, source source_kind NOT NULL,
  ns_tran_id TEXT, created_at TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE cost_types (                              -- normalized GL "Simplified" categories
  id SERIAL PRIMARY KEY, name TEXT UNIQUE,
  is_influencer BOOLEAN DEFAULT false,                -- influencer grouping flag
  department_id INT REFERENCES departments(id)        -- nullable; cost types can span depts
);
CREATE TABLE benchmark_rules (
  id SERIAL PRIMARY KEY, name TEXT, department_id INT REFERENCES departments(id),
  numerator_filter JSONB,                             -- {cost_type|vendor|gl}
  denominator_channel_id INT REFERENCES revenue_channels(id),
  threshold_type TEXT,                                -- 'vs_sply_ratio' | 'abs_pct'
  threshold_value NUMERIC, message_template TEXT, active BOOLEAN DEFAULT true
);
CREATE TABLE benchmark_results (
  id BIGSERIAL PRIMARY KEY, rule_id INT REFERENCES benchmark_rules(id),
  period DATE, department_id INT, ratio_cy NUMERIC, ratio_py NUMERIC,
  flagged BOOLEAN, computed_at TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE recommendation_rules (
  id SERIAL PRIMARY KEY, code TEXT UNIQUE,            -- 'vendor_concentration', 'cost_type_yoy'...
  params JSONB,                                       -- thresholds
  message_template TEXT, severity TEXT, active BOOLEAN DEFAULT true
);
CREATE TABLE generated_recommendations (
  id BIGSERIAL PRIMARY KEY, run_at TIMESTAMPTZ DEFAULT now(),
  department_id INT, rule_code TEXT, severity TEXT,
  issue TEXT, why TEXT, metric TEXT, entity TEXT,
  entity_type TEXT,                                   -- vendor|cost_type|gl|department
  action TEXT, drill_type TEXT, drill_value TEXT, flagged BOOLEAN DEFAULT true
);
CREATE TABLE department_dashboard_configs (
  department_id INT PRIMARY KEY REFERENCES departments(id),
  headline_order TEXT[], default_comparison TEXT DEFAULT 'sply',
  revenue_channel_id INT REFERENCES revenue_channels(id), settings JSONB
);
-- pre-computed metrics powering pages fast (refreshed each sync)
CREATE TABLE vendor_department_monthly_metrics (
  id BIGSERIAL PRIMARY KEY,
  period DATE NOT NULL, department_id INT, vendor_id INT, cost_type_id INT, gl_account_id INT,
  cy_value NUMERIC(16,2), py_value NUMERIC(16,2),
  abs_change NUMERIC(16,2), pct_change NUMERIC,
  revenue_denominator NUMERIC(16,2), benchmark_threshold NUMERIC,
  flagged_for_review BOOLEAN DEFAULT false
);
CREATE INDEX ix_vdmm ON vendor_department_monthly_metrics(department_id, period);
```

Add `expense_type` (from v1) and ensure `cost_type_id` FK on `expense_transactions`. Each computed metric row stores period, department, optional vendor/cost-type/GL, CY value, PY value, abs change, pct change, revenue denominator, benchmark threshold, and the flagged boolean — exactly as requested.

---

## 13. NetSuite integration updates

The daily job now: (a) keeps the folder base import (history); (b) syncs **expenses** from NetSuite (≥ 2026-05); (c) syncs **revenue** from NetSuite and maps to channels (Amazon/Shopify/DTC/wholesale where available); (d) maps expenses to dept/vendor/cost-type/GL/invoice; (e) **recomputes `vendor_department_monthly_metrics` and `benchmark_results`** after each sync; (f) **regenerates `generated_recommendations`**; (g) logs status/errors; (h) preserves folder-vs-NetSuite auditability (source discriminator + reconciliation from v1). Revenue uses the same TBA OAuth1 auth and dedupe/watermark pattern.

---

## 14. Revised MVP

1. Company-wide overview page. 2. Department-specific pages. 3. Left-side filters. 4. Daily NetSuite sync for **expenses and revenue**. 5. Shared-folder base import. 6. **SPLY bar charts**. 7. Vendor share of department. 8. Cost-type growth analysis. 9. **Amazon FBA % of Amazon revenue** *(if fields available)*. 10. **Facebook ads % of Shopify/DTC revenue** *(if fields available)*. 11. **Influencers grouped under Marketing**. 12. Marketing dashboard. 13. **Recommendation cards**. 14. Drilldown from recs → vendor / cost type / GL / invoice / transaction.

Revenue-dependent items (9, 10, influencer-%) ship in **stub state** with clear "Revenue not connected" labels until the NetSuite channel mapping is confirmed; all expense-only items are fully live in the companion dashboard today.

---

## 15. Design intent

Not a passive report. Each department head should get direct answers to: *What grew the most? Which vendor takes the largest share of my budget? What cost type is growing faster than last year? Which expenses are growing faster than revenue? What do I investigate first? Which invoices/GL accounts caused the increase? Where can we save?* The headline cards + recommendation engine answer the first six on expense data today; the seventh (vs-revenue) activates with NetSuite revenue.

---
*Companion: `kitsch_expense_dashboard.html` (live on real folder data) and `expense-tracker-starter/` (schema, importer, NetSuite expense+revenue sync, recommendation engine).*
