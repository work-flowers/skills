---
name: ordinaryfolk-data-engineering
description: >
  BigQuery SQL patterns for Ordinary Folk healthcare subscription analytics.
  Use when writing queries for OF data warehouse including subscription metrics
  (MRR, churn, LTV), contribution margin analysis, multi-channel sales
  consolidation (Stripe, TikTok, Shopee, Lazada, Atome, COD), customer lifecycle
  analysis, marketing spend, and cost calculations. Regions SG, HK, JP.
---

# Ordinary Folk Data Engineering

BigQuery SQL patterns for healthcare subscription business operating in SG, HK, JP.

## Schema Reference

For detailed table/column documentation, see [references/schema.md](references/schema.md).

### Quick Reference — Key Tables

| Dataset | Key Tables | Purpose |
|---------|------------|---------|
| `all_stripe` | charge, invoice, subscription_history, price, product | Unified Stripe data (all regions) |
| `all_postgres` | patient, order, evaluation | Application DB |
| `finance_metrics` | contribution_margin, customer_lifecycle_monthly, ltv_cac | Analytics views |
| `ref` | fx_rates, tax_rate_history, stripe_currency_subunits | Lookups |
| `cac` | marketing_spend | Ad spend |
| `google_sheets` | tiktok_orders, shopee_orders, lazada_orders, opex | Manual inputs |

### Key Relationships

```
charge.customer_id → customer.id (+ region match)
charge.invoice_id → invoice.id → invoice_line_item.invoice_id
invoice.subscription_id → subscription_history.id
price.product_id → product.id
patient.stripe_customer_id → customer.id
```

## Naming Conventions

### View Naming
- Analytics views: `vw_<descriptive_name>.sql` → `CREATE VIEW <dataset>.<view_name>`
- Drop-create pattern: Always use `DROP VIEW IF EXISTS` before `CREATE VIEW`

```sql
DROP VIEW IF EXISTS finance_metrics.contribution_margin;
CREATE VIEW finance_metrics.contribution_margin AS
```

### Column Naming
- Snake_case for all columns: `customer_id`, `purchase_date`, `mrr_usd`
- USD-converted amounts: suffix with `_usd` (e.g., `line_item_amount_usd`, `mrr_usd`)
- Local currency amounts: suffix with `_local` or use column name alone
- Date columns: `purchase_date`, `obs_date` (observation), `acq_date` (acquisition), `from_date`/`to_date`
- Counts: prefix with `n_` (e.g., `n_customers`, `n_orders`)
- Rates: suffix with `_rate` (e.g., `fee_rate`, `refund_rate`)

### CTE Naming
- Descriptive, snake_case: `customers_monthly`, `sub_starts`, `patient_brand_stripe_ids`
- Suffix patterns: `_base`, `_final`, `_lagged`, `_tagged`

## Common SQL Patterns

### Currency Conversion with FX Rates

```sql
-- Standard conversion pattern
amount / fx.fx_to_usd AS amount_usd

-- With Stripe subunits (cents to dollars)
ch.amount / fx.fx_to_usd / COALESCE(sub.subunits, 100) AS amount_usd

-- Join pattern
LEFT JOIN ref.fx_rates AS fx
    ON LOWER(source.currency) = fx.currency
LEFT JOIN ref.stripe_currency_subunits AS sub
    ON fx.currency = sub.currency
```

### Date Range Validation for Costs

Costs, taxes, and FX rates have effective date ranges. Always validate:

```sql
LEFT JOIN all_stripe.product_cost AS pc
    ON px.id = pc.price_id
    AND DATE(ch.created) BETWEEN pc.from_date AND pc.to_date

LEFT JOIN ref.tax_rate_history AS t
    ON ch.region = t.region
    AND DATE(ch.created) BETWEEN t.from_date AND t.to_date
```

### Creating Date Range Views from Point-in-Time Data

```sql
SELECT
    *,
    effective_date AS from_date,
    COALESCE(
        LEAD(effective_date) OVER (PARTITION BY sku_id ORDER BY effective_date),
        '9999-12-31'
    ) AS to_date
FROM source_table
```

### QUALIFY for Deduplication

```sql
-- Get latest record per entity
SELECT *
FROM all_stripe.subscription_history
QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY _fivetran_end DESC) = 1

-- Get first record per customer
SELECT *
FROM source
WHERE mrr_usd > 0
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY obs_date ASC) = 1
```

