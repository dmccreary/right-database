---
title: Chapter 16 - Database Selection and Polyglot Persistence
description: Capstone chapter — ATAM-based database selection in practice, scoring matrices, polyglot persistence architectures, migration planning, schema evolution, operational runbooks, and producing an Architectural Decision Record.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 16: Database Selection and Polyglot Persistence

## Summary

This capstone chapter brings together all prior content through the lens of ATAM-based database selection. Students apply scoring matrices, total cost of ownership analysis, and architectural risk assessment to choose among competing database candidates for realistic system scenarios. The chapter covers polyglot persistence — deliberately combining multiple database types within one system — and addresses the practical challenges of migration planning, schema evolution, operational runbooks, and vendor lock-in. Students complete the chapter by conducting a full ATAM database selection exercise and producing an architectural decision record.

## Concepts Covered

This chapter covers the following 12 concepts from the learning graph:

1. Polyglot Persistence
2. Database Selection Framework
3. Scoring Matrix
4. Total Cost of Ownership
5. Vendor Lock-In Risk
6. Database Migration Plan
7. Schema Migration
8. Multi-Model Database
9. Operational Runbook
10. Team Expertise Factor
11. Database Deprecation Risk
12. Data Access Pattern Analysis

## Prerequisites

This chapter builds on concepts from:

- [Chapter 1: The ATAM Method](../01-atam-method/index.md)
- [Chapter 3: Relational Databases](../03-relational-databases/index.md)
- [Chapter 4: Analytical Databases](../04-analytical-databases/index.md)
- [Chapter 5: Key-Value Stores](../05-key-value-stores/index.md)
- [Chapter 6: Column-Family Databases](../06-column-family-databases/index.md)
- [Chapter 7: Document Databases](../07-document-databases/index.md)
- [Chapter 8: Graph Databases](../08-graph-databases/index.md)
- [Chapter 13: High Availability Architecture](../13-high-availability/index.md)
- [Chapter 15: LLM-Generated Embeddings](../15-llm-embeddings/index.md)

---

!!! mascot-welcome "Welcome to Chapter 16 — the Capstone!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Every chapter in this book has built toward this one. You now have a working mental model of six database paradigms, the distributed systems theory that governs them, the high-availability engineering that keeps them running, and the vector search and embedding capabilities that extend them. This chapter puts all of it to work in a structured, repeatable database selection process — the kind that produces decisions you can defend in an architecture review, explain to a CTO, and revisit in five years without embarrassment.

## The Database Selection Framework

A **database selection framework** is a structured process for choosing among competing database candidates given a set of system requirements. Without such a framework, database selection degenerates into advocacy: the engineer who has the most experience with MongoDB argues for MongoDB; the PostgreSQL expert argues for PostgreSQL; the decision is made by seniority or enthusiasm rather than by evidence.

The ATAM-based selection framework this book has built toward has five steps:

1. **Data Access Pattern Analysis** — characterize the workload precisely.
2. **Utility Tree Construction** — elicit and prioritize quality attribute scenarios.
3. **Candidate Identification** — identify which paradigms are viable given the patterns and scenarios.
4. **Scoring Matrix Evaluation** — score each candidate against weighted criteria.
5. **Risk Assessment and ADR** — identify risks, document the decision.

Each step is examined in depth below, using a worked example: a **real-time e-commerce platform** that must handle product catalog browsing, order processing, user session management, product recommendations, and fraud detection.

## Step 1: Data Access Pattern Analysis

**Data access pattern analysis** is the discipline of characterizing what a system actually does with its data — before choosing how to store it. The analysis produces a workload profile that makes database selection evidence-based rather than opinion-based.

The key dimensions to characterize:

- **Read/write ratio:** What fraction of operations are reads vs. writes? A 95% read, 5% write workload favors read-optimized databases (Redis for caching, replicated PostgreSQL) over write-optimized ones (Cassandra).
- **Query patterns:** Are queries point lookups (by primary key), range scans, full-text search, graph traversals, or aggregations? Each favors different index structures and paradigms.
- **Data volume and growth rate:** How much data today? In three years? A 10GB relational database and a 50TB relational database have very different architecture implications.
- **Latency requirements:** What are the p99 latency budgets for each operation? Sub-millisecond (Redis), single-digit ms (well-indexed PostgreSQL), tens of ms (DynamoDB), hundreds of ms (Cassandra wide scans)?
- **Consistency requirements:** Is eventual consistency acceptable, or does the operation require read-your-writes, strong consistency, or serializability?

