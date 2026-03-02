# central_db_gold_layer.md
## Novo Business Bank — `METABASE_DB` Gold Layer Technical Specification

> **Purpose:** Authoritative table-level specification for the functional mart Gold Layer.
> All Cube `sql_table` references must point to `METABASE_DB.<MART>.<TABLE>` as defined here.
> Silver (`prod_db`) must never be referenced directly from the semantic layer.
>
> **Build pattern:** All tables are materialized dbt models. Views are forbidden at the Gold layer.
> `FACT_UNIT_ECONOMICS` is the single most critical pre-join model — it must never be a view.

---

## Database: `METABASE_DB`

### Schema Index

| Schema | Tables | Owner | Purpose |
|---|---|---|---|
| `CONFORMED` | 4 | Data Platform | Shared dimensions — every mart joins here |
| `MART_BANKING` | 8 | Banking / Payments | Money movement: ACH, wires, mRDC, balances |
| `MART_CARDS` | 7 | Cards | Debit + credit card lifecycle, ops, fulfillment |
| `MART_FUNDING` | 5 | Funding / MCA Risk | MCA portfolio: draws, repayment, delinquency |
| `MART_FINANCE` | 4 | Finance / Treasury | Unit economics, interchange, float revenue |
| `MART_MARKETING` | 6 | Marketing | Acquisition funnel, ad spend, LTV |
| `MART_PRODUCT` | 8 | Product | Feature adoption, activation, customer success |

---

## CONFORMED

### `CONFORMED.DIM_BUSINESSES`
**Grain:** One row per business account.
**Build:** Incremental, SCD Type 1. `ACCOUNT_STATUS` uses SCD Type 2 when status changes.
**Source:** `PROD_DB.DATA.BUSINESSES` + `PROD_DB.DATA.FRAUD_BUSINESSES`

| Column | Type | Description | Metric Home |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR PK | UUID for the business account | All marts |
| `APPLICATION_ID` | VARCHAR FK | FK to `DIM_APPLICATIONS` | — |
| `EMAIL` | VARCHAR | Primary business email | Identity |
| `ACCOUNT_STATUS` | VARCHAR | `active`, `suspended`, `frozen` | Active Account (DDA) |
| `CUSTOMER_SUCCESS_ACCOUNT_STATUS` | VARCHAR | CS hierarchy status (9 levels) | CS Account Status |
| `ACCOUNT_CREATED_AT` | TIMESTAMP_TZ | Account open timestamp | OA Cohort |
| `ACCOUNT_OPENED_MONTH` | DATE | `DATE_TRUNC('month', ACCOUNT_CREATED_AT)` | OA Cohort |
| `ACCOUNT_CLOSED_AT` | TIMESTAMP_TZ | Account close timestamp | Closure Churn |
| `FIRST_TRANSACTION_AT` | DATE | Date of first financial transaction | Financially Active |
| `LAST_TRANSACTION_AT` | DATE | Date of most recent transaction | Financially Active, Inactive Churn |
| `LAST_DEBIT_CARD_TRANSACTION_AT` | DATE | Date of most recent debit card transaction | DC Active Account |
| `LAST_CREDIT_CARD_TRANSACTION_AT` | TIMESTAMP_TZ | Date of most recent CC transaction | CC Active Account |
| `ACCOUNT_1K_FUNDED_AT` | TIMESTAMP_NTZ | Date account first reached $1K cumulative credits | Funded Account |
| `IS_ACCOUNT_1K_FUNDED` | BOOLEAN | TRUE if account has >= $1K cumulative credits | Funded Account |
| `IS_ACCOUNT_INACTIVE_60DAYS` | BOOLEAN | TRUE if no transaction in last 60 days | Inactive Account |
| `IS_ACCOUNT_CLOSED` | BOOLEAN | TRUE if ACCOUNT_STATUS = 'suspended' | Closed Account |
| `IS_ACCOUNT_CLOSED_FOR_SUSPECTED_FRAUD` | BOOLEAN | TRUE if closed due to fraud suspicion | Fraud sub-status |
| `CLOSURE_IS_SUSPECT_FRAUD` | BOOLEAN | Fraud closure flag | CS Account Status |
| `EXPENSED_FRAUD_LOSS` | FLOAT | Dollar value of expensed fraud loss | Fraud Loss |
| `DELINQUENCY_DAYS_PAST_DUE` | NUMBER | DPD from credit product (for CS status) | CS Delinquent |
| `IS_EMAIL_IN_USERS` | BOOLEAN | Email cross-reference flag | CS Prospect/Applicant |
| `IS_EMAIL_IN_APPLICATIONS` | BOOLEAN | Email cross-reference flag | CS Prospect/Applicant |
| `LAST_MODIFIED_AT` | TIMESTAMP_TZ | For incremental builds | — |

---

### `CONFORMED.DIM_APPLICATIONS`
**Grain:** One row per application.
**Build:** Full refresh daily (low cardinality).
**Source:** `PROD_DB.DATA.APPLICATIONS` + `PROD_DB.INTERMEDIATE.INT_APPLICATION_EVENTS_PIVOTED_TO_APPLICATION_IDS` + `PROD_DB.INTERMEDIATE.INT_BUSINESS_INDUSTRY_MAPPING`

