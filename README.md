# 🥃 Vendor Performance Analysis

![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-SQLite-07405E?logo=sqlite&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)

An end-to-end data analytics project that turns raw purchase, sales, and inventory records from a retail/wholesale distributor into vendor- and brand-level performance insights — built with **SQL (SQLite)**, **Python**, and **Power BI**.

---

## 📌 Overview

Retailers and distributors deal with hundreds of vendors and thousands of SKUs, and small inefficiencies in pricing or inventory management quietly erode profitability. This project builds a single analytical view of vendor performance — combining purchasing, sales, freight, and pricing data — to answer a few concrete business questions:

- Which brands are underpriced or under-promoted relative to their margin potential?
- Which vendors actually drive the business, and how dependent is the company on them?
- Does buying in bulk meaningfully lower unit cost?
- Where is money sitting idle in slow-moving stock?
- Do high-volume and low-volume vendors really operate at different profit margins, or is that just noise?

The project covers the full pipeline: raw data → SQLite database → a cleaned, vendor-level summary table → statistical analysis in Python → a written report and an interactive Power BI dashboard.

## 🎯 Objectives

- Identify brands that need promotional or pricing adjustments (low sales, high margin).
- Determine which vendors contribute the most to sales, purchases, and gross profit.
- Quantify the impact of bulk purchasing on unit cost.
- Flag vendors with low inventory turnover and excess/slow-moving stock.
- Statistically test whether profit margins genuinely differ between high- and low-performing vendors.

## 🗂️ Repository Structure

```
Vendor-Performance-Analysis/
├── ingestion_db.py                    # Loads raw CSVs from data/ into a SQLite database (inventory.db)
├── get_vendor_summary.py              # Builds & cleans the vendor_sales_summary table (SQL + pandas)
├── Exploratory Data Analysis.ipynb    # First pass: inspects raw tables, designs the summary table
├── Vendor Performance Analysis.ipynb  # Main analysis: stats, visuals, business questions, hypothesis test
├── vendor_sales_summary.csv           # Final cleaned dataset (10,692 rows x 18 columns)
├── Vendor Performance Report.pdf      # Written report: problem statement, findings, recommendations
├── vendor_performance.pbix            # Interactive Power BI dashboard
└── README.md
```

> **Note:** the raw source CSVs (`purchases.csv`, `purchase_prices.csv`, `sales.csv`, `vendor_invoice.csv`, `begin_inventory.csv`, `end_inventory.csv`) and the `logs/` folder the scripts write to aren't included in this repo — see **Getting Started** below.

## 🧰 Tech Stack

| Layer | Tools |
|---|---|
| Data storage | SQLite |
| ETL / data modeling | Python, SQLAlchemy, SQL (CTEs, joins, aggregations) |
| Analysis & statistics | pandas, NumPy, SciPy (confidence intervals, two-sample t-test) |
| Visualization (notebook) | Matplotlib, Seaborn |
| Dashboard | Power BI (DAX measures) |
| Environment | Jupyter Notebook |

## 🔄 Data Pipeline

### 1. Raw data → SQLite (`ingestion_db.py`)

Reads every CSV in a local `data/` folder and loads each one into `inventory.db` as a table named after the file (e.g. `purchases.csv` → `purchases`), logging progress and timing to `logs/ingestion_db.log`. The six source tables expected are:

| Table | Contents |
|---|---|
| `purchases` | Line-level purchase transactions — vendor, brand, quantity, dollars, date |
| `purchase_prices` | Reference purchase/actual price and bottle volume per brand |
| `vendor_invoice` | Invoice-level totals per vendor, including freight cost |
| `sales` | Line-level sales transactions — quantity, price, dollars, excise tax |
| `begin_inventory` | Inventory on hand at the start of the period |
| `end_inventory` | Inventory on hand at the end of the period |

