# Ordinary Folk Schema Reference

## Dataset Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     UNIFIED VIEWS (use these)                   │
├─────────────────────────────────────────────────────────────────┤
│  all_stripe.*          │  all_postgres.*                        │
│  (charge, invoice,     │  (patient, order, evaluation,          │
│   subscription, etc.)  │   consultation_sessions, etc.)         │
└────────────┬───────────┴────────────────┬───────────────────────┘
             │                            │
┌────────────▼───────────┐  ┌─────────────▼──────────────┐
│   REGIONAL RAW DATA    │  │    REGIONAL RAW DATA       │
├────────────────────────┤  ├────────────────────────────┤
│  sg_stripe.*           │  │  sg_postgres_rds_public.*  │
│  hk_stripe.*           │  │  hk_postgres_rds_public.*  │
│  jp_stripe.*           │  │  jp_postgres_rds_public.*  │
└────────────────────────┘  └────────────────────────────┘
```

## Core Datasets

### `all_stripe` — Unified Stripe Data

Multi-region Stripe data with `region` column for filtering.

#### charge
Primary transaction record for all payments.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Charge ID (primary key) |
| region | STRING | sg, hk, jp |
| customer_id | STRING | FK → customer.id |
| invoice_id | STRING | FK → invoice.id (NULL for OTC) |
| payment_intent_id | STRING | FK → payment_intent.id |
| balance_transaction_id | STRING | FK → balance_transaction.id |
| amount | INT64 | Charge amount in subunits (cents) |
| amount_refunded | INT64 | Refunded amount in subunits |
| currency | STRING | 3-letter code (sgd, hkd, jpy) |
| status | STRING | succeeded, pending, failed |
| created | TIMESTAMP | Charge creation time |
| metadata | STRING | JSON with orderId, etc. |
| _fivetran_synced | TIMESTAMP | Last sync time |

**Key filters:** `WHERE status = 'succeeded'`

#### invoice
Subscription billing records.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Invoice ID (primary key) |
| region | STRING | sg, hk, jp |
| subscription_id | STRING | FK → subscription_history.id |
| customer_id | STRING | FK → customer.id |
| billing_reason | STRING | subscription_create, subscription_cycle, manual |
| subtotal | INT64 | Amount before discounts (subunits) |
| total | INT64 | Final amount (subunits) |
| amount_paid | INT64 | Amount collected (subunits) |
| currency | STRING | 3-letter code |
| status | STRING | paid, open, void, uncollectible |
| created | TIMESTAMP | Invoice creation time |

#### invoice_line_item
Line items on invoices.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Line item ID |
| invoice_id | STRING | FK → invoice.id |
| subscription_id | STRING | FK → subscription_history.id |
| price_id | STRING | FK → price.id |
| amount | INT64 | Line item amount (subunits) |
| quantity | INT64 | Quantity purchased |
| region | STRING | sg, hk, jp |

#### subscription_history
Subscription state over time (SCD Type 2).

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Subscription ID |
| region | STRING | sg, hk, jp |
| customer_id | STRING | FK → customer.id |
| status | STRING | active, canceled, past_due, trialing |
| start_date | TIMESTAMP | Subscription start |
| ended_at | TIMESTAMP | Cancellation date (NULL if active) |
| created | TIMESTAMP | Record creation |
| _fivetran_end | TIMESTAMP | SCD end timestamp |

**Deduplication:** `QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY _fivetran_end DESC) = 1`

#### subscription_item
Items within a subscription (links to pricing).

| Column | Type | Description |
|--------|------|-------------|
| subscription_id | STRING | FK → subscription_history.id |
| plan_id | STRING | FK → plan.id |
| quantity | INT64 | Quantity subscribed |

#### price
Stripe price objects.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Price ID (primary key) |
| product_id | STRING | FK → product.id |
| region | STRING | sg, hk, jp |
| unit_amount | INT64 | Price in subunits |
| currency | STRING | 3-letter code |
| recurring_interval | STRING | month, year, week, day |
| recurring_interval_count | INT64 | Interval multiplier |
| metadata | STRING | JSON with boxes count |

#### plan
Legacy pricing (similar to price).

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Plan ID |
| product_id | STRING | FK → product.id |
| amount | INT64 | Price in subunits |
| currency | STRING | 3-letter code |
| interval | STRING | month, year, week, day |
| interval_count | INT64 | Interval multiplier |
| metadata | STRING | JSON |

#### product
Product catalog.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Product ID (primary key) |
| region | STRING | sg, hk, jp |
| name | STRING | Product display name |
| metadata | STRING | JSON with condition, boxes, etc. |

**Metadata extraction:** `JSON_VALUE(metadata, '$.condition')`

#### customer
Stripe customer records.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Customer ID (primary key) |
| region | STRING | sg, hk, jp |
| email | STRING | Customer email |
| name | STRING | Customer name |
| created | TIMESTAMP | Account creation |

#### balance_transaction
Fee and payout tracking.

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Transaction ID |
| region | STRING | sg, hk, jp |
| amount | INT64 | Transaction amount (subunits) |
| fee | INT64 | Stripe fee (subunits) |
| net | INT64 | Net amount (subunits) |
| type | STRING | charge, payout, refund, etc. |

#### payment_intent
Payment intent records (for OTC charges).

| Column | Type | Description |
|--------|------|-------------|
| id | STRING | Payment intent ID |
| region | STRING | sg, hk, jp |
| metadata | STRING | JSON with priceIds, stripePriceIds |

### `all_postgres` — Unified Application Data

#### patient
Core patient/customer record from application DB.

| Column | Type | Description |
|--------|------|-------------|
| sys_id | STRING | Internal patient ID |
| stripe_customer_id | STRING | FK → all_stripe.customer.id |
| email | STRING | Patient email |
| from_platform_env | STRING | Brand identifier (noah, zoey) |
| region | STRING | sg, hk, jp |

**Brand extraction:** `INITCAP(from_platform_env) AS brand`

#### order
Order records from application.

| Column | Type | Description |
|--------|------|-------------|
| sys_id | STRING | Order ID |
| short_id | INT64 | Human-readable order number |
| patient_id | STRING | FK → patient.sys_id |
| price_id | STRING | FK → all_stripe.price.id |
| prescription_price_id | STRING | Override price for prescriptions |
| stripe_payment_intent_id | STRING | FK → payment_intent.id |
| stripe_subscription_id | STRING | FK → subscription_history.id |
| status | STRING | Order status |
| region | STRING | sg, hk, jp |

#### evaluation
Medical evaluation/questionnaire records.

| Column | Type | Description |
|--------|------|-------------|
| sys_id | STRING | Evaluation ID |
| name | STRING | Evaluation name/type |
| type | STRING | Condition category |
| region | STRING | sg, hk, jp |

#### consultation_sessions
Teleconsultation records.

| Column | Type | Description |
|--------|------|-------------|
| sys_id | STRING | Session ID |
| order_sys_id | STRING | FK → order.sys_id |
| consultation_session_status | STRING | Status |
| consultation_session_type | STRING | Type of consult |
| region | STRING | sg, hk, jp |

---

### `finance_metrics` — Analytics Views

#### contribution_margin
**Primary sales view** — consolidated multi-channel transaction data.

| Column | Type | Description |
|--------|------|-------------|
| sales_channel | STRING | Stripe, TikTok, Shopee, Lazada, Atome, SG COD, HK COD |
| region | STRING | sg, hk, jp |
| purchase_date | DATE | Transaction date |
| purchase_type | STRING | Subscription, One-Time |
| billing_reason | STRING | subscription_create, subscription_cycle, manual |
| customer_id | STRING | Customer identifier |
| charge_id | STRING | Unique transaction ID |
| product_id | STRING | Product identifier |
| product_name | STRING | Product display name |
| condition | STRING | Medical condition (ED, Hair Loss, etc.) |
| brand | STRING | Noah, Zoey, N/A |
| quantity | INT64 | Units purchased |
| currency | STRING | Local currency code |
| line_item_amount_usd | FLOAT64 | Revenue in USD |
| total_charge_amount_usd | FLOAT64 | Total charge in USD |
| cogs | FLOAT64 | Cost of goods sold (USD) |
| packaging | FLOAT64 | Packaging cost (USD) |
| cashback | FLOAT64 | Cashback rate (decimal) |
| fee_rate | FLOAT64 | Payment gateway fee rate |
| gst_vat | FLOAT64 | Tax rate (decimal, e.g., 0.09) |
| refund_rate | FLOAT64 | Portion refunded |
| amount_refunded_usd | FLOAT64 | Refund amount (USD) |
| acquisition_date | DATE | Customer first purchase date |
| new_existing | STRING | New, Existing |
| subscription_id | STRING | Stripe subscription ID |
| subscription_created_date | DATE | Subscription start date |

#### monthly_contribution_margin
Aggregated P&L by month with margin calculations.

| Column | Type | Description |
|--------|------|-------------|
| date | DATE | Month (first of month) |
| exact_date | DATE | Original transaction date |
| country | STRING | sg, hk, jp |
| condition | STRING | Medical condition |
| sales_channel | STRING | Sales channel |
| source | STRING | sales, marketing, delivery, opex, teleconsult_cogs |
| amount | FLOAT64 | Gross revenue |
| cogs | FLOAT64 | Cost of goods |
| packaging | FLOAT64 | Packaging costs |
| gateway_fees | FLOAT64 | Payment fees |
| tax_paid_usd | FLOAT64 | GST/VAT paid |
| refunds | FLOAT64 | Refund amounts |
| marketing_cost | FLOAT64 | Marketing spend |
| delivery_cost | FLOAT64 | Shipping costs |
| dispensing_fees | FLOAT64 | Pharmacy fees |
| operating_expense | FLOAT64 | Opex |
| staff_cost | FLOAT64 | Staff costs |
| gross_revenue | FLOAT64 | = amount |
| net_revenue | FLOAT64 | = amount - refunds - tax |
| gross_profit | FLOAT64 | = net_revenue - cogs - dispensing_fees |
| cm2 | FLOAT64 | = gross_profit - packaging - delivery - gateway_fees |
| cm3 | FLOAT64 | = cm2 - marketing |
| ebitda | FLOAT64 | = cm3 - opex - staff |

#### customer_lifecycle_monthly
Subscription customer cohort analysis.

| Column | Type | Description |
|--------|------|-------------|
| region | STRING | sg, hk, jp |
| obs_date | DATE | Observation month |
| lifecycle | STRING | New, Churn, Reactivation, Expansion, Contraction, Retention |
| condition | STRING | Medical condition |
| brand | STRING | Noah, Zoey |
| n_customers | INT64 | Customer count |
| current_mrr | FLOAT64 | Current month MRR |
| lagged_mrr | FLOAT64 | Previous month MRR |

#### acquisition_details
First subscription details per customer.

| Column | Type | Description |
|--------|------|-------------|
| customer_id | STRING | Stripe customer ID |
| region | STRING | sg, hk, jp |
| condition | STRING | Initial condition (grouped) |
| acquired_date | DATE | First subscription date |

#### ltv_cac
LTV/CAC metrics by region, condition, brand.

| Column | Type | Description |
|--------|------|-------------|
| region | STRING | sg, hk, jp |
| obs_date | DATE | Month |
| condition | STRING | Medical condition |
| brand | STRING | Noah, Zoey |
| n_new_customers | INT64 | New customers acquired |
| current_mrr | FLOAT64 | Active MRR |
| churned_mrr | FLOAT64 | Lost MRR |
| marketing_cost | FLOAT64 | Marketing spend |
| net_revenue | FLOAT64 | Net revenue |
| cogs | FLOAT64 | Cost of goods |

---

### `ref` — Reference/Lookup Tables

#### fx_rates
Currency conversion rates (annual averages).

| Column | Type | Description |
|--------|------|-------------|
| currency | STRING | 3-letter code (lowercase) |
| fx_to_usd | FLOAT64 | Multiplier to convert TO USD |
| year | INT64 | Rate year |

**Usage:** `amount / fx_to_usd AS amount_usd`

#### stripe_currency_subunits
Stripe currency decimal handling.

| Column | Type | Description |
|--------|------|-------------|
| currency | STRING | 3-letter code |
| subunits | INT64 | Subunits per unit (100 for most, 1 for JPY) |

**Usage:** `amount / COALESCE(subunits, 100)`

#### tax_rate_history
GST/VAT rates by region over time.

| Column | Type | Description |
|--------|------|-------------|
| region | STRING | sg, hk, jp |
| from_date | DATE | Rate effective from |
| to_date | DATE | Rate effective until |
| rate | FLOAT64 | Tax rate as decimal (0.09 = 9%) |

**Usage:** `WHERE date BETWEEN from_date AND to_date`

---

### `cac` — Marketing Analytics

#### marketing_spend
Unified ad spend across channels.

| Column | Type | Description |
|--------|------|-------------|
| channel | STRING | google_ads, facebook_ads, taboola, manual suppliers |
| brand | STRING | Noah, Zoey, N/A |
| date | DATE | Spend date |
| campaign_name | STRING | Campaign identifier |
| condition | STRING | Target condition |
| country_code | STRING | Target country |
| currency_code | STRING | Spend currency |
| cost_local | FLOAT64 | Spend in local currency |
| cost_usd | FLOAT64 | Spend in USD |
| clicks | FLOAT64 | Click count |
| impressions | INT64 | Impression count |
| reach | INT64 | Unique reach |

---

### `google_sheets` — Manual Data Inputs

#### Key cost tables (all have from_date/to_date pattern):
- `tiktok_cogs` → `finance_metrics.tiktok_product_costs`
- `shopee_cogs` → `finance_metrics.shopee_product_costs`
- `lazada_cogs` → `finance_metrics.lazada_product_costs`
- `sg_product_cost_stripe` / `hk_product_cost_stripe` / `jp_product_cost_stripe` → `all_stripe.product_cost_per_box`

#### Order data:
- `tiktok_orders` — TikTok Shop transactions
- `shopee_orders` — Shopee transactions
- `lazada_orders` — Lazada transactions
- `atome_manual` — Atome BNPL transactions

#### Operational data:
- `delivery_cost` — Shipping costs by date/country
- `opex` — Operating expenses (dispensing, staff, teleconsult fees)
- `tax_rates` → `ref.tax_rate_history`
- `campaign_condition_map` — Campaign to condition mapping

---

## Key Derived Views in `all_stripe`

#### subscription_metrics_monthly
Monthly subscription snapshots with MRR.

| Column | Type | Description |
|--------|------|-------------|
| subscription_id | STRING | Subscription ID |
| customer_id | STRING | Customer ID |
| region | STRING | sg, hk, jp |
| obs_date | DATE | Observation month |
| mrr_usd | FLOAT64 | Monthly recurring revenue |
| mrr_local | FLOAT64 | MRR in local currency |
| condition | STRING | Medical condition |
| brand | STRING | Noah, Zoey |
| new_existing | STRING | New or Existing at subscription start |

#### subscription_details
Current subscription state with product info.

| Column | Type | Description |
|--------|------|-------------|
| subscription_id | STRING | Subscription ID |
| currency | STRING | Billing currency |
| plan_id | STRING | Price/plan ID |
| interval | STRING | month, year, week, day |
| interval_count | INT64 | Billing frequency |
| product_id | STRING | Product ID |
| product_name | STRING | Product name |
| condition | STRING | Medical condition |
| subscription_mrr | FLOAT64 | Normalised MRR |

#### product_cost
COGS by price_id with date ranges.

| Column | Type | Description |
|--------|------|-------------|
| region | STRING | sg, hk, jp |
| price_id | STRING | FK → price.id |
| product_id | STRING | FK → product.id |
| from_date | DATE | Cost effective from |
| to_date | DATE | Cost effective until |
| cogs | FLOAT64 | Cost of goods |
| packaging | FLOAT64 | Packaging cost |
| cashback | FLOAT64 | Cashback rate |

#### otc_price_id
Price IDs for one-time (non-invoice) charges.

| Column | Type | Description |
|--------|------|-------------|
| payment_intent_id | STRING | FK → payment_intent.id |
| price_id | STRING | Extracted price ID |
| quantity | INT64 | Quantity |
| description | STRING | Line description |

---

## Common Join Patterns

### Stripe charge to product details
```sql
FROM all_stripe.charge AS ch
LEFT JOIN all_stripe.invoice AS inv
    ON ch.invoice_id = inv.id AND ch.region = inv.region
