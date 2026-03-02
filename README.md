# Novo Semantic Layer — Rearchitected

> **Status:** Parallel deployment candidate — safe to run alongside existing `prod_db` architecture
> using the strangler fig pattern.
>
> **Owner:** Data Engineering / Analytics Engineering
> **Last updated:** February 2026

---

## What This Is

This repository contains two things:

1. **The existing Cube.js semantic layer** — currently pointing at `DEV_DB.DEV_ANWAR.*` and `PROD_DB.DATA.*`
2. **`/reworked_cube_models/`** — a fully rearchitected Cube layer pointing at `NOVO_GOLD_DB.*` functional marts (the new Gold layer)

The goal is to migrate from a **Generic Table** architecture (`prod_db` → `DATA.BUSINESSES` with 682 relationships) to a **Functional Mart** architecture (`NOVO_GOLD_DB` → purpose-built fact and dimension tables organized by business function). We are leaving our existing `prod_db` → `metabase_db` pipelines as-is and building this as a parallel, dedicated structure.

---

## Repo Structure

```
novo_semantic_layer-main/
│
├── README.md                       ← you are here
├── gold_layer_spec.md              ← full column-level spec for every Gold table
├── ARCHITECTURE_DECISIONS.md       ← 1-pager on all key architectural decisions
│
├── model/                          ← EXISTING Cube models (do not modify)
│   └── *.js / *.yml               ← currently pointing at DEV_DB / PROD_DB
│
└── reworked_cube_models/           ← NEW rearchitected layer (Gold-pointed)
    │
    ├── cubes/                      ← Private cubes (public: false)
    │   │                              Not visible to Metabase directly
    │   ├── conformed/
    │   │   ├── dim_businesses.yml  ← Shared account + status dimensions
    │   │   └── dim_applications.yml← Shared acquisition + onboarding dimensions
    │   ├── banking/
    │   │   ├── fact_transactions.yml        ← ACH, wires, mRDC, checks (gross)
    │   │   └── fact_ach_wires_balances.yml  ← Rail-specific decisions + ADB
    │   ├── cards/
    │   │   └── fact_card_transactions.yml   ← Debit txns + Credit txns + Lithic settlement
    │   ├── funding/
    │   │   └── fact_mca_cubes.yml           ← Loan tape + MCA revenue + MCA events
    │   ├── finance/
    │   │   └── fact_finance_cubes.yml       ← Unit economics + float revenue
    │   ├── marketing/
    │   │   └── fact_marketing_cubes.yml     ← Ad spend + acquisition cost + LTV
    │   └── product/
    │       └── fact_product_cubes.yml       ← Feature adoption + Zendesk CS
    │
    └── views/                      ← PUBLIC views (Metabase-facing)
        │                              These are the ONLY objects Metabase queries
        ├── v_core_financial_and_payments.yml  ← Account lifecycle + revenue + payments
        ├── v_marketing_and_credit_risk.yml    ← Funnel + CAC + LTV + MCA/CC risk
        └── v_product_and_cs.yml               ← Feature adoption + support tickets
```

---

## Target Database Layout (`NOVO_GOLD_DB`)

The new Gold layer lives in a new dedicated database (e.g., `NOVO_GOLD_DB`) in Snowflake. It is built by dbt, never queried directly from Bronze or Silver.

| Schema | Purpose |
|---|---|
| `CONFORMED` | Shared dimensions — `DIM_BUSINESSES`, `DIM_APPLICATIONS`, `DIM_USERS`, `DIM_DATE` |
| `MART_BANKING` | Money movement: ACH, wires, mRDC, balance snapshots |
| `MART_CARDS` | Debit + credit card: transactions, settings, fulfillment, disputes |
| `MART_FUNDING` | MCA portfolio: loan tape, repayment, delinquency, vintage |
| `MART_FINANCE` | Unit economics: all revenue + cost components pre-joined |
| `MART_MARKETING` | Acquisition funnel, ad spend (Google/Facebook/Microsoft), LTV |
| `MART_PRODUCT` | Feature adoption, activation milestones, Zendesk CS |

Full column-level specs, source mappings, and metric homes: **[`gold_layer_spec.md`](./gold_layer_spec.md)**

---

## The Cube View Pattern

```
Metabase (end user)
      │
      ▼
   VIEWS  (public: true)         ← what Metabase sees
   e.g., v_core_financial
      │
      ▼
   CUBES  (public: false)        ← private, modular, reusable
   e.g., dim_businesses
        + fact_unit_economics
      │
      ▼
   NOVO_GOLD_DB.*                 ← Gold layer tables built by dbt
```

All `cubes/` are `public: false`. Metabase users only ever see the 5 clean Views. This keeps the underlying mart logic invisible and gives you full flexibility to refactor cubes without breaking Metabase dashboards.

---

