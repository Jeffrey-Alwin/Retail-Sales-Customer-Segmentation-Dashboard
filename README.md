# Retail Sales & Customer Segmentation Dashboard (Power BI)

An interactive 3-page Power BI dashboard that tracks retail sales performance, product contribution and customer value. It combines a PostgreSQL SQL layer, a Power BI star-style data model, DAX measures and RFM customer segmentation.

**Live dashboard:** [add Power BI public link]
**Screenshots:** see the `/screenshots` folder

---

## Business Questions

1. How much revenue does the store generate, and is it growing?
2. Which departments, brands and products drive revenue?
3. Which products are high-volume traffic drivers, and which are high-revenue premium items?
4. Who are the most valuable customers, and which customers are slipping away?

## Headline Results

| Metric | Value |
|---|---|
| Total revenue (merchandise only) | $7.39M |
| Total baskets | 252K |
| Average basket value | $29.28 |
| Customers | 2,500 |
| Revenue growth, latest vs. previous complete 4-week period | 7.90% |

**Key insights**
- Grocery is the largest department by a wide margin, followed by Drug GM and Produce.
- Fluid milk, soft drinks and beer/ale lead revenue. High-volume items are not always the highest earners per unit.
- **Champions are about 20% of customers but generate about 43% of revenue.**
- **Hibernating customers are the largest group (about 26% of customers) but contribute only about 8% of revenue**, which makes them a low-cost reactivation target, while At Risk customers are the priority win-back group.

---

## Dashboard Pages

1. **Sales Overview:** KPI cards, revenue by four-week period, revenue by department, a top-10 revenue contributors table, and Department and Brand slicers. Every visual on this page responds to both slicers.
2. **Product Performance:** top products by revenue and a revenue vs. units sold scatter plot.
3. **Customer Segmentation (RFM):** customers and revenue by segment, a segment summary table, and auto-generated insight lines.

## Tech Stack

- **Database:** PostgreSQL (source tables and SQL views)
- **Modeling / ETL:** Power Query (M), Power BI data model with relationships
- **Analytics:** DAX measures, SQL window functions (`ntile`, `lag`)
- **Visualization:** Power BI Desktop

## Data

- Source: Dunnhumby "The Complete Journey" retail dataset [confirm and link], with household transactions, products, campaigns and coupons.
- Main tables: `transaction_data` and `product`.

## Data Model

| Table / view | Role |
|---|---|
| `vw_merch_transactions` | Fact table: merchandise-only transaction lines |
| `vw_product_clean` | Product dimension (brand, department, commodity, sub-commodity), related on `product_id` |
| `vw_customer_rfm` | One row per household with RFM scores and segment, related on `household_key` |
| `vw_period_sales` | Four-week period lookup (`period_number`, `weeks_in_period`) |
| `vw_daily_sales` | Daily lookup, related on `day` |

All page 1 KPIs are measures on `vw_merch_transactions`, so the Department and Brand slicers filter them consistently through the product dimension.

---

## Methodology

### 1. Data cleaning: one shared rule
Fuel, coupons and miscellaneous sales are not merchandise, and they distorted rankings (gasoline quantity dwarfed every product). They are excluded once, in the SQL view `vw_merch_transactions`, so every downstream view and measure uses the same definition:

- Department `KIOSK-GAS` and `MISC SALES TRAN`
- Commodity `COUPON/MISC ITEMS`

The same rule is mirrored in Power Query as an `is_merchandise` flag, so rows are flagged rather than deleted. Product names were also cleaned (trimmed, capitalized, abbreviations expanded, and a small manual override list for names that were truncated in the source).

### 2. Four-week periods
Weeks are grouped as `floor((week_no - 1) / 4) + 1`. The final period contains only 2 weeks, so it is excluded from the trend chart and the growth KPI with a `weeks_in_period = 4` filter. This avoids a false revenue drop at the end of the chart.

### 3. Revenue growth KPI
Growth compares revenue in the latest complete four-week period with the previous one. It is calculated in DAX from the transaction data, so it matches the line chart and responds to the Department and Brand slicers.

### 4. RFM segmentation
For each household, calculated on merchandise transactions only:
- **Recency:** days since the last purchase, relative to the latest purchase day in the data
- **Frequency:** number of distinct baskets
- **Monetary:** total merchandise spend

