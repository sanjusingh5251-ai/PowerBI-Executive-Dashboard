# Olist E-Commerce Analytics: PostgreSQL to Power BI

An end-to-end analytics project on the public Olist Brazilian e-commerce dataset. Raw CSV files were loaded into **PostgreSQL**, modelled with primary and foreign keys, shaped into **BI-ready SQL views**, and presented in a **6-page Power BI dashboard** built on a DAX measure library.

> Currency: Brazilian Real (R$). Data period: 2016 to 2018.

## Project at a glance

| Metric | Value |
|---|---|
| Orders with items | 98,666 |
| Order items | 112,650 |
| Item revenue | R$13.59M |
| Freight charged | R$2.25M (16.6% of item revenue) |
| Average order value | R$137.75 |
| Active sellers | 3,095 |
| Late deliveries | 6.8% of delivered orders (6,534 of 96,478) |

## Business questions

- How are revenue and orders trending, and how do they compare with last year?
- Which product categories drive revenue and orders, and how do customers rate them?
- How reliable is delivery, where are the delays, and how does shipping route (same state vs cross state) affect delivery time, freight and lateness?
- How do customers pay, and how is revenue spread across sellers and regions?

## Key findings

- **Delivery is reliable overall:** 93.2% of delivered orders arrived on or before the estimated date. Late rates vary widely by state, from about 21.4% in AL (highest) to about 2.9% in RO (lowest).
- **Cross-state orders are slower and costlier.** About 64% of orders ship across state lines. They take roughly 15 days versus 8 days for same-state orders, have a late rate of about 8% versus 4.5%, and carry about R$24 freight per item versus about R$13.5.
- **Freight is a visible cost:** R$2.25M, equal to about R$16.6 for every R$100 of product value.
- **Credit cards dominate payments:** 75.4% of orders (78.3% of payment value), and 66.9% of credit card orders are paid in installments.
- **Most orders are single-item** (1.14 items per order), so growth depends mainly on winning more orders.
- **Orders vs revenue by category:** bed_bath_table has the most orders, while health_beauty and watches_gifts lead on revenue.

Route-type and late-rate figures are read from the dashboard charts and are approximate.

## Tech stack

- **PostgreSQL** (pgAdmin, psql): tables, constraints, views
- **SQL:** joins, aggregation, window-style logic (`DISTINCT ON`, `ARRAY_AGG`), CASE logic
- **Power BI:** data modelling, DAX measures, calendar table, maps and interactive visuals

## Data model

Nine source tables, each loaded with an explicit primary key and foreign keys:

| Table | Grain (one row per) |
|---|---|
| customers_data | customer_id |
| orders_data | order |
| order_items_data | order item (`order_id` + `order_item_id`) |
| order_payments_data | payment (`order_id` + `payment_sequential`) |
| order_reviews_data | review |
| products_data | product |
| sellers_data | seller |
| product_category_name_translation | category |
| geolocations_data | zip prefix |

Foreign keys: orders reference customers; order items reference orders, products and sellers; payments and reviews reference orders.

## SQL layer: BI views

Business logic lives in five views, so the dashboard imports one clean fact table.

1. `bi_dim_product`: English category name (falls back to Portuguese) and product volume
2. `bi_dim_latest_review`: latest review per order (`DISTINCT ON`)
3. `bi_payments_order`: payments collapsed to one row per order (total value, max installments, primary payment type)
4. `bi_fact_order`: order-level dates, `delivery_days`, `is_late` flag, customer city and state
5. `bi_fact_sales`: item-grain fact table joining everything above plus seller location

**Design decision, controlling grain:** joining raw payments directly to order items multiplied rows (112,650 items became 117,604) and overstated payment value by about 27%. Payments and reviews are therefore aggregated to one row per order before joining.

## DAX layer

Measures are grouped as sales (Total Revenue, Total Order, AOV), delivery (Late Orders, Late Delivery %, On Time %, Avg Delivery Days), time intelligence (Revenue LY, YTD, Rolling 30D, YoY %), sellers and payments, and reviews. A `Dim_Date` calendar table and a `Route Type` column (same state vs cross state) support the analysis.

Because the fact table is item grain, order-based measures use `DISTINCTCOUNT(order_id)` so multi-item orders are never double counted. Order-level fields such as payment value are aggregated once per order:

```dax
Late Delivery % =
DIVIDE([Late Orders], [Order Delivered])

Total Payment =
SUMX(VALUES(bi_fact_sales[order_id]),
     CALCULATE(MAX(bi_fact_sales[payment_value])))
```

`Late Orders` counts delivered orders whose actual delivery date is after the estimated date, and `Late Delivery %` divides by delivered orders only.

## Dashboard pages

1. **Overview (executive dashboard):** KPI cards, sales trend, revenue trend, top categories
2. **Sales Trend:** YTD vs last year, rolling 30-day revenue, month by category matrix
3. **Category and Product:** review sentiment, treemap, category matrix
4. **Delivery and Logistics:** late delivery map by state, late vs on-time split, late orders by review bucket
5. **Sellers and Payments:** same-state vs cross-state comparison, freight and seller maps, payment breakdown
6. **Insights and Recommendations:** summary of findings and next actions

Screenshots are in the [`images`](images) folder.

![Overview](images/overview.png)
![Delivery and Logistics](images/delivery.png)
![Sellers and Payments](images/sellers_payments.png)

## Data challenges solved

| Problem | Cause | Fix |
|---|---|---|
| Duplicate key error on items and payments | One order has several items or payments, so `order_id` is not unique | Composite primary keys |
| Date import errors | Dates are month/day/year | Set `datestyle` to `MDY` |
| Category file rejected | `.csv` extension but tab-separated | Import with a tab delimiter |
| Encoding error on reviews | Accented Portuguese text in a Windows code page | Set client encoding before import |
| Doubled rows after repeat import | Same file loaded twice | `TRUNCATE` before reload; the primary key acts as a safety net |
| Row multiplication in joins | Payments are order-level, items are item-level | Aggregate to order grain before joining |

## How to run

1. Download the dataset from Kaggle: *Brazilian E-Commerce Public Dataset by Olist*.
2. Create a PostgreSQL database and run the table scripts (`create_table_sql`), then the foreign keys script.
3. Import each CSV into its table (see the data challenges above for date format, delimiter and encoding settings).
4. Run the view scripts in this order: `bi_dim_product`, `bi_dim_latest_review`, `bi_payments_order`, `bi_fact_order`, `bi_fact_sales`.
5. Open the `.pbix` file in Power BI Desktop, point it to your PostgreSQL database, and refresh.

## Repository structure

```
├── sql/
│   ├── create_table_sql.sql
│   ├── foreign_keys.sql
│   └── bi_views_sql.sql
├── dax/
│   └── bi_measure_sql.txt
├── dashboard/
│   └── Olist_dashboard.pbix
├── images/
│   └── (dashboard screenshots)
├── report/
│   └── Olist_project_report.docx
└── README.md
```

## Limitations and next steps

- Total Revenue sums item price across all order statuses, so it includes cancelled and unavailable orders. A delivered-only revenue measure would be stricter.
- Early and late months of the data are partial, so year-over-year comparisons are only reliable for periods complete in both years.
- Route type and lateness show association, not proof of cause; distance, carrier and seller quality can all contribute.
- Next: split delivery time into seller handling and carrier transit using the carrier handover date.

## Author

**[yogita]**, aspiring Data Analyst
github:(https://github.com/sanjusingh5251-ai)
Email: (yogita.sin231@gmail.com)