## How to Deploy Alongside Existing Architecture (Parallel Deployment)

This is safe because the new models use a **different `sql_table` prefix** (`NOVO_GOLD_DB.*`) from the existing models (`DEV_DB.DEV_ANWAR.*` and `PROD_DB.DATA.*`). They can coexist in the same Cube instance and parallel to the current `metabase_db` architecture with zero collision.

### Step-by-Step

**Step 1 — dbt: Build the Gold Tables**
```bash
# Run the dbt models that populate NOVO_GOLD_DB in Snowflake
# Requires: INT_MCA_LOAN_TAPE_RECONCILED, INT_AD_SPEND_NORMALIZED, float_rate seed
dbt run --select mart_banking mart_cards mart_funding mart_finance mart_marketing mart_product conformed
dbt test --select mart_banking mart_cards mart_funding mart_finance mart_marketing mart_product conformed
```

**Step 2 — Cube: Register the new models**
```bash
# Point CUBE_DB_TYPE at Snowflake, confirm NOVO_GOLD_DB is in scope
# Add reworked_cube_models/ to your cubejs.config.js schemaPath or copy files into model/
```

**Step 3 — Validate in Cube Playground**
Confirm each View resolves without SQL errors:
- `v_core_financial` → spot-check `opened_accounts` by `account_opened_month`
- `v_payments` → spot-check `gross_transactions` by `derived_medium`
- `v_marketing` → spot-check `cost_per_opened_account` by `application_channel_derived`
- `v_credit_risk_mca` → spot-check `outstanding_balance_mca` by `delinquency_bin`
- `v_credit_risk_cc` → spot-check `cc_outstanding_balance` (new measure)
- `v_product_adoption` → spot-check `invoice_attach` and `is_f90d` (new dimension)
- `v_customer_success` → spot-check `contact_volume` by `created_channel`

**Step 4 — Wire to Metabase**
Connect the Views to new Metabase questions/dashboards. Run in parallel against existing dashboards to validate parity.

**Step 5 — Parallel Run & Cutover**
Once parity is confirmed per dashboard:
1. Point the Metabase question at the new View
2. Deprecate the old Cube model (`public: false`)
3. Remove the old Silver reference after 2 sprint cycles

---

## Three Pre-Processing Gates (dbt — must complete before Gold)

Before the Gold marts are reliable, three Silver-layer problems must be resolved:

| Gate | Problem | Required dbt Work |
|---|---|---|
| **MCA Loan Tape** | `CANOPY_LENDING.*` (33 tables, 3 source layers, 167 cross-refs) conflicts with `DATA.LOAN_TAPE` | Build `INT_MCA_LOAN_TAPE_RECONCILED` intermediate model |
| **Ad Spend** | Google (22 tables), Facebook (16 tables), Microsoft (22 tables) — inconsistent schemas | Build `INT_AD_SPEND_NORMALIZED` union model |
| **Float Rate** | `PROD_DB.MODELS.FLOAT_RATE_TABLE` is manually edited in Snowflake with zero version control | Migrate to versioned dbt seed file |

---

## Key Files for New Team Members

| File | What it is |
|---|---|
| [`gold_layer_spec.md`](./gold_layer_spec.md) | Column-level table specs for every Gold mart, metric homes, and a validation table confirming all 37 legacy Cube measures have a Gold home |
| [`ARCHITECTURE_DECISIONS.md`](./ARCHITECTURE_DECISIONS.md) | 1-pager summary of every major architectural decision and the rationale |
| [`reworked_cube_models/cubes/conformed/`](./reworked_cube_models/cubes/conformed/) | Start here — `DIM_BUSINESSES` is the universal join key for all marts |
| [`reworked_cube_models/views/`](./reworked_cube_models/views/) | The 5 public Cube views that Metabase will query |

---

## Metrics Coverage

All metrics from `novo critical metrics.csv` are documented with:
- Their **Gold table home** (mart + table + column)
- **Aggregation type** (COUNT DISTINCT, SUM, AVG)
- **Full SQL logic** extracted from the existing Cube repo

See [`re-architected_semantic_layer.md`](../../.gemini/antigravity/brain/1ce10039-8448-4169-9b36-03a23ffb0c7c/re-architected_semantic_layer.md) for the complete metrics catalog.

> **Note for new team members:** 5 metrics were found **missing from the legacy Cube implementation** and are newly added here:
> `CC Outstanding Balance`, `CC Exposure / Credit Limit`, `CC Utilization`, `CC Total Spend`, `First 90 Day Customer (F90D)`.

---

## Questions / Owners

| Area | Owner |
|---|---|
| Data Platform / dbt | tbd |
| Banking & Cards | tbd |
| Funding / MCA Risk | tbd |
| Finance / Treasury | tbd |
| Marketing / Growth | tbd |
| Product | tbd |
| Customer Success | tbd |
