# 🏭 FMCG Data Engineering Pipeline — Sports Bar

A production-grade, end-to-end data engineering pipeline built on **Databricks** and **Delta Lake**, implementing the **Medallion Architecture (Bronze → Silver → Gold)** for an FMCG analytics platform. The pipeline processes customer, product, pricing, and order data from raw CSV files on Amazon S3 into clean, analytics-ready Delta tables.

---

## 📸 Architecture Overview

> _Add architecture diagram here (e.g., a flow diagram showing S3 → Bronze → Silver → Gold → Parent Catalog)_

![Architecture Diagram](./images/architecture_diagram.png)

---

## 📁 Project Structure

```
consolidated_pipeline/
│
├── 01_Setup/
│   ├── utilities.py                  # Shared schema name constants
│   └── setup_catalogs.py             # Unity Catalog & schema setup
│
├── 02_Dimensions/
│   ├── dim_date_table_creation.py    # Date dimension (monthly grain)
│   ├── 1_customer_data_processing.py # Customer dimension pipeline
│   ├── 2_products_data_processing.py # Products dimension pipeline
│   └── 3_pricing_data_processing.py  # Gross price dimension pipeline
│
└── 03_Facts/
    ├── 1_full_load_fact.py           # Full historical load for orders fact
    └── 2_incremental_load_fact.py    # Incremental daily load for orders fact
```

---

## 🗂️ Data Sources

All raw data is ingested from **Amazon S3** (`s3://sportsbar-jam/`) in CSV format.

| Source | S3 Path | Description |
|---|---|---|
| Customers | `s3://sportsbar-jam/customers/*.csv` | Customer master data |
| Products | `s3://sportsbar-jam/products/*.csv` | Product catalogue |
| Gross Price | `s3://sportsbar-jam/gross_price/*.csv` | Monthly product pricing |
| Orders | `s3://sportsbar-jam/orders/landing/` | Daily transactional orders |

---

## 🏗️ Medallion Architecture

### Bronze Layer — Raw Ingestion

Raw CSV files are ingested as-is into Delta tables with minimal transformation. File metadata (`file_name`, `file_size`, `read_timestamp`) is captured for auditability.

### Silver Layer — Cleansed & Conformed

Business logic, data quality rules, and standardisation are applied at this layer:

- **Deduplication** on natural keys
- **Null handling** and business-confirmed corrections
- **Spelling normalisation** (e.g., city names, product category typos)
- **Date parsing** across multiple input formats
- **Schema alignment** with the parent company data model

### Gold Layer — Analytics-Ready

Curated, dimension-modelled tables ready for BI consumption and merged into the parent company's unified catalog (`fmcg.gold`).

---

## 📸 Medallion Layer Screenshot

> _Add a screenshot of your Delta table explorer or data flow here_

![Medallion Layers](./images/medallion_layers.png)

---

## 🧱 Data Model

### Dimension Tables

| Table | Key Column | Description |
|---|---|---|
| `fmcg.gold.dim_customers` | `customer_code` | Enriched customer dimension with city, market, channel |
| `fmcg.gold.dim_products` | `product_code` | Product catalogue with division, category, variant |
| `fmcg.gold.dim_gross_price` | `product_code`, `year` | Latest annual price per product |
| `fmcg.gold.dim_date` | `date_key` | Monthly date dimension (2024–2025) |

### Fact Table

| Table | Grain | Description |
|---|---|---|
| `fmcg.gold.fact_orders` | Monthly × Product × Customer | Aggregated sold quantities from the Sports Bar child entity |

---

## 📸 Gold Layer Tables

> _Add a screenshot of your gold-layer tables or a sample query result here_

![Gold Tables](./images/gold_tables.png)

---

## ⚙️ Pipeline Details

### 1. Setup (`setup_catalogs.py`)

Creates the Unity Catalog `fmcg` with three schemas: `bronze`, `silver`, and `gold`. Only needs to be run once per environment.

### 2. Date Dimension (`dim_date_table_creation.py`)

Generates a monthly date spine from `2024-01-01` to `2025-12-01` with surrogate keys, year, quarter, and month name columns. Saved to `fmcg.gold.dim_date`.

### 3. Customer Pipeline (`1_customer_data_processing.py`)

**Key transformations:**
- City name standardisation (e.g., `Bengaluruu` → `Bengaluru`, `NewDelhi` → `New Delhi`)
- Business-confirmed manual city fixes for 4 customers with unresolvable city nulls
- Name title-casing and whitespace trimming
- Derived `customer` column: `CustomerName-City`
- Static attributes: `market = India`, `platform = Sports Bar`, `channel = Acquisition`
- Merged into parent `fmcg.gold.dim_customers` via Delta MERGE

### 4. Products Pipeline (`2_products_data_processing.py`)

