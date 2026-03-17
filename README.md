📌 FMCG Data Pipeline (End-to-End Data Engineering Project)
🚀 Overview

This project implements a complete data pipeline using PySpark, Delta Lake, and Databricks following the Medallion Architecture (Bronze → Silver → Gold).

The pipeline processes FMCG business data including customers, products, pricing, and orders, and builds a scalable analytics-ready data model.

🏗️ Architecture

Bronze Layer → Raw data ingestion from AWS S3

Silver Layer → Data cleaning, transformation, and standardization

Gold Layer → Business-ready fact and dimension tables

Orchestration → Databricks Workflows (DAG-based execution)

🔄 Data Flow
S3 → Bronze → Silver → Gold → Parent Tables
📊 Key Features

✅ Medallion Architecture implementation

✅ Incremental data processing (optimized pipelines)

✅ Delta Lake MERGE operations (upsert logic)

✅ Data cleaning and standardization

✅ Window functions for business logic (latest price selection)

✅ Fact & Dimension modeling (Star Schema)

✅ Orchestration using Databricks Workflows

🧠 Pipeline Components
1. Customer Pipeline

Deduplication

Data standardization

City corrections

2. Product Pipeline

Data cleaning

Category normalization

Product code generation (hashing)

3. Pricing Pipeline

Multi-format date parsing

Price validation & correction

Latest price selection using window functions

4. Orders Pipeline

Full load + Incremental load

Data cleaning and joins

Monthly aggregation

⚙️ Technologies Used

PySpark

Delta Lake

Databricks

AWS S3

SQL

🔁 Orchestration

The pipeline is orchestrated using Databricks Workflows, ensuring:

Dependency-based execution

Dimension-first processing

Reliable data consistency

📈 Output

dim_customers

dim_products

dim_gross_price

fact_orders

🚀 Future Enhancements

Airflow integration

Data quality checks

Monitoring & alerting

Dashboard (Power BI)

🧠 3. Architecture Diagram (Simple)

Create using draw.io or PowerPoint:

S3
 ↓
Bronze (Raw Tables)
 ↓
Silver (Cleaned Data)
 ↓
Gold (Fact + Dimensions)
 ↓
Databricks Workflow

👉 Export as image → upload in /architecture

📸 4. Add Your Orchestration Screenshot

![Screenshot_17-3-2026_15355_dbc-aa304915-2a15 cloud databricks com](https://github.com/user-attachments/assets/58b322a6-b2f8-4480-9c2f-4f4395e28909)
