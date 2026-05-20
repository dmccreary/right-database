---
title: Chapter 3 - Relational Databases
description: Deep dive into the relational model — normalization, SQL, indexing strategies, MVCC and lock-based concurrency, write-ahead logging, and where PostgreSQL and MySQL excel and hit their limits.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 3: Relational Databases

## Summary

This chapter covers the relational model in depth — from normalization and SQL through indexing strategies, concurrency control, and write-ahead logging. Students learn how relational databases achieve ACID guarantees through locking and MVCC, where they excel (structured data, complex joins, strong consistency), and where their scalability limits begin to matter. PostgreSQL and MySQL are examined as canonical representatives.

## Concepts Covered

This chapter covers the following 20 concepts from the learning graph:

1. Relational Data Model
2. SQL
3. Primary Key
4. Foreign Key
5. Normalization
6. First Normal Form
7. Third Normal Form
8. Join Operation
9. B-Tree Index
10. Query Execution Plan
11. Stored Procedure
12. Database View
13. Transaction Log
14. Write-Ahead Log
15. Multi-Version Concurrency
16. Lock-Based Concurrency
17. Deadlock Detection
18. Connection Pooling
19. PostgreSQL
20. MySQL

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)

---

!!! mascot-welcome "Welcome to Chapter 3!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Relational databases have been the default choice for structured data since the 1970s, and understanding *why* they work — and where they don't — is essential before evaluating any alternative. This chapter builds the technical vocabulary you'll use to compare every paradigm that follows. Pay particular attention to normalization, MVCC, and the write-ahead log: these three ideas appear in disguised form in nearly every modern database, even the ones that market themselves as "NoSQL."

## The Relational Model: Data as Tables

Edgar Codd's 1970 paper "A Relational Model of Data for Large Shared Data Banks" introduced an idea that still shapes database design today: represent all data as relations — tables with rows and columns — and use a declarative query language to retrieve it. The revolutionary insight was *independence*: application code should not know how data is physically stored or retrieved. The database engine handles those details.

A **relational data model** organizes data into tables (also called relations). Each table has a fixed schema: a named set of columns (attributes), each with a declared data type. Each row (tuple) represents one instance of the entity the table describes. Two properties must hold for a table to be a proper relation: every column must contain atomic (indivisible) values, and no two rows may be identical.

The **primary key** is one or more columns whose values uniquely identify every row in a table. The engine enforces uniqueness automatically — attempting to insert a duplicate primary key raises a constraint violation. When a column in one table stores values that correspond to primary keys in another table, it is called a **foreign key**. The engine can enforce that every foreign key value refers to an existing primary key, a guarantee called referential integrity.

Before examining a live example, note that three terms appear in almost every relational diagram — table, row, and column — and confusing them is a common source of schema design errors.

- A **table** defines the structure (what columns exist, what types they have).
- A **row** is one record: one complete set of column values.
- A **column** stores one attribute across all rows.

#### Diagram: Interactive Relational Model Explorer

<details markdown="1">
<summary>Interactive Relational Model Explorer</summary>
Type: MicroSim
**sim-id:** relational-model-explorer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify the structural components of the relational model (table, row, column, primary key, foreign key) and explain how foreign keys create relationships between tables. (Bloom L1–L2: Remember/Understand)

**Canvas:** 760px wide × 460px tall. CANVAS_HEIGHT: 460.

**Description:**
Display two tables side by side: `orders` (left) and `customers` (right). Each table is rendered as a grid with a header row (column names in bold, light-gray background) and 4 data rows. Columns in `orders`: `order_id` (PK, highlighted gold), `customer_id` (FK, highlighted blue), `amount`, `status`. Columns in `customers`: `customer_id` (PK, highlighted gold), `name`, `city`.

Draw an animated dashed arrow from the `customer_id` FK column in `orders` to the `customer_id` PK column in `customers` when the user hovers over any FK cell. The arrow should animate (dash offset scrolling) to indicate the referential link.