| Column | Type | Description | Metric Home |
|---|---|---|---|
| `APPLICATION_ID` | VARCHAR PK | UUID for the application | Identity |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `APPLICATION_CREATED_AT` | TIMESTAMP_NTZ | Step 1 started (email + password) | Started Application |
| `APPLICATION_STARTED_AT` | TIMESTAMP_NTZ | Application start timestamp | Started Application |
| `APPLICATION_COMPLETED_AT` | TIMESTAMP_NTZ | All steps completed (e-signed) | Completed Application |
| `APPLICATION_DECISION` | VARCHAR | `auto_approved`, `straight_through`, `manual_approved`, `auto_denied`, `manual_denied`, `incomplete`, `abandoned` | Application Decision |
| `APPLICATION_STATUS` | VARCHAR | Current raw status | — |
| `CURRENT_STEP` | VARCHAR | Last step reached in onboarding | — |
| `BUSINESS_TYPE` | VARCHAR | `Sole Proprietorship`, `LLC`, `Corporation`, `Partnership` | Business Type |
| `B2B_SERVICES_FLAG` | VARCHAR | B2B services indicator | Segment |
| `BUSINESS_NAME` | VARCHAR | Business name at application | — |
| `BUSINESS_INDUSTRY_CATEGORY` | VARCHAR | 6-digit NAICS industry category | Industry Category Name |
| `BUSINESS_INDUSTRY_NAME` | VARCHAR | 8-digit NAICS name *(add — source TBD)* | Industry Name |
| `BUSINESS_ESTABLISHMENT_DATE` | DATE | Self-reported business founding date | Age of Business |
| `APPLICATION_CHANNEL_RAW` | VARCHAR | Raw channel string | — |
| `APPLICATION_CHANNEL_DERIVED` | VARCHAR | Last-touch channel: `paid_search`, `paid_social`, `organic_direct`, `bd_affiliate`, `referral` | Acquisition Channel |
| `IS_AUTO_APPROVED` | NUMBER(1) | 1 if auto-approved | Application Decision |
| `IS_AUTO_DENIED` | NUMBER(1) | 1 if auto-denied | Application Decision |
| `IS_STRAIGHT_THROUGH` | NUMBER(1) | 1 if approved without manual review | Application Decision |

---

### `CONFORMED.DIM_USERS`
**Grain:** One row per user.
**Source:** `PROD_DB.DATA.USERS`

| Column | Type | Description |
|---|---|---|
| `USER_ID` | VARCHAR PK | UUID for the user |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` |
| `APPLICATION_ID` | VARCHAR FK | FK to `DIM_APPLICATIONS` |
| `SEGMENT_USER_ID` | VARCHAR | Third-party system link ID *(currently missing from Gold)* |
| `EMAIL` | VARCHAR | User email |
| `CUSTOMER_FIRST_NAME` | VARCHAR | First name |
| `CUSTOMER_LAST_NAME` | VARCHAR | Last name |

---

### `CONFORMED.DIM_DATE`
**Grain:** One row per calendar day.
**Build:** dbt date spine (`generate_date_spine`). Static, full rebuild weekly.

| Column | Type | Description |
|---|---|---|
| `DATE_DAY` | DATE PK | Calendar day |
| `WEEK_START` | DATE | ISO week start (Monday) |
| `MONTH_START` | DATE | First day of calendar month |
| `QUARTER_START` | DATE | First day of quarter |
| `FISCAL_YEAR` | NUMBER | Fiscal year |
| `FISCAL_QUARTER` | NUMBER | Fiscal quarter |
| `IS_WEEKEND` | BOOLEAN | Saturday or Sunday |
| `IS_BUSINESS_DAY` | BOOLEAN | NOT weekend AND NOT holiday |

---

## MART_BANKING

### `MART_BANKING.FACT_TRANSACTIONS`
**Grain:** One row per transaction on the Novo checking account (all rails).
**Source:** `METABASE_DB.PAYMENTS.TRANSACTIONS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `TRANSACTION_ID` | VARCHAR PK | Transaction UUID | Gross Transactions Count |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `ACCOUNT_ID` | VARCHAR | Checking account ID | — |
| `TRANSACTION_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `TRANSACTION_AT` | TIMESTAMP_TZ | Full timestamp | — |
| `CREDIT_OR_DEBIT` | VARCHAR | `credit` (inbound) or `debit` (outbound) | Transaction Type |
| `TRANSFER_TYPE` | VARCHAR | Transfer type classification | — |
| `MEDIUM` | VARCHAR | Raw medium string | Transaction Medium |
| `DERIVED_MEDIUM` | VARCHAR | Cleaned: `ach`, `wire`, `debit_card`, `check`, `mrdc`, `internal` | Transaction Medium |
| `VVC_MEDIUM` | VARCHAR | VVC medium classification | — |
| `ORIGINATION_TYPE` | VARCHAR | `push` or `pull` | Origination/Receipt |
| `TRANSACTION_STATUS` | VARCHAR | `pending`, `settled`, `returned`, `failed` | — |
| `TRANSACTION_AMOUNT` | FLOAT | Amount in dollars | Gross Transaction Volume |
| `DESCRIPTION` | VARCHAR | Transaction description | — |
| `CATEGORY_ID` | VARCHAR | Novo category ID | — |
| `EXPRESS_ACH_FEE` | FLOAT | Express ACH fee if applicable | Express ACH Revenue |

**Key measures:**
- `Gross Transactions Count` = `COUNT(TRANSACTION_ID)`
- `Gross Transaction Volume` = `SUM(ABS(TRANSACTION_AMOUNT))`

---

### `MART_BANKING.FACT_ACH_TRANSFERS`
**Grain:** One row per ACH transfer event (push or pull).
**Source:** `METABASE_DB.PAYMENTS.TRANSFERS` + `METABASE_DB.PAYMENTS.PULL_FUNDS_REQUESTS` + `FIVETRAN_DB.PROD_NOVO_API_PUBLIC.TRANSFER_EVENTS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `TRANSFER_ID` | VARCHAR PK | Transfer UUID | — |
| `TRANSACTION_ID` | VARCHAR FK | FK to `FACT_TRANSACTIONS` | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TRANSFER_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `TRANSFER_TYPE` | VARCHAR | `ach_push`, `ach_pull`, `express_ach` | — |
| `TRANSFER_STATUS` | VARCHAR | `pending`, `approved`, `completed`, `returned`, `rejected` | ACH Pull/Push Status |
| `ACH_PUSH_DECISION` | VARCHAR | `auto_approved`, `manual_review_approved`, `manual_review_rejected`, `auto_rejected`, `discarded` | ACH Push Decision |
| `ACH_PULL_DECISION` | VARCHAR | `auto_approved`, `manual_review_approved`, `manual_review_rejected`, `auto_rejected`, `canceled` | ACH Pull Decision |
| `AMOUNT` | FLOAT | Transfer amount | — |
| `EXPRESS_ACH_FLAG` | BOOLEAN | Was sent as Express ACH | — |
| `REVENUE_EXPRESS_ACH` | FLOAT | Express ACH fee revenue | Express ACH Revenue |
| `RETURNED_DESCRIPTION` | VARCHAR | Return reason code | — |
| `REJECTED_DESCRIPTION` | VARCHAR | Rejection reason | — |
| `CREATED_AT` | TIMESTAMP_TZ | Initiation timestamp | — |
| `APPROVED_AT` | TIMESTAMP_TZ | Approval timestamp | — |
| `COMPLETED_AT` | TIMESTAMP_TZ | Settlement timestamp | — |
| `RETURNED_AT` | TIMESTAMP_TZ | Return timestamp | — |