For the e-commerce platform, the analysis produces distinct profiles per domain:

| Domain | Read/Write | Query Pattern | Latency | Consistency |
|--------|-----------|---------------|---------|-------------|
| Product catalog | 99% read | Point + full-text | <50ms | Eventual OK |
| Orders | 60% read | Point lookup, joins | <100ms | Serializable |
| Sessions | 99% read | Point lookup by key | <5ms | Read-your-writes |
| Recommendations | 95% read | Vector ANN | <50ms | Eventual OK |
| Fraud detection | 70% read | Graph traversal | <200ms | Strong |

The different profiles immediately signal that no single database paradigm will serve all five domains optimally — a polyglot architecture is indicated.

#### Diagram: Data Access Pattern Analyzer

<details markdown="1">
<summary>Interactive Data Access Pattern Analyzer</summary>
Type: MicroSim
**sim-id:** data-access-pattern-analyzer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Analyze a system's data access patterns and identify which database paradigms are viable candidates for each domain. (Bloom L4: Analyze)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
A form-based MicroSim with 5 sliders/dropdowns per domain (user configures the system):
- Read/Write Ratio slider (0% = all writes to 100% = all reads)
- Query Pattern dropdown: Point Lookup | Range Scan | Full-Text | Graph Traversal | ANN Similarity | Aggregation
- Data Volume dropdown: <1GB | 1GB–1TB | 1TB–100TB | >100TB
- Latency Budget dropdown: <5ms | 5–50ms | 50–500ms | >500ms
- Consistency Requirement dropdown: Eventual | Read-Your-Writes | Strong | Serializable

After configuring all sliders, clicking "Analyze" produces a ranked list of recommended database paradigms for this profile, with color-coded fit scores (green=strong, yellow=moderate, red=poor):
- Relational: Strong for Serializable consistency + joins. Poor for >100TB + high write.
- Key-Value: Strong for <5ms + point lookup + eventual. Poor for graph traversal.
- Column-Family: Strong for high write + time-series. Poor for complex joins.
- Document: Strong for flexible schema + moderate consistency. Poor for graph traversal.
- Graph: Strong for graph traversal. Poor for aggregation at scale.
- Analytical: Strong for aggregation + large volume. Poor for <5ms latency.
- Vector (add-on): Strong for ANN similarity. Neutral on others.

**Responsive:** Redraws on window resize.
</details>

## Step 2: Utility Tree Construction

The **utility tree** translates business requirements into precise, measurable quality attribute scenarios that drive the database selection. A full utility tree was covered in Chapter 1; here we apply it specifically to database selection.

For the e-commerce platform, the utility tree's database-relevant branches produce the following high-priority scenarios (H,H ratings):

- **Performance:** "The product catalog search returns results within 50ms at p99 for 10,000 concurrent users during peak shopping events." → Drives: full-text + vector search, read replicas.
- **Durability:** "No order data is lost under any single-node failure mode." → Drives: synchronous replication, WAL on durable storage, multi-AZ deployment.
- **Availability:** "The order placement flow must remain available during a single availability zone failure." → Drives: active-active or active-passive topology, automated failover.
- **Consistency:** "Account balances and inventory counts must reflect the most recent committed transaction." → Drives: serializable isolation, CP system for these domains.
- **Scalability:** "The recommendation engine must serve 50,000 vector similarity queries per second during peak." → Drives: ANN index, horizontal sharding, possibly dedicated vector infrastructure.

## Step 3: Candidate Identification and the Scoring Matrix

A **scoring matrix** is a structured evaluation tool that scores each database candidate against a weighted set of criteria derived from the utility tree. The process:

1. List the quality attributes from the utility tree as columns, weighted by their (H/M/L) priority.
2. List the candidate databases as rows.
3. Score each cell (1–5): how well does this database satisfy this quality attribute?
4. Multiply scores by weights and sum to produce a weighted total.
5. Examine the results — and then examine your assumptions.

For the e-commerce platform's Order domain (requiring serializable consistency, joins, multi-AZ HA, < 100ms latency):

