# Data Engineering Case Study

This repository contains the end-to-end data engineering pipeline for the Case Study. The project involves ingesting raw CSV data from a public S3 bucket, building a 3-layer (Bronze-Silver-Gold) data lakehouse in Databricks, and creating a final aggregated datamart.

## Project Overview

* **Objective:** Build a robust, production-style ETL pipeline.
* **Source Data:** Two CSV files (`item.csv` and `event.csv`) hosted in a public S3 bucket.
* **Technology:** Databricks (using PySpark and Databricks SQL).
* **Architecture:** Medallion (Bronze, Silver, Gold).
* **Final Datamart:** `gold_db.top_item`, which calculates the top-viewed items per year, their rank, and the most-used platform.

## Data Sources

* **Item Data:** `https://merkle-de-interview-case-study.s3.eu-central-1.amazonaws.com/de/item.csv`
    * A product dimension table containing item attributes (name, category, price).
* **Event Data:** `https://merkle-de-interview-case-study.s3.eu-central-1.amazonaws.com/de/event.csv`
    * A large fact table of user behavioral events (e.g., `view_item`, `test_assignment`) with a complex, double-escaped JSON payload.

## Architecture: The Medallion Pipeline

This pipeline follows the Bronze-Silver-Gold architecture:

### 1. Bronze Layer (`bronze_db`)

* **Purpose:** Raw, immutable data ingestion.
* **Tables:** `item`, `event`.
* **Process:** Data is read from the source CSVs using the `multiLine` and `escape` options to correctly handle the malformed JSON. All columns are intentionally left as `STRING` to preserve the raw data.

### 2. Silver Layer (`silver_db`)

* **Purpose:** Cleansed, conformed, and query-ready data.
* **Tables:** `item`, `event`.
* **Process:**
    * **`item` table:** Casts `id` and `price` to correct numeric types, handles `NULL` values.
    * **`event` table:** Casts `user_id` and timestamps. The core logic is flattening the `event_payload` JSON string into proper columns (`event_name`, `platform`, `parameter_name`, `parameter_value`). This table is partitioned by `event_year` for performance.

### 3. Gold Layer (`gold_db`)

* **Purpose:** Business-level aggregated datamarts.
* **Tables:** `top_item`.
* **Process:** This table joins the Silver `item` and `event` tables to calculate the final metrics as requested:
    1.  `total_views`: A count of `view_item` events per item/year.
    2.  `most_used_platform`: The top platform (e.g., "web", "android") for each item/year.
    3.  `item_rank_in_year`: A `RANK()` window function to rank items by `total_views` within each year.

## Key Data Discoveries

Our initial data exploration revealed several critical insights:

* **A/B Testing Framework:** The data includes a full A/B testing framework, identified by the `test_assignment` and `test_id` events.
* **3 PM Server Spike:** A massive spike in `server`-platform events at 3 PM was identified. This is not user traffic, but rather the daily **A/B test assignment job** running.
* **User Discovery Funnels:** The `referrer` parameter for `view_item` events showed that the primary user funnels are (in order):
    1.  **Home Feed (`home`):** Algorithmic discovery.
    2.  **Shopping Cart (`shopping_cart`):** A high-intent "review and cross-sell" moment.
    3.  **Search (`search`):** Active, high-intent discovery.

## How to Run This Pipeline

This project is built as a Databricks Notebook.

### Prerequisites

1.  A Databricks workspace.
2.  A Databricks cluster (e.g., a "Single User" cluster).
3.  Your writable Unity Catalog is named `workspace`.

### Instructions

1.  **Create Volume:** In your Databricks workspace, go to **Data > `workspace` > `default`** and create a **Managed Volume** named `merkle_landing_zone`.
2.  **Run ETL Pipeline:** Run the `COMMAND` blocks of the script cell by cell. They will copy the files from S3 into your new Volume, execute the Bronze, Silver, and Gold transformations.