**Interactions:**
- Hovering any cell highlights the entire row that cell belongs to (yellow tint).
- Clicking a FK cell in `orders` highlights the matching PK row in `customers` with a green tint and draws a bold connecting line between the two rows.
- A bottom info panel updates on hover/click: "This FK value (3) refers to customer row 3: Alice / San Francisco." If the user clicks a row with no matching FK (null), display: "NULL foreign key — this order has no associated customer."
- A "Referential Integrity" toggle button: when toggled ON, an attempt to insert a new `orders` row with an invalid `customer_id` is shown (row flashes red and a tooltip reads "Constraint violation: customer 99 does not exist").

**Responsive:** Redraws on window resize maintaining aspect ratio.

**Color conventions:** PK columns: `#FFD700` (gold). FK columns: `#4682B4` (steel blue). Selected rows: `#c8e6c9` (light green). Hover rows: `#FFF9C4` (light yellow). Header: `#ECEFF1`.
</details>

## SQL: The Query Language

**SQL (Structured Query Language)** is the standardized language for defining, manipulating, and querying relational data. It is declarative: you describe *what* you want, not *how* to retrieve it. The database's query planner decides the physical retrieval strategy.

SQL divides into three sublanguages that together cover every operation a relational database needs:

- **DDL (Data Definition Language):** `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` — define the schema.
- **DML (Data Manipulation Language):** `SELECT`, `INSERT`, `UPDATE`, `DELETE` — manipulate data.
- **DCL (Data Control Language):** `GRANT`, `REVOKE` — control permissions.

The most important and complex DML statement is `SELECT`. A query that retrieves all orders over $1,000 placed by customers in New York looks like this:

```sql
SELECT o.order_id, c.name, o.amount
FROM   orders o
JOIN   customers c ON o.customer_id = c.customer_id
WHERE  c.city = 'New York'
  AND  o.amount > 1000
ORDER BY o.amount DESC;
```

Before running this query, the database's query planner examines statistics about table sizes, available indexes, and column cardinalities to produce an execution plan — a tree of physical operators (index scan, hash join, sort) that retrieves the result as efficiently as possible.

## Normalization: Eliminating Redundancy

**Normalization** is the process of structuring a relational schema to reduce data redundancy and prevent update anomalies. The theory defines a series of **normal forms**, each a stronger constraint on how data is organized. For most production systems, reaching **Third Normal Form (3NF)** is the practical target.

The problem normalization solves is best illustrated by what happens without it. Suppose an `orders` table stores customer name and city in every row. When a customer moves cities, every one of their order rows must be updated. Miss one, and the data is inconsistent. Normalization eliminates such redundancy by ensuring each fact is stored in exactly one place.

**First Normal Form (1NF)** requires that every column hold atomic values — no lists, no comma-separated strings, no repeating groups. A column `phone_numbers` containing "555-1234, 555-5678" violates 1NF; the fix is a separate `phone_numbers` table with one phone per row.

**Second Normal Form (2NF)** applies to tables with composite primary keys. Every non-key column must depend on the *entire* composite key, not just part of it.

**Third Normal Form (3NF)** requires that every non-key column depend directly on the primary key — and *only* on the primary key. No non-key column may determine another non-key column (no transitive dependencies). Storing `city` alongside `zip_code` in a customer table violates 3NF if zip code already determines city; the fix is a separate `zip_codes` table.

#### Diagram: Normalization Visualizer

<details markdown="1">
<summary>Normalization Visualizer — 1NF to 3NF Step-by-Step</summary>
Type: MicroSim
**sim-id:** normalization-visualizer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Apply the rules of 1NF and 3NF to transform a denormalized table into a properly normalized schema. (Bloom L3: Apply)

**Canvas:** 780px wide × 520px tall. CANVAS_HEIGHT: 520.

**Description:**
Display a step-through wizard with 4 stages: Unnormalized → 1NF → 2NF → 3NF.