Each metric is scored 1-5 with `ntile(5)`. A `household_key` tiebreaker keeps scores stable when many households share the same value.

| Segment | Rule |
|---|---|
| Champions | R ≥ 4, F ≥ 4, M ≥ 4 |
| Loyal Customers | R ≥ 3, F ≥ 4 |
| Potential Loyalists | R ≥ 4, F between 2 and 3 |
| At Risk | R ≤ 2, F ≥ 3 |
| Hibernating | R ≤ 2, F ≤ 2 |
| Others | Everything else |

Rules are evaluated top to bottom, and the first match wins.

### 5. Reconciliation checks
- Total Revenue, the sum of period revenue, the sum of daily revenue and the RFM `monetary` total agree.
- Average basket value × total baskets equals total revenue.
- Segment customer counts add up to the customer total, and revenue shares add up to 100%.
- A weighted-average revenue per customer is used in totals, so the total row is not a simple average of segment values.

---

## Assumptions & Limitations

- Basket value covers merchandise lines only. A trip that included fuel is valued without the fuel.
- Customer and basket counts are distinct counts, so they do not add up across departments or brands. Almost every household buys both private-label and national-brand items, so customer counts barely change under the Brand slicer.
- The Customer Segmentation page is not filtered by the Department and Brand slicers, because segments are scored per household across all merchandise.
- RFM thresholds use quintiles, so segment sizes depend on this dataset's distribution.
- The data is historical, so recency is measured from the last transaction day, not today.
- The top-10 table ranks products by revenue within the current slicer selection, so its Total row sums only those 10 rows and will not equal the Total Revenue card.

## Repository Structure

```
├── README.md
├── sql/
│   ├── vw_merch_transactions.sql
│   ├── vw_period_sales.sql
│   ├── vw_daily_sales.sql
│   └── vw_customer_rfm.sql
├── powerbi/
│   ├── retail_dashboard.pbix      (if file size allows)
│   └── measures.md
└── screenshots/
    ├── 01_sales_overview.png
    ├── 02_product_performance.png
    └── 03_customer_segmentation.png
```

## Key DAX Measures

```dax
Total Revenue = SUM('public vw_merch_transactions'[sales_value])

Total Baskets = DISTINCTCOUNT('public vw_merch_transactions'[basket_id])

Average Basket Value = DIVIDE([Total Revenue], [Total Baskets])

Total Customers = DISTINCTCOUNT('public vw_merch_transactions'[household_key])

Units Sold = SUM('public vw_merch_transactions'[quantity])

Latest Revenue Growth % =
VAR LatestPeriod =
    CALCULATE(
        MAX('public vw_period_sales'[period_number]),
        ALL('public vw_period_sales'),
        'public vw_period_sales'[weeks_in_period] = 4
    )
VAR CurrentRev =
    CALCULATE([Total Revenue], ALL('public vw_period_sales'),
        'public vw_period_sales'[period_number] = LatestPeriod)
VAR PriorRev =
    CALCULATE([Total Revenue], ALL('public vw_period_sales'),
        'public vw_period_sales'[period_number] = LatestPeriod - 1)
RETURN DIVIDE(CurrentRev - PriorRev, PriorRev)

Customers = DISTINCTCOUNT('public vw_customer_rfm'[household_key])

Segment Revenue = SUM('public vw_customer_rfm'[monetary])

% of Customers =
DIVIDE([Customers], CALCULATE([Customers], ALL('public vw_customer_rfm'[customer_segment])))

% of Revenue =
DIVIDE([Segment Revenue], CALCULATE([Segment Revenue], ALL('public vw_customer_rfm'[customer_segment])))

Revenue per Customer = DIVIDE([Segment Revenue], [Customers])
```

## What I Learned

- Outliers like fuel can break a chart without any error, so check units and categories before trusting a ranking.
- Precomputed values in SQL views cannot respond to slicers, so KPIs that must be filterable belong on the transaction-level table.
- Distinct counts cannot be summed from pre-aggregated rows, so baskets and customers are counted from transactions.
- Reconciling totals across pages catches filter and definition mismatches that charts alone never show.

## Possible Next Steps

- Market basket analysis (products frequently bought together).
- Track segment movement over time (for example, Champions slipping to At Risk).
- Use the campaign and coupon tables to measure win-back response.

---

*Author: [Your name] · [LinkedIn] · [Email]*
