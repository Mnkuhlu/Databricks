# Medallion Pipeline — Bronze / Silver / Gold Sales Analytics

A production-style Databricks Asset Bundle implementing a Bronze → Silver → Gold medallion
architecture on retail sales transaction data, orchestrated as a scheduled multi-task job.

## Architecture

```
raw_sales_transactions.csv
        │
        ▼
┌───────────────┐     ┌──────────────────┐     ┌────────────────────────┐
│    BRONZE     │ ──▶ │      SILVER      │ ──▶ │          GOLD          │
│ Raw ingestion │     │ Cleaned, deduped │     │ Business-level metrics │
│ (Auto Loader) │     │  star schema     │     │  (customer/product/    │
│               │     │                  │     │       sales)           │
└───────────────┘     └──────────────────┘     └────────────────────────┘
```

The pipeline is deployed and orchestrated via **Databricks Asset Bundles** (`databricks.yml` +
`resources/BronzeToGold_Pipeline.yml`) as a single job, `medallion_pipeline_job`, with five
dependent tasks running on serverless compute.

## Layers

### Bronze — `workspace.bronze.bronze_sales_transactions`
Raw ingestion from CSV via Auto Loader, with `ingestion_timestamp` and source file lineage
attached. No transformation — this layer preserves the data exactly as received, so any
downstream issue can always be traced back to the original source.

### Silver — `workspace.silver`
Cleans and conforms the bronze data into a star schema:

| Table | Description |
|---|---|
| `dim_customers` | One row per customer, deduplicated, with derived `customer_full_name` |
| `dim_products` | One row per product, with derived `profit_margin` |
| `dim_stores` | One row per store |
| `fact_transactions` | Deduplicated, validated transaction grain (nulls and invalid quantities/amounts filtered out) |

Data quality rules applied at this stage: drop duplicate `transaction_id`s, and filter out
records with null keys, negative amounts, or non-positive quantities.

### Gold — `workspace.gold`
Business-ready aggregates, split across three notebooks by domain:

**Customer analytics** (`Customer-Silver To Gold.ipynb`)
- `customer_lifetime_value` — total spend and transaction count per customer
- `customer_rfm_segmentation` — Recency/Frequency/Monetary scoring
- `customer_cohorts` — monthly signup-cohort retention and revenue
- `customer_churn_risk` — customers flagged by purchase inactivity

**Product analytics** (`Product-Silver To Gold.ipynb`)
- `product_performance` — units sold, revenue, and margin per product
- `category_performance` — rollups by category/subcategory
- `product_affinity` — market-basket "bought together" pairs
- `brand_performance` — brand-level revenue and pricing comparison
- `inventory_insights` — sales velocity signals per product

**Sales analytics** (`Sales-Silver To Gold.ipynb`)
- `daily_sales_summary` / `monthly_sales_summary` — time-series rollups
- `store_performance` — revenue and transaction volume per store
- `category_trends` — category revenue trends over time

## Orchestration

Defined in `resources/BronzeToGold_Pipeline.yml` as `medallion_pipeline_job`:

- **Schedule:** daily at 03:00 `Africa/Johannesburg` (currently `PAUSED` — flip to `UNPAUSED`
  once a manual run has been validated)
- **Compute:** serverless (no cluster configuration required)
- **Task graph:** `bronze_ingest → silver_transform → {gold_customer, gold_product, gold_sales}`
  (the three gold tasks fan out in parallel once silver completes)
- **Retries:** bronze/silver tasks retry twice on failure with a 60s backoff; gold tasks retry
  once
- **Alerting:** job failures notify `notification_email` (configured per target in `databricks.yml`)

## Project structure

```
Databricks/
├── databricks.yml                  # Bundle root config (targets: dev, prod)
├── resources/
│   └── BronzeToGold_Pipeline.yml   # Job + task definitions
├── src/
│   ├── bronze/ingest_bronze.ipynb
│   ├── silver/Bronze to Silver Ingestion.ipynb
│   ├── gold/
│   │   ├── Customer-Silver To Gold.ipynb
│   │   ├── Product-Silver To Gold.ipynb
│   │   └── Sales-Silver To Gold.ipynb
│   └── run_pipeline.py             # Local/CLI entry point (in progress)
├── tests/                          # Unit tests per layer (in progress)
├── data/raw/                       # Sample raw CSV for local development
└── config/settings.yaml            # Environment-specific settings (in progress)
```

## Getting started

**Prerequisites**
- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/index.html) v0.230+ (bundle-capable)
- Access to a Databricks workspace with Unity Catalog and serverless compute enabled

**Deploy**
```bash
databricks auth login --host <your-workspace-url>
databricks bundle validate -t dev
databricks bundle deploy -t dev
```

**Run**
```bash
databricks bundle run -t dev medallion_pipeline_job
```

## Status

This is an active portfolio project demonstrating a medallion architecture with Databricks
Asset Bundles, Auto Loader, and Delta Lake. Bronze, Silver, and Gold layers are functionally
complete; `run_pipeline.py`, unit tests, `config/settings.yaml`, and CI are still in progress.

## Author

Muzi Mngadi — Data Engineer