### Window Functions for Lifecycle Analysis

```sql
-- Lagged MRR for churn detection
LAG(mrr_usd) OVER (PARTITION BY customer_id ORDER BY obs_date) AS lagged_mrr

-- First purchase date per customer
MIN(purchase_date) OVER (PARTITION BY customer_id) AS acquisition_date

-- Running aggregation check
MAX(ch._fivetran_synced) OVER() AS as_of
```

### Customer Lifecycle Classification

```sql
CASE
    WHEN current_mrr > 0 AND obs_date = acq_date THEN 'New'
    WHEN current_mrr = 0 AND lagged_mrr > 0 THEN 'Churn'
    WHEN current_mrr > 0 AND lagged_mrr = 0 AND obs_date != acq_date THEN 'Reactivation'
    WHEN current_mrr > lagged_mrr AND lagged_mrr > 0 THEN 'Expansion'
    WHEN current_mrr < lagged_mrr AND current_mrr > 0 THEN 'Contraction'
    WHEN current_mrr = lagged_mrr AND current_mrr > 0 THEN 'Retention'
    ELSE NULL
END AS lifecycle
```

### Monthly Time Series Generation

```sql
-- Generate monthly observation dates
SELECT
    s.subscription_id,
    DATE_ADD(DATE_TRUNC(s.created_at, MONTH), INTERVAL n MONTH) AS obs_date
FROM subscriptions AS s
JOIN UNNEST(
    GENERATE_ARRAY(0, DATE_DIFF(
        DATE_TRUNC(s.end_date, MONTH),
        DATE_TRUNC(s.created_at, MONTH),
        MONTH
    ))
) AS n
```

### Multi-Region Handling

```sql
-- Always include region in joins
LEFT JOIN all_stripe.invoice AS inv
    ON ch.invoice_id = inv.id
    AND ch.region = inv.region

-- Filter for active markets
WHERE region IN ('sg', 'hk', 'jp')
-- or
WHERE country IN ('sg', 'hk', 'jp')
```

### Patient Brand Mapping

```sql
WITH patient_brand_stripe_ids AS (
    SELECT DISTINCT
        stripe_customer_id,
        INITCAP(from_platform_env) AS brand
    FROM all_postgres.patient
)
-- Then join on customer_id
LEFT JOIN patient_brand_stripe_ids AS pbsi
    ON ch.customer_id = pbsi.stripe_customer_id
```

## Business Logic

### MRR Calculation

```sql
CASE
    WHEN pl.interval = 'month' THEN pl.amount * si.quantity / COALESCE(subs.subunits, 100) / COALESCE(pl.interval_count, 1)
    WHEN pl.interval = 'year' THEN pl.amount * si.quantity / COALESCE(subs.subunits, 100) / (12 * COALESCE(pl.interval_count, 1))
    WHEN pl.interval = 'week' THEN pl.amount * si.quantity / COALESCE(subs.subunits, 100) * (52 / 12) / COALESCE(pl.interval_count, 1)
    WHEN pl.interval = 'day' THEN pl.amount * si.quantity / COALESCE(subs.subunits, 100) * (365 / 12) / COALESCE(pl.interval_count, 1)
    ELSE 0
END AS subscription_mrr
```

### Condition Grouping

```sql
CASE
    WHEN condition IN ('ED', 'PE') THEN 'ED + PE'
    WHEN condition IN ('Brand', 'OTC', 'Smoking Cessation', 'Sex Toys') THEN 'Other'
    WHEN condition IS NOT NULL THEN condition
    ELSE 'Other'
END AS condition
```

### New vs Existing Customer

```sql
-- Based on acquisition date window (7 days)
CASE
    WHEN purchase_date <= DATE_ADD(acquisition_date, INTERVAL 7 DAY) THEN 'New'
    WHEN acquisition_date IS NOT NULL THEN 'Existing'
END AS new_existing

-- Based on prior purchase history
CASE
    WHEN first_purchase_date < subscription_created_at THEN 'Existing'
    ELSE 'New'
END AS new_existing
```

### Contribution Margin Calculations