**Unnormalized stage:** Show a single `order_details` table with columns: `order_id`, `customer_name`, `customer_city`, `zip_code`, `products` (a multi-valued cell showing "Laptop, Mouse"). Highlight the `products` column in red with a tooltip "Violation: non-atomic value."

**1NF stage:** Expand the multi-valued `products` into separate rows. Each product gets its own row. Highlight the change in green. Show a callout: "Fixed: atomic values — one product per row."

**2NF stage:** Show that `customer_name` depends only on `order_id`, not on the full composite key. Split into `orders` and `order_items`. Show the callout: "Fixed: no partial dependencies."

**3NF stage:** Show that `customer_city` is determined by `zip_code` (transitive dependency — red highlight). Split to create a `zip_codes` reference table. Show the final schema of 4 normalized tables.

**Controls:**
- "Next Step" and "Prev Step" buttons navigate stages.
- Violations show a red glow on offending column(s); fixed columns glow green.
- A bottom text panel describes the current violation and the fix applied.

**Responsive:** Redraws on window resize.
</details>

## Join Operations

A **join operation** combines rows from two or more tables based on a related column. It is the mechanism that makes normalized schemas useful: data stored in separate tables is re-assembled at query time. SQL supports several join types, each with different semantics for handling rows with no match.

Before examining the variants, two terms need definitions: the **driving table** is the one whose rows are read first; the **probing table** is the one searched for matches.

| Join Type | Rows Returned |
|-----------|--------------|
| INNER JOIN | Only rows with matching values in both tables |
| LEFT JOIN | All rows from left table; NULLs for unmatched right rows |
| RIGHT JOIN | All rows from right table; NULLs for unmatched left rows |
| FULL OUTER JOIN | All rows from both tables; NULLs wherever no match |
| CROSS JOIN | Every combination of rows from both tables (Cartesian product) |

The physical implementation of a join is determined by the query planner. Three common algorithms are:

- **Nested-loop join:** For each row in the driving table, scan the probing table for matches. Simple but O(n×m).
- **Hash join:** Build a hash table from the smaller relation; probe it with rows from the larger. O(n+m) for unsorted data.
- **Merge join:** Sort both relations on the join key, then merge. Efficient when both inputs are already sorted by an index.

## B-Tree Indexes and Query Execution Plans

Without indexes, every query requires a full table scan — reading every row to find matching ones. For tables with millions of rows, this is prohibitively slow. **Indexes** are auxiliary data structures that allow the engine to jump directly to relevant rows.

The **B-Tree index** (balanced tree) is the default index type in PostgreSQL, MySQL, and most other relational databases. A B-Tree stores index entries in sorted order across a balanced multi-level tree. Each leaf node contains the indexed column value plus a pointer (row ID or primary key) to the heap row. The tree remains balanced after inserts and deletes through node splits and merges, guaranteeing O(log n) lookup regardless of table size.

B-Trees excel at equality lookups (`WHERE customer_id = 42`), range scans (`WHERE amount BETWEEN 100 AND 500`), and ORDER BY acceleration when the sort column matches the index.

A **query execution plan** is the tree of physical operations the engine will use to execute a query. Engineers use `EXPLAIN` (PostgreSQL) or `EXPLAIN ANALYZE` to inspect plans, identify unintentional full table scans, and diagnose slow queries.

#### Diagram: B-Tree Index Traversal

<details markdown="1">
<summary>Interactive B-Tree Index Traversal</summary>
Type: MicroSim
**sim-id:** btree-index-traversal<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Demonstrate how a B-Tree index traverses from root to leaf to locate a specific key, and explain the O(log n) performance characteristic. (Bloom L2–L3: Understand/Apply)

**Canvas:** 780px wide × 480px tall. CANVAS_HEIGHT: 480.

**Description:**
Render a 3-level B-Tree with fan-out 3. Root node contains 2 key values (30, 70) and 3 child pointers. Each internal node has 2 keys and 3 child pointers. Leaf nodes contain 2–3 key-value pairs and a "next leaf" pointer (dashed horizontal arrow).

