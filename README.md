# Distribution Company Data Warehouse

An end-to-end data warehousing project built on a fictional mid-sized product distribution company. Covers the full supply chain loop — procurement → inventory → orders → shipment — across a normalized OLTP source, a star schema warehouse, an ETL pipeline, and an analytical layer.

Built to demonstrate data engineering and analytics skills for junior/intern roles.

---

## Project Overview

| Layer | Technology | Status |
|---|---|---|
| OLTP Source Database | MySQL | ✅ Complete |
| Star Schema (Warehouse) | MySQL | 🔄 In Progress |
| Staging Area | MySQL | 🔄 In Progress |
| ETL Pipeline | Python (Pandas, SQLAlchemy) | 🔄 In Progress |
| Analytical SQL Queries | MySQL | 🔄 In Progress |
| Dashboard | Power BI | ⏳ Planned |

---

## Business Context

A mid-sized product distribution company that:
- Receives orders from customers across multiple regions
- Manages inventory across multiple warehouses
- Ships orders through multiple carriers
- Sources products from multiple suppliers

---

## Data Source

Real-world public datasets used as source data:

| Dataset | Source | Used For |
|---|---|---|
| DataCo Smart Supply Chain | [Kaggle](https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis) | Orders, customers, products, shipments |
| Supply Chain Logistics Problem | [Brunel University / Figshare](https://brunel.figshare.com/articles/dataset/Supply_Chain_Logistics_Problem_Dataset/7558679) | Warehouses, freight rates, inventory |

Both datasets are flat files. Part of this project involves normalizing them into a relational schema — extracting unique entities, assigning surrogate keys, and loading into structured tables.

---
## Schema

<img width="5288" height="3452" alt="Supply Chain" src="https://github.com/user-attachments/assets/aa8c599f-7390-40e9-acd3-907af041005a" />

## OLTP Schema (Phase 1)

Normalized to 3NF. 12 tables covering the full supply chain.

<img width="3713" height="2989" alt="OLTP Schema" src="https://github.com/user-attachments/assets/7b439095-8a1b-4d3e-a6df-020873c6c09b" />

### Tables

| Table | Description |
|---|---|
| `customers` | Customer master — name, email, segment, region |
| `suppliers` | Supplier master — name, country, rating |
| `carriers` | Carrier master — name, service type |
| `warehouses` | Warehouse master — location, region, capacity |
| `categories` | Product categories and subcategories |
| `products` | Product catalog — price, cost, weight, linked to supplier and category |
| `orders` | Customer order headers — date, status, promised delivery |
| `order_items` | Order line items — quantity, unit price at time of order, discount |
| `shipments` | Shipment records — shipped/delivered dates, carrier, warehouse, cost |
| `inventory` | Current stock levels per product per warehouse |
| `supplier_orders` | Purchase orders to suppliers — dates, status |
| `supplier_order_items` | PO line items — quantity ordered vs received, unit cost |

### Key Design Decisions

- `order_items.unit_price` stores price at the time of sale, not the live product price — preserves historical accuracy
- `inventory` has a unique constraint on `(warehouse_id, product_id)` — prevents duplicate stock rows at the database level
- Supplier procurement mirrors the customer order structure: `supplier_orders` / `supplier_order_items` parallel `orders` / `order_items`

---

## Star Schema (Phase 2)

Optimized for analytical queries. Fact tables at the center, dimension tables around them.

### Fact Tables

| Fact Table | Grain | Key Measures |
|---|---|---|
| `fact_orders` | One order line item | Revenue, quantity, discount |
| `fact_shipments` | One shipment | Delivery performance, shipping cost, days to deliver |
| `fact_inventory` | One product per warehouse per day | Stock levels, reorder alerts |
| `fact_supplier_orders` | One purchase order line | Procurement cost, lead time, quantity received |

### Dimension Tables

| Dimension | SCD Type | Key Attributes |
|---|---|---|
| `dim_customer` | Type 2 | Segment, city, region |
| `dim_product` | Type 2 | Category, subcategory, supplier |
| `dim_supplier` | Type 1 | Name, country, rating |
| `dim_warehouse` | Type 1 | Location, capacity, region |
| `dim_carrier` | Type 1 | Name, service type |
| `dim_date` | Static | Year, month, quarter, week, is_weekend |

### SCD Strategy

- **Type 1** (overwrite): Used for dimensions where history is not needed — suppliers, warehouses, carriers
- **Type 2** (track history): Used for customers and products — adds `effective_from`, `effective_to`, and `is_current` columns to capture changes over time

---

## ETL Pipeline (Phase 5)

Python pipeline that extracts from the OLTP source, transforms into warehouse format, and loads into the star schema.

```
Raw CSVs (DataCo + Brunel)
        │
        ▼
  Staging Tables          ← raw load, no transformation
        │
        ▼
  Dimension Tables        ← deduplication, surrogate keys, SCD logic
        │
        ▼
  Fact Tables             ← join to dimension keys, compute measures
```

**Tech stack:** Python · Pandas · SQLAlchemy · Parquet (intermediate storage)

---

## Analytical Questions

Every SQL query and Power BI visual answers one of these business questions:

1. Which product categories generate the most revenue by region and quarter?
2. What is the on-time delivery rate by carrier and warehouse?
3. Which products are at risk of stockout across warehouses?
4. What is the average supplier lead time by supplier and product category?
5. Which customer segments have the highest order frequency and average order value?
6. What is the month-over-month revenue trend by region?
7. Which warehouses have the highest inventory turnover?
8. What percentage of orders are fulfilled within the promised delivery window?

---

## Repository Structure

```
distribution-data-warehouse/
│
├── oltp/
│   ├── schema.sql              # OLTP DDL — all 12 tables
│   └── sample_queries.sql      # Exploratory queries on raw data
│
├── warehouse/
│   ├── star_schema.sql         # Fact and dimension table DDL
│   └── staging.sql             # Staging area DDL
│
├── etl/
│   ├── extract.py              # Loads raw CSVs into staging
│   ├── transform.py            # Cleans and normalizes data
│   ├── load_dimensions.py      # Builds dimension tables with SCD logic
│   └── load_facts.py           # Builds fact tables
│
├── sql/
│   ├── q1_revenue_by_category_region.sql
│   ├── q2_ontime_delivery_by_carrier.sql
│   ├── q3_stockout_risk.sql
│   ├── q4_supplier_lead_time.sql
│   ├── q5_customer_segment_analysis.sql
│   ├── q6_mom_revenue_trend.sql
│   ├── q7_inventory_turnover.sql
│   └── q8_order_fulfillment_rate.sql
│
├── data/
│   └── README.md               # Instructions to download source datasets
│
├── dashboard/
│   └── distribution_warehouse.pbix   # Power BI dashboard (planned)
│
└── README.md
```

---

## Setup

### Prerequisites

- MySQL 8.0+
- Python 3.9+
- MySQL Workbench (recommended)
- Power BI Desktop (for dashboard phase)

### Python dependencies

```bash
pip install pandas sqlalchemy pymysql pyarrow
```

### Database setup

```bash
# Run OLTP schema
mysql -u root -p < oltp/schema.sql

# Run warehouse schema
mysql -u root -p < warehouse/star_schema.sql
mysql -u root -p < warehouse/staging.sql
```

### Data setup

Download the source datasets (see `data/README.md`) and place the CSV files in the `data/raw/` folder. Then run the ETL pipeline:

```bash
python etl/extract.py
python etl/transform.py
python etl/load_dimensions.py
python etl/load_facts.py
```

---

## Skills Demonstrated

- Relational database design (3NF normalization)
- Data warehousing concepts (star schema, SCD Type 1 and 2, fact/dimension modeling)
- ETL pipeline design and implementation in Python
- Advanced SQL (window functions, CTEs, aggregations, date logic)
- Real dataset normalization from flat files to relational tables
- Power BI dashboard design

---

## Status

This is an active portfolio project being built incrementally. Each phase is committed separately so the build progression is visible in the commit history.
