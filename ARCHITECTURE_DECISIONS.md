# Novo Semantic Layer — Architecture Decisions
### 1-Pager for Engineering Leadership & New Team Members

> **Scope:** Migration from generic-table Silver (`prod_db`) to a new functional Mart Gold layer (`NOVO_GOLD_DB` or equivalent), while leaving the legacy `metabase_db` undisturbed.
> **Date:** February 2026 | **Authors:** Analytics Engineering

---

## The Problem We Are Solving

The current semantic layer queries `PROD_DB.DATA.BUSINESSES` — a 98-column table with **682 upstream relationships**. Every dashboard query fans out through this single table, making the layer fragile, slow, and impossible to own clearly. `DEV_DB.DEV_ANWAR.*` models have no production promotion path, so "dev" work is quietly powering production dashboards.

---

## Decision 1 — Adopt a Functional Mart Architecture

| | Before | After |
|---|---|---|
| Grain | Generic tables organized by system (`DATA.*`, `MODELS.*`) | Purpose-built marts organized by business function |
| Join pattern | Fan out through `DATA.BUSINESSES` (682 relationships) | All marts join through `CONFORMED.DIM_BUSINESSES` (single, incremental dimension) |
| Cube `sql_table` | `DEV_DB.DEV_ANWAR.*` (dev, unversioned) | `NOVO_GOLD_DB.<MART>.<TABLE>` (production Gold) |
| Metabase access | Direct cube queries | Cube Views only — underlying cubes are `public: false` |

**Why:** Decouples metric ownership by team, enforces schema contracts, eliminates the 682-relationship bottleneck.

---

## Decision 2 — MART_CARDS Gets Its Own Schema

Cards is not split into Banking (debit) and Funding (credit). It gets its own schema:

**Rationale:**
- Card ordering, fulfilment, and dispute management don't belong in either MART_BANKING or MART_FUNDING
- Cross-card analytics (debit vs credit attach rate, dispute outcomes across programs) require zero cross-mart joins
- `DIM_CARD_SETTINGS` (freeze, PIN, limits) and `FACT_CARD_DISPUTES` span both card types — one mart owns them cleanly
- Cards team owns the full card lifecycle regardless of whether the underlying product is a debit or credit account

---

## Decision 3 — MART_FUNDING ≠ MART_CREDIT

MCA and Credit Card are separate marts:

| Dimension | MCA (MART_FUNDING) | Credit Card (MART_CARDS) |
|---|---|---|
| Underwriting system | Canopy LMS + Aurora | Lithic issuer |
| Repayment | Daily ACH sweep | Monthly billing cycle |
| Risk metric | DPD from Canopy loan tape | DPD from CC loan tape |
| Revenue | Factor fee × balance | Interest + late fees |
| Team owner | Funding / Risk | Cards / Banking |

**Why:** One `MART_CREDIT` would create the same kind of multi-owner, multi-grain chaos that `DATA.BUSINESSES` already is.

---

## Decision 4 — Fraud Has No Mart

Fraud is cross-cutting, not a business function. Fraud attributes live on the tables they describe:

- `EXPENSED_FRAUD_LOSS` → `CONFORMED.DIM_BUSINESSES` (account-level flag)
- `IS_ACCOUNT_CLOSED_FOR_SUSPECTED_FRAUD` → `CONFORMED.DIM_BUSINESSES`
- `SARDINE_TRANSACTION_RISK_LEVEL` → `MART_CARDS.FACT_DEBIT_CARD_TRANSACTIONS` + `FACT_CREDIT_CARD_TRANSACTIONS`
- `FRAUD_LOSS` (P&L view) → `MART_FINANCE.FACT_UNIT_ECONOMICS`

**Why:** A standalone MART_FRAUD would duplicate columns from six other marts. Fraud signals are attributes of other facts, not facts themselves.

---

## Decision 5 — `FACT_UNIT_ECONOMICS` Must Be Materialized

`MART_FINANCE.FACT_UNIT_ECONOMICS` joins 7+ upstream fact tables into one pre-resolved monthly business × period row. **This must be a fully materialized dbt model — never a view.**