**Interactive search:** A text input at top lets the user type a key (1–99). Clicking "Search" animates the traversal: highlight root, evaluate against root values (tooltip: "42 < 70, go left"), highlight child pointer, move to next node, repeat to leaf. Each step: 600ms delay. Leaf hits show green; misses show red.

**Range scan mode:** A "Range Scan" button reveals min/max inputs. Highlight all leaf nodes in range and animate leaf-to-leaf pointer traversal.

**Counter panel:** Shows "Nodes visited: 3 of 13" and "Steps: O(log₃ 13) ≈ 2.3" after each search.

**Responsive:** Redraws on window resize.
</details>

!!! mascot-tip "Index Selectivity Matters"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex offers a tip">
    A B-Tree index on a boolean column is nearly useless — the engine still reads half the table. Index high-cardinality columns (many distinct values). The query planner uses statistics to decide when an index scan beats a full table scan; never assume an index is being used until you've run `EXPLAIN ANALYZE`.

## Stored Procedures and Views

A **stored procedure** is a named, pre-compiled block of SQL stored in the database and executed on demand. Stored procedures reduce network round trips, can enforce business rules at the database level, and allow the engine to cache the execution plan.

A **database view** is a named query stored in the database's catalog. When queried, the view executes its underlying SELECT and returns the result as if it were a table. Views serve two purposes: they simplify complex queries (encapsulating a multi-table join) and they provide a security layer (users can access a view without accessing the underlying tables). A **materialized view** physically stores the view's result set and refreshes it periodically — useful for expensive aggregations that don't need to be real-time.

## The Write-Ahead Log and Crash Recovery

The **transaction log** is a sequential, append-only file that records every change made to the database. Before a change is applied to actual data pages on disk, it is first written to the log. This "write-ahead" ordering is what gives the **Write-Ahead Log (WAL)** its name and its power.

The WAL is the foundation of crash recovery. If the database crashes mid-transaction, on restart the engine scans the WAL from the last checkpoint: fully committed transactions (COMMIT record present) are **redone**; uncommitted transactions (no COMMIT record) are **undone**. The WAL also enables point-in-time recovery by shipping log files to a standby server, and in PostgreSQL it is the foundation of logical replication.

## Concurrency Control: Locking vs. MVCC

When multiple transactions access the same data simultaneously, the database must prevent anomalies — lost updates, dirty reads, phantom reads. Relational databases use two primary concurrency control mechanisms.

**Lock-based concurrency control** (pessimistic concurrency) prevents conflicts by acquiring locks before accessing data. Shared locks allow concurrent reads; exclusive locks prevent both concurrent reads and writes. Transactions hold locks until they commit or roll back. Conflicts are prevented before they occur, at the cost of throughput — long-running transactions block other transactions on the same rows.

**Multi-Version Concurrency Control (MVCC)** takes a different approach: rather than blocking readers, the engine maintains multiple versions of each row simultaneously. Each transaction sees a consistent snapshot of the database as it existed at its start time. Readers see old versions; writers create new versions. Readers never block writers; writers never block readers.

PostgreSQL stores old row versions in the heap itself (as "dead tuples") and reclaims them via a background VACUUM process. MySQL's InnoDB stores old versions in a separate undo log. Both approaches deliver read consistency without blocking, at the cost of storage overhead and periodic maintenance.

#### Diagram: MVCC vs. Lock-Based Concurrency Timeline

<details markdown="1">
<summary>MVCC vs. Lock-Based Concurrency — Interactive Timeline</summary>
Type: MicroSim
**sim-id:** mvcc-vs-locking<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Compare how MVCC and lock-based concurrency handle concurrent read and write transactions, and explain why readers don't block writers in MVCC. (Bloom L4: Analyze)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
Two side-by-side panels (left: "Lock-Based", right: "MVCC"), each showing a time-axis diagram of two concurrent transactions (Txn A = reader, Txn B = writer).

