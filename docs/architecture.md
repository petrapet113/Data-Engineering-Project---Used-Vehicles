# 🏗️ Architecture Documentation

## Used Vehicles Data Platform — Technical Architecture

---

## 1. Overview

This platform implements the **Medallion Architecture** (Bronze → Silver → Gold) to progressively refine raw vehicle data into business-ready analytics tables.

### Design Principles
- **Idempotency** — Every notebook can be re-run safely without side effects
- **Separation of concerns** — Each layer has a single responsibility
- **Auditability** — Every record carries metadata about its origin and transformation
- **Schema enforcement** — Delta Lake ensures data quality at the storage level

---

## 2. Data Flow

```
Azure Data Lake (ADLS Gen2)
    │
    │  [01_bronze_ingestion]
    │  Reads: CSV, Parquet, Avro, JSON
    │  Adds: _ingestion_timestamp, _source_path
    ▼
BRONZE LAYER (hive_metastore.bronze.*)
    │
    │  [02_silver_transformations]
    │  Actions: Type casting, date parsing, NULL handling,
    │           dedup, union all sources, brand mapping,
    │           currency conversion
    ▼
SILVER LAYER (hive_metastore.silver.*)
    │
    │  [03_gold_dimensional_model]
    │  [04_gold_reliability_score]
    │  Actions: Star schema, fact/dim tables,
    │           reliability algorithm, rankings
    ▼
GOLD LAYER (hive_metastore.gold.*)
    │
    ▼
BI / Analytics / Dashboards
```

---

## 3. Layer Details

### 3.1 Bronze Layer

**Purpose:** Store raw data exactly as received, with minimal transformation.

**Strategy:** Full overwrite (CTAS) on each run.

**Tables:**

| Table | Source Format | Rows | Notes |
|-------|--------------|------|-------|
| `bronze.first_source_system` | CSV | 111,332 | All columns STRING |
| `bronze.second_source_system` | Parquet | 106,970 | Typed columns |
| `bronze.third_source_system` | Avro | 106,970 | Typed columns |
| `bronze.fourth_source_system` | JSON | 106,970 | Typed columns |
| `bronze.model_to_brand_lookup` | CSV | 23,848 | Reference data |
| `bronze.service_book` | CSV | 8,312,490 | Service history |

**Schema (Source Systems 1-4):**

| Column | Type (Source 1) | Type (Sources 2-4) | Description |
|--------|----------------|--------------------|--------------|
| id | STRING | STRING | Unique listing ID |
| url | STRING | STRING | Original listing URL |
| region | STRING | STRING | Geographic region |
| price | STRING | DOUBLE | Listing price |
| year | STRING | DOUBLE | Vehicle year |
| model | STRING | STRING | Vehicle model |
| condition | STRING | STRING | Vehicle condition |
| cylinders | STRING | STRING | Engine cylinders |
| fuel | STRING | STRING | Fuel type |
| odometer | STRING | DOUBLE | Mileage |
| title_status | STRING | STRING | Title status |
| transmission | STRING | STRING | Transmission type |
| VIN | STRING | STRING | Vehicle ID Number |
| drive | STRING | STRING | Drive type |
| size | STRING | STRING | Vehicle size |
| type | STRING | STRING | Vehicle type |
| paint_color | STRING | STRING | Color |
| state | STRING | STRING | US state |
| lat | STRING | DOUBLE | Latitude |
| long | STRING | DOUBLE | Longitude |
| posting_date | STRING | STRING | Date posted (5+ formats!) |
| source_system_id | STRING | STRING | Source system number |
| source_system_name | STRING | STRING | Source system name |
| currency | STRING | STRING | Price currency |
| SourceSystemId | STRING | STRING | UUID identifier |

**Metadata Columns (added during ingestion):**
| Column | Type | Description |
|--------|------|-------------|
| _ingestion_timestamp | TIMESTAMP | When data was ingested |
| _source_path | STRING | ADLS path of source file |

---

### 3.2 Silver Layer

**Purpose:** Clean, standardize, and combine all source data into a single unified dataset.