(`begin_inventory` / `end_inventory` are ingested for completeness but aren't used in the vendor summary below.)

### 2. Vendor summary table (`get_vendor_summary.py`)

Joins `purchases`, `purchase_prices`, `sales`, and `vendor_invoice` through a multi-CTE SQL query into one vendor-and-brand-level table, then cleans it:

- casts `Volume` to numeric and fills missing values with 0
- strips stray whitespace from `VendorName` / `Description`
- derives four new metrics:

  ```
  GrossProfit          = TotalSalesDollars - TotalPurchaseDollars
  ProfitMargin         = GrossProfit / TotalSalesDollars * 100
  StockTurnover        = TotalSalesQuantity / TotalPurchaseQuantity
  SalesToPurchaseRatio = TotalSalesDollars / TotalPurchaseDollars
  ```

The result is written back to SQLite as `vendor_sales_summary` and exported to `vendor_sales_summary.csv`.

### Final dataset at a glance

| | |
|---|---|
| Rows | 10,692 (vendor–brand pairs) |
| Vendors | 126 |
| Brands | 10,663 |
| Total purchases | ~$321.9M |
| Total sales | ~$451.6M |

<details>
<summary>Column reference (18 columns)</summary>

| Column | Description |
|---|---|
| `VendorNumber`, `VendorName` | Vendor identifier and name |
| `Brand`, `Description` | Brand/product identifier and name |
| `PurchasePrice` | Price paid per unit |
| `ActualPrice` | List/reference price per unit |
| `Volume` | Bottle/package volume |
| `TotalPurchaseQuantity`, `TotalPurchaseDollars` | Units and dollars purchased |
| `TotalSalesQuantity`, `TotalSalesDollars` | Units and dollars sold |
| `TotalSalesPrice` | Sum of per-transaction sale prices |
| `TotalExciseTax` | Excise tax collected on sales |
| `FreightCost` | Freight cost billed by the vendor |
| `GrossProfit` | `TotalSalesDollars - TotalPurchaseDollars` |
| `ProfitMargin` | Gross profit as a % of sales |
| `StockTurnover` | Units sold ÷ units purchased |
| `SalesToPurchaseRatio` | Sales dollars ÷ purchase dollars |

</details>

## 📓 Analysis Workflow

**`Exploratory Data Analysis.ipynb`** — inspects the raw tables, checks row counts and relationships, and works out the joins needed to build `vendor_sales_summary` (this is where the SQL in `get_vendor_summary.py` comes from).

**`Vendor Performance Analysis.ipynb`** — the main analysis:
- Summary statistics, distribution plots, and boxplots to catch outliers and bad data (negative gross profit, zero sales, etc.), followed by filtering down to profitable, actually-sold records
- A correlation heatmap across all numeric columns
- Brands worth a pricing/promotion review (bottom-15%-sales, top-15%-margin)
- Top vendors and brands by sales
- A Pareto view of purchase concentration across vendors
- Unit-price impact of small vs. medium vs. large order sizes
- Vendors with stock turnover below 1 (slow-moving inventory)
- Capital tied up in unsold inventory, by vendor
- 95% confidence intervals for profit margin, comparing top-quartile vs. bottom-quartile vendors by sales
- A two-sample t-test on whether that margin difference is statistically significant

## 📊 Key Findings

1. **Pricing/promotion candidates** — 198 brands show low sales but above-average margins, suggesting room for promotion or price tuning without hurting profitability.
2. **Vendor concentration** — the top 10 vendors account for **65.69%** of total purchase dollars; the other 116 vendors share the remaining 34.31%, a meaningful supply-chain concentration risk.
3. **Bulk purchasing pays off** — moving from small to large order sizes cuts the average unit price by roughly **72%** (down to ~$10.78/unit), confirming bulk buying as an effective cost lever.
4. **Capital stuck in unsold stock** — vendors with stock turnover under 1 collectively tie up about **$2.71M** in unsold inventory.
5. **Margin vs. volume trade-off** — top-quartile vendors by sales average a 31.17% margin (95% CI 30.74–31.61%), while bottom-quartile vendors average 41.55% (95% CI 40.48–42.62%). Lower-volume vendors charge more but sell less.
6. **Statistically confirmed** — a two-sample t-test rejects the null hypothesis that these two groups share the same mean margin (p < 0.05): high- and low-performing vendors operate under genuinely different profitability profiles, not just random variation.

## 💡 Recommendations

- Re-evaluate pricing for low-sales, high-margin brands to grow volume without giving up margin.
- Diversify the vendor base to reduce dependence on a small group of suppliers.
- Lean further into bulk-purchasing agreements to keep unit costs down.
- Work down slow-moving inventory through adjusted order quantities, clearance pricing, or revised storage strategy.
- Invest in marketing/distribution for low-performing vendors rather than relying on margin alone.

## 📈 Power BI Dashboard

`vendor_performance.pbix` is a single-page interactive dashboard built on `vendor_sales_summary`, with:

- KPI cards for total purchases, total sales, gross profit, profit margin, and unsold capital
- A donut chart of purchase contribution % by vendor
- Bar charts for top vendors and top brands by sales
- A funnel of low-performing vendors and a scatter plot of low-performing brands

Open it in Power BI Desktop to filter and drill into any of the above.

> 📸 *Consider adding a screenshot here (e.g. `assets/dashboard_preview.png`) once you export one.*

## 🚀 Getting Started

### Prerequisites
- Python 3.x
- Power BI Desktop (optional, Windows-only — only needed to open `vendor_performance.pbix`)

### Setup

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>

pip install pandas numpy matplotlib seaborn scipy sqlalchemy jupyter
```

### Run the pipeline

The scripts expect a `data/` folder (raw CSVs) and a `logs/` folder (for log output) in the project root — add both before running:

```bash
mkdir -p data logs
# copy purchases.csv, purchase_prices.csv, sales.csv,
# vendor_invoice.csv, begin_inventory.csv and end_inventory.csv into data/

python ingestion_db.py        # raw CSVs -> inventory.db
python get_vendor_summary.py  # builds & cleans vendor_sales_summary
```

Then either:
- open the two notebooks in Jupyter to step through the EDA and full analysis, or
- skip straight to `vendor_sales_summary.csv` / `vendor_performance.pbix` if you just want the results.

## 🔧 Possible Extensions

- Add a `requirements.txt` / `environment.yml` for reproducible installs
- Parameterize the database path and raw-data location instead of hard-coding them
- Automate the raw-data refresh and re-run the pipeline on a schedule
- Publish the Power BI dashboard to the Power BI Service for easier sharing

## 🙋 Author

**Sourav Mishra** — Data Analyst / Analytics Engineer (Python, SQL, ETL, Power BI, Tableau)

## 📄 License

No license file is included yet — add one (e.g. MIT) if you'd like to set explicit reuse terms.
