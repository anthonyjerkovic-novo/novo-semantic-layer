# TEST_MATRIX.md — Metric Validation: `prod_db` → `metabase_db`

> **Purpose:** Side-by-side comparison of old metric logic (Silver/DEV layer) vs. new metric logic
> (Gold functional marts) for the 5 most critical Novo business metrics.
> Each validation query is Snowflake-ready and can be run directly against production.
>
> **How to use:** Run each validation query, compare `OLD_VALUE` vs `NEW_VALUE`.
> A variance of 0 (or within acceptable rounding tolerance) confirms the migration is safe.

---

## Why These 5 Metrics

| Rank | Metric | Rationale |
|---|---|---|
| 1 | **Financially Active Accounts** | Primary board-level health KPI — denominator for all retention analysis |
| 2 | **Marginal Net Income** | Core profitability signal — the single most complex aggregation in the stack |
| 3 | **Average Daily Balance (ADB)** | Treasury-critical — drives float revenue and the interbank reserve calculation |
| 4 | **Total Interchange Revenue** | Largest predictable revenue line — any variance immediately flags a settlement data gap |
| 5 | **MCA Outstanding Balance** | Risk team's portfolio exposure signal — a variance here means a charge-off model error |

---

## 1. Financially Active Accounts

| | Detail |
|---|---|
| **Metric Name** | Financially Active Accounts (#) |
| **Definition** | Open accounts with at least one financial transaction in the last 60 days |
| **Aggregation** | COUNT DISTINCT |

### Old Logic (DEV_DB.DEV_ANWAR → prod_db Silver)
**Source file:** `model/cubes/core/businesses.yml`
**`sql_table`:** `DEV_DB.DEV_ANWAR.DIM_BUSINESSES`

```sql
-- Cube measure: businesses.financially_active_accounts
COUNT(
  CASE
    WHEN account_status = 'active'
     AND last_transaction_at >= DATEADD(day, -60, CURRENT_DATE)
    THEN business_id
  END
)
-- Underlying Silver source: PROD_DB.DATA.BUSINESSES
```

### New Logic (METABASE_DB Gold)
**Source table:** `METABASE_DB.CONFORMED.DIM_BUSINESSES`
**Column:** `LAST_TRANSACTION_AT`, `ACCOUNT_STATUS`

```sql
-- Cube measure (reworked): dim_businesses.financially_active_accounts
COUNT(DISTINCT
  CASE
    WHEN ACCOUNT_STATUS = 'active'
     AND LAST_TRANSACTION_AT >= CURRENT_DATE - 60
    THEN BUSINESS_ID
  END
)
-- Underlying Gold source: METABASE_DB.CONFORMED.DIM_BUSINESSES
```

### Validation Query

```sql
-- ============================================================
-- VALIDATION 1: Financially Active Accounts
-- Compares DEV_DB Silver vs METABASE_DB Gold
-- Expected variance: 0 (same grain, same logic, different source)
-- ============================================================

WITH old_logic AS (
    SELECT
        DATE_TRUNC('month', account_created_at)::DATE   AS cohort_month,
        COUNT(CASE
                WHEN account_status = 'active'
                 AND last_transaction_at >= DATEADD('day', -60, CURRENT_DATE)
                THEN business_id
              END)                                       AS old_financially_active
    FROM DEV_DB.DEV_ANWAR.DIM_BUSINESSES
    GROUP BY 1
),

new_logic AS (
    SELECT
        DATE_TRUNC('month', ACCOUNT_CREATED_AT)::DATE   AS cohort_month,
        COUNT(DISTINCT CASE
                WHEN ACCOUNT_STATUS = 'active'
                 AND LAST_TRANSACTION_AT >= CURRENT_DATE - 60
                THEN BUSINESS_ID
              END)                                       AS new_financially_active
    FROM METABASE_DB.CONFORMED.DIM_BUSINESSES
    GROUP BY 1
)

SELECT
    COALESCE(o.cohort_month, n.cohort_month)            AS cohort_month,
    COALESCE(o.old_financially_active, 0)               AS old_value,
    COALESCE(n.new_financially_active, 0)               AS new_value,
    new_value - old_value                               AS variance,
    ROUND(100.0 * (new_value - old_value)
          / NULLIF(old_value, 0), 2)                   AS variance_pct
FROM old_logic   o
FULL OUTER JOIN new_logic n USING (cohort_month)
ORDER BY cohort_month DESC;
```

**Pass condition:** `ABS(variance_pct) < 0.01` for all months.
**If variance exists:** Check `ACCOUNT_STATUS` casing differences and whether `LAST_TRANSACTION_AT` is a DATE vs TIMESTAMP in the two sources.

---

## 2. Marginal Net Income

| | Detail |
|---|---|
| **Metric Name** | Marginal Net Income ($) |
| **Definition** | Total Revenue Excl. Interchange − All COGS & Loss items |
| **Aggregation** | SUM |

### Old Logic (DEV_DB.DEV_ANWAR → prod_db Silver)
**Source file:** `model/cubes/financial/total_revenue.yml`
**`sql_table`:** `DEV_DB.DEV_ANWAR.FACT_TOTAL_REVENUE`

```sql
-- Cube measure: total_revenue.marginal_net_income
-- = revenue_excluding_interchange - expenses

-- Revenue (excl. interchange):
COALESCE(float_revenue_approximated,  0)
+ COALESCE(total_factor_fee_collected, 0)
+ COALESCE(total_late_fee_collected,   0)
+ COALESCE(payment_allocated_fees,     0)
+ COALESCE(payment_allocated_interest, 0)
+ COALESCE(credit_card_mgmt_fee,       0)
+ COALESCE(express_ach_fee,            0)
+ COALESCE(wire_fee,                   0)

-- Expenses:
- COALESCE(cost_to_open_cogs,              0)
- COALESCE(cost_to_run_cogs,              0)
- COALESCE(onboarding_costs,             0)
- COALESCE(interchange_expenses_cogs,    0)
- COALESCE(atm_reimbursement,            0)
- COALESCE(mca_charge_off_loss_guco,     0)
- COALESCE(credit_card_charge_off_loss_guco, 0)
```

### New Logic (METABASE_DB Gold)
**Source table:** `METABASE_DB.MART_FINANCE.FACT_UNIT_ECONOMICS`
**Column:** `MARGINAL_NET_INCOME` (pre-computed, fully materialized)

```sql
-- Cube measure (reworked): fact_unit_economics.marginal_net_income
SUM(MARGINAL_NET_INCOME)
-- METABASE_DB.MART_FINANCE.FACT_UNIT_ECONOMICS
-- Pre-joined at build time — no runtime fan-out joins required
```

### Validation Query

```sql
-- ============================================================
-- VALIDATION 2: Marginal Net Income
-- Compares DEV_DB pre-join Silver vs METABASE_DB pre-materialized Gold
-- Expected variance: 0 (identical formula, different materialization)
-- ============================================================

WITH old_logic AS (
    SELECT
        DATE_TRUNC('month', reporting_month)::DATE  AS reporting_month,
        SUM(
            COALESCE(float_revenue_approximated,  0)
          + COALESCE(total_factor_fee_collected, 0)
          + COALESCE(total_late_fee_collected,   0)
          + COALESCE(payment_allocated_fees,     0)
          + COALESCE(payment_allocated_interest, 0)
          + COALESCE(credit_card_mgmt_fee,       0)
          + COALESCE(express_ach_fee,            0)
          + COALESCE(wire_fee,                   0)
          - COALESCE(cost_to_open_cogs,          0)
          - COALESCE(cost_to_run_cogs,           0)
          - COALESCE(onboarding_costs,           0)
          - COALESCE(interchange_expenses_cogs,  0)
          - COALESCE(atm_reimbursement,          0)
          - COALESCE(mca_charge_off_loss_guco,   0)
          - COALESCE(credit_card_charge_off_loss_guco, 0)
        )                                           AS old_marginal_net_income
    FROM DEV_DB.DEV_ANWAR.FACT_TOTAL_REVENUE
    GROUP BY 1
),

new_logic AS (
    SELECT
        REPORTING_MONTH                             AS reporting_month,
        SUM(MARGINAL_NET_INCOME)                    AS new_marginal_net_income
    FROM METABASE_DB.MART_FINANCE.FACT_UNIT_ECONOMICS
    GROUP BY 1
)

SELECT
    COALESCE(o.reporting_month, n.reporting_month)      AS reporting_month,
    ROUND(COALESCE(o.old_marginal_net_income, 0), 2)    AS old_value,
    ROUND(COALESCE(n.new_marginal_net_income, 0), 2)    AS new_value,
    ROUND(new_value - old_value, 2)                     AS variance,
    ROUND(100.0 * (new_value - old_value)
          / NULLIF(ABS(old_value), 0), 4)               AS variance_pct
FROM old_logic   o
FULL OUTER JOIN new_logic n USING (reporting_month)
ORDER BY reporting_month DESC;
```

**Pass condition:** `ABS(variance_pct) < 0.01` per month.
**If variance exists:** Compare the `credit_card_mgmt_fee` component — this was split into `cc_late_fee_revenue` + `cc_other_fee_revenue` in the Gold layer. Check if the split introduces a rounding difference.

---

## 3. Average Daily Balance (ADB)

| | Detail |
|---|---|
| **Metric Name** | Average Daily Balance Aggregated Monthly ($) |
| **Definition** | Average of all daily balance snapshots within a calendar month, per business |
| **Aggregation** | AVG (then SUM at portfolio level) |

### Old Logic (DEV_DB.DEV_ANWAR → prod_db Silver)
**Source file:** `model/cubes/financial/monthly_balances.yml`
**`sql_table`:** `DEV_DB.DEV_ANWAR.FACT_MONTHLY_BALANCES`

```sql
-- Cube measure: monthly_balances.avg_daily_balance_aggr_monthly
SUM(avg_daily_balance_aggr_monthly)
-- Note: Cube uses type: sum on a pre-averaged column.
-- Average is computed in the underlying DEV_ANWAR model, not at query time.
```

### New Logic (METABASE_DB Gold)
**Source table:** `METABASE_DB.MART_BANKING.FACT_MONTHLY_BALANCES`
**Column:** `AVG_DAILY_BALANCE_AGGR_MONTHLY`

```sql
-- Cube measure (reworked): fact_monthly_balances.avg_daily_balance_aggr_monthly
AVG(AVG_DAILY_BALANCE_AGGR_MONTHLY)
-- Grain: one row per business per month — same pre-averaged column, same grain
```

### Validation Query

```sql
-- ============================================================
-- VALIDATION 3: Average Daily Balance (ADB)
-- Compares DEV_DB Silver vs METABASE_DB.MART_BANKING Gold
-- Expected variance: 0 (same underlying computation, promoted table)
-- ============================================================

WITH old_logic AS (
    SELECT
        balance_month::DATE                         AS balance_month,
        SUM(avg_daily_balance_aggr_monthly)         AS old_total_adb,
        COUNT(DISTINCT business_id)                 AS old_account_count
    FROM DEV_DB.DEV_ANWAR.FACT_MONTHLY_BALANCES
    GROUP BY 1
),

new_logic AS (
    SELECT
        BALANCE_MONTH                               AS balance_month,
        SUM(AVG_DAILY_BALANCE_AGGR_MONTHLY)         AS new_total_adb,
        COUNT(DISTINCT BUSINESS_ID)                 AS new_account_count
    FROM METABASE_DB.MART_BANKING.FACT_MONTHLY_BALANCES
    GROUP BY 1
)

SELECT
    COALESCE(o.balance_month, n.balance_month)          AS balance_month,
    ROUND(COALESCE(o.old_total_adb, 0), 2)              AS old_total_adb,
    ROUND(COALESCE(n.new_total_adb, 0), 2)              AS new_total_adb,
    ROUND(new_total_adb - old_total_adb, 2)             AS variance,
    ROUND(100.0 * (new_total_adb - old_total_adb)
          / NULLIF(old_total_adb, 0), 4)                AS variance_pct,
    o.old_account_count,
    n.new_account_count,
    n.new_account_count - o.old_account_count           AS account_count_diff
FROM old_logic   o
FULL OUTER JOIN new_logic n USING (balance_month)
ORDER BY balance_month DESC;
```

**Pass condition:** `ABS(variance_pct) < 0.01` and `account_count_diff = 0`.
**If variance exists:** `account_count_diff ≠ 0` means the Gold table has a different population (check for closed accounts being excluded in one layer but not the other).

---

## 4. Total Interchange Revenue

| | Detail |
|---|---|
| **Metric Name** | Total Interchange Revenue ($) |
| **Definition** | Net interchange revenue from both debit card and credit card programs |
| **Aggregation** | SUM |

### Old Logic (DEV_DB.DEV_ANWAR → prod_db Silver)
**Source file:** `model/cubes/financial/lithic_settlement_daily.yml`
**`sql_table`:** `DEV_DB.DEV_ANWAR.FACT_LITHIC_SETTLEMENT_DAILY`

```sql
-- Cube measure: lithic_settlement_daily.interchange_revenue_total
SUM(
  COALESCE(interchange_gross_amount_cc, 0)
  + COALESCE(interchange_gross_amount_dc, 0)
)
```

### New Logic (METABASE_DB Gold)
**Source table:** `METABASE_DB.MART_CARDS.FACT_LITHIC_SETTLEMENT`
**Columns:** `INTERCHANGE_GROSS_AMOUNT_CC` + `INTERCHANGE_GROSS_AMOUNT_DC`

```sql
-- Cube measure (reworked): fact_lithic_settlement.total_interchange_revenue
SUM(
  COALESCE(INTERCHANGE_GROSS_AMOUNT_CC, 0)
  + COALESCE(INTERCHANGE_GROSS_AMOUNT_DC, 0)
)
-- Same source data, promoted from DEV_DB to METABASE_DB.MART_CARDS
```

### Validation Query

```sql
-- ============================================================
-- VALIDATION 4: Total Interchange Revenue
-- Compares DEV_DB Lithic settlement vs METABASE_DB.MART_CARDS Gold
-- Expected variance: 0 (same Lithic source file, direct promotion)
-- ============================================================

WITH old_logic AS (
    SELECT
        DATE_TRUNC('month', report_date)::DATE          AS report_month,
        SUM(COALESCE(interchange_gross_amount_cc, 0)
          + COALESCE(interchange_gross_amount_dc, 0))   AS old_interchange_revenue,
        SUM(COALESCE(interchange_gross_amount_dc, 0))   AS old_dc_interchange,
        SUM(COALESCE(interchange_gross_amount_cc, 0))   AS old_cc_interchange
    FROM DEV_DB.DEV_ANWAR.FACT_LITHIC_SETTLEMENT_DAILY
    GROUP BY 1
),

new_logic AS (
    SELECT
        DATE_TRUNC('month', REPORT_DATE)::DATE          AS report_month,
        SUM(COALESCE(INTERCHANGE_GROSS_AMOUNT_CC, 0)
          + COALESCE(INTERCHANGE_GROSS_AMOUNT_DC, 0))   AS new_interchange_revenue,
        SUM(COALESCE(INTERCHANGE_GROSS_AMOUNT_DC, 0))   AS new_dc_interchange,
        SUM(COALESCE(INTERCHANGE_GROSS_AMOUNT_CC, 0))   AS new_cc_interchange
    FROM METABASE_DB.MART_CARDS.FACT_LITHIC_SETTLEMENT
    GROUP BY 1
)

SELECT
    COALESCE(o.report_month, n.report_month)            AS report_month,
    ROUND(COALESCE(o.old_interchange_revenue, 0), 2)    AS old_total_interchange,
    ROUND(COALESCE(n.new_interchange_revenue, 0), 2)    AS new_total_interchange,
    ROUND(new_total_interchange - old_total_interchange, 2) AS total_variance,
    ROUND(100.0 * (new_total_interchange - old_total_interchange)
          / NULLIF(old_total_interchange, 0), 4)        AS variance_pct,
    -- Component-level drill for diagnosis
    ROUND(n.new_dc_interchange - o.old_dc_interchange, 2)  AS dc_variance,
    ROUND(n.new_cc_interchange - o.old_cc_interchange, 2)  AS cc_variance
FROM old_logic   o
FULL OUTER JOIN new_logic n USING (report_month)
ORDER BY report_month DESC;
```

**Pass condition:** `total_variance = 0` for all months (no rounding expected — same source file).
**If variance exists:** Check `dc_variance` vs `cc_variance` separately. A gap in one program signals a date filter or settlement-file-join issue in the Gold promotion model.

---

## 5. MCA Outstanding Balance

| | Detail |
|---|---|
| **Metric Name** | MCA Outstanding Balance ($) |
| **Definition** | Total outstanding principal owed across all active MCA accounts at the snapshot date |
| **Aggregation** | SUM |

### Old Logic (DEV_DB.DEV_ANWAR → prod_db Silver)
**Source file:** `model/cubes/credit_risk/funding_credit_risk.yml`
**`sql_table`:** `DEV_DB.DEV_ANWAR.FACT_LOAN_TAPE_HISTORY`

```sql
-- Cube measure: funding_credit_risk.outstanding_balance_mca
SUM(principal_balance_outstanding)
-- Source table has one row per account per snap_date
-- No deduplication logic applied in the Cube layer
```

### New Logic (METABASE_DB Gold)
**Source table:** `METABASE_DB.MART_FUNDING.FACT_MCA_LOAN_TAPE`
**Column:** `PRINCIPAL_BALANCE_OUTSTANDING`

```sql
-- Cube measure (reworked): fact_mca_loan_tape.outstanding_balance_mca
SUM(PRINCIPAL_BALANCE_OUTSTANDING)
-- Requires INT_MCA_LOAN_TAPE_RECONCILED upstream to resolve
-- CANOPY_LENDING (Aurora) vs DATA.LOAN_TAPE (LMS) source conflicts
-- before Gold values are reliable
```

### Validation Query

```sql
-- ============================================================
-- VALIDATION 5: MCA Outstanding Balance
-- Compares DEV_DB FACT_LOAN_TAPE_HISTORY vs METABASE_DB.MART_FUNDING Gold
-- Expected variance: 0 after INT_MCA_LOAN_TAPE_RECONCILED is built
-- NOTE: non-zero variance on first run may indicate the reconciliation
-- intermediate model is still required — flag for the Funding team.
-- ============================================================

WITH old_logic AS (
    SELECT
        snap_date::DATE                             AS snap_date,
        SUM(principal_balance_outstanding)          AS old_outstanding_balance,
        COUNT(DISTINCT business_id)                 AS old_account_count
    FROM DEV_DB.DEV_ANWAR.FACT_LOAN_TAPE_HISTORY
    GROUP BY 1
),

new_logic AS (
    SELECT
        SNAP_DATE                                   AS snap_date,
        SUM(PRINCIPAL_BALANCE_OUTSTANDING)          AS new_outstanding_balance,
        COUNT(DISTINCT BUSINESS_ID)                 AS new_account_count
    FROM METABASE_DB.MART_FUNDING.FACT_MCA_LOAN_TAPE
    GROUP BY 1
),

comparison AS (
    SELECT
        COALESCE(o.snap_date, n.snap_date)              AS snap_date,
        ROUND(COALESCE(o.old_outstanding_balance, 0), 2) AS old_value,
        ROUND(COALESCE(n.new_outstanding_balance, 0), 2) AS new_value,
        ROUND(new_value - old_value, 2)                 AS variance,
        ROUND(100.0 * (new_value - old_value)
              / NULLIF(old_value, 0), 4)                AS variance_pct,
        o.old_account_count,
        n.new_account_count,
        n.new_account_count - o.old_account_count       AS account_count_diff
    FROM old_logic   o
    FULL OUTER JOIN new_logic n USING (snap_date)
)

SELECT *
FROM comparison
WHERE snap_date >= DATEADD('month', -3, CURRENT_DATE)  -- last 3 months
ORDER BY snap_date DESC;

-- Diagnostic: Identify which accounts are causing the variance
-- Run this if ABS(variance_pct) > 0.01 in the query above:
/*
SELECT
    'old_only'          AS source,
    business_id,
    snap_date,
    principal_balance_outstanding
FROM DEV_DB.DEV_ANWAR.FACT_LOAN_TAPE_HISTORY
WHERE snap_date = CURRENT_DATE - 1

EXCEPT

SELECT
    'new_only',
    BUSINESS_ID,
    SNAP_DATE,
    PRINCIPAL_BALANCE_OUTSTANDING
FROM METABASE_DB.MART_FUNDING.FACT_MCA_LOAN_TAPE
WHERE SNAP_DATE = CURRENT_DATE - 1;
*/
```

**Pass condition:** `ABS(variance_pct) < 0.01` across all recent snap dates.
**If variance exists:** Run the diagnostic EXCEPT query to isolate which `business_id` + `snap_date` combinations are missing. This will confirm whether `INT_MCA_LOAN_TAPE_RECONCILED` needs to be built before the Gold table is reliable.

---

## Summary Checklist

| # | Metric | Query | Pass Condition | Likely Failure Cause |
|---|---|---|---|---|
| 1 | Financially Active Accounts | Validation 1 | `ABS(variance_pct) < 0.01` | `ACCOUNT_STATUS` casing or Timestamp vs Date type mismatch |
| 2 | Marginal Net Income | Validation 2 | `ABS(variance_pct) < 0.01` per month | `credit_card_mgmt_fee` split across multiple Gold columns |
| 3 | Average Daily Balance (ADB) | Validation 3 | `variance = 0`, `account_count_diff = 0` | Closed accounts scoped differently in DEV vs Gold |
| 4 | Total Interchange Revenue | Validation 4 | `total_variance = 0` | Settlement file missing from Gold ingestion window |
| 5 | MCA Outstanding Balance | Validation 5 | `ABS(variance_pct) < 0.01` | `INT_MCA_LOAN_TAPE_RECONCILED` not yet built — Aurora/LMS conflict |
