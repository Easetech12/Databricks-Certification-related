In Databricks, a Materialized View (MV) is a managed database object that allows you to precompute and save the results of a query to a physical table on disk [1].
However, unlike traditional databases (where you have to manually write scripts to manage and refresh the data), Databricks Materialized Views are smart, declarative, and managed automatically by Delta Live Tables (DLT) and Unity Catalog [1].

Key Characteristics in Databricks

1. Declarative Management: You only define what data you want using a SQL query. Databricks takes care of creating the tables, scheduling the updates, and tracking data dependencies [1].
2. Incremental Refreshing: Where traditional MVs often recalculate the entire dataset from scratch, Databricks leverages the Delta Lake transaction log to perform incremental updates whenever possible [1]. It only processes the new or changed data, saving immense amounts of compute costs.
3. Unity Catalog Integration: They are fully governed assets [1]. You can apply row/column-level security, track data lineage, and share them across your workspace [1].

💻 Code Example

To create a Materialized View in Databricks, you must execute the query within a SQL warehouse (using Pro or Serverless tiers) or as part of a Delta Live Tables pipeline [1].
sql
CREATE MATERIALIZED VIEW sales_workspace.mv_daily_category_revenue
AS
SELECT
    order_date,
    category,
    SUM(price * quantity) AS total_revenue,
    COUNT(order_id) AS transaction_count
FROM sales_workspace.raw_transactions  -- Base table
GROUP BY order_date, category;
Use code with caution.

How to Refresh It:

By default, you can trigger a refresh manually:
sql
REFRESH MATERIALIZED VIEW sales_workspace.mv_daily_category_revenue;
Use code with caution.
Alternatively, you can schedule it to refresh automatically directly in the definition:
sql
CREATE MATERIALIZED VIEW sales_workspace.mv_daily_category_revenue
SCHEDULE CRON '0 0 * * * ?' -- Refreshes every day at midnight
AS ...
Use code with caution.

🏃‍♂️ The Dry Run (How Databricks Handles It Under the Hood)

Let's do a dry run tracking how Databricks optimizes this pipeline.

Step 1: The Base Table (raw_transactions)

You have an upstream Delta table tracking raw sales data streaming into your lakehouse. It contains 500 million rows.

Step 2: Creation of the MV

When you run CREATE MATERIALIZED VIEW, Databricks compiles the query, maps out the data lineage in Unity Catalog, runs the aggregation over the 500 million rows, and writes a highly compressed summary Delta table to your cloud storage.

Step 3: New Data Arrives

It's Black Friday, and 10 million new rows pour into raw_transactions.
• If a analyst runs a query against mv_daily_category_revenue, they see the old data. The view is technically stale until a refresh occurs.

Step 4: The Intelligent Refresh Execution

An automated cron job triggers REFRESH MATERIALIZED VIEW.
• What a traditional database does: Scans 510 million rows, drops the old view table, and rebuilds it. (High compute cost).
• What Databricks does: The Delta Live Tables engine inspects the transaction log of raw_transactions. It realizes 500 million rows haven't changed. It only reads the 10 million new rows, computes their aggregates, and merges ("appends/updates") them directly into the physical Materialized View table.

Step 5: The Read

Your BI tools (like Power BI or Tableau) query the Materialized View. It responds in milliseconds because it reads a small, perfectly indexed table, without ever touching the raw 510-million-row dataset.

⚖️ Databricks Materialized Views vs. Streaming Tables

Databricks introduces two closely related objects. It's crucial to understand when to use which:

Feature	Materialized View (MV)	Streaming Table (ST)
Primary Use Case	Aggregations & Analytics (e.g., SUM, COUNT, GROUP BY, JOIN) [1].	Data Ingestion & Appending (e.g., pulling raw JSON files from cloud storage).
How it treats source data	Treats the source as a static snapshot at the moment of refresh [1].	Treats the source as an infinite stream of append-only data.
Data Correction	Automatically handles upstream updates, changes, and overwrites [1].	Expects append-only data. Changes upstream require a full pipeline refresh.

Are you planning to build these views using Databricks SQL Warehouses or inside a Delta Live Tables (DLT) pipeline? I can show you how to configure the cluster settings or handle data quality expectations (expectations syntax) depending on your choice.