**Key transformations:**
- Category title-casing and `Protien` → `Protein` spelling correction (via regex)
- `division` mapped from `category` (e.g., `Energy Bars` → `Nutrition Bars`)
- `variant` extracted from product name parentheses
- Deterministic `product_code` generated via SHA-256 hash of `product_name`
- Invalid `product_id` values replaced with fallback `999999`
- Merged into parent `fmcg.gold.dim_products` via Delta MERGE

### 5. Pricing Pipeline (`3_pricing_data_processing.py`)

**Key transformations:**
- Month field normalised across 4 date formats
- Negative gross prices converted to positive; non-numeric values set to 0
- Enriched with `product_code` via join to `fmcg.silver.products`
- Latest non-zero price selected per `product_code` per year using a Window function
- Merged into parent `fmcg.gold.dim_gross_price` via Delta MERGE

### 6. Orders — Full Load (`1_full_load_fact.py`)

Performs a one-time historical load of all order data from the landing zone:

- Files moved from `landing/` to `processed/` after ingestion
- Order date parsed across 4 formats (including weekday prefix stripping)
- Invalid `customer_id` values defaulted to `999999`
- Deduplication on `(order_id, date, customer_id, product_id, order_qty)`
- Joined with `fmcg.silver.products` to resolve `product_code`
- Daily orders aggregated to monthly grain before merging into `fmcg.gold.fact_orders`

### 7. Orders — Incremental Load (`2_incremental_load_fact.py`)

Handles daily incremental data with staging tables to avoid reprocessing:

- Writes to both the main `bronze.orders` table (append) and a `bronze.staging_orders` table (overwrite)
- All transformations run only on staged (new) data
- Incremental months identified; affected months fully recalculated from the gold table to ensure accuracy
- Parent `fmcg.gold.fact_orders` updated via Delta MERGE
- Staging tables (`bronze.staging_orders`, `silver.staging_orders`) dropped after load

---

## 📸 Incremental Load Flow

> _Add a diagram or screenshot showing the staging → silver → gold → parent merge flow_

![Incremental Load](./images/incremental_load_flow.png)

---

## 🔧 Technologies Used

| Technology | Purpose |
|---|---|
| **Databricks** | Compute & notebook orchestration |
| **Apache Spark (PySpark)** | Distributed data processing |
| **Delta Lake** | ACID transactions, Change Data Feed, MERGE operations |
| **Amazon S3** | Raw data landing zone and file archive |
| **Unity Catalog** | Centralised data governance across child entities |

---

## 🚀 Getting Started

### Prerequisites

- Databricks workspace with Unity Catalog enabled
- Access to the S3 bucket `s3://sportsbar-jam/`
- Cluster with Delta Lake runtime (DBR 12.0+ recommended)

### Execution Order

Run the notebooks in the following sequence for a fresh environment:

```
1. 01_Setup/setup_catalogs.py
2. 01_Setup/utilities.py          ← %run this in dependent notebooks
3. 02_Dimensions/dim_date_table_creation.py
4. 02_Dimensions/1_customer_data_processing.py
5. 02_Dimensions/2_products_data_processing.py
6. 02_Dimensions/3_pricing_data_processing.py
7. 03_Facts/1_full_load_fact.py   ← Run once for historical data
8. 03_Facts/2_incremental_load_fact.py  ← Schedule for daily runs
```

### Widgets / Parameters

All notebooks accept the following Databricks widgets:

| Widget | Default | Description |
|---|---|---|
| `catalog` | `fmcg` | Unity Catalog name |
| `data_source` | varies | Source table name (e.g., `customers`, `orders`) |

---

## 🏢 Multi-Entity Architecture

This pipeline is designed for a **parent-child data model** where multiple child entities (like Sports Bar) feed into a centralised parent catalog. Each dimension and fact table in the Sports Bar namespace is merged into the corresponding parent Gold table using Delta MERGE with upsert semantics.

```
Sports Bar (Child)              Parent Company (fmcg.gold)
─────────────────────           ──────────────────────────
sb_dim_customers       ──▶      dim_customers
sb_dim_products        ──▶      dim_products
sb_dim_gross_price     ──▶      dim_gross_price
sb_fact_orders         ──▶      fact_orders
```

---

## 📸 Parent-Child Merge

> _Add a screenshot showing the parent gold table after a successful merge from the child entity_

![Parent Merge](./images/parent_child_merge.png)

---

## 📝 Notes

- Change Data Feed (`delta.enableChangeDataFeed = true`) is enabled on all Delta tables for downstream CDC support.
- The `product_code` SHA-256 hash ensures a stable, surrogate-key-free join key that survives source system changes.
- The incremental orders pipeline recalculates entire affected months (not just new rows) to handle late-arriving data correctly.
- All schema corrections (city names, product spellings) are documented inline and confirmed with the business team.
