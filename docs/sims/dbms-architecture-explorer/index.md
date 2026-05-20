---
title: DBMS Architecture Explorer
description: Layered vis-network diagram of a DBMS stack — click any component to see how it contributes to query processing, concurrency, buffering, indexing, storage, and durability.
status: implemented
library: vis-network
bloom_level: 2
bloom_verb: explain
---

# DBMS Architecture Explorer

<iframe src="main.html" width="100%" height="642" scrolling="no"></iframe>

## Learning Objective

Understanding (Bloom's level 2) — students explain the role of each DBMS component and describe how the components interact to deliver persistence, concurrency, query processing, and durability.

!!! warning "Scaffold"
    This MicroSim has been scaffolded from its specification. The interactive
    implementation has not been built yet.

## Learning Objective

Understanding (Bloom's level 2) — students explain the role of each DBMS component and describe how the components interact to deliver persistence, concurrency, query processing, and durability.

- **Bloom Level:** TBD
- **Bloom Verb:** TBD
- **Library:** vis-network

## Preview

<iframe src="main.html" width="100%" height="600"></iframe>

[Run MicroSim in Fullscreen](main.html){ .md-button .md-button--primary }

## Specification

The full specification below is extracted from
[Chapter 2: Chapter 2 - Database Foundations](../../chapters/02-database-foundations/index.md).

```text
Type: interactive-infographic
**sim-id:** dbms-architecture-explorer
**Library:** vis-network
**Status:** Specified

**Learning objective:** Understanding (Bloom's level 2) — students explain the role of each DBMS component and describe how the components interact to deliver persistence, concurrency, query processing, and durability.

**Canvas:** 900 × 520px, responsive. A layered architecture diagram rendered as a vis-network graph with a fixed hierarchical layout (direction: UD, physics: false). An info panel (300px) appears to the right when a node is clicked.

**Node layers and data (top to bottom):**

Layer 1 — Application interface (y=60, color #4682B4, shape: box):
- id: 1, label: "Client / Application", x: 450

Layer 2 — Query interface (y=160, color #E65100, shape: box):
- id: 2, label: "Query Language\nParser", x: 220
- id: 3, label: "Query Optimizer", x: 680

Layer 3 — Execution (y=260, color #6f42c1, shape: box):
- id: 4, label: "Query Executor", x: 450

Layer 4 — Data management (y=360, color #28a745, shape: box):
- id: 5, label: "Concurrency\nController", x: 220
- id: 6, label: "Buffer Pool\nManager", x: 450
- id: 7, label: "Index Manager", x: 680

Layer 5 — Storage (y=460, color #6c757d, shape: box):
- id: 8, label: "Storage Engine\n(B-tree / LSM)", x: 220
- id: 9, label: "Write-Ahead Log\n(WAL)", x: 680

**Edges (all arrows pointing downward):**
- 1 → 2 (label: "SQL / query text")
- 2 → 3 (label: "parse tree")
- 3 → 4 (label: "execution plan")
- 4 → 5, 4 → 6, 4 → 7
- 6 → 8, 6 → 9

**Info panel content per node:**

Node 1 (Client / Application): "The application issues queries or commands using the database's query language or API. Every request enters through this boundary — it is the only interface the application should use, because it ensures all DBMS guarantees apply."

Node 2 (Query Language Parser): "The parser tokenizes and validates the query text, then produces a structured representation (parse tree or AST) the optimizer can reason about. This is where syntax errors are caught. The query language abstracts the application from storage details — the same SQL can be executed against very different storage layouts."

Node 3 (Query Optimizer): "The optimizer transforms the parse tree into a physical execution plan by selecting join algorithms, access paths (index scan vs. full table scan), and operation ordering. A good optimizer can make a naive query run 1000× faster than the obvious execution path. A poor optimizer — or an optimizer operating without statistics — can make the same query run 1000× slower."

Node 4 (Query Executor): "The executor runs the physical plan produced by the optimizer, pulling data from the buffer pool and passing it through operator pipelines (scan, filter, join, aggregate). Execution performance is heavily influenced by whether intermediate results fit in memory and whether sequential I/O patterns allow hardware prefetch."

Node 5 (Concurrency Controller): "The concurrency controller ensures that concurrent reads and writes produce results consistent with some defined isolation level. It implements either lock-based control (two-phase locking) or multiversion concurrency control (MVCC). The choice of isolation level is a sensitivity point: stronger isolation prevents more anomalies but reduces throughput."

Node 6 (Buffer Pool Manager): "The buffer pool is a memory cache of disk pages. It decides which pages to keep in RAM and which to evict when memory is full. Hit rate is the dominant factor in read performance. Buffer pool management is a critical sensitivity point: too small, and every read goes to disk; too large, and the OS is memory-starved."

Node 7 (Index Manager): "The index manager maintains secondary data structures (B-trees, hash tables, inverted indexes, ANN indexes) that allow the query executor to find records without scanning every row. Indexes dramatically accelerate selective queries but consume storage and slow down writes, since every write must update all relevant indexes."

Node 8 (Storage Engine): "The storage engine implements how data is physically organized on disk. B-tree engines (InnoDB, PostgreSQL heap) optimize for read performance and range queries. LSM-tree engines (RocksDB, Cassandra) optimize for write throughput. The choice of storage engine is a fundamental tradeoff point between write performance and read amplification."

Node 9 (Write-Ahead Log / WAL): "The WAL is an append-only log of every change made to the database, written before the change is applied to the data files. It enables crash recovery (replay the log from the last checkpoint) and replication (ship the log to replicas). WAL sync frequency is a critical sensitivity point for the durability-vs-write-throughput tradeoff."

**Interaction:** Click any node to open the info panel. Hovering a node highlights its direct connections. A "Reset" button centers the view.
```

## Related Resources

- [Chapter 2: Chapter 2 - Database Foundations](../../chapters/02-database-foundations/index.md)
