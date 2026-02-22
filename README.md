# FMCG DataBricks Tutorial Project -- using Databricks (Free Edition)

An end-to-end data engineering project built on **Databricks Free Edition** and  **Amazon Web Services Free Tier** for the FMCG domain. This project simulates a real-world post-merger data consolidation scenario.

---

## 📋 Project Overview

**Atlon**, a leading sports equipment manufacturer, acquired **Sports Bar** — a fast-growing startup in the energy bars and athletic nutrition space. The two companies had vastly different data infrastructures, causing reporting chaos and misaligned metrics.

The goal of this project is to build a **reliable, scalable data pipeline** that unifies both companies' data into a single analytical layer for business reporting and dashboarding.

---

## 🛠️ Tech Stack

| Tool / Technology | Purpose |
|---|---|
| **Python / PySpark** | Data transformation and pipeline logic |
| **SQL** | Querying and table creation |
| **AWS S3** | Data lake / object store for raw files |
| **Databricks (Free Edition)** | Unified platform for ETL, orchestration, and dashboarding |
| **Medallion Architecture** | Bronze → Silver → Gold data layering |
| **Databricks Dashboard** | BI reporting |
| **Databricks Genie** | AI-powered natural language querying |

---

## 🏗️ Architecture

The project follows a **Medallion Architecture** with three layers:

- **Bronze** — Raw ingested data from S3, stored as-is with metadata columns (`read_timestamp`, `file_name`, `file_size`)
- **Silver** — Cleaned and transformed data (deduplication, null handling, type casting, standardization)
- **Gold** — BI-ready, denormalized tables with aggregations and derived columns; merged data from both companies

### Data Flow

```
Atlon (Parent)          Sports Bar (Child)
OLTP → Gold Layer       OLTP → CSV → AWS S3
  (pre-existing)                ↓
                         Bronze → Silver → Gold
                                          ↓
                         Merged into Parent Gold Layer
                                          ↓
                         Databricks Dashboard / Genie
```

---

## 📁 Data Model

The project uses a **Star Schema** centered on a fact table, with several dimension tables.

### Parent Company (Atlon) — Gold Layer

| Table | Description |
|---|---|
| `dim_customers` | Customer code, name, market, platform, channel |
| `dim_products` | Product code, division, category, product, variant |
| `gross_price` | Product code, year, price (INR) |
| `fact_orders` | Date, product code, customer code, sold quantity (monthly aggregate) |
| `dim_date` | Date key, year, month, quarter (generated via script) |

### Child Company (Sports Bar) — Staging / Merge

| Table | Description |
|---|---|
| `sb_dim_customers` | Customer ID, name, city (with city appended to name) |
| `sb_dim_products` | Product code (SHA hash), division (mapped), category, product, variant |
| `sb_gross_price` | Product code, month, gross price (latest month used as yearly price) |
| `sb_fact_orders` | Order-level daily data, rolled up to monthly before merging |

---

## 🔄 Pipeline Stages

### 1. Setup
- Create `FMCG` catalog with `bronze`, `silver`, and `gold` schemas
- Import parent company CSVs directly into the gold layer
- Generate `dim_date` table via PySpark script

### 2. AWS S3 Ingestion
- Upload child company CSV files to an S3 bucket
- Establish a Databricks external connection to S3

### 3. Dimension Data Processing (Child Company)
For each dimension table (customers, products, gross price):
- **Bronze**: Read from S3 with metadata columns, write raw data
- **Silver**: Clean and transform (dedup, trim, fix typos, handle nulls, standardize formats, type casting)
- **Gold**: Select BI-ready columns, write child gold table, then **upsert** into parent gold table

### 4. Fact Data Processing

**Full Load (Historical Backfill — July to November)**
- Read all files from S3 landing folder
- Apply transformations (date normalization, null handling, deduplication, join with product for SHA code)
- Write to bronze (append), silver (upsert), gold (upsert), then merge with parent fact table at monthly granularity
- Move processed files from `landing/` to `processed/` in S3

