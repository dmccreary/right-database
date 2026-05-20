---
title: Chapter 4 - Analytical Databases
description: Columnar storage, massively parallel processing, star and snowflake schemas, ETL pipelines, Parquet, Inmon vs. Kimball, and modern cloud data warehouses — Snowflake and BigQuery.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 4: Analytical Databases

## Summary

This chapter covers the architectural patterns that make databases fast for analytical (OLAP) workloads: columnar storage, massively parallel processing, and data warehouse design. Students learn the difference between the Inmon and Kimball architectural philosophies, how star and snowflake schemas optimize analytical queries, and how modern cloud data warehouses such as Snowflake and BigQuery implement these ideas at scale. ETL pipelines and the Apache Parquet file format are examined as the data ingestion backbone of analytical systems.

## Concepts Covered

This chapter covers the following 15 concepts from the learning graph:

1. Columnar Storage
2. Massively Parallel Processing
3. Data Warehouse
4. Star Schema
5. Snowflake Schema
6. OLAP Cube
7. Apache Parquet Format
8. ETL Pipeline
9. Inmon Architecture
10. Kimball Architecture
11. Bitmap Index
12. Materialized View
13. Query Pushdown
14. Snowflake Database
15. BigQuery

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)
- [Chapter 3: Relational Databases](../03-relational-databases/index.md)

---

!!! mascot-welcome "Welcome to Chapter 4!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Analytical databases are optimized for a completely different adversary than the one relational databases fight. Relational databases win at many small, fast reads and writes on individual rows. Analytical databases win at a few very large reads across millions of rows, aggregating and summarizing as they go. The architectural choices that produce those wins — columnar storage, MPP, and schema denormalization — are the opposite of what Chapter 3 taught you was best practice. That tension is the point.

## Why OLTP and OLAP Demand Different Architectures

Before examining analytical databases, two terms need precise definitions. An **OLTP (Online Transaction Processing)** workload consists of many short, concurrent transactions that read or write a small number of rows at a time. Point-of-sale transactions, user account updates, order placements — these are OLTP workloads. A relational database with row-oriented storage and B-Tree indexes handles them well.

An **OLAP (Online Analytical Processing)** workload consists of complex queries that scan large fractions of a dataset to compute aggregates, trends, and summaries. "Total revenue by product category by quarter across all 50 million orders" is an OLAP query. Running that query on an OLTP row-oriented database would read every column of every order row, even though it only needs `category`, `amount`, and `order_date` — columns that represent perhaps 5% of the row's data.

**Data warehouse** is the term for a database system designed explicitly for OLAP workloads. A data warehouse consolidates data from multiple operational systems (OLTP databases, SaaS applications, event streams) into a single queryable repository, typically refreshed periodically via ETL pipelines. The design philosophy is read-optimized at the cost of write throughput and storage efficiency.

## Columnar Storage: The Core Analytical Innovation