If built as a view, any Metabase query that touches multiple revenue + cost lines triggers 7 simultaneous fan-out joins at query time. The table collapses under normal dashboard load.

---

## Decision 6 — Cube Views Gate All Metabase Access

End users and Metabase only see Cube **Views**. All underlying Cubes are `public: false`.

```
Metabase → Views (public) → Cubes (private) → NOVO_GOLD_DB Gold Tables
```

**Why:** Lets us refactor the underlying cube logic or swap Gold table structures without breaking any Metabase question. Metabase builds on the View contract, not the cube implementation.

---

## Decision 7 — Three Silver Pre-Processing Gates

Three Silver sources are **too messy to promote directly** to Gold:

| Source | Problem | Resolution |
|---|---|---|
| `CANOPY_LENDING.*` (33 tables) | Aurora vs. LMS source conflicts on the same accounts | `INT_MCA_LOAN_TAPE_RECONCILED` dbt intermediate |
| `MODELS.FLOAT_RATE_TABLE` | Manually edited in Snowflake, zero version control or lineage | Migrate to versioned dbt seed file before Gold is reliable |
| 60+ ad platform raw tables | Google / Facebook / Microsoft have incompatible column schemas | `INT_AD_SPEND_NORMALIZED` union model |

**Why this matters:** Promoting dirty Silver directly to Gold compounds the existing lineage problem rather than solving it.

---

## Decision 8 — Deployment via Parallel Run

The new `reworked_cube_models/` layer uses `NOVO_GOLD_DB.*` as its `sql_table` prefix. The existing models use `DEV_DB.DEV_ANWAR.*` and `PROD_DB.DATA.*`. **They coexist safely in the same Cube instance and parallel to the current `metabase_db` architecture with zero collision.**

Proposed cutover sequence:
1. Build Gold dbt models in Snowflake (parallel to Silver)
2. Register `reworked_cube_models/` in Cube (parallel to existing `model/`)
3. Validate each View in Cube Playground against existing dashboard parity
4. Migrate Metabase questions one dashboard at a time
5. Deprecate old Cube models as each is decommissioned
6. Remove Silver references from Cube after 2 sprint cycles of clean operation

---

## 5 Missing Metrics Found During This Work

These metrics exist in `novo critical metrics.csv` but had **no Gold home and no Cube implementation**:

| Metric | Status | New Home |
|---|---|---|
| `CC Outstanding Balance` | ⚠️ Missing from Cube | `MART_CARDS.FACT_CREDIT_CARD_TRANSACTIONS` |
| `CC Exposure / Credit Limit` | ⚠️ Missing from Cube | `MART_CARDS.DIM_CREDIT_CARD_APPLICATION_FUNNEL` |
| `CC Utilization Rate` | ⚠️ Missing from Cube | `MART_CARDS.FACT_CREDIT_CARD_TRANSACTIONS` |
| `CC Total Spend` | ⚠️ Missing from Cube | `MART_CARDS.FACT_CREDIT_CARD_TRANSACTIONS` |
| `First 90 Day Customer (F90D)` | ⚠️ Missing from Cube | `MART_PRODUCT.FACT_ACTIVATION_MILESTONES` |

---

## Open Questions for Leadership

1. **Canopy source stability** — During any migration, is Canopy (LMS) the stable source, or is Aurora still in flight? Answer determines whether `INT_MCA_LOAN_TAPE_RECONCILED` de-prioritizes LMS or Aurora as the tie-breaker.

2. **`PROD_DB.MODELS.*` version control** — Are ML model output tables in `MODELS.*` version-controlled dbt outputs, or analyst-managed worksheets? If the latter, none of them are safe for direct Gold promotion.

3. **Cube Push-Down Boundary** — Where exactly does `NOVO_GOLD_DB` materialization end and Cube push-down SQL begin? Recommendation: everything computed at a per-business-per-month grain is materialized in dbt. Cube handles only cross-period aggregations and user-facing filters.
