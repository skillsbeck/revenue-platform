## Business Impact

This platform demonstrates how a governed analytics layer can:

* Eliminate metric inconsistency across teams

* Standardize CAC and LTV definitions

* Enable data-driven budget allocation

* Improve marketing ROI visibility

* Provide forward-looking revenue forecasting

It models how a Senior Analytics Engineer partners with Finance, Marketing, and Leadership to deliver scalable analytics infrastructure.

## Technologies Used

* Snowflake (Data Warehouse)

* dbt (Transformation + Testing)

* SQL

* Streamlit (Executive Application)

* Git/GitHub

---

## Metric Definitions & Calculations 

This project treats metrics as **governed, auditable definitions** implemented in the dbt marts layer.

Each metric includes:

- **Definition** (what it means)
- **Grain** (what level it’s valid at)
- **Formula**
- **Implementation notes** (where/how it’s computed)
- **Pitfalls avoided** (how we protect trust)

---

### Revenue Metrics

#### 1) Gross Revenue
- **Definition:** Total revenue before refunds, discounts, and fees.
- **Grain:** `order_id` (transaction)
- **Formula:** `gross_revenue = SUM(item_price * quantity)`
- **Implementation:** Aggregated from line items into `fact_revenue`.
- **Pitfalls avoided:** Prevents double counting multi-line orders by aggregating at `order_id`.

#### 2) Discounts
- **Definition:** Total promotional/discount amount applied to transactions.
- **Grain:** `order_id`
- **Formula:** `discount_amount = SUM(discount_value)`
- **Implementation:** Applied at a single level (line *or* order) then rolled into `fact_revenue`.
- **Pitfalls avoided:** Avoids subtracting discounts twice.

#### 3) Refunds
- **Definition:** Total refunded amount applied to prior purchases.
- **Grain:** `refund_id` (refund event)
- **Formula:** `refund_amount = SUM(refund_value)`
- **Implementation:** Tracked in `stg_refunds` and joined to orders; netted by `refund_date` in `fact_revenue`.
- **Pitfalls avoided:** Refund timing handled explicitly; refunds aren’t “lost” in the original purchase month.

#### 4) Net Revenue
- **Definition:** Revenue after discounts and refunds.
- **Grain:** `customer_id + order_date + order_id`
- **Formula:** `net_revenue = gross_revenue - discount_amount - refund_amount`
- **Implementation:** Materialized in `fact_revenue`.
- **Pitfalls avoided:** Reconciles to raw totals; refunds netted by event date.

#### 5) Gross Margin / Gross Margin %
- **Definition:** Profitability after COGS.
- **Grain:** `order_id`
- **Formula:**  
  - `gross_margin = net_revenue - cogs`  
  - `gross_margin_pct = gross_margin / NULLIF(net_revenue, 0)`
- **Implementation:** COGS joined from `seed_product_costs` (transparent + version-controlled).
- **Pitfalls avoided:** Divide-by-zero handled; COGS applied at the same grain as revenue.

#### 6) MRR (Monthly Recurring Revenue)
- **Definition:** Monthly recurring subscription revenue.
- **Grain:** Month
- **Formula:**  
  - Monthly plan: `MRR = SUM(monthly_plan_amount)`  
  - Annual plan: `MRR = SUM(annual_amount / 12)`
- **Implementation:** Built from subscription facts into `mart_revenue_monthly`.
- **Pitfalls avoided:** One-time purchases excluded; annual plans normalized to avoid spikes.

#### 7) ARR (Annual Recurring Revenue)
- **Definition:** Annualized recurring subscription revenue.
- **Grain:** Month
- **Formula:** `ARR = MRR * 12`
- **Implementation:** Computed in `mart_revenue_monthly`.

---

### Marketing Metrics

#### 8) Marketing Spend
- **Definition:** Total paid media spend by channel and day.
- **Grain:** `channel + date`
- **Formula:** `spend = SUM(spend_amount)`
- **Implementation:** Materialized in `fact_marketing_spend`.
- **Pitfalls avoided:** Channel naming normalized via `dim_channel`.

#### 9) Attributed Conversions
- **Definition:** Conversions attributed to a channel based on explicit attribution weights.
- **Grain:** `channel + date`
- **Formula:** `attributed_conversions = SUM(conversions * channel_weight)`
- **Implementation:** Weights stored in `seed_channel_weights` (auditable).
- **Pitfalls avoided:** Prevents multi-channel double counting without clear weighting.

#### 10) CAC (Customer Acquisition Cost)
- **Definition:** Average cost to acquire a new customer.
- **Grain:** Month (or channel-month)
- **Formula:** `CAC = marketing_spend / NULLIF(new_customers, 0)`
- **Where:** `new_customers = COUNT(DISTINCT customer_id WHERE first_purchase_month = month)`
- **Implementation:** `first_purchase_date` in `dim_customer`, CAC in `mart_marketing_monthly`.
- **Pitfalls avoided:** Uses first purchase only (repeat buyers aren’t “new”).

#### 11) ROAS (Return on Ad Spend)
- **Definition:** Attributed revenue per $1 spent.
- **Grain:** `channel + month` (or `channel + date`)
- **Formula:** `ROAS = attributed_revenue / NULLIF(marketing_spend, 0)`
- **Implementation:** Attributed revenue computed using acquisition channel or attribution mapping.
- **Pitfalls avoided:** Separates total revenue from attributed revenue.

---

### Customer Metrics

#### 12) LTV (Lifetime Value, margin-based)
- **Definition:** Total gross margin generated by a customer over their lifetime (or horizon).
- **Grain:** Customer
- **Formula:** `LTV = SUM(gross_margin) OVER customer lifetime`
- **Implementation:** Materialized in `fact_customer_ltv`.
- **Pitfalls avoided:** Uses gross margin (not revenue) to reflect profitability.

#### 13) LTV:CAC Ratio
- **Definition:** Acquisition efficiency.
- **Grain:** Customer (or cohort)
- **Formula:** `LTV_CAC = LTV / NULLIF(CAC, 0)`
- **Implementation:** CAC assigned via cohort/channel average; rolled into cohort marts.
- **Pitfalls avoided:** No divide-by-zero; mapping is documented and versioned.

#### 14) Cohort Retention Rate
- **Definition:** % of customers in a cohort who purchase again after N months.
- **Grain:** Cohort-month
- **Formula:** `retention_rate_m = retained_customers_m / cohort_size`
- **Implementation:** Computed in `mart_cohorts`.
- **Pitfalls avoided:** Cohort denominator fixed at cohort start to avoid drift.

---

### Forecasting Metrics

#### 15) 30/60/90 Day Revenue Forecast (Run-Rate Baseline)
- **Definition:** Expected revenue based on recent daily run rate.
- **Grain:** Day or month
- **Formula:** `forecast = AVG(daily_net_revenue_last_28_days) * forecast_days`
- **Implementation:** `forecast_params` in seeds; forecast in `mart_forecasting`.
- **Pitfalls avoided:** Parameters are explicit + version-controlled (no hidden spreadsheet logic).

---

### ✅ Metric Quality Checks (Examples)

To protect trust in executive metrics, marts include validations such as:

- `net_revenue >= 0` (except refund-only edge cases)
- `CAC >= 0`
- `gross_margin_pct BETWEEN -1 AND 1` (configurable bounds)
- `MRR` excludes one-time purchases
- Reconciliation checks between raw totals and marts

## Author

Sarah Connolly
Senior Analytics Engineer | Revenue Intelligence | Snowflake + dbt