**Decision logic (pre-computed in dbt):**
- ACH Push: sub-select `TRANSFER_EVENTS WHERE EVENT = 'manual_review'` as flag, then CASE on final event
- ACH Pull: CASE on `APPROVED_DESCRIPTION` / `REJECTED_DESCRIPTION` patterns

---

### `MART_BANKING.FACT_DOMESTIC_WIRES`
**Grain:** One row per domestic wire.
**Source:** `METABASE_DB.PAYMENTS.TRANSFERS` + `FIVETRAN_DB.PROD_NOVO_API_PUBLIC.DOMESTIC_WIRE_EVENTS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `DOMESTIC_WIRE_ID` | VARCHAR PK | Wire UUID | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `WIRE_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `WIRE_STATUS` | VARCHAR | Latest event state | Wire Status |
| `WIRE_DECISION` | VARCHAR | `auto_approved`, `manual_review_approved`, `manual_review_rejected`, `auto_rejected`, `discarded`, `no_decision` | Wire Decision |
| `WIRE_AMOUNT` | NUMBER | Wire amount in dollars | — |
| `WIRE_FEE` | NUMBER | Fee charged | Domestic Wire Revenue |
| `WIRE_CREATED_AT` | TIMESTAMP_TZ | Initiation timestamp | — |
| `PROCESSED_AT` | TIMESTAMP_TZ | Processing timestamp | — |

**Decision/Status logic (pre-computed):** `ROW_NUMBER() OVER (PARTITION BY DOMESTIC_WIRE_ID ORDER BY CREATED_AT DESC) = 1` on wire events, then CASE on EVENT string.

---

### `MART_BANKING.FACT_REMOTE_DEPOSIT_CHECKS`
**Grain:** One row per mobile check deposit.
**Source:** `METABASE_DB.PAYMENTS.REMOTE_DEPOSIT_CHECKS` + `FIVETRAN_DB.PROD_NOVO_API_PUBLIC.REMOTE_DEPOSIT_CHECK_EVENTS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `REMOTE_DEPOSIT_CHECK_ID` | VARCHAR PK | mRDC UUID | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `DEPOSIT_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `MRDC_STATUS` | VARCHAR | `pending`, `approved`, `rejected`, `returned` | mRDC Status |
| `MRDC_DECISION` | VARCHAR | `auto_approved`, `manual_review_approved`, `manual_review_rejected`, `auto_rejected` | mRDC Decision |
| `AMOUNT` | FLOAT | Check amount | — |
| `CREATED_AT` | TIMESTAMP_NTZ | Submission timestamp | — |

---

### `MART_BANKING.FACT_MONTHLY_BALANCES`
**Grain:** One row per business per month.
**Source:** `DEV_DB.DEV_ANWAR.FACT_MONTHLY_BALANCES` → promote to `METABASE_DB`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `BALANCE_MONTH` | DATE FK | FK to `DIM_DATE` — first day of month | — |
| `MONTH_END_BALANCE` | FLOAT | Balance at month end | — |
| `AVG_DAILY_BALANCE_AGGR_MONTHLY` | FLOAT | Average daily balance for the month | **Average Monthly Balance (ADB)** |

---

### `MART_BANKING.FACT_EXPRESS_ACH_ACTIVITY`
**Grain:** One row per business per month with Express ACH activity.
**Source:** `DEV_DB.DEV_ANWAR.FACT_EXPRESS_ACH_ACTIVITY`

