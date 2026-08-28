# Case Study: Building a Banking Analytics Lakehouse on Azure Databricks

## The Problem

A retail bank's operational data lives scattered across ten disconnected systems — customer records, accounts, transactions, loans, credit cards, branches, employees, fraud cases, insurance policies, and support tickets. Raw CSV exports from these systems are inconsistent, unvalidated, and impossible to query efficiently at scale. Leadership needs governed, trustworthy, dashboard-ready data to answer questions like: *Which customers are prime cross-sell targets? Which branches underperform? Where is fraud concentrated?*

I designed and built a lakehouse platform to solve this: ingesting 10 raw datasets (~17,150 records) and transforming them into a governed, analytics-ready layer using Azure Databricks, Delta Lake, and Unity Catalog, with Azure Synapse as the serving layer for BI tools.

## Approach: Medallion Architecture

I structured the pipeline around the **Bronze → Silver → Gold** medallion pattern, which separates raw ingestion from cleaning/enrichment from business aggregation — keeping each layer auditable and independently reprocessable.

**Bronze** — Raw CSVs are read with explicit, type-safe schemas and written to Delta tables unmodified, with ingestion metadata (`ingestion_timestamp`, `source_file`, `batch_id`) attached for lineage and reprocessing. No business logic is applied here, so Bronze always reflects exactly what the source systems sent.

**Silver** — This is where the real engineering happens: over 50 transformation rules across the 10 tables. Examples include deriving `age_group` and `income_bracket` from raw customer fields, standardizing inconsistent status strings (`"Active"`, `"active"`, `"ACTIVE"` → one canonical value), calculating `credit_utilization_pct` and flagging cards nearing expiry, computing loan `total_payable` and `interest_rate_category`, and parsing transaction timestamps into year/month/day/quarter/day-of-week fields for time-series analysis. In total, 75+ new derived columns were created.

**Gold** — Ten aggregated tables map directly to business questions: `customer_360` (a single view of every customer's accounts, loans, cards, and computed risk score), `fraud_analysis` (loss and confirmation-rate patterns), `branch_performance` (deposits, loans, and employee efficiency per branch), and others feeding directly into dashboards.

## Governance: Unity Catalog

Rather than leaving tables as loose files in a data lake, I registered everything in **Unity Catalog** — one catalog with `bronze`, `silver`, and `gold` schemas holding 20 governed tables. This gave the platform centralized access control, column-level lineage, and audit history, and meant analysts could query Gold tables without needing to know anything about the underlying storage paths. Access to ADLS Gen2 was granted through a Databricks **Access Connector** with a managed identity — no storage keys hardcoded anywhere in the pipeline.

## Serving Layer: Azure Synapse

To make the Gold layer consumable by standard BI tools, I connected **Azure Synapse Serverless SQL** on top of the Delta tables, exposing them as SQL views without duplicating data or provisioning a dedicated database — keeping the "one copy of the data" principle intact across the whole platform.

## Key Engineering Decisions

- **Delta Lake over plain Parquet** — ACID transactions, schema evolution, and time-travel were worth the modest overhead, especially for a pipeline that will be re-run incrementally.
- **Explicit schemas over `inferSchema`** — after an initial pass reading data with inferred types, I redefined every table with explicit `StructType` schemas. This caught type mismatches early and meaningfully improved read performance.
- **Metadata-first Bronze layer** — keeping Bronze untouched (aside from metadata columns) meant any downstream bug in the Silver logic could be fixed and replayed without re-ingesting from source.
- **Managed identity over storage keys** — using a Databricks Access Connector with RBAC-scoped permissions kept credentials out of notebooks entirely.

## Results

- **17,150** records processed end-to-end across 10 domains
- **30 Delta tables** (10 Bronze + 10 Silver + 10 Gold), all governed in Unity Catalog
- **50+ transformation rules**, **75+ derived columns**
- **10–100x** query performance improvement moving from raw CSV scans to partitioned Delta tables
- **10 Gold aggregation tables** directly powering business dashboards (customer segmentation, fraud analysis, branch performance, cross-sell targeting, and more)

## Business Impact (Illustrative Insights Surfaced)

The Gold layer surfaced several actionable patterns: 60% of customers hold only a savings account — a clear cross-sell opportunity for loans, cards, and insurance. UPI transactions make up 45% of transaction volume but have the lowest success rate (94%) of any payment channel, pointing to a reliability gap worth investing in. The top 5 branches account for 40% of total deposits, suggesting an opportunity to study and replicate what those branches are doing differently.

## What I'd Do Differently

With more time, I'd add automated data-quality checks (e.g., Great Expectations or Databricks' native constraints) between Bronze and Silver, and orchestrate the notebook sequence with Databricks Workflows instead of manual execution, to make the pipeline fully production-ready and schedulable.

---

*Stack: Azure Databricks · PySpark · Delta Lake · Azure Data Lake Storage Gen2 · Unity Catalog · Azure Synapse Analytics*