In a **row-oriented** storage engine (like PostgreSQL's heap), all values for a given row are stored contiguously on disk. Reading a single column requires scanning past all other columns in every row. For an OLAP query that reads 3 of 100 columns across 50 million rows, row storage performs 97% wasted I/O.

**Columnar storage** inverts this layout: all values for a given column are stored contiguously, with each column in its own storage file or segment. Reading 3 of 100 columns requires touching only 3 files — 97% of I/O is eliminated. This is the primary reason analytical databases are orders of magnitude faster than row-oriented databases for aggregation queries.

Columnar storage also compresses extremely well. Because all values in a column share the same data type — and often have low cardinality (many repeated values) — compression algorithms like run-length encoding and dictionary encoding typically achieve 5–10x size reduction. Smaller data means more data fits in memory and cache, further accelerating queries.

#### Diagram: Row vs. Columnar Storage I/O Comparison

<details markdown="1">
<summary>Interactive Row vs. Columnar Storage I/O Visualizer</summary>
Type: MicroSim
**sim-id:** row-vs-columnar-storage<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Explain how columnar storage reduces I/O for analytical queries by reading only the columns a query needs. (Bloom L2: Understand)

**Canvas:** 780px wide × 480px tall. CANVAS_HEIGHT: 480.

**Description:**
Two side-by-side panels: "Row Storage" (left) and "Columnar Storage" (right). Each panel shows a grid representing 6 rows and 6 columns (order_id, customer_id, product, amount, category, date).

The user selects which columns to query via checkboxes at the top (default: amount, category, date — 3 of 6 checked). Clicking "Run Query" animates a scan highlight across both panels:

- **Row Storage panel:** Highlight sweeps across all columns of every row (orange). A counter shows "Bytes read: 6 × 6 = 36 cells."
- **Columnar Storage panel:** Highlight touches only the checked columns (green). Counter shows "Bytes read: 3 × 6 = 18 cells."

A "Savings" indicator at the bottom shows the percentage reduction: "50% I/O saved."

When fewer columns are checked, the savings increase. With 1 column checked: "83% I/O saved."

Each cell is color-coded by column: order_id = gray, customer_id = blue, product = teal, amount = gold, category = orange, date = purple.

**Responsive:** Redraws on window resize.
</details>

## Massively Parallel Processing

A single server, no matter how large, eventually runs out of CPU cores and memory for the most demanding analytical queries. **Massively Parallel Processing (MPP)** addresses this by distributing both data and query execution across a cluster of nodes, each processing its local data slice in parallel.

In an MPP architecture:

1. Data is distributed across nodes using a hash or range partition on a **distribution key** (often a high-cardinality column like customer_id).
2. When a query arrives at the **leader node**, it decomposes the query into a parallel execution plan.
3. Each **compute node** executes its portion of the plan against its local data slice.
4. Intermediate results are shuffled between nodes as needed (for joins across distribution keys).
5. The leader collects and merges final results.

The practical result is that a 100-node MPP cluster can scan 100x more data per second than a single node, making queries that would take hours on a single machine complete in minutes.

## Data Warehouse Design Philosophies

Two schools of thought have dominated data warehouse design for three decades. Both start from the same observation — that analytical schemas should be organized differently from OLTP schemas — but reach different conclusions about how.

### The Inmon Architecture

**Bill Inmon's architecture** (the "top-down" approach) builds a central, enterprise-wide data warehouse first. This warehouse is normalized (typically 3NF) and serves as the single source of truth for all enterprise data. Downstream **data marts** — subject-specific subsets — are derived from the central warehouse and may use denormalized schemas optimized for specific business domains. The central warehouse is authoritative; data marts are projections.

Inmon's strength is consistency: every data mart draws from one authoritative source, so reporting discrepancies between departments are eliminated. Its weakness is time-to-value: building a fully normalized enterprise warehouse before any reports can be produced takes months or years.

### The Kimball Architecture

**Ralph Kimball's architecture** (the "bottom-up" approach) builds dimensional models directly. Each project starts by identifying the **business process** being analyzed (e.g., sales, inventory, customer service), then designing a **star schema** or **snowflake schema** optimized for that process. Multiple dimensional models are joined through **conformed dimensions** — shared dimension tables (like a common date or customer dimension) that make cross-process analysis possible.

Kimball's strength is speed: a single business process can be modeled and delivered in weeks. Its weakness is that inconsistencies can emerge if conformed dimensions are not rigorously maintained across projects.

| Dimension | Inmon | Kimball |
|-----------|-------|---------|
| Schema style | Normalized (3NF) | Denormalized (star/snowflake) |
| Approach | Top-down | Bottom-up |
| First deliverable | Enterprise warehouse | Dimensional model for one process |
| Time to first value | Months to years | Weeks to months |
| Risk of inconsistency | Low (single source) | Moderate (conformed dimensions required) |
| Suitable for | Enterprises with long planning horizons | Agile teams, specific domain focus |

## Star and Snowflake Schemas

Dimensional modeling — whether following Kimball or a hybrid approach — produces two canonical schema patterns. Both center on a **fact table** surrounded by **dimension tables**. The fact table contains the quantitative measurements (sales amounts, page views, sensor readings) and foreign keys to the dimension tables. Dimension tables contain descriptive attributes (customer name, product category, date details) that provide context for analysis.

Before examining the schemas, define three terms you'll need: a **fact** is a measurable business event (a sale); a **dimension** is a descriptive context for that fact (who bought it, what was purchased, when); and **grain** is the level of detail at which facts are recorded (one row per individual line item vs. one row per order).

A **star schema** has the fact table at the center with one layer of denormalized dimension tables radiating outward. The geometry resembles a star. Queries are simple: join the fact table to each needed dimension. The cost is redundancy — a customer's city and region are stored in every row of the customer dimension.

A **snowflake schema** normalizes the dimension tables, splitting them into multiple layers. The customer dimension might reference a separate city dimension, which references a region dimension. This reduces redundancy at the cost of more joins in every query. In practice, most modern analytical databases favor star schemas because the storage savings from snowflaking are less significant than they were when disk was expensive, and query planners handle the additional joins efficiently.

#### Diagram: Star vs. Snowflake Schema Explorer

<details markdown="1">
<summary>Interactive Star vs. Snowflake Schema Explorer</summary>
Type: MicroSim
**sim-id:** star-vs-snowflake-schema<br/>
**Library:** vis-network<br/>
**Status:** Specified

**Learning Objective:** Distinguish between star and snowflake schema structures and explain the tradeoff between query simplicity and storage efficiency. (Bloom L4: Analyze)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
Two toggle-able views: "Star Schema" and "Snowflake Schema" (controlled by a button or tab).

**Star Schema view:** Central fact node (`sales_fact`) colored `#4682B4` (blue). Four dimension nodes radiating outward: `dim_date`, `dim_customer`, `dim_product`, `dim_store` — all colored `#E65100` (orange). Edges labeled with the FK relationship. Each dimension node is large (denormalized, showing 6–8 attribute labels inside the node or as a tooltip).

**Snowflake Schema view:** The same fact node, but `dim_customer` now branches to `dim_city` (which branches to `dim_region`), and `dim_product` branches to `dim_category`. New nodes colored `#28a745` (green) to indicate normalized child dimensions.

**Interactions:**
- Hovering a node shows a tooltip listing its columns (e.g., "dim_customer: customer_id, name, email, city, region, country").
- Clicking any dimension node opens a sidebar: "This dimension has 8 attributes. In the snowflake schema, city and region are split out, saving storage but requiring 2 additional joins."
- Clicking the fact node shows: "sales_fact grain: one row per line item. 50M rows. FK columns: date_key, customer_key, product_key, store_key. Measures: quantity, unit_price, discount, net_amount."
- A "Run Sample Query" button highlights the path traversed to answer "Total revenue by product category by quarter" — highlighting which nodes and edges are joined.

**Node conventions:** Fact = `#4682B4`, Dimension = `#E65100`, Normalized sub-dimension = `#28a745`.

**Responsive:** Redraws on window resize.
</details>

## OLAP Cubes and Materialized Views

An **OLAP cube** is a pre-aggregated, multi-dimensional data structure that stores summary values at every combination of dimension values. A sales cube with dimensions (product, region, quarter) pre-computes the total revenue for every product × region × quarter combination. Query response times drop to milliseconds because no aggregation is needed at query time — the answer is already stored.

The cost is pre-computation time and storage: computing all combinations across many dimensions is expensive and can produce enormous data volumes (the "combinatorial explosion" problem). Modern cloud warehouses have largely replaced traditional OLAP cubes with **materialized views** — pre-computed query results stored as tables and refreshed on a schedule or on-demand. Materialized views are more flexible than fixed-schema cubes and integrate naturally with SQL.

**Query pushdown** is a complementary optimization: when a query involves filtering or aggregation that can be evaluated before data is transferred between storage and compute (or between nodes in an MPP cluster), the engine "pushes" those operations down to where the data lives. A `WHERE category = 'Electronics'` predicate pushed down to columnar storage means only rows matching that predicate are decompressed and returned — dramatically reducing data movement.

## ETL Pipelines and the Apache Parquet Format

Data does not arrive in a data warehouse ready to query. **ETL (Extract, Transform, Load)** pipelines extract data from source systems (OLTP databases, SaaS APIs, event streams), transform it (clean, join, normalize, cast types, apply business rules), and load it into the warehouse. Modern architectures often invert this to **ELT** — extract and load raw data first, then transform using the warehouse's own compute power.

Two terms in the Transform phase need definitions before examining the pipeline: **data cleansing** removes or corrects malformed records (nulls where they shouldn't be, out-of-range values, encoding errors); **schema mapping** converts source data types and column names into the target warehouse schema.

**Apache Parquet** is the de-facto standard file format for analytical data at rest. It is an open-source, columnar file format with the following properties:

- **Columnar layout:** Data is stored by column, enabling column pruning (skip unneeded columns at read time).
- **Nested data support:** Parquet represents nested records (arrays and maps within rows) using Dremel-style encoding — a significant advantage for semi-structured source data.
- **Built-in compression:** Parquet files are typically compressed with Snappy or Zstandard, achieving 5–10x size reduction over raw CSV.
- **Predicate pushdown metadata:** Row group statistics (min/max per column per chunk) allow engines to skip entire row groups that don't match query predicates.

Parquet files are the lingua franca between tools: Apache Spark produces Parquet files, Snowflake and BigQuery read them natively from object storage, and data catalogs like Apache Iceberg manage Parquet file collections as queryable tables.

#### Diagram: ETL Pipeline Architecture

<details markdown="1">
<summary>Interactive ETL Pipeline Architecture Explorer</summary>
Type: MicroSim
**sim-id:** etl-pipeline-explorer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify the stages of an ETL/ELT pipeline and explain what happens to data at each stage. (Bloom L2–L3: Understand/Apply)

**Canvas:** 780px wide × 400px tall. CANVAS_HEIGHT: 400.

**Description:**
A horizontal pipeline flow diagram with 5 stages: Source Systems → Extract → Transform → Load → Data Warehouse. Each stage is a rounded rectangle with an icon and label.

**Source Systems:** Three source icons (PostgreSQL logo, Salesforce icon, Kafka logo) stacked vertically. Clicking any source shows a tooltip: "Orders DB: 50K rows/day. Salesforce CRM: 5K records/day. Kafka event stream: 2M events/day."

**Extract:** Shows an animated data-flow arrow (dashed, scrolling) pulling data from sources. Tooltip: "Raw data extracted without transformation. May include duplicates, nulls, type mismatches."

**Transform:** Shows a gear icon with three sub-steps: Cleanse → Map → Aggregate. Hovering Cleanse: "Remove nulls, fix encoding errors, deduplicate." Hovering Map: "Convert source column names and types to warehouse schema." Hovering Aggregate: "Pre-compute daily totals, join to dimension keys."

**Load:** Shows data flowing into Parquet files (cloud object storage icon). Tooltip: "Parquet files written to S3/GCS with Snappy compression. Row group size: 128 MB. Predicate pushdown enabled."

**Data Warehouse:** Final destination icon (Snowflake or BigQuery logo, generic). Tooltip: "External tables or native table load. Query-ready."

An "ELT Mode" toggle swaps the Transform and Load stages, with a note: "In ELT, raw data lands in the warehouse first; SQL transforms run inside the warehouse engine."

**Responsive:** Redraws on window resize.
</details>

## Bitmap Indexes

The **bitmap index** is an index type optimized for low-cardinality columns — those with a small number of distinct values (gender, status, region). For each distinct value, a bitmap index stores one bit per row: 1 if the row has that value, 0 if it doesn't.

Bitmap indexes excel at multi-column boolean filters common in OLAP queries: `WHERE region = 'West' AND status = 'Active' AND product_line = 'Electronics'` becomes three bitmap AND operations — extremely fast and cache-friendly. They are a poor choice for high-cardinality columns (like order_id), where almost every row has a unique value and the bitmap provides no filtering benefit.

Modern columnar databases often achieve similar results through columnar statistics and predicate pushdown rather than explicit bitmap indexes, but the concept remains important for understanding how analytical engines accelerate filter-heavy queries.

## Snowflake and BigQuery: The Modern Cloud Warehouses

**Snowflake** is a cloud-native analytical database built on a three-layer architecture: storage (data in object storage as Parquet files), compute (virtual warehouses — independently scalable MPP clusters), and services (query planning, metadata, security). The key innovation is that storage and compute are separated — you can scale compute up for a heavy query job and back down immediately, paying only for what you use. Multiple virtual warehouses can query the same data simultaneously without contention.

**BigQuery** (Google Cloud) takes an even more radical approach: there are no persistent clusters. Every query is automatically distributed across thousands of slots (units of compute) pulled from Google's shared pool. Users pay per terabyte scanned rather than for reserved compute. BigQuery uses a proprietary columnar storage format (Capacitor) and the Dremel query execution engine, which achieves massive parallelism by executing queries as a tree of workers.

| Feature | Snowflake | BigQuery |
|---------|-----------|---------|
| Pricing model | Credits for compute + storage | Per-TB scanned + storage |
| Compute model | Virtual warehouses (persistent clusters) | Serverless (auto-scaled per query) |
| Storage format | Parquet-compatible object storage | Proprietary Capacitor format |
| Data sharing | Native (zero-copy cross-account) | Analytics Hub |
| Semi-structured support | VARIANT type (JSON/XML/Avro) | Native JSON columns |
| SQL dialect | ANSI SQL + extensions | Standard SQL (ANSI-compliant) |

!!! mascot-tip "Right-Size Your Warehouse"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex offers a tip">
    In Snowflake, a query that takes 10 minutes on an XS virtual warehouse takes roughly 30 seconds on a 2XL — but costs 16× more per second. Run exploratory queries on small warehouses; reserve large warehouses for production ETL jobs with SLA requirements. In BigQuery, focus on reducing bytes scanned (column pruning, partition filters) rather than compute size — scanned bytes is the pricing unit.

## The ATAM Lens: Analytical Database Tradeoffs

From an ATAM perspective, analytical databases present a distinctive set of quality attribute tradeoffs that must appear explicitly in any utility tree that includes reporting or BI requirements.

**Sensitivity points — where architecture choices matter most:**

- **Query latency vs. freshness:** Pre-aggregated OLAP cubes and materialized views dramatically reduce query latency but introduce data staleness. Systems requiring real-time analytics must accept higher query latency or invest in streaming ETL (Kafka → Spark Streaming → warehouse).
- **Columnar storage vs. write throughput:** Columnar storage is read-optimized and write-pessimized. Bulk loads (via Parquet files) are fast; individual row updates are slow or unsupported. Analytical databases are not OLTP replacements.

**Tradeoff point — the HTAP question:**

HTAP (Hybrid Transactional/Analytical Processing) systems attempt to serve both OLTP and OLAP workloads from a single database. Products like Google Spanner, TiDB, and SingleStore advertise HTAP capability. In ATAM terms, HTAP is a tradeoff point: gaining OLAP capability in an OLTP system usually costs something in query latency, consistency, or operational complexity. Treat HTAP claims as a sensitivity point that requires load-testing with your actual query mix before including in an architecture decision.

## Key Takeaways

Analytical databases invert nearly every choice relational databases make: column orientation instead of row orientation, denormalized schemas instead of normalized ones, batch writes instead of transactional writes, eventual freshness instead of immediate consistency. These inversions are intentional — they are the precise tradeoffs that make billion-row aggregation queries complete in seconds.

The concepts from this chapter that recur throughout the book:

- **Columnar storage** — the foundational I/O efficiency mechanism for analytical workloads
- **MPP** — the scaling strategy that turns multi-hour queries into multi-minute ones
- **Star schema / Snowflake schema** — the dimensional modeling patterns that organize analytical data
- **ETL / ELT** — the pipeline that moves data from operational systems to the warehouse
- **Parquet** — the open columnar file format that connects every tool in the analytical stack
- **Materialized view** — pre-computed query results that eliminate aggregation at query time

!!! mascot-celebration "Chapter 4 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You've navigated the inversion: everything Chapter 3 told you was a strength of relational databases — row orientation, normalization, write performance — analytical databases deliberately trade away. The tradeoffs exist for good reason, and understanding both sides is what lets you make defensible architectural decisions. Next up: databases that trade even more of the relational model's guarantees for extreme throughput — starting with the simplest possible data model: key-value stores.