**Transformations:**
1. **Type casting** — Convert STRING columns from Source 1 to proper types (DOUBLE, INT)
2. **Date parsing** — Handle 5+ date formats in `posting_date`:
   - `2024-02-12` (ISO)
   - `21.01.2022` (European)
   - `02/07/2021` (US)
   - `November 14, 2020` (Full month name)
   - `March 30, 2020` (Full month name)
3. **NULL handling** — Strategy per column (drop, fill, flag)
4. **Deduplication** — Remove exact duplicates based on ID
5. **Union** — Combine all 4 source systems into one table
6. **Brand mapping** — Join with lookup table to add brand column
7. **Currency conversion** — Add columns for GBP, USD, AUD prices
8. **Exchange rates** — Daily rates from public API

**Tables:**
| Table | Description |
|-------|-------------|
| `silver.vehicles` | Unified, cleaned vehicle listings |
| `silver.exchange_rates` | Daily exchange rate history |

---

### 3.3 Gold Layer

**Purpose:** Business-ready tables optimized for analytics and reporting.

**Dimensional Model (Star Schema):**

```
                    ┌─────────────────┐
                    │  dim_vehicle     │
                    │  (make, model,   │
                    │   type, fuel...) │
                    └────────┬────────┘
                             │
┌─────────────────┐  ┌──────┴────────┐  ┌─────────────────┐
│  dim_location    │  │ fact_listings  │  │  dim_source       │
│  (state, region, │──│ (price, date, │──│  (source_name,   │
│   lat, long)     │  │  FK_vehicle,  │  │   currency)      │
└─────────────────┘  │  FK_location, │  └─────────────────┘
                    │  FK_source)   │
                    └────────┬───────┘
                             │
                    ┌────────┴────────┐
                    │  dim_date        │
                    │  (year, month,   │
                    │   quarter, day)  │
                    └─────────────────┘
```

**Reliability Score (0–100):**
- Based on ServiceBook analysis (8.3M records)
- Highest scores: vehicles with only regular maintenance
- Lowest scores: vehicles with frequent non-maintenance issues
- Aggregated per make/model for rankings

**Tables:**
| Table | Description |
|-------|-------------|
| `gold.fact_listings` | Central fact table with measures |
| `gold.dim_vehicle` | Vehicle attributes dimension |
| `gold.dim_location` | Geographic dimension |
| `gold.dim_source` | Source system dimension |
| `gold.dim_date` | Calendar dimension |
| `gold.reliability_scores` | Per-vehicle reliability metrics |
| `gold.best_value_rankings` | Best car for price by make/model |

---

## 4. Storage Architecture

| Layer | Catalog | Schema | Storage |
|-------|---------|--------|---------|
| Bronze | hive_metastore | bronze | DBFS (managed) |
| Silver | hive_metastore | silver | DBFS (managed) |
| Gold | hive_metastore | gold | DBFS (managed) |
| Mirror | used_vehicle_databricks_ws | bronze/silver/gold | ADLS (external) |

**Why dual storage?**
- `hive_metastore` — Reliable, always works, independent of Azure subscription
- `used_vehicle_databricks_ws` — Visible in Unity Catalog Explorer UI

---

## 5. Data Quality Rules

| Rule | Applied In | Action |
|------|-----------|--------|
| Valid year (1900–2026) | Silver | Filter out invalid |
| Non-negative price | Silver | Filter out |
| Valid coordinates | Silver | NULL if out of range |
| Non-null ID | Bronze/Silver | Required |
| Deduplicated IDs | Silver | Keep first occurrence |
| Valid currency code | Silver | Must be in (EUR, USD, GBP, AUD) |

---

## 6. Refresh Strategy

| Notebook | Frequency | Strategy |
|----------|-----------|----------|
| 01_bronze_ingestion | On-demand | Full overwrite |
| 02_silver_transformations | Daily | Full rebuild |
| 05_api_exchange_rates | Daily | Append new rates |
| 03_gold_dimensional_model | Daily (after silver) | Full rebuild |
| 04_gold_reliability_score | Weekly | Full rebuild |