| Database | Consistency (30%) | Join Support (20%) | Write Throughput (15%) | HA/Failover (20%) | Ops Complexity (15%) | Weighted Score |
|---------|------------------|-------------------|----------------------|-------------------|---------------------|---------------|
| PostgreSQL (multi-AZ) | 5 | 5 | 3 | 4 | 4 | **4.25** |
| CockroachDB | 5 | 4 | 4 | 5 | 3 | **4.25** |
| MySQL (InnoDB) | 4 | 5 | 4 | 3 | 5 | **4.10** |
| DynamoDB | 2 | 1 | 5 | 5 | 4 | **3.05** |
| MongoDB | 3 | 2 | 4 | 4 | 4 | **3.25** |

PostgreSQL and CockroachDB tie on weighted score. The distinguishing factors become operational: team PostgreSQL expertise (favoring PostgreSQL), global distribution requirement (favoring CockroachDB), and cost (PostgreSQL is cheaper to run unless global distribution is needed). These factors must appear in the final ADR as the reasoning behind the tiebreak.

#### Diagram: Interactive Database Scoring Matrix

<details markdown="1">
<summary>Interactive Database Selection Scoring Matrix</summary>
Type: MicroSim
**sim-id:** database-scoring-matrix<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Apply a weighted scoring matrix to evaluate database candidates against quality attribute criteria and identify the top candidate for a given workload. (Bloom L3–L5: Apply/Evaluate)

**Canvas:** 780px wide × 520px tall. CANVAS_HEIGHT: 520.

**Description:**
A spreadsheet-style grid: 5 database rows × 6 criteria columns + a "Weighted Score" column.

**Criteria columns with adjustable weights (shown as sliders 0–40%):**
Consistency, Query Flexibility, Write Throughput, HA/Failover, Schema Flexibility, Ops Complexity. Weights must sum to 100% (constraint enforced with live feedback).

**Score cells:** Each cell is a clickable stepper (1–5) with a color gradient (red=1, yellow=3, green=5).

**Weighted Score column:** Recomputes live whenever any cell or weight changes. The highest-scoring row is highlighted with a gold border.

**Database rows:** PostgreSQL, CockroachDB, Cassandra, MongoDB, DynamoDB.

**Pre-set scenarios:** Dropdown with 4 pre-configured scenarios: "E-Commerce Orders", "IoT Time-Series", "Social Network", "Analytics Dashboard." Each pre-fills the weights and scores and explains why.

A "Export Decision" button shows a text summary: "Based on this scoring, PostgreSQL is recommended for the E-Commerce Orders domain. Key differentiators: Consistency (5/5), Join Support (5/5). Primary risks: Write throughput at scale — mitigate with read replicas and connection pooling."

**Responsive:** Redraws on window resize.
</details>

## Total Cost of Ownership

**Total cost of ownership (TCO)** is the full cost of a database selection decision over its operational lifetime — not just the licensing fee. Engineers who evaluate databases only on infrastructure cost routinely underestimate the true cost by 3–5×.

TCO components for a database selection decision:

- **Licensing:** Open-source (PostgreSQL, Cassandra) = $0 license. Commercial (Oracle, Snowflake) = significant per-CPU or consumption-based fees.
- **Infrastructure:** Compute (CPU, RAM), storage (NVMe SSD vs. HDD vs. object storage), network (replication traffic, cross-AZ data transfer).
- **Operations:** DBA hours for backup management, index tuning, upgrade planning, incident response.
- **Migration:** One-time cost to migrate from an existing system. Includes development, testing, dual-run period, and rollback planning.
- **Training:** Time for the engineering team to learn the new database, its operational idioms, and its failure modes.
- **Vendor lock-in risk:** The cost (financial and schedule) of migrating *away* from the chosen database in the future if requirements change.

**Team expertise factor** is one of the most underweighted TCO components in practice. A team with five years of PostgreSQL production experience choosing Cassandra for a new system is not choosing a free database — it is choosing a database whose operational cost includes months of learning curve, unfamiliar failure modes (compaction storms, SSTable accumulation, tombstone handling), and higher incident resolution times until expertise accumulates.

## Vendor Lock-In and Database Deprecation Risk

**Vendor lock-in risk** is the risk that choosing a proprietary or closed-source database makes future migration prohibitively expensive. The risk manifests in several ways:

- **Proprietary SQL extensions:** A database that uses non-standard SQL syntax (Oracle's `ROWNUM`, SQL Server's `TOP`) makes the application code tightly coupled to that vendor.
- **Proprietary wire protocol:** An application built against a custom API (DynamoDB's API, Cassandra's CQL over the Cassandra driver) is harder to migrate than one built against an open standard.
- **Data format lock-in:** Data stored in a proprietary binary format (some analytical databases) cannot be read without the vendor's tools.