```sql
-- Net revenue
amount - refunds - tax_paid_usd AS net_revenue

-- Gross profit
amount - refunds - tax_paid_usd - cogs - dispensing_fees AS gross_profit

-- CM2 (after variable costs)
amount - refunds - tax_paid_usd - cogs - dispensing_fees - packaging - delivery_cost - gateway_fees AS cm2

-- CM3 (after marketing)
cm2 - marketing_cost AS cm3

-- Tax extraction from gross amount
(amount - amount_refunded_usd) * (1 - 1 / (1 + gst_vat)) AS tax_paid_usd
```

### COGS with Refund Adjustment

```sql
cogs * (1 - SAFE_DIVIDE(amount_refunded_usd, amount)) AS adjusted_cogs
```

## Multi-Channel Sales Patterns

### Stripe Data Structure

```sql
SELECT
    'Stripe' AS sales_channel,
    ch.region,
    CASE
        WHEN ii.subscription_id IS NULL THEN 'One-Time'
        ELSE 'Subscription'
    END AS purchase_type,
    COALESCE(inv.billing_reason, 'manual') AS billing_reason,
    ch.customer_id,
    ch.id AS charge_id,
    JSON_VALUE(ch.metadata, '$.orderId') AS order_sys_id,
    DATE(ch.created) AS purchase_date,
    -- Price ID fallback chain
    COALESCE(
        ii.price_id,
        otc.price_id,
        JSON_VALUE(pi.metadata, '$.priceIds'),
        JSON_VALUE(pi.metadata, '$.stripePriceIds'),
        JSON_VALUE(pi.metadata, '$.paymentIntentPriceId')
    ) AS price_id
FROM all_stripe.charge AS ch
WHERE ch.status = 'succeeded'
```

### Marketplace Channels (TikTok, Shopee, Lazada)

All use:
- `'One-Time'` for purchase_type
- Singapore region (`'sg'`)
- Custom product cost tables with date ranges
- Fee calculations vary by platform

### BNPL (Atome)

```sql
-- Aggregates multiple transactions per order
SUM(GREATEST(am.transaction_amount, 0)) AS total_charge
SUM(ABS(LEAST(am.transaction_amount, 0))) AS refunds
-- Link via order short_id to PostgreSQL
LEFT JOIN sg_postgres_rds_public.order AS o
    ON am.e_commerce_platform_order_id = o.short_id
```

### Union Pattern

All channel CTEs output identical column structure, then:

```sql
SELECT * FROM stripe_data
UNION ALL
SELECT * FROM tiktok_data
UNION ALL
SELECT * FROM shopee_data
-- etc.
```

## Comment Style

```sql
-- Single line for brief explanations
-- CTE to get subscription start dates from Stripe subscription history

-- Block comments for complex logic
-- Calculate line item amount proportionally based on invoice breakdown
-- This handles cases where the charge amount differs from invoice subtotal

-- Inline comments for non-obvious logic
.02 AS cashback,  -- Fixed 2% cashback for COD orders

-- Document external references
-- https://docs.stripe.com/currencies#zero-decimal
```

## Metadata Extraction

```sql
-- Product condition from metadata
JSON_VALUE(prod.metadata, '$.condition') AS condition
COALESCE(JSON_VALUE(prod.metadata, '$.condition'), 'N/A') AS condition

-- Boxes per product
COALESCE(CAST(JSON_VALUE(px.metadata, '$.boxes') AS FLOAT64), 1) AS n_boxes

-- Order ID from charge
JSON_VALUE(ch.metadata, '$.orderId') AS order_sys_id
```

## Safe Division

Always use `SAFE_DIVIDE` for ratios that might divide by zero:

```sql
SAFE_DIVIDE(refunds, line_item_amount_usd) AS refund_rate
SAFE_DIVIDE(fees, amount) AS fee_rate
```

## Performance Patterns

### Filter Early
```sql
WHERE ch.status = 'succeeded'  -- Apply filters in base CTEs
```

### Use INNER JOIN for Required Relationships
```sql
-- Only include subscriptions with successful charges
INNER JOIN paid_subs AS ps USING(subscription_id)
```

### Aggregate Before Joining
```sql
-- Pre-aggregate in CTE, then join
WITH aggregated AS (
    SELECT customer_id, SUM(amount) AS total
    FROM charges
    GROUP BY 1
)
SELECT ... FROM customers LEFT JOIN aggregated USING(customer_id)
```