| Column | Type | Description | Metric |
|---|---|---|---|
| `EXPRESS_ACH_BUSINESS_MONTH_ID` | VARCHAR PK | Composite key | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TRANSACTION_MONTH` | DATE FK | Reporting month | — |
| `EXPRESS_ACH_FEE` | FLOAT | Total Express ACH fees this month | Express ACH Revenue |
| `BUSINESS_SPECIFIC_ORDINAL` | NUMBER | 1 = first month of usage | Express ACH Attach |

---

## MART_CARDS

### `MART_CARDS.FACT_DEBIT_CARD_TRANSACTIONS`
**Grain:** One row per debit card transaction event.
**Source:** `METABASE_DB.CARDS.DEBIT_CARD_TRANSACTIONS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `CARD_TRANSACTION_ID` | VARCHAR PK | Transaction event UUID | Debit Card Status/Decision |
| `CARD_ID` | VARCHAR FK | FK to debit card | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TRANSACTION_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `TRANSACTION_CREATED_AT` | TIMESTAMP_TZ | Full timestamp | — |
| `MERCHANT_CATEGORY_CODE` | VARCHAR | MCC code | — |
| `POS_TYPE` | VARCHAR | In-person, online, etc. | — |
| `TRANSACTION_STATUS` | VARCHAR | `approved`, `declined`, `settled`, `reversed` | **Debit Card Status** |
| `RESULT` | VARCHAR | Decision outcome | **Debit Card Decision** |
| `DECLINE_REASON` | VARCHAR | Decline reason | — |
| `AMOUNT` | NUMBER | Transaction amount | — |
| `SETTLED_AMOUNT` | NUMBER | Final settled amount | — |
| `INTERCHANGE_FEE` | NUMBER | Interchange fee earned | Interchange Eligible Spend (DC) |
| `SARDINE_TRANSACTION_RISK_LEVEL` | VARCHAR | Fraud risk signal | — |
| `IS_DISPUTE` | BOOLEAN | Dispute flag | — |

---

### `MART_CARDS.FACT_CREDIT_CARD_TRANSACTIONS`
**Grain:** One row per credit card transaction event.
**Source:** `METABASE_DB.CARDS.CREDIT_CARD_TRANSACTIONS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `TRANSACTION_ID` | VARCHAR PK | Transaction UUID | CC Status / Decision |
| `CREDIT_CARD_ID` | VARCHAR | CC identifier | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TRANSACTION_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `TRANSACTION_CREATED_AT` | TIMESTAMP_TZ | Full timestamp | — |
| `MERCHANT_CATEGORY_CODE` | VARCHAR | MCC code | — |
| `TRANSACTION_STATUS` | VARCHAR | `approved`, `declined`, `settled`, `reversed` | **Credit Card Status** |
| `RESULT` | VARCHAR | Decision outcome | **Credit Card Decision** |
| `DECLINE_REASON` | VARCHAR | Decline reason if declined | — |
| `AUTHORIZATION_AMOUNT` | NUMBER | Authorized amount | — |
| `SETTLED_AMOUNT` | NUMBER | Settled amount | **CC Total Spend** |
| `SPEND_AMOUNT` | NUMBER | Spend amount | — |
| `OUTSTANDING_BALANCE` | NUMBER | Balance at statement date (from loan tape join) | **CC Outstanding Balance** |
| `CREDIT_LIMIT` | NUMBER | Credit limit (from loan tape join) | **CC Exposure** |
| `UTILIZATION` | FLOAT | `OUTSTANDING_BALANCE / CREDIT_LIMIT` | **CC Utilization** |
| `SARDINE_TRANSACTION_RISK_LEVEL` | VARCHAR | Fraud signal | — |
| `IS_DISPUTE` | BOOLEAN | Dispute flag | — |

---

### `MART_CARDS.FACT_LITHIC_SETTLEMENT`
**Grain:** One row per daily card network settlement report.
**Source:** `DEV_DB.DEV_ANWAR.FACT_LITHIC_SETTLEMENT_DAILY` + `FIVETRAN_DB` settlement reports

| Column | Type | Description | Metric |
|---|---|---|---|
| `REPORT_DATE` | DATE PK | Settlement report date | — |
| `TRANSACTIONS_GROSS_AMOUNT_DC` | NUMBER | Debit card gross eligible spend | **Interchange Eligible Spend (Debit)** |
| `INTERCHANGE_GROSS_AMOUNT_DC` | NUMBER | Debit card interchange revenue | **Interchange Revenue (Debit)** |
| `TRANSACTIONS_GROSS_AMOUNT_CC` | NUMBER | Credit card gross eligible spend | **Interchange Eligible Spend (Credit)** |
| `INTERCHANGE_GROSS_AMOUNT_CC` | NUMBER | Credit card interchange revenue | **Interchange Revenue (Credit)** |

---

### `MART_CARDS.FACT_DEBIT_CARD_REQUESTS`
**Grain:** One row per card request event per business.
**Source:** `DEV_DB.DEV_ANWAR.FACT_DEBIT_CARD_REQUESTS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `DEBIT_CARD_REQUESTED_AT` | TIMESTAMP_TZ | First card request timestamp | **Debit Card Attach** |

---