**Database deprecation risk** is the risk that the database vendor discontinues the product, changes pricing dramatically, or is acquired. Open-source databases eliminate deprecation risk (the software can be self-hosted indefinitely), but introduce different risks: the project may lose maintainers, security patches may slow, or the community may fork.

**Mitigation strategies:**

- Use databases with open-source cores (PostgreSQL, MySQL, Cassandra) to minimize both vendor lock-in and deprecation risk.
- Design a thin database abstraction layer (repository pattern) so the application is not tightly coupled to a specific database API.
- Maintain data in standard, exportable formats (PostgreSQL's `pg_dump`, Parquet files in S3) to preserve migration optionality.

## Polyglot Persistence: Combining Multiple Database Types

**Polyglot persistence** is the architectural pattern of using multiple database types within a single system, assigning each data domain to the database paradigm best suited to its access patterns. The term was coined by Martin Fowler and Pramod Sadalage in 2011 and has become the dominant architecture pattern for complex production systems.

Returning to the e-commerce platform, the polyglot architecture assigns:

- **Orders and payments** → PostgreSQL (serializable ACID, joins, complex queries)
- **Product catalog** → Elasticsearch + pgvector (full-text + semantic search)
- **User sessions** → Redis (sub-millisecond key-value reads, TTL-based expiration)
- **Recommendations** → PostgreSQL + pgvector (vector ANN on product embeddings)
- **Fraud detection graph** → Neo4j or TigerGraph (graph traversal for relationship patterns)
- **Analytics and reporting** → Snowflake or BigQuery (OLAP aggregations over historical orders)

Each domain uses the database that makes its quality attribute scenarios achievable. No single database is asked to be everything.

#### Diagram: Polyglot Persistence Architecture

<details markdown="1">
<summary>Interactive Polyglot Persistence Architecture — E-Commerce Platform</summary>
Type: MicroSim
**sim-id:** polyglot-persistence-architecture<br/>
**Library:** vis-network<br/>
**Status:** Specified

**Learning Objective:** Design a polyglot persistence architecture that assigns each data domain to the most appropriate database paradigm, with documented tradeoff justification. (Bloom L6: Create)

**Canvas:** 780px wide × 540px tall. CANVAS_HEIGHT: 540.

**Description:**
A vis-network graph showing the e-commerce platform's polyglot architecture. Nodes:

**Service nodes** (rounded rectangles, `#4682B4`): Order Service, Product Service, Session Service, Recommendation Service, Fraud Service, Analytics Service.

**Database nodes** (cylinders, color by paradigm):
- PostgreSQL: `#336791` (relational blue) — connected to Order Service, Recommendation Service
- Redis: `#DC382D` (redis red) — connected to Session Service
- Elasticsearch: `#F04E98` (elastic pink) — connected to Product Service
- Neo4j: `#018BFF` (graph blue) — connected to Fraud Service
- Snowflake: `#29B5E8` (snowflake blue) — connected to Analytics Service

**Edges:** Labeled with the access pattern: "ACID writes", "Key lookup <5ms", "Full-text + vector search", "Graph traversal", "OLAP aggregation."

**Interactions:**
- Clicking any database node shows a tooltip: "PostgreSQL: handles Order and Recommendation data. Provides serializable transactions for order placement. pgvector extension handles product embedding similarity queries."
- Clicking any service node highlights all databases it connects to and shows the access patterns.
- A "Show Data Flow" button animates an order placement: data flows from Order Service → PostgreSQL (ACID write) → event published to Analytics Service → Snowflake (async ETL).
- A "Show Risk" toggle overlays risk badges: "Redis: single point of failure for sessions — mitigate with Redis Cluster." "Neo4j: community edition has limited HA — evaluate TigerGraph for HA requirements."

**Node color conventions:** Relational = `#4682B4`, Key-Value = `#E65100`, Document = `#28a745`, Graph = `#6f42c1`, Analytical = `#6c757d`.

**Responsive:** Redraws on window resize.
</details>

## The Accidental vs. Essential Complexity Test

Polyglot persistence introduces genuine operational complexity: more systems to monitor, back up, upgrade, and troubleshoot. Before adding any database to the architecture, apply the **accidental vs. essential complexity test**:

- **Essential complexity** is complexity that is inherent to the problem. Fraud detection genuinely requires graph traversal; adding Neo4j addresses an essential requirement.
- **Accidental complexity** is complexity that is not inherent to the problem but introduced by the solution. Adding a document database for data that is naturally relational — just to avoid schema migrations — introduces accidental complexity.

The test: could this requirement be met at acceptable quality by a database already in the architecture, with reasonable effort? If yes, resist the temptation to add a new database. Three databases operating well are better than six databases operating poorly.

!!! mascot-thinking "The 'Just Use Postgres' Heuristic"
    <img src="../../img/mascot/thinking.png" class="mascot-admonition-img" alt="Dex thinks">
    PostgreSQL in 2025 is a remarkably capable polyglot database: it supports JSON documents (JSONB), time-series data (TimescaleDB extension), vector search (pgvector), graph-like queries (recursive CTEs), and full-text search — all with ACID guarantees. For many systems, "just use Postgres" is not intellectual laziness but genuine architectural wisdom. Add a second database type only when you have a quality attribute scenario that PostgreSQL cannot meet at any configuration. That discipline keeps your architecture simple and your operational burden manageable.

## Database Migration Planning

When the correct database for a domain is different from what the system currently uses, a **database migration plan** is required. Database migrations are among the most operationally risky activities in software engineering — more data, more risk. The plan must address:

- **Dual-write period:** Both the old and new database receive writes simultaneously. This ensures the new database accumulates current data before cutover without downtime.
- **Backfill:** Historical data is migrated from the old to the new database, typically via a batch job.
- **Validation:** Before cutover, read the same records from both databases and compare. Discrepancies indicate bugs in the migration logic.
- **Cutover:** Switch the application to read from the new database. Keep the old database in read-only mode for a rollback window.
- **Rollback plan:** If the new database shows unexpected behavior in production, describe precisely how to revert. (Without a rollback plan, "revert" means "emergency heroics.")

**Schema migration** — evolving a database schema while keeping existing data valid and the system running — is a related challenge. For relational databases, the discipline is **online schema changes** (using tools like `pg_repack` or `pt-online-schema-change` for MySQL) that add columns, create indexes, and transform data without locking the table. For document databases, schema evolution is handled at the application layer (schema-on-read), but this creates the inverse problem: old and new document shapes must coexist in code until all documents are migrated.

## Multi-Model Databases

A **multi-model database** is a single database system that natively supports multiple data models — typically some combination of relational, document, key-value, and graph. ArangoDB supports document + graph + key-value. Cosmos DB supports document, column-family, graph, and key-value interfaces. FaunaDB supports document + relational.

Multi-model databases are attractive because they reduce the number of distinct systems to operate. The practical tradeoffs:

- **Depth vs. breadth:** A multi-model database's graph engine is rarely as capable as a dedicated graph database; its full-text search is rarely as capable as Elasticsearch. You trade best-of-breed depth for operational simplicity.
- **Ecosystem maturity:** Purpose-built databases (PostgreSQL, Cassandra, Neo4j) have larger communities, more tooling, and more operational knowledge than multi-model alternatives.
- **ATAM fit:** Multi-model makes sense when the system's quality attribute scenarios are satisfied by a good-enough implementation of multiple models, and operational simplicity is itself a high-priority quality attribute.

## Operational Runbooks

An **operational runbook** is a documented procedure for operating, monitoring, and recovering a database in production. Runbooks are the institutional knowledge that makes a database team's operations reproducible, auditable, and survivable when key engineers are unavailable.

A complete database runbook includes:

- **Health check procedures:** How to verify the database is healthy (connection count, replication lag, WAL disk usage, cache hit rate).
- **Common failure modes and remediation:** "If WAL disk is > 90% full: identify long-running transactions holding WAL, notify on-call DBA, escalate to DB restart if disk hits 95%."
- **Backup and restore procedures:** Backup schedule, retention policy, restore test frequency, RTO/RPO targets.
- **Upgrade procedure:** How to upgrade to a new database version with minimal downtime (rolling upgrade vs. blue/green).
- **Escalation matrix:** Who is paged at what severity, and who has root-cause authority.

## Producing an Architectural Decision Record

An **Architectural Decision Record (ADR)** is the written artifact that documents a significant architectural decision — its context, the options considered, the tradeoffs evaluated, and the decision reached. ADRs are the output of ATAM analysis applied to database selection.

A database selection ADR has four sections:

1. **Context:** What problem are we solving? What quality attribute scenarios drive the decision? What is the current state of the system?
2. **Considered options:** What database candidates were evaluated? What were their key strengths and weaknesses relative to the utility tree scenarios?
3. **Decision:** What database was selected, and for which domain? What were the deciding factors?
4. **Consequences:** What risks does this decision introduce? What mitigation strategies are in place? What will this decision make harder or easier in the future? When should this decision be revisited?

The ADR is not a marketing document for the chosen database. It is an honest account of the tradeoffs, written so that a future engineer can understand why the decision was made and whether the assumptions that drove it still hold.

#### Diagram: ATAM Database Selection Decision Tree

<details markdown="1">
<summary>Interactive ATAM Database Selection Decision Tree</summary>
Type: MicroSim
**sim-id:** database-selection-decision-tree<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Apply the ATAM database selection framework to a novel system scenario by working through a structured decision tree of quality attribute questions. (Bloom L6: Create)

**Canvas:** 780px wide × 560px tall. CANVAS_HEIGHT: 560.

**Description:**
A vertical decision tree MicroSim. The user answers a series of questions by clicking Yes/No buttons. Based on answers, the tree narrows to a database recommendation.

**Question sequence:**
1. "Does the system require ACID transactions across multiple records?" → Yes: keep relational/NewSQL. No: consider NoSQL.
2. "Is the primary query pattern point lookup by a single key?" → Yes: consider Key-Value. No: continue.
3. "Does the data have a natural graph structure (relationships between entities are first-class)?" → Yes: consider Graph DB. No: continue.
4. "Is the workload primarily analytical (complex aggregations over large datasets)?" → Yes: consider Analytical DB. No: continue.
5. "Does the schema change frequently or vary significantly across records?" → Yes: consider Document DB. No: consider Relational.
6. "Is write throughput > 100K writes/second at scale?" → Yes: consider Column-Family. No: relational is likely sufficient.

**Visual:** Each question is a diamond node; edges are labeled Yes/No. Answered questions turn green. The current question pulses. Recommended paradigm appears in a gold badge at the bottom with a one-line justification.

A "Reset" button starts over. An "Export" button shows the full question/answer path as text — useful for documenting the reasoning in an ADR.

**Responsive:** Redraws on window resize.
</details>

## Key Takeaways

!!! mascot-encourage "You Have Built the Toolkit"
    <img src="../../img/mascot/encouraging.png" class="mascot-admonition-img" alt="Dex encourages">
    You have covered six database paradigms, the distributed systems theory that governs them, consensus protocols, high availability engineering, vector search, LLM embeddings, and a structured decision framework for choosing among them. That is not a small thing. The field moves fast — new databases appear constantly, and new capabilities (vector search, HTAP, serverless) blur the lines between paradigms. But the analytical framework you have built — characterize access patterns, build a utility tree, score candidates, document the decision — is durable. It works regardless of what the database market looks like next year.

This chapter's core contributions to your toolkit:

- **Data access pattern analysis** — characterize read/write ratio, query patterns, volume, latency, and consistency requirements per domain before choosing a database
- **Scoring matrix** — weighted, evidence-based comparison of candidates against utility tree quality attributes
- **TCO** — full cost including operations, training, migration, and vendor lock-in risk
- **Team expertise factor** — the hidden multiplier on every "operationally simple" claim
- **Polyglot persistence** — the right architecture when no single database serves all domains; resisted by the accidental-vs-essential complexity test
- **Database migration plan** — dual-write, backfill, validate, cutover, rollback — in that order
- **Multi-model database** — breadth over depth; use when operational simplicity outweighs best-of-breed depth
- **Operational runbook** — the institutional knowledge that makes a database survivable at 2am
- **ADR** — the written record that context, options, decision, and consequences are all captured honestly

!!! mascot-celebration "Congratulations — You've Completed the Course!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates big">
    From ATAM utility trees to vector ANN indexes, from WAL-based crash recovery to distributed saga compensation, from B-Tree index traversal to consensus protocol leader election — you have built a comprehensive, theoretically grounded, practically applicable understanding of database architecture and selection. The next time a team asks "which database should we use?", you will not answer from habit or intuition. You will answer from evidence, with a documented decision and a clear account of the tradeoffs. That is what this course set out to teach. Go build something with it.
