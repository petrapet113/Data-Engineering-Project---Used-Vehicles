# 🚗 Used Vehicles Data Platform

[![Databricks](https://img.shields.io/badge/Platform-Databricks-red?logo=databricks)](https://databricks.com)
[![Azure](https://img.shields.io/badge/Cloud-Azure-blue?logo=microsoftazure)](https://azure.microsoft.com)
[![Delta Lake](https://img.shields.io/badge/Storage-Delta%20Lake-00ADD8?logo=delta)](https://delta.io)
[![Python](https://img.shields.io/badge/Language-Python%203.x-yellow?logo=python)](https://python.org)
[![PySpark](https://img.shields.io/badge/Engine-Apache%20Spark-E25A1C?logo=apachespark)](https://spark.apache.org)

---

## 📝 Overview

Production-ready data pipeline for used vehicle data, built using the **Medallion Architecture** (Bronze → Silver → Gold) on Azure Databricks. The platform ingests raw vehicle listings from 4 different source systems, cleans and harmonizes the data, integrates live exchange rates, calculates vehicle reliability scores, and delivers business-ready analytics tables.

**Key highlights:**
- Processes **8.7M+ records** from 6 data sources (CSV, Parquet, Avro, JSON)
- Multi-currency support with daily exchange rate refresh (EUR, USD, GBP, AUD)
- Reliability scoring algorithm based on 8.3M service book records
- Star schema dimensional model optimized for BI/Analytics
- Fully idempotent — safe to re-run at any time

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DATA SOURCES (Azure Data Lake Gen2)                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │  Source 1  │ │  Source 2  │ │  Source 3  │ │  Source 4  │  │
│  │   (CSV)    │ │ (Parquet)  │ │   (Avro)   │ │   (JSON)   │  │
│  └──────┴─────┘ └──────┴─────┘ └──────┴─────┘ └──────┴─────┘  │
│        │              │              │              │           │
└────────┼──────────────┼──────────────┼──────────────┼───────────┘
         │              │              │              │
         └──────────────┴──────────────┴──────────────┘
                              │
                    ┌─────────┴──────────┐
                    │  🥉 BRONZE LAYER  │
                    │  Raw Data As-Is  │
                    │  (Delta Tables)  │
                    └─────────┬──────────┘
                              │
                    ┌─────────┴──────────┐
                    │  🥈 SILVER LAYER │
                    │  Cleaned & Typed │
                    │  + Currency Conv │
                    │  + Brand Mapping │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────┴─────┐ ┌────┴──────────┐ ┌────┴──────────┐
    │ 🥇 GOLD LAYER │ │ Reliability  │ │  Best Value   │
    │  Star Schema  │ │    Score     │ │   Rankings   │
    │  (Dim/Fact)   │ │   (0-100)    │ │ (Make/Model) │
    └───────────────┘ └───────────────┘ └───────────────┘
              │               │               │
              └───────────────┴───────────────┘
                              │
                    ┌─────────┴──────────┐
                    │  📊 BI / Analytics │
                    │  Dashboards &    │
                    │  Reporting        │
                    └────────────────────┘
```

---

## 📁 Project Structure

```
Data-Engineering-Project---Used-Vehicles/
│
├── README.md                          ← This file
├── notebooks/
│   ├── 01_bronze_ingestion            ← Raw data ingestion from ADLS
│   ├── 02_silver_transformations      ← Cleaning, typing, combining
│   ├── 03_gold_dimensional_model      ← Star schema & fact/dim tables
│   ├── 04_gold_reliability_score      ← Reliability scoring algorithm
│   └── 05_api_exchange_rates          ← Live currency rate integration
│
├── includes/
│   └── config                         ← Shared configuration & helpers
│
├── docs/
│   └── architecture.md                ← Detailed architecture docs
│
└── tests/
    └── data_quality_checks            ← Data quality validation
```

---

## 📚 Data Sources

| Source | Format | Records | Description |
|--------|--------|---------|-------------|
| FirstSourceSystem | CSV | 111,332 | Vehicle listings (all STRING types) |
| SecondSourceSystem | Parquet | 106,970 | Vehicle listings (typed) |
| ThirdSourceSystem | Avro | 106,970 | Vehicle listings (typed) |
| FourthSourceSystem | JSON | 106,970 | Vehicle listings (typed) |
| model_to_brand_lookup | CSV | 23,848 | Model → Brand mapping reference |
| ServiceBook | CSV | 8,312,490 | Vehicle service history |

**Total: 8,768,580 records across 6 sources**

Data originates from: AutoScout24, CarGurus, Craigslist, CarsGuide

---

## 🚀 Getting Started

### Prerequisites
- Azure Databricks workspace
- Azure Data Lake Storage Gen2 with raw data
- Storage credential configured for ADLS access
- Databricks Runtime 13.x+ (or Serverless)

### Setup
1. Clone this repository into your Databricks workspace:
   ```
   Workspace → Repos → Add Repo → paste GitHub URL
   ```
2. Run notebooks in order (01 → 02 → 03 → 04 → 05)
3. Each notebook is idempotent — safe to re-run at any time

### Configuration
All paths, catalog names, and source definitions are centralized in `includes/config`. Update values there if your environment differs.

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| Cloud | Microsoft Azure |
| Storage | Azure Data Lake Storage Gen2 |
| Compute | Azure Databricks (Serverless) |
| Processing | Apache Spark (PySpark) |
| Table Format | Delta Lake |
| Catalog | Unity Catalog |
| Orchestration | Databricks Workflows |
| Version Control | Git / GitHub |
| Secrets | Azure Key Vault |
| Architecture | Medallion (Bronze/Silver/Gold) |

---

## 📊 Pipeline Layers

### 🥉 Bronze (Raw)
- Ingests all 6 sources as-is into Delta tables
- Adds audit metadata (`_ingestion_timestamp`, `_source_path`)
- Full refresh (overwrite) strategy

### 🥈 Silver (Cleaned)
- Type casting & standardization
- Date format parsing (5+ formats)
- NULL handling & deduplication
- Brand/model mapping via lookup table
- Multi-currency price conversion (EUR, USD, GBP, AUD)
- Daily exchange rate integration via public API

### 🥇 Gold (Business-Ready)
- Star schema dimensional model
- Reliability Score (0–100) per vehicle based on service history
- Best value rankings by make/model
- Optimized for BI queries and dashboards

---

## 📄 License

A personal data engineering portfolio project.
