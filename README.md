# E-Commerce Data Engineering Project

An end-to-end e-commerce data engineering project built with **Databricks, PySpark, Delta Lake, and Unity Catalog**. It demonstrates how raw e-commerce CSV files can be ingested, cleaned, transformed, and organized into Bronze, Silver, and Gold layers for analytics.

## Project Overview

The project uses the **Medallion Architecture** to process dimension data (products, brands, categories, customers, and dates) and order-item fact data. It includes a separate set of notebooks for experimenting with incremental fact loading.

### Architecture

```text
CSV files in a Unity Catalog Volume
              |
              v
      Databricks + PySpark
              |
              v
     Bronze Layer (raw data)
              |
              v
   Silver Layer (cleaned data)
              |
              v
 Gold Layer (analytics-ready tables)
              |
              v
     BI / SQL Analytics
```

- **Bronze:** Reads source CSV files with explicit schemas, adds ingestion metadata, stores data as Delta tables, and archives processed files.
- **Silver:** Cleans and standardizes the data, handles duplicates and nulls, fixes inconsistent values, and converts columns to suitable data types.
- **Gold:** Joins dimensions, creates useful analytical fields, calculates order amounts, and creates tables designed for reporting and BI analysis.

## Technology Stack

- **Azure Databricks / Databricks notebooks** — development and execution environment
- **Apache Spark and PySpark** — distributed data processing and transformations
- **Delta Lake** — storage format for processed tables
- **Unity Catalog** — catalog, schema, table, and Volume organization
- **Spark SQL** — SQL-based transformations and table creation
- **CSV** — source data format

## Repository Structure

```text
e-commerce-dataEngeering/
├── README.md
└── project_ecommerce/
    ├── 1_setup/
    │   └── setup_catalog.ipynb
    ├── 2_medallion_processing_dim/
    │   ├── 1_dim_bronze.ipynb
    │   ├── 2_dim_silver.ipynb
    │   └── 3_dim_gold.ipynb
    ├── 3_medallion_processing_fact/
    │   ├── 1_fact_bronze.ipynb
    │   ├── 2_fact_silver.ipynb
    │   └── 3_fact_gold.ipynb
    └── 4_incremental_load_fact/
        ├── 1_inc_fact_bronze.ipynb
        ├── 2_inc_fact_silver.ipynb
        └── 3_inc_fact_gold.ipynb
```

## Data Processing Details

### 1. Environment Setup

The setup notebook creates the `ecommerce` catalog and the `bronze`, `silver`, and `gold` schemas.

### 2. Dimension Processing

The dimension notebooks process the following source entities:

- **Brands** — standardizes brand and category codes and trims names.
- **Categories** — removes duplicate category codes and normalizes code casing.
- **Products** — normalizes product attributes, converts measurements to numeric types, corrects selected spelling inconsistencies, and handles invalid rating counts.
- **Customers** — removes rows with missing customer IDs and fills missing phone values.
- **Calendar / Date** — converts dates, removes duplicate dates, normalizes day names, and creates calendar attributes such as date IDs, month names, quarters, weeks, and weekend flags.

The Gold dimension notebook creates analytics-ready dimension tables, including a product dimension enriched with brand/category names, a customer dimension with regional mappings, and a calendar dimension.

### 3. Fact Processing — Full Load

The full-load notebooks process order-item records:

- Read CSV files from the order-items landing Volume.
- Add source-file and ingestion timestamp metadata.
- Write data to the Bronze Delta table.
- Archive processed source files into a date-based archive folder.
- Clean duplicate order lines and normalize quantity, prices, discounts, taxes, channels, coupon codes, dates, and timestamps in Silver.
- Calculate gross amount, discount amount, net/sale amount, date ID, and coupon flag in Gold.
- Convert amounts to INR using the fixed exchange-rate mapping defined in the notebook.
- Save the analytics-ready order-item fact table.

### 4. Incremental Fact Loading

