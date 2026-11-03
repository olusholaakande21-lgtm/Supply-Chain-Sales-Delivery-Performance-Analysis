# Supply Chain Sales & Delivery Performance Analysis

An end-to-end analysis of a global supply chain dataset, from raw CSV to a one-page Power BI dashboard. The project answers three questions: **where does the money come from, where does it leak, and are orders arriving on time?**


![Uploading Supply Chain Dashboard.png…]()


---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Business Questions](#business-questions)
3. [Dataset](#dataset)
4. [Tools Used](#tools-used)
5. [Repository Structure](#repository-structure)
6. [Data Cleaning](#data-cleaning)
7. [Data Model](#data-model)
8. [Key DAX Measures](#key-dax-measures)
9. [Key Findings](#key-findings)
10. [Recommendations](#recommendations)
11. [Data Quality Notes](#data-quality-notes)
12. [How to Reproduce](#how-to-reproduce)
13. [Skills Demonstrated](#skills-demonstrated)
14. [Contact](#contact)

---

## Project Overview

The source data is a single wide table of 180,519 order items and 53 columns, covering January 2015 to January 2018 across 164 countries and 118 products. It contained encoding problems, duplicated and empty columns, location errors and incomplete records.

This project cleans the data, reshapes it into a star schema, builds DAX measures, and presents the results in a single-page Power BI dashboard, supported by a written pre-, in- and post-analysis document.

## Business Questions

- What are total sales, profit, margin, orders and average order value?
- Which products, departments and markets drive revenue?
- How did performance change across years and months?
- How concentrated is revenue among customers?
- What share of deliveries arrive late, and which shipping modes fail most often?
- Which payment types are linked to cancelled or fraud-flagged orders?
- Do discounts affect margin, and how many items are sold at a loss?
- Where are the data-quality gaps?

## Dataset

| Item | Detail |
|---|---|
| Records | 180,519 order items (65,752 orders) |
| Columns | 53 in the raw file |
| Period | January 2015 to January 2018 |
| Coverage | 164 countries, 118 products, 20,652 customers |
| Source | DataCo Smart Supply Chain dataset ([link to source](#)) |

> The raw file is about 96 MB, so it is not committed to this repository. Download it from the source link above and place it in `data/raw/`.

## Tools Used

- **Python (pandas):** data cleaning and validation
- **Power BI Desktop:** data model, DAX and dashboard
- **DAX:** measures and time intelligence
- **Star schema modelling**

## Repository Structure

```
├── data/
│   ├── raw/                  # original CSV (not committed)
│   └── cleaned/
│       ├── Fact_OrderItems.csv
│       ├── Dim_Customer.csv
│       ├── Dim_Product.csv
│       ├── Dim_Geography.csv
│       └── Dim_Date.csv
├── notebooks/
│   └── cleaning.ipynb        # cleaning and modelling steps
├── powerbi/
│   └── Supply_Chain_Dashboard.pbix
├── docs/
│   ├── Analysis_Document.pdf # pre-, in- and post-analysis boards
│   └── Data_Dictionary_and_Cleaning_Log.md
├── images/
│   └── dashboard.png
└── README.md
```

## Data Cleaning

- Read the file with the correct encoding (it was not UTF-8) and exported as UTF-8.
- Trimmed stray whitespace across text columns.
- **Dropped 14 columns:** masked customer email and password, customer street, empty product description, constant product status, product image URL, order zipcode (86% empty), and six exact duplicates of other columns.
- Repaired 3 customer records where city, state and zipcode were shifted.
- Fixed **105 US customer longitudes** with the wrong sign and nulled 9 impossible coordinate pairs.
- Translated Spanish country names to English and merged duplicates (for example "EE. UU." and "Estados Unidos").
- Parsed order and ship dates, and added helper columns: delay days, loss-making flag and cancelled/fraud flag.
- Validated the output: no duplicate rows, unique order-item IDs, and totals matching the raw data (net sales $33.05M, profit $3.97M).

The full list is in [`docs/Data_Dictionary_and_Cleaning_Log.md`](docs/Data_Dictionary_and_Cleaning_Log.md).

## Data Model

A star schema with one fact table and four dimensions:

```
Dim_Customer ─┐
Dim_Product  ─┼──< Fact_OrderItems >── Dim_Date
Dim_Geography ┘
```

| Table | Key | Role |
|---|---|---|
| Fact_OrderItems | Order Item Id | One row per order item: sales, profit, discount, shipping and status |
| Dim_Customer | Customer Id | Customer name and segment |
| Dim_Product | Product Id | Product, category and department |
| Dim_Geography | Geography Key | Market, region, country, state, city |
| Dim_Date | Date Key | Calendar attributes; related to Order Date (active) |

All relationships are one-to-many with single-direction filtering.

## Key DAX Measures

```dax
Net Sales = SUM(Fact_OrderItems[Net Sales])

Profit Margin % = DIVIDE([Total Profit], [Net Sales])

Orders = DISTINCTCOUNT(Fact_OrderItems[Order Id])

Late Delivery % =
DIVIDE(
    CALCULATE(SUM(Fact_OrderItems[Late Delivery Flag]),
        Fact_OrderItems[Cancelled / Fraud Flag] = 0),
    CALCULATE(COUNTROWS(Fact_OrderItems),
        Fact_OrderItems[Cancelled / Fraud Flag] = 0)
)

Cancelled / Fraud % =
DIVIDE(SUM(Fact_OrderItems[Cancelled / Fraud Flag]), COUNTROWS(Fact_OrderItems))

Net Sales LY = CALCULATE([Net Sales], SAMEPERIODLASTYEAR(Dim_Date[Date]))
```

## Key Findings

| Metric | Result |
|---|---|
| Net sales | $33.05M |
| Total profit | $3.97M (12.00% margin) |
| Orders / customers | 65,752 / 20,652 |
| Average order value | $502.71 |
| Late deliveries | **57.29%** of delivered items |
| Cancelled / fraud items | 4.30%, **all from Transfer payments** |
| Loss-making items | 18.7% (about $3.88M in lost profit) |

**Delivery by shipping mode**

| Mode | Late % |
|---|---|
| First Class | 100% |
| Second Class | 79.8% |
| Same Day | 47.9% |
| Standard Class | 39.8% |

**Other highlights**

- Fan Shop generates 46.5% of net sales, and the top five products about 60.6%.
- Europe (29.6%) and LATAM (27.9%) are the largest markets; margins are similar across all markets.
- Lateness is almost identical across markets (56.8% to 57.7%), which points to a company-wide process issue.
- 15.54% of Transfer-payment items are cancelled or flagged; Cash, Debit and Payment show none.
- Margin is flat across discount bands (11.7% to 12.4%).

## Recommendations

1. **Fix the delivery promise:** reset lead times for First and Second Class or improve fulfilment.
2. **Protect core products and departments** that drive most revenue.
3. **Reduce loss-making sales** and set a minimum-margin rule for discounts.
4. **Verify Transfer payments** to cut cancellations and fraud.
5. **Improve data capture** so order-item records stay complete.

## Data Quality Notes

- From **October 2017**, orders contain about 1 item on average, compared with about 3 before. Order counts held steady, but item-level sales appear to collapse. The dashboard trend is filtered to September 2017, and order counts are used for later months.
- Late Delivery Flag is inconsistent for cancelled shipments, so delivery KPIs exclude cancelled and fraud-flagged orders.
- Order State and City remain partly in Spanish.
- Latitude and longitude are customer locations, not delivery destinations.

## How to Reproduce

1. Download the raw dataset and place it in `data/raw/`.
2. Run `notebooks/cleaning.ipynb` to produce the five CSVs in `data/cleaned/`.
3. Open `powerbi/Supply_Chain_Dashboard.pbix` in Power BI Desktop.
4. If the file paths break, go to **Transform data > Data source settings** and point each table to your local `data/cleaned/` folder.
5. Mark `Dim_Date` as a date table and confirm the relationships in Model view.

## Skills Demonstrated

- Data cleaning and validation with pandas
- Dimensional modelling (star schema)
- DAX and time intelligence
- Dashboard design and data storytelling
- Business problem framing and written recommendations
- Data-quality detection and documentation

## Contact

**Shola**
[LinkedIn](#) | [Medium](#) | [Portfolio](#)