LEFT JOIN all_stripe.invoice_line_item AS ii
    ON inv.id = ii.invoice_id AND ch.region = ii.region
LEFT JOIN all_stripe.price AS px
    ON ii.price_id = px.id AND ch.region = px.region
LEFT JOIN all_stripe.product AS prod
    ON px.product_id = prod.id AND ch.region = prod.region
```

### Customer to brand mapping
```sql
WITH patient_brand AS (
    SELECT DISTINCT stripe_customer_id, INITCAP(from_platform_env) AS brand
    FROM all_postgres.patient
)
SELECT ...
FROM all_stripe.charge AS ch
LEFT JOIN patient_brand AS pb ON ch.customer_id = pb.stripe_customer_id
```

### Cost lookup with date validation
```sql
LEFT JOIN all_stripe.product_cost AS pc
    ON px.id = pc.price_id
    AND DATE(ch.created) BETWEEN pc.from_date AND pc.to_date
```

### FX conversion
```sql
LEFT JOIN ref.fx_rates AS fx ON LOWER(ch.currency) = fx.currency
LEFT JOIN ref.stripe_currency_subunits AS sub ON fx.currency = sub.currency
...
SELECT ch.amount / fx.fx_to_usd / COALESCE(sub.subunits, 100) AS amount_usd
```
