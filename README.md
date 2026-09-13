# Azure Data Engineering Project — Olist E-Commerce Pipeline

An end-to-end enterprise data engineering platform on **Microsoft Azure**, implementing a **Medallion Architecture** (Bronze → Silver → Gold) using **Azure Data Factory**, **Azure Databricks**, **ADLS Gen2**, **MongoDB**, and **Synapse Analytics** — built on the Olist Brazilian e-commerce dataset.

---

## 🏗️ Architecture

![Architecture Diagram](ss/Screenshot%202026-09-05%20221907.png)

**Flow:**
1. **Data Sources** — Raw CSVs pulled via HTTP from GitHub + a relational **SQL table**.
2. **Data Ingestion** — Azure Data Factory orchestrates ingestion into ADLS Gen2.
3. **Raw Data (Bronze)** — Landed in ADLS Gen2 as-is.
4. **Data Transformation** — Azure Databricks cleans, joins, and enriches the data, pulling in a lookup/enrichment table from a **NoSQL (MongoDB) database**.
5. **Transformed Data (Silver)** — Written back to ADLS Gen2 in Parquet format.
6. **Serving (Gold)** — Azure Synapse Analytics (serverless SQL pool) exposes curated views on top of the Silver layer.
7. **Visualization** — Power BI, Tableau, and Microsoft Fabric consume the Gold layer for reporting.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | Azure Data Factory |
| Storage | Azure Data Lake Storage Gen2 |
| Compute / Transformation | Azure Databricks (PySpark) |
| Enrichment Store | MongoDB (NoSQL) |
| Relational Source | MySQL |
| Serving Layer | Azure Synapse Analytics (Serverless SQL Pool) |
| Security | Azure AD App Registration (Service Principal), IAM RBAC |
| Visualization | Power BI / Tableau / Microsoft Fabric |

---

## 📊 Dataset

The pipeline uses the **Olist Brazilian E-Commerce** public dataset, made up of 8 relational CSV files:

- `olist_customers_dataset`
- `olist_geolocation_dataset`
- `olist_order_items_dataset`
- `olist_order_payments_dataset`
- `olist_order_reviews_dataset`
- `olist_orders_dataset`
- `olist_products_dataset`
- `olist_sellers_dataset`

### Entity Relationship Diagram

![ER Diagram](ss/Screenshot%202026-09-10%20202029.png)

---

## 1️⃣ Ingestion — Azure Data Factory & App Registration

Data ingestion logic (SQL → ADLS Gen2, and MongoDB enrichment setup) lives in:
📓 [`DataIngestionToSql&MongoDB.ipynb`](DataIngestionToSql&MongoDB.ipynb)

A **parametrized ADF pipeline** dynamically loops through source tables using `Lookup` → `ForEach` → `Copy Data` activities, pulling data from the SQL source into ADLS Gen2.

![ADF Pipeline Run](ss/Screenshot%202026-09-07%20182406.png)

### Bronze Layer in ADLS Gen2

All 8 raw datasets landed in the `olist-data/bronze` container.

![Bronze Layer](ss/Screenshot%202026-09-06%20154313.png)

### Security — Service Principal & Access Control

An **Azure AD App Registration** authenticates Databricks against ADLS Gen2 via OAuth (Service Principal).

![App Registration](ss/Screenshot%202026-09-08%20151852.png)

Access is scoped through **IAM role assignments** — the app registration and Databricks managed identity are granted `Storage Blob Data Contributor` on the storage account.

![Storage IAM Roles](ss/Screenshot%202026-09-12%20190608.png)

> ⚠️ **Security note:** Never hardcode client secrets in notebooks or commit them to source control. Use **Azure Key Vault** and reference secrets via `dbutils.secrets.get()` instead. All credentials used in this project have since been rotated and the underlying Azure resources have been decommissioned.

---

## 2️⃣ Transformation — Azure Databricks

Transformation logic lives in:
📓 [`DataBricks Code For Transformation.ipynb`](DataBricks%20Code%20For%20Transformation.ipynb)

**Workflow:**

![Databricks Workflow](ss/Screenshot%202026-09-08%20140928.png)

1. Read raw data from ADLS Gen2
2. Perform basic transformations — cleaning, renaming, and filtering
3. Join multiple datasets together
4. Enrich data via a MongoDB lookup table
5. Perform aggregations and derive insights
6. Write the final dataset back to ADLS Gen2 (Silver layer)

### Final Joined DataFrame

Output of the join across orders, customers, products, sellers, and order items:

![Final DataFrame](ss/Screenshot%202026-09-10%20214447.png)

### Exploratory Visualization

In-notebook visualization of delivery delay time by payment type:

![Delay Time Visualization](ss/Screenshot%202026-09-10%20214430.png)

### Enrichment Sources (MySQL + MongoDB)

The relational source table and the NoSQL enrichment table are hosted on managed cloud database instances.

![Shared Databases](ss/Screenshot%202026-09-12%20200554.png)

---

## 3️⃣ Serving — Azure Synapse Analytics

SQL scripts for the Gold-layer views are in [`sql_scripts/`](sql_scripts/). Synapse's **serverless SQL pool** reads the Silver-layer Parquet files directly via `OPENROWSET`, and curated views are built on top for consumption.

### Workspace Overview

![Synapse Workspace](ss/Screenshot%202026-09-12%20173902.png)

### Querying the Silver Layer

```sql
SELECT *
FROM
    OPENROWSET(