**Lock-Based panel:** Txn B acquires X-lock at t=2. Txn A's read at t=3 is a blocked dashed red arrow. Txn B commits at t=5 (lock released). Txn A proceeds at t=6. Blocked period highlighted orange: "Txn A blocked 3 time units."

**MVCC panel:** Txn B starts writing at t=2, creating version v2. Txn A reads at t=3 and sees version v1 (snapshot at start time). Both proceed simultaneously with no blocking. A "version store" sidebar shows v1 and v2 coexisting, with each transaction's pointer to its visible version.

**Controls:**
- "Play" button animates both timelines simultaneously at 1 step per second.
- "Pause" stops animation.
- Clicking any event shows a tooltip explaining what happens at that moment.
- A "Show Dirty Read" button demonstrates what happens at Read Uncommitted isolation: Txn A reads Txn B's uncommitted v2, then Txn B rolls back, leaving Txn A with stale data.

**Responsive:** Redraws on window resize.
</details>

### Deadlock Detection

A **deadlock** occurs when two or more transactions each hold a lock that another transaction needs. Transaction A holds a lock on row 1 and waits for row 2; Transaction B holds a lock on row 2 and waits for row 1. Neither can proceed.

Relational databases detect deadlocks by periodically scanning the **wait-for graph** — a directed graph where each node is a transaction and each edge represents "is waiting for a lock held by." A cycle in this graph indicates a deadlock. Upon detection, the engine chooses a victim transaction (usually the one that has done the least work) and rolls it back, releasing its locks and allowing the other transaction to proceed. Applications must be written to handle deadlock errors and retry rolled-back transactions.

## Connection Pooling

Database connections are expensive to establish: creating a TCP connection, authenticating, and initializing session state can take 20–100ms. For web applications handling hundreds of requests per second, opening a new connection per request is prohibitive.

**Connection pooling** maintains a pool of pre-established connections that are leased to application threads as needed and returned on completion. PgBouncer (for PostgreSQL) and ProxySQL (for MySQL) are widely used external pool managers; application frameworks also include built-in pools. Key configuration parameters are pool size (too small causes queuing; too large exhausts database memory), idle timeout, and maximum connection lifetime.

## PostgreSQL and MySQL: The Canonical Representatives

**PostgreSQL** is the gold standard for open-source relational feature completeness. It supports full ACID transactions with MVCC, advanced data types (arrays, JSONB, ranges, geometric types), table partitioning, logical and streaming replication, row-level security, and an extension ecosystem that includes PostGIS (geospatial) and pgvector (vector search, covered in Chapter 14). Its query planner is among the most sophisticated of any open-source database.

**MySQL** (and its community fork MariaDB) emphasizes read throughput and operational simplicity. With the InnoDB storage engine, MySQL delivers ACID-compliant transactions, row-level locking, and MVCC. Its replication setup is simpler to operate and it remains the dominant database behind LAMP-stack applications. MySQL's JSON support and generated columns have closed much of the feature gap with PostgreSQL, though edge cases in type coercion and standards compliance still distinguish them.

| Dimension | PostgreSQL | MySQL (InnoDB) |
|-----------|-----------|----------------|
| ACID compliance | Full (all isolation levels) | Full (Read Committed default) |
| Default isolation level | Read Committed | Repeatable Read |
| MVCC implementation | Heap-based (dead tuples + VACUUM) | Undo log |
| JSON support | JSONB (indexed, binary) | JSON (text, partial index) |
| Extension ecosystem | Rich (pgvector, PostGIS, Citus) | Moderate |
| Replication | Streaming + logical | Binary log (async/semi-sync/group) |
| Typical primary use case | Complex OLTP, mixed workloads | Web apps, read-heavy OLTP |

!!! mascot-warning "The Scalability Wall"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex warns">
    Single-node relational databases scale vertically beautifully — add CPU, RAM, and faster storage and throughput improves. But they hit a ceiling. A single PostgreSQL node on the largest cloud VM typically handles 50,000–100,000 transactions per second under favorable conditions. Beyond that, you are sharding (Chapter 11), routing to replicas, or re-evaluating whether relational is the right paradigm for this workload. Knowing this limit is not a reason to avoid relational databases — it is a reason to account for it in your ATAM utility tree.

