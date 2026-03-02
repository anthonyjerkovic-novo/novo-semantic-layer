# PRD: Novo Semantic Layer (Data Platform)

**Status:** Draft / RFC
**Owner:** Analytics Engineering (Handing off to Data Engineering)
**Last updated:** March 2026

## **1. Executive Summary & Motivation**
We are migrating our data stack from a fragile "Generic Table" architecture (`prod_db.data.businesses` with 682 upstream relationships) to a robust "Functional Mart" architecture. Currently, `prod_db` feeds into `metabase_db` (our existing Silver/Gold structure). We will leave `metabase_db` entirely as-is.

The new target architecture utilizes purpose-built fact and dimension tables organized cleanly by business function. The Analytics Engineering team has defined the end-state semantic layer (using Cube.js). **This PRD outlines the requirements for the Data Engineering team to construct the underlying *new* Gold marts in Snowflake (e.g., `NOVO_GOLD_DB`) to power this new layer.**

### Core Philosophy: Iteration & Extensibility
The new Data Platform infrastructure is built for iteration. The initial set of metrics and attributes mapped in the Gold layer may not be perfectly exhaustive on Day 1. **However, the structural setup is the critical deliverable.** By enforcing strict schema contracts and modular functionality, the infrastructure is designed so the Analytics team can easily amend, add, or remove metrics in the future without risking system-wide regressions. 

---

## **2. Target Architecture (The "Gold" Layer)**

The new Gold layer will reside in a new dedicated database in Snowflake (e.g., `NOVO_GOLD_DB`), built entirely via dbt. This layer is never queried directly from Bronze or Silver. The Cube layer will sit on top of this, exposing only public `Views` to Metabase, abstracting the underlying tables.

### Database Layout Requirements

Data Engineering must construct the following schemas and their respective tables according to the specifications in `gold_layer_spec.md`.

| Schema | Purpose / Scope |
|---|---|
| `CONFORMED` | Shared dimensions acting as universal join keys (`DIM_BUSINESSES`, `DIM_APPLICATIONS`, `DIM_USERS`, `DIM_DATE`). |
| `MART_BANKING` | Money movement: ACH, wires, mRDC, balance snapshots. |
| `MART_CARDS` | Debit + credit card lifecycle: transactions, settings, fulfillment, disputes. (Note: Kept separate from Banking/Funding deliberately). |
| `MART_FUNDING` | MCA portfolio: loan tape, repayment, delinquency, vintage. |
| `MART_FINANCE` | Unit economics: all revenue + cost components pre-joined. |
| `MART_MARKETING`| Acquisition funnel, ad spend (Google/Facebook/Microsoft), LTV. |
| `MART_PRODUCT` | Feature adoption, activation milestones, Zendesk CS. |

### Architectural Invariants
1. **Single Entry Point for Joins:** All functional marts must join through `CONFORMED.DIM_BUSINESSES`. No cross-mart dependency spaghetti.
2. **Materialized Unit Economics:** `MART_FINANCE.FACT_UNIT_ECONOMICS` joins 7+ upstream fact tables. **This must be a fully materialized table**, not a view, to prevent massive query-time fan-outs.
3. **Decoupled Fraud Processing:** Fraud is cross-cutting. Fraud attributes must live on the tables they describe (e.g., `FACT_DEBIT_CARD_TRANSACTIONS`), rather than in a detached "Fraud" mart.
4. **Business Alignment:** `@Master_Metrics.csv` is the final authority on metric definitions. The current Cube repo serves as the reference for transformation logic but *not* the structure.

---

## **3. Implementation Phases & Hand-off Instructions**

Data Engineering will inherit the transformation logic currently captured in the existing Cube models (`@novo_semantic_layer-main`). The target state is defined by the new `reworked_cube_models/`. 

### Phase 1: Pre-Processing Gates (Silver Layer Cleanup)
Before constructing the Gold marts, three critical upstream data issues must be resolved to ensure reliability. Data Engineering must build dbt intermediate models to solve these:

1. **MCA Loan Tape Reconciliation:** Create `INT_MCA_LOAN_TAPE_RECONCILED` to resolve conflicts between Aurora and LMS sources across 33 `CANOPY_LENDING.*` tables.
2. **Ad Spend Normalization:** Create `INT_AD_SPEND_NORMALIZED` as a union model to standardize the incompatible schemas from Google, Facebook, and Microsoft (60+ raw tables).
3. **Float Rate Version Control:** Migrate `PROD_DB.MODELS.FLOAT_RATE_TABLE` (currently manually edited) to a version-controlled dbt seed file.

### Phase 2: Gold Mart Construction
Build the schemas outlined in Section 2 using dbt inside the new Gold layer (`NOVO_GOLD_DB`). 
- Use `gold_layer_spec.md` for column-level requirements and source mappings.
- Ensure all 37 legacy Cube measures have a mapped Gold home.
- Implement the 5 newly identified metrics: `CC Outstanding Balance`, `CC Exposure / Credit Limit`, `CC Utilization Rate`, `CC Total Spend`, `First 90 Day Customer (F90D)`.

---

## **4. Release Strategy & Parallel Deployment**

**Migration Strategy:** We will leave both the legacy (`prod_db` / `metabase_db`) and new (`NOVO_GOLD_DB`) systems running in tandem. We will migrate all business usage (Metabase dashboards) sequentially to the new system. Once the migration is complete and parity is validated, we will then define the deprecation and sunset plan for the old setup.

### Deployment Steps
1. **Build & Parallel Run:** Deploy new Gold dbt models to Snowflake alongside the existing `metabase_db` tables.
2. **Cube Registration:** Register the `reworked_cube_models/` in the Cube instance. They are safely isolated by the `NOVO_GOLD_DB.*` prefix and public `Views`, having zero collision with the existing `PROD_DB` / `METABASE_DB` models.
3. **Validation:** Analytics Engineering validates the new Views in the Cube Playground ensuring parity with existing dashboard calculations.
4. **Migration:** Point Metabase dashboards one-by-one to the new target Views. 
5. **Post-Migration Assessment:** Once 100% of traffic is routed through the new layer, the combined engineering/analytics teams will strategize the teardown of the legacy `DATA.BUSINESSES` dependencies.

---

## **5. Open RFC Items for Data Engineering**

Please review and provide feedback on the following implementation details:
1. **Canopy Source Stability:** Is Canopy (LMS) the stable source for the MCA Loan Tape, or is Aurora still in flight? This dictates the tie-breaker logic for `INT_MCA_LOAN_TAPE_RECONCILED`.
2. **Cube Push-Down Boundary:** Analytics Engineering recommends that everything computed at a per-business-per-month grain is materialized in dbt, leaving only cross-period aggregations and UI filters to Cube. Do we agree on this physical boundary?
3. **`PROD_DB.MODELS.*` Governance:** How do we transition ML model output tables into proper version-controlled dbt models before they are promoted to Gold?