The `4_incremental_load_fact` directory contains a separate incremental-processing path. Its Bronze notebook appends incoming order-item data and enables Delta Change Data Feed (CDF). The Silver and Gold notebooks use append-mode writes and include CDF configuration.

> **Important:** This is an implementation example of incremental ingestion. Before treating the pipeline as production-ready, validate how each run identifies only new/changed records and how duplicates or updates are handled across Silver and Gold. Enabling CDF alone does not automatically make downstream processing incremental.

## Unity Catalog Objects

The notebooks use the catalog name `ecommerce` and create/use these schemas:

- `ecommerce.bronze`
- `ecommerce.silver`
- `ecommerce.gold`

Examples of tables referenced in the notebooks include:

| Layer | Example tables |
|---|---|
| Bronze | `brz_brands`, `brz_category`, `brz_products`, `brz_customers`, `brz_calendar`, `brz_order_items` |
| Silver | `slv_brands`, `slv_category`, `slv_products`, `slv_customers`, `slv_calendar`, `slv_order_items` |
| Gold | `gld_dim_products`, `gld_dim_customers`, `gld_dim_date`, `gld_fact_order_items` |

The notebooks expect source files to be available in a Unity Catalog Volume using paths similar to:

```text
/Volumes/ecommerce/source_data/raw/...
/Volumes/ecommerce/source_data/archive/<YYYY-MM-DD>/...
```

The project does not include the source CSV datasets in this repository, so these Volumes and source files must be configured in your Databricks workspace before running the notebooks.

## How to Run

1. **Open Databricks** and import the notebooks from `project_ecommerce/` into your workspace.
2. **Configure Unity Catalog permissions** and make sure you can create/use the `ecommerce` catalog and its schemas.
3. **Create or configure the source Volume** to match the paths used in the notebooks. Place the required CSV files in their expected folders, including the order-items landing folder.
4. **Run the setup notebook**: `1_setup/setup_catalog.ipynb`.
5. **Run dimension notebooks in order**:
   - `2_medallion_processing_dim/1_dim_bronze.ipynb`
   - `2_medallion_processing_dim/2_dim_silver.ipynb`
   - `2_medallion_processing_dim/3_dim_gold.ipynb`
6. **Run the full-load fact notebooks in order**:
   - `3_medallion_processing_fact/1_fact_bronze.ipynb`
   - `3_medallion_processing_fact/2_fact_silver.ipynb`
   - `3_medallion_processing_fact/3_fact_gold.ipynb`
7. For the separate incremental-load exercise, run the notebooks in `4_incremental_load_fact/` in order after reviewing its append/CDF behavior and ensuring the landing folder contains only the intended incoming files.

Notebook execution order matters because later notebooks read Delta tables created by earlier notebooks.

## Skills Demonstrated

- Designing a Medallion Architecture data pipeline
- Reading CSV files with explicit PySpark schemas
- Working with Databricks Unity Catalog and Volumes
- Writing and reading Delta tables
- Cleaning data and correcting data-quality issues
- Managing duplicate and null records
- Applying joins and reference mappings
- Creating dimension and fact tables for analytics
- Building derived measures and currency conversions
- File archival and ingestion metadata
- Exploring append-based ingestion and Delta Change Data Feed

## Notes and Limitations

- The notebook paths, catalog name, schema names, table names, and file names must match your Databricks environment.
- Exchange rates are hard-coded in the Gold fact notebooks as an example; update them and their effective date before using the values for real financial reporting.
- Some notebook cells contain saved execution outputs or errors from previous runs. Review and re-run the notebooks in your own workspace rather than assuming every saved output represents a successful current run.
- Review overwrite and append modes before re-running notebooks, especially if the tables already contain data.
- Do not commit credentials, tokens, or other secrets to the repository.

## Author

**Jyothish Mypati**

GitHub: [JyothishMypati03](https://github.com/JyothishMypati03)

---

If you find this project useful, feel free to explore the notebooks and adapt the pipeline to your own e-commerce analytics use cases.