**Incremental Load (Daily — December onwards)**
- Read only new daily file from S3 `landing/` folder
- Create a **staging table** for just the new records
- Apply same transformations as full load
- Append to bronze, upsert to silver, upsert to child gold
- Re-aggregate all records for the current month from child gold, then upsert into parent gold
- Clean up staging tables

### 5. Denormalized View
A single flat view (`fact_orders_consolidated` or similar) joins all fact and dimension tables, making it easy to build BI dashboards without complex joins.

### 6. Orchestration
A Databricks Job pipeline runs tasks in sequence:
1. Customer dimension processing
2. Products dimension processing
3. Gross price dimension processing
4. Fact orders incremental load

Scheduled to run **daily at 11 PM** via cron syntax.

---

## 🧹 Key Data Quality Transformations

| Issue | Fix Applied |
|---|---|
| Duplicate records | `dropDuplicates()` on key columns |
| Leading/trailing spaces | `f.trim()` on string columns |
| City name typos | Manual mapping dictionary + `isin()` filter |
| Mixed case customer/category names | `f.initcap()` |
| Null city values | Business-confirmed mapping + `coalesce()` join |
| Non-uniform date formats | `coalesce(try_to_date(...), ...)` with multiple format patterns |
| Negative gross prices | Multiply by `-1`; unknown prices set to `0` |
| Invalid product IDs (non-numeric) | Replace with `9999`; create SHA surrogate key |
| Spelling errors (e.g., "protien") | `regexp_replace()` with case-insensitive flag |
| Different pricing granularity | Window function to select latest month price per year |
| Daily vs. monthly aggregation | `trunc(date, 'MM')` + `groupBy` + `sum(sold_quantity)` |

---

## 📊 Dashboard & Genie

The **Databricks Dashboard** includes:
- KPI counters: Total Revenue, Total Quantity Sold
- Filters: Year, Quarter, Month, Channel, Category
- Top 10 Products by Revenue (bar chart)
- Revenue Share by Channel (pie chart)
- Monthly revenue trend

**Databricks Genie** allows natural-language queries such as:
- *"What are the top 5 customers by sold quantity?"*
- *"Show total revenues by quarter."*
- *"What are the top products by revenue?"*

---

## 📂 Project Structure

```
consolidated_pipeline/
│
├── setup/
│   ├── setup_catalogs_and_tables.ipynb
│   └── dim_date_table_creation.ipynb
│
├── dimension_data_processing/
│   ├── customer_data_processing.ipynb
│   ├── products_data_processing.ipynb
│   └── gross_price_data_processing.ipynb
│
├── fact_data_processing/
│   ├── 1_orders_full_load.ipynb
│   └── 2_orders_incremental_load.ipynb
│
└── utilities.ipynb          ← Shared schema/config variables
```

---

## 🚀 Getting Started

1. **Sign up** for [Databricks Free Edition](https://www.databricks.com/) (no credit card required)
2. **Create an AWS account** and set up an S3 bucket with a globally unique name
3. **Upload** child company CSV files to the S3 bucket (`full_load/` and `incremental_load/` folders)
4. **Connect** Databricks to S3 via the External Connection wizard (AWS Quick Start)
5. **Run notebooks** in order: setup → dimension processing → fact full load → fact incremental load
6. **Schedule** the orchestration job for daily incremental updates
7. **Build** your dashboard using the denormalized gold view

---

## 📦 Data

The project uses two companies' data across a timeline of **January 2024 – December 2025**:

- **Parent (Atlon)**: Full load (Jan 2024 – Nov 2025) + Incremental (Dec 2025)
- **Child (Sports Bar)**: Full load (Jul – Nov 2025) + Daily incremental (Dec 2025)

---

## 🙏 Credits

Project built following the [CodeBasics](https://www.youtube.com/@codebasics) Databricks Data Engineering tutorial. Thanks to **Databricks** for providing the free edition that makes this kind of learning accessible to everyone.