## The ATAM Lens: When Relational Databases Win and Lose

Applying ATAM thinking to relational databases means asking: for which quality attribute scenarios does the relational model create sensitivity or tradeoff points?

**Where relational databases win:**

- **Strong consistency:** ACID transactions with serializable isolation are correct for financial systems, inventory management, and any scenario where lost or duplicated writes have business consequences.
- **Complex query expressiveness:** Ad-hoc joins, window functions, and aggregations are first-class SQL citizens. Systems requiring flexible reporting without pre-defined access patterns benefit from SQL's generality.
- **Schema enforcement:** When data quality and type safety matter — regulated industries, multi-team systems — the enforced schema catches bad data at the database boundary.

**Where relational databases lose:**

- **Write throughput at scale:** Single-leader architectures have one writable node. Once write volume saturates that node, options are limited.
- **Schema rigidity:** Rapidly evolving schemas create friction with migrations and ALTER TABLE locks that can block production traffic.
- **Hierarchical or graph-shaped data:** Representing deeply nested object graphs in relational tables requires many joins or denormalization — both hurt readability and performance compared to document or graph databases.

#### Diagram: Relational Database ATAM Radar

<details markdown="1">
<summary>Relational Database Quality Attribute Radar Chart</summary>
Type: MicroSim
**sim-id:** relational-quality-radar<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Evaluate relational databases against six quality attributes used throughout ATAM analysis, and identify tradeoffs compared to a reference NoSQL baseline. (Bloom L5: Evaluate)

**Canvas:** 660px wide × 520px tall. CANVAS_HEIGHT: 520.

**Description:**
A radar (spider) chart with six axes: Write Throughput, Read Throughput, Consistency, Query Flexibility, Schema Flexibility, and Horizontal Scalability. Scale 0–10 per axis.

Two overlaid polygons:
- "Relational (PostgreSQL)": Consistency=9, Query Flexibility=9, Read Throughput=7, Write Throughput=6, Schema Flexibility=4, Horizontal Scalability=4. Fill: `#1565C0` 40% opacity.
- "Key-Value (Redis)": Consistency=5, Query Flexibility=2, Read Throughput=10, Write Throughput=10, Schema Flexibility=9, Horizontal Scalability=9. Fill: `#E65100` 30% opacity.

**Interactions:**
- Hovering any vertex label shows a tooltip explaining the score: "Relational / Consistency: 9 — Full ACID with serializable isolation."
- A dropdown lets the user swap the comparison paradigm (Key-Value, Column-Family, Document, Graph, Analytical). Each paradigm shows its own scored polygon.
- Clicking any axis label opens a sidebar with a 2-sentence explanation of what that axis measures.

**Responsive:** Redraws on window resize.
</details>

## Key Takeaways

The relational model's durability over fifty years reflects genuine strengths that few alternatives match. ACID transactions, a declarative query language, enforced referential integrity, and mature tooling make relational databases the correct default for structured data with complex query requirements. The tradeoffs are real but manageable within their design envelope.

The vocabulary built in this chapter recurs throughout the rest of the book:

- **Primary key / Foreign key** — unique identity and referential integrity
- **Normalization (1NF, 3NF)** — single source of truth for each fact
- **B-Tree index** — O(log n) lookup, range scans, order optimization
- **MVCC** — readers never block writers; snapshot isolation
- **WAL** — the backbone of crash recovery and replication
- **Connection pooling** — amortize connection setup cost across requests

!!! mascot-celebration "Chapter 3 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now have a working mental model of the engine room behind every relational database. Normalization keeps data clean; B-Trees keep queries fast; MVCC keeps readers from blocking writers; and the WAL keeps data safe through crashes. The next three chapters examine paradigms that deliberately sacrifice some of these guarantees — and deliver something else in return. Understanding what you're trading away starts here.