### `MART_CARDS.FACT_CREDIT_CARD_REQUESTS`
**Grain:** One row per CC request event per business.
**Source:** `DEV_DB.DEV_ANWAR.FACT_CREDIT_CARD_REQUESTS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `CREDIT_CARD_REQUESTED_AT` | TIMESTAMP_TZ | First CC request timestamp | **Credit Card Attach** |

---

### `MART_CARDS.DIM_CREDIT_CARD_APPLICATION_FUNNEL`
**Grain:** One row per business (latest CC funnel state).
**Source:** `DEV_DB.DEV_ANWAR.DIM_CREDIT_CARD_APPLICATION_FUNNEL`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR PK | FK to `DIM_BUSINESSES` | — |
| `INVITATION_CREATED_AT` | TIMESTAMP_TZ | CC invite sent date | **CC Invite Sent** |
| `APPLICATION_SUBMITTED_AT` | TIMESTAMP_TZ | CC application submitted date | **CC Application Submitted** |
| `DECISION_CREATED_AT` | TIMESTAMP_TZ | Underwriting decision date | **CC Application Approved** |
| `DECISION` | VARCHAR | `approved`, `denied` | — |
| `CREDIT_RISK_BIN` | NUMBER | FICO-derived risk bin 1-5 | **Model Risk Bin** |
| `LINE_ASSIGNMENT_SEGMENT` | VARCHAR | `A`, `B`, or `C` based on FICO × limit | **Line Assignment Segment** |
| `OPEN_COHORT` | DATE | `MIN(STATEMENT_DATE)` — month first underwritten | **Open Cohort (CC)** |

---

## MART_FUNDING

### `MART_FUNDING.FACT_MCA_LOAN_TAPE`
**Grain:** One row per MCA account per snapshot date.
**Source:** `DEV_DB.DEV_ANWAR.FACT_LOAN_TAPE` (requires `INT_MCA_LOAN_TAPE_RECONCILED` upstream)

| Column | Type | Description | Metric |
|---|---|---|---|
| `LOAN_TAPE_ID` | VARCHAR PK | Composite key (account + snap date) | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `SNAP_DATE` | DATE FK | FK to `DIM_DATE` | — |
| `PRINCIPAL_BALANCE_OUTSTANDING` | FLOAT | Outstanding principal | **Outstanding Balance (MCA)** |
| `TOTAL_REVOLVING_FUNDS` | FLOAT | Total revolving credit facility | **Exposure (MCA)** |
| `DAYS_PAST_DUE` | NUMBER | Days past due at snapshot | — |
| `DELINQUENCY_BIN` | VARCHAR | `bucket 0`–`bucket 7` (30-day increments) | **Delinquency Bin (MCA)** |
| `TOTAL_FACTOR_FEE_COLLECTED` | FLOAT | Factor fee collected to date | MCA Factor Fee Revenue |
| `TOTAL_LATE_FEES_COLLECTED` | FLOAT | Late fees collected to date | MCA Late Fee Revenue |
| `TOTAL_PRINCIPAL_RECOVERED` | FLOAT | Principal recovered after charge-off | MCA Recovery |
| `TOTAL_FACTOR_FEE_RECOVERED` | FLOAT | Factor fee recovered after charge-off | MCA Recovery |

---

### `MART_FUNDING.FACT_MCA_REVENUE`
**Grain:** One row per MCA account per reporting month.
**Source:** Derived from `FACT_MCA_LOAN_TAPE` monthly snapshot

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `REPORTING_MONTH` | DATE FK | FK to `DIM_DATE` | — |
| `FACTOR_FEE_REVENUE` | FLOAT | Factor fees collected this month | **MCA Factor Fee Revenue** |
| `LATE_FEE_REVENUE` | FLOAT | Late fees collected this month | **MCA Late Fee Revenue** |
| `CHARGE_OFF_LOSS` | FLOAT | Principal at DPD=180 | **MCA Charge Off Loss (GUCO)** |
| `RECOVERY` | FLOAT | Recovered amounts this month | **MCA Recovery** |

---

### `MART_FUNDING.FACT_MCA_TRANSACTIONS`
**Grain:** One row per MCA payment event.
**Source:** `PROD_DB.DATA.LENDING_TRANSACTIONS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `FUNDING_TRANSACTION_ID` | VARCHAR PK | Transaction UUID | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TRANSACTION_AT` | TIMESTAMP_TZ | Event timestamp | — |
| `TRANSACTION_TYPE` | VARCHAR | `draw`, `cash_in` (repayment), `fee` | MCA Active |
| `AMOUNT` | FLOAT | Transaction amount | **Payments Made (MCA)** |
| `BUSINESS_SPECIFIC_ORDINAL` | NUMBER | 1 = first draw event | **MCA Attach** |

---

### `MART_FUNDING.DIM_MCA_APPLICATION_FUNNEL`
**Grain:** One row per business (latest MCA funnel state).
**Source:** `DEV_DB.DEV_ANWAR.DIM_FUNDING_APPLICATION_FUNNEL`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR PK | FK to `DIM_BUSINESSES` | — |
| `INVITATION_CREATED_AT` | TIMESTAMP_TZ | MCA invite sent date | **MCA Invite Sent** |
| `OFFER_ACCEPTED_AT` | TIMESTAMP_TZ | Offer acceptance date | **MCA Offer Accepted** |
| `OPEN_COHORT` | DATE | `DATE_TRUNC('month', OFFER_ACCEPTED_AT)` | **Open Cohort (MCA)** |

---

## MART_FINANCE

### `MART_FINANCE.FACT_UNIT_ECONOMICS`
**Grain:** One row per business per reporting month. **Fully pre-joined materialized table.**
**Source:** Union of all revenue + cost component models. Never build as a view.

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `APPLICATION_ID` | VARCHAR FK | FK to `DIM_APPLICATIONS` | — |
| `REPORTING_MONTH` | DATE FK | FK to `DIM_DATE` | — |
| `ACCOUNT_CREATED_MONTH` | DATE | Month account was opened | — |
| `FLOAT_REVENUE` | FLOAT | `AVG_DAILY_BALANCE × FLOAT_RATE` | **Float Revenue** |
| `MCA_FACTOR_FEE_REVENUE` | FLOAT | Factor fees collected | **MCA Factor Fee Revenue** |
| `MCA_LATE_FEE_REVENUE` | FLOAT | MCA late fees | **MCA Late Fee Revenue** |
| `MCA_CHARGE_OFF_LOSS` | FLOAT | MCA charge-off at DPD=180 | **MCA Charge Off Loss** |
| `MCA_RECOVERY` | FLOAT | MCA recovery amounts | **MCA Recovery** |
| `CC_INTEREST_REVENUE` | FLOAT | CC interest (PAYMENT_ALLOCATED_INTEREST deduped) | **CC Interest Revenue** |
| `CC_LATE_FEE_REVENUE` | FLOAT | CC late fees (PAYMENT_ALLOCATED_FEES deduped) | **CC Late Fee Revenue** |
| `CC_OTHER_FEE_REVENUE` | FLOAT | Foreign txn + returned payment fees | **CC Other Fee Revenue** |
| `CC_REWARDS_ACCRUED` | FLOAT | CC rewards cost (`REWARDS/100`) | **CC Rewards Accrued** |
| `CC_CHARGE_OFF_LOSS` | FLOAT | CC charge-off at DPD=180 | **CC Charge Off Loss** |
| `EXPRESS_ACH_REVENUE` | FLOAT | Express ACH fee revenue | **Express ACH Revenue** |
| `DOMESTIC_WIRE_REVENUE` | FLOAT | Wire fee revenue | **Domestic Wire Revenue** |
| `INTERCHANGE_REVENUE_DC` | FLOAT | Debit card interchange (from settlement) | Interchange Revenue (DC) |
| `INTERCHANGE_REVENUE_CC` | FLOAT | Credit card interchange (from settlement) | Interchange Revenue (CC) |
| `COST_TO_OPEN_COGS` | FLOAT | Hard-coded: 20.06 in open month, else 0 | **Cost to Open** |
| `COST_TO_RUN_COGS` | FLOAT | Hard-coded: 1.59 per active month | **Cost to Run** |
| `INTERCHANGE_EXPENSES_COGS` | FLOAT | Hard-coded: 0.7026 per month | **Interchange Expenses** |
| `ATM_REIMBURSEMENT` | FLOAT | Hard-coded: 0.0025 per month | **ATM Reimbursement** |
| `ONBOARDING_COSTS` | FLOAT | Hard-coded: 2.71 in open month, else 0 | **Onboarding Costs** |
| `FRAUD_LOSS` | FLOAT | Expensed fraud loss | **Fraud Loss** |
| `TOTAL_REVENUE_EXCL_INTERCHANGE` | FLOAT | Sum of all non-interchange revenue | **Total Revenue Excl. Interchange** |
| `MARGINAL_NET_INCOME` | FLOAT | Revenue − all COGS line items | **Marginal Net Income** |

---

### `MART_FINANCE.FACT_FLOAT_REVENUE`
**Grain:** One row per business per month.
**Source:** `PROD_DB.MODELS.HISTORIC_CASHFLOWS_LATEST` × `PROD_DB.MODELS.FLOAT_RATE_TABLE` (dbt seed)

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `MONTH` | DATE FK | FK to `DIM_DATE` | — |
| `AVERAGE_DAILY_BALANCE` | FLOAT | Average daily balance | **Average Monthly Balance (ADB)** |
| `FLOAT_RATE` | FLOAT | Fed rate proxy for this month | — |
| `FLOAT_REVENUE_APPROXIMATED` | FLOAT | `AVG_DAILY_BALANCE × FLOAT_RATE` | **Float Revenue** |

---

## MART_MARKETING

### `MART_MARKETING.FACT_AD_SPEND`
**Grain:** One row per day per platform per campaign.
**Source:** `INT_AD_SPEND_NORMALIZED` (union of Google, Facebook, Microsoft)

| Column | Type | Description | Metric |
|---|---|---|---|
| `AD_SPEND_ID` | VARCHAR PK | Composite surrogate key | — |
| `REPORT_DATE` | DATE FK | FK to `DIM_DATE` | **Day** |
| `PLATFORM` | VARCHAR | `google`, `facebook`, `microsoft` | — |
| `CAMPAIGN_ID` | VARCHAR | Campaign identifier | — |
| `CAMPAIGN_NAME` | VARCHAR | Campaign name | — |
| `CHANNEL_TYPE` | VARCHAR | `paid_search`, `paid_social`, `display` | — |
| `TOTAL_IMPRESSIONS` | NUMBER | Ad impressions | **Impressions** |
| `TOTAL_TRAFFIC` | NUMBER | Website sessions | **Traffic** |
| `TOTAL_CLICK` | NUMBER | CTA clicks | **Clicks** |
| `TOTAL_DOLLAR_COST` | FLOAT | Ad spend in dollars | **Daily Spend** |

---

### `MART_MARKETING.FACT_ACQUISITION_COST`
**Grain:** One row per application (attributed CAC).
**Source:** `PROD_DB.DATA.GTM_ACQUISITION_BUSINESS_COST`

| Column | Type | Description | Metric |
|---|---|---|---|
| `APPLICATION_ID` | VARCHAR PK | FK to `DIM_APPLICATIONS` | — |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `COST_PER_STARTED_APPLICATION` | FLOAT | Attributed CPSA | **Cost Per Started Application** |
| `COST_PER_OPENED_ACCOUNT` | FLOAT | Attributed CPOA | **Cost Per Opened Account** |
| `CONTRIBUTION_LTV_M1` | FLOAT | First-month LTV contribution | — |
| `CONTRIBUTION_LTV_M3` | FLOAT | Third-month LTV contribution | **ANPV (M3 LTV)** |

---

### `MART_MARKETING.FACT_APPLICATION_EVENTS`
**Grain:** One row per application.
**Source:** `METABASE_DB.ONBOARDING.APPLICATIONS` + `PROD_DB.DATA.GTM_ACQUISITION_FUNNEL`

| Column | Type | Description | Metric |
|---|---|---|---|
| `APPLICATION_ID` | VARCHAR PK | FK to `DIM_APPLICATIONS` | — |
| `APPLICATION_STARTED_AT` | TIMESTAMP_NTZ | Step 1 (email + password) | **Started Application** |
| `APPLICATION_COMPLETED_AT` | TIMESTAMP_NTZ | All steps e-signed | **Completed Application** |
| `ACCOUNT_OPENED_AT` | TIMESTAMP_TZ | Account approved and created | Opened Account |

---

### `MART_MARKETING.FACT_LTV_CONTRIBUTION`
**Grain:** One row per business per LTV type.
**Source:** `DEV_DB.DEV_ANWAR.FACT_LTV_COMPONENTS_MONTHLY`

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `APPLICATION_ID` | VARCHAR FK | FK to `DIM_APPLICATIONS` | — |
| `LTV_TYPE` | VARCHAR | `M1` or `M3` | — |
| `CONTRIBUTION` | FLOAT | Dollar LTV contribution | **18M_ANPV** |
| `MONTH_ON_BOOKS` | NUMBER | Months since account opened | 18M filter (`< 18`) |

---

## MART_PRODUCT

### `MART_PRODUCT.FACT_ACTIVATION_MILESTONES`
**Grain:** One row per business. First-ever activation dates for each product.
**Source:** Composite of all activity fact tables (first ordinal = 1 records)

| Column | Type | Description | Metric |
|---|---|---|---|
| `BUSINESS_ID` | VARCHAR PK | FK to `DIM_BUSINESSES` | — |
| `DEBIT_CARD_ATTACHED_AT` | TIMESTAMP_TZ | First debit card request event | **Debit Card Attach** |
| `CREDIT_CARD_ATTACHED_AT` | TIMESTAMP_TZ | First credit card request event | **Credit Card Attach** |
| `MCA_ATTACHED_AT` | TIMESTAMP_TZ | First MCA draw event | **MCA Attach** |
| `EXPRESS_ACH_ATTACHED_AT` | TIMESTAMP_TZ | First express ACH fee event | **Express ACH Attach** |
| `INVOICE_ATTACHED_AT` | TIMESTAMP_TZ | First invoice created | **Invoice Attach** |
| `RESERVES_ATTACHED_AT` | TIMESTAMP_TZ | First reserve created | **Reserves Attach** |
| `BOOKKEEPING_ATTACHED_AT` | TIMESTAMP_TZ | First categorized transaction | **Bookkeeping Attach** |
| `INTEGRATION_ATTACHED_AT` | TIMESTAMP_TZ | First integration connected | **Integrations Attach** |
| `BOOST_ATTACHED_AT` | TIMESTAMP_TZ | First boost received | **Boost Users Attach** |
| `IS_F90D` | BOOLEAN | TRUE if within 90 days of account open | **First 90 Day Customer (F90D)** |

---

### `MART_PRODUCT.FACT_ZENDESK_TICKETS`
**Grain:** One row per Zendesk ticket.
**Source:** `METABASE_DB.CS.ZENDESK_TICKET_METRICS`

| Column | Type | Description | Metric |
|---|---|---|---|
| `TICKET_ID` | NUMBER PK | Zendesk ticket ID | **Contact Volume** |
| `BUSINESS_ID` | VARCHAR FK | FK to `DIM_BUSINESSES` | — |
| `TICKET_CREATED_AT` | TIMESTAMP_NTZ | Ticket creation timestamp | — |
| `CREATED_CHANNEL` | VARCHAR | Contact channel. Exclude `side_conversation` | **Contact Channel** |
| `TICKET_SATISFACTION_SCORE` | VARCHAR | `good` or `bad` | **CSAT** |
| `TIME_TO_FIRST_REPLY_CALENDAR_MINUTES` | NUMBER | Calendar FRT | **First Reply Time** |
| `TIME_TO_FIRST_REPLY_BUSINESS_MINUTES` | NUMBER | Business hours FRT | **First Reply Time** |
| `TIME_TO_FINAL_RESOLUTION_CALENDAR_MINUTES` | NUMBER | Calendar TTFR | **TTFR** |
| `TIME_TO_FINAL_RESOLUTION_BUSINESS_MINUTES` | NUMBER | Business hours TTFR | **TTFR** |

---

### `MART_PRODUCT.FACT_INVOICE_ACTIVITY`
**Source:** `DEV_DB.DEV_ANWAR.FACT_INVOICE_ACTIVITY`
**Metrics:** Invoice Active (`COUNT DISTINCT BUSINESS_ID` with any invoice in period), Invoice Attach.

### `MART_PRODUCT.FACT_BOOKKEEPING_ACTIVITY`
**Source:** `DEV_DB.DEV_ANWAR.FACT_BOOKKEEPING_ACTIVITY`
**Metrics:** Bookkeeping Active, Bookkeeping Attach.

### `MART_PRODUCT.FACT_INTEGRATION_CONNECTIONS`
**Source:** `DEV_DB.DEV_ANWAR.FACT_INTEGRATIONS_ACTIVITY`
**Metrics:** Integrations Active, Integrations Attach.

### `MART_PRODUCT.FACT_PROVISIONAL_CREDITS`
**Source:** `DEV_DB.DEV_ANWAR.FACT_PROVISIONAL_CREDITS_ACTIVITY`
**Metrics:** Active Boost Users, Boost Users Attach.

### `MART_PRODUCT.FACT_RESERVE_LEDGER`
**Source:** `DEV_DB.DEV_ANWAR.FACT_RESERVES_ACTIVITY`
**Metrics:** Reserves Active, Reserves Attach.

---

## Validation: Existing Cube Coverage → Gold Home

| Existing Cube | Measure / Dimension | Gold Table |
|---|---|---|
| `businesses` | `opened_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `businesses` | `funded_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `businesses` | `active_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `businesses` | `financially_active_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `businesses` | `closed_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `businesses` | `inactive_accounts` | `CONFORMED.DIM_BUSINESSES` |
| `applications` | all dims | `CONFORMED.DIM_APPLICATIONS` |
| `mca_revenue` | `factor_fee_revenue`, `late_fee_revenue`, `charge_off_loss_mca` | `MART_FUNDING.FACT_MCA_REVENUE` |
| `total_revenue` | `revenue_excl_interchange`, `marginal_net_income` | `MART_FINANCE.FACT_UNIT_ECONOMICS` |
| `float_revenue` | `float_revenue_approximated` | `MART_FINANCE.FACT_FLOAT_REVENUE` |
| `monthly_balances` | `avg_daily_balance_aggr_monthly` | `MART_BANKING.FACT_MONTHLY_BALANCES` |
| `lifetime_values` | `contribution_ltv_m1`, `contribution_ltv_m3` | `MART_MARKETING.FACT_LTV_CONTRIBUTION` |
| `lithic_settlement_daily` | all interchange measures | `MART_CARDS.FACT_LITHIC_SETTLEMENT` |
| `variable_costs` | all 5 cost measures | `MART_FINANCE.FACT_UNIT_ECONOMICS` |
| `express_ach_revenue` | `express_ach_revenue` | `MART_BANKING.FACT_EXPRESS_ACH_ACTIVITY` |
| `credit_card_primary_revenue` | `interest_revenue_cc`, `late_fee_revenue_cc`, `charge_off_loss_cc` | `MART_FINANCE.FACT_UNIT_ECONOMICS` |
| `credit_card_other_revenue` | `credit_card_other_fee_revenue` | `MART_FINANCE.FACT_UNIT_ECONOMICS` |
| `credit_card_rewards` | `credit_card_rewards_accrued` | `MART_FINANCE.FACT_UNIT_ECONOMICS` |
| `domestic_wire_revenue` | `domestic_wire_revenue` | `MART_BANKING.FACT_DOMESTIC_WIRES` |
| `gtm_acquisition_cost` | `cost_per_started_application`, `cost_per_opened_account` | `MART_MARKETING.FACT_ACQUISITION_COST` |
| `gtm_acquisition_funnel` | all funnel measures + `spend` | `MART_MARKETING.FACT_AD_SPEND` + `FACT_APPLICATION_EVENTS` |
| `ltv_components_18m` | `ltv_18m_m1`, `ltv_18m_m3` | `MART_MARKETING.FACT_LTV_CONTRIBUTION` |
| `expected_payback` | `e18m_net_income`, `nip_score`, `e90d_net_income` | `MART_MARKETING.FACT_LTV_CONTRIBUTION` |
| `credit_card_credit_risk` | all dims + `payments_made_cc` | `MART_CARDS.DIM_CREDIT_CARD_APPLICATION_FUNNEL` + `FACT_CREDIT_CARD_TRANSACTIONS` |
| `funding_credit_risk` | all dims + `outstanding_balance_mca`, `payments_made_mca` | `MART_FUNDING.FACT_MCA_LOAN_TAPE` + `FACT_MCA_TRANSACTIONS` |
| `credit_card_funnel_cohort_based` | all funnel measures | `MART_CARDS.DIM_CREDIT_CARD_APPLICATION_FUNNEL` |
| `funding_funnel_cohort_based` | all funnel measures | `MART_FUNDING.DIM_MCA_APPLICATION_FUNNEL` |
| `zendesk_tickets` | all measures | `MART_PRODUCT.FACT_ZENDESK_TICKETS` |
| `transactions` | `gross_transactions`, `gross_volume`, all dims | `MART_BANKING.FACT_TRANSACTIONS` |
| `bookkeeping_activity` | `bookkeeping_attach`, `bookkeeping_active` | `MART_PRODUCT.FACT_BOOKKEEPING_ACTIVITY` |
| `boosts_activity` | `boost_attach`, `boost_active` | `MART_PRODUCT.FACT_PROVISIONAL_CREDITS` |
| `express_ach_activity` | `express_ach_attach`, `express_ach_active` | `MART_BANKING.FACT_EXPRESS_ACH_ACTIVITY` |
| `funding_activity` | `mca_attach`, `mca_active` | `MART_PRODUCT.FACT_ACTIVATION_MILESTONES` |
| `invoice_activity` | `invoice_attach`, `invoice_active` | `MART_PRODUCT.FACT_INVOICE_ACTIVITY` |
| `integrations_activity` | `integrations_attach`, `integrations_active` | `MART_PRODUCT.FACT_INTEGRATION_CONNECTIONS` |
| `reserves_activity` | `reserves_attach`, `reserves_active` | `MART_PRODUCT.FACT_RESERVE_LEDGER` |

**✅ All 37 existing Cube definitions have a designated Gold home. No orphaned measures or dimensions.**
