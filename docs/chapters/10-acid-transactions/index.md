---
title: Chapter 10 - ACID Transactions
description: The four ACID properties, all four isolation levels, concurrency anomalies each prevents, two-phase locking, MVCC, savepoints, rollback, and distributed sagas for microservices.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 10: ACID Transactions

## Summary

This chapter covers the ACID transaction model in depth — the gold standard for data integrity in relational and NewSQL systems. Students learn all four isolation levels, the concurrency anomalies each prevents (dirty reads, phantom reads, non-repeatable reads), and how both pessimistic (two-phase locking) and optimistic (MVCC) concurrency control mechanisms achieve those guarantees. The chapter closes with distributed sagas as the primary alternative to ACID for microservices systems that cannot afford the latency of distributed locking.

## Concepts Covered

This chapter covers the following 18 concepts from the learning graph:

1. Atomicity
2. Consistency (ACID)
3. Isolation
4. Durability
5. Transaction Isolation Level
6. Read Uncommitted
7. Read Committed
8. Repeatable Read
9. Serializable Isolation
10. Dirty Read
11. Phantom Read
12. Non-Repeatable Read
13. Two-Phase Locking
14. Optimistic Concurrency Control
15. Savepoint
16. Rollback
17. Commit Protocol
18. Distributed Saga

## Prerequisites

This chapter builds on concepts from:

- [Chapter 3: Relational Databases](../03-relational-databases/index.md)
- [Chapter 9: CAP Theorem and Consensus Protocols](../09-cap-theorem-consensus/index.md)

---

!!! mascot-welcome "Welcome to Chapter 10!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    ACID is the reason relational databases have powered banking, e-commerce, and healthcare for fifty years. It is also one of the most frequently misunderstood guarantees in software engineering — developers often believe their database is giving them ACID when it is not, or believe they need ACID when a weaker model would suffice. This chapter gives you the precision to know the difference, and to make the right tradeoff in every system you design.

## The Four ACID Properties

A **transaction** is a sequence of database operations (reads and writes) that the database treats as a single logical unit of work. The **ACID** properties — Atomicity, Consistency, Isolation, Durability — define what guarantees a transactional database provides for each unit of work.

### Atomicity

**Atomicity** means that all operations within a transaction either all succeed or all fail together. There is no partial commit — no state where half the operations took effect and half did not. If a bank transfer involves two steps — debit account A, credit account B — atomicity guarantees that if the credit fails for any reason (network error, constraint violation, system crash), the debit is automatically rolled back. The database returns to its state before the transaction began.

Atomicity is enforced through the **Write-Ahead Log** (covered in Chapter 3): the engine writes all changes to the WAL before applying them to data pages, and uses the WAL to undo incomplete transactions on recovery.

### Consistency (ACID)

**Consistency** (the ACID sense, distinct from CAP consistency) means that a transaction takes the database from one valid state to another valid state. "Valid" means all declared constraints are satisfied — primary key uniqueness, foreign key referential integrity, check constraints, not-null constraints. A transaction that would violate any constraint is aborted entirely. The database never exposes a state that violates its declared integrity rules.

Note the responsibility boundary: the database enforces *declared* constraints. Correctness of business logic (e.g., that a product price is never negative) is the application's responsibility unless explicitly declared as a CHECK constraint.

### Isolation

**Isolation** means that concurrent transactions do not interfere with each other. Each transaction sees a consistent view of the database as if it were running alone. In practice, full isolation is expensive — it is relaxed through **isolation levels** that trade isolation guarantees for concurrency and performance. The isolation levels are the most nuanced and operationally consequential part of the ACID model.

### Durability

**Durability** means that once a transaction is committed, its changes are permanent. They survive crashes, power failures, and restarts. Durability is achieved by forcing the WAL record of a commit to stable storage (disk, SSD, or battery-backed RAM) before acknowledging success to the client. The `fsync` system call is the mechanism; skipping it for performance is the source of many dramatic data loss incidents.

#### Diagram: ACID Bank Transfer Atomicity Visualizer

<details markdown="1">
<summary>ACID Atomicity Visualizer — Bank Transfer Failure Scenario</summary>
Type: MicroSim
**sim-id:** acid-atomicity-visualizer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Demonstrate atomicity by showing how a partial failure in a multi-step transaction is automatically rolled back, leaving the database in its original state. (Bloom L2: Understand)

**Canvas:** 760px wide × 440px tall. CANVAS_HEIGHT: 440.

**Description:**
Two bank account panels: Account A (left, balance $1,000) and Account B (right, balance $500). A transaction panel in the center shows two steps: Step 1 "Debit A: -$300" and Step 2 "Credit B: +$300."

**Happy path animation:** "Run Transaction" button. Step 1 executes: Account A balance animates to $700 (green flash). Step 2 executes: Account B balance animates to $800 (green flash). "COMMIT" label appears. Both balances stay updated.

**Failure path animation:** "Inject Failure at Step 2" button. Step 1 executes: Account A animates to $700. Step 2 fails: Account B shows a red X (constraint violation). A "ROLLBACK" label appears. Account A animates back to $1,000 (orange flash). "Transaction rolled back — database returned to consistent state."

**Without ACID toggle:** Shows what happens without atomicity — Account A stays at $700 even after Step 2 fails. "Money disappeared." Red warning banner.

**Responsive:** Redraws on window resize.
</details>

## Transaction Isolation Levels and Concurrency Anomalies

The SQL standard defines four isolation levels, each preventing a different set of concurrency anomalies. Before examining the levels, define the three anomalies they protect against.

A **dirty read** occurs when a transaction reads a value written by another transaction that has not yet committed. If the writing transaction later rolls back, the reading transaction has seen data that never officially existed.

A **non-repeatable read** occurs when a transaction reads the same row twice and gets different values because another transaction modified and committed that row between the two reads. The row exists in both reads — it just changed.

A **phantom read** occurs when a transaction re-executes a query and finds different rows — not because existing rows changed, but because another transaction inserted or deleted rows that match the query's predicate.

With those definitions in hand, the four isolation levels are:

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Typical Use |
|----------------|-------------|---------------------|---------------|-------------|
| Read Uncommitted | Possible | Possible | Possible | Rarely used; analytics where staleness is acceptable |
| Read Committed | Prevented | Possible | Possible | Default in PostgreSQL; most OLTP workloads |
| Repeatable Read | Prevented | Prevented | Possible | Default in MySQL; inventory, financial reporting |
| Serializable | Prevented | Prevented | Prevented | Financial transactions, anything requiring full isolation |

**Read Uncommitted** allows transactions to read uncommitted changes from other transactions. This is the weakest level and is almost never correct for transactional workloads.

**Read Committed** ensures a transaction only sees committed data. Each read within a transaction sees the latest committed value at the moment of that read. This is the practical default for most OLTP applications.

**Repeatable Read** adds the guarantee that if you read a row, you will get the same value every time you read it within the same transaction. In PostgreSQL's implementation, this is achieved via MVCC snapshots. In MySQL, it is the default isolation level.

**Serializable Isolation** is the strongest level: transactions behave exactly as if they ran one at a time in some serial order. No concurrency anomalies of any kind are possible. PostgreSQL implements this via Serializable Snapshot Isolation (SSI), a highly concurrent implementation that detects and aborts only conflicting transactions.

#### Diagram: Isolation Level Anomaly Explorer

<details markdown="1">
<summary>Interactive Isolation Level Anomaly Explorer</summary>
Type: MicroSim
**sim-id:** isolation-level-explorer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Apply isolation level knowledge to predict which concurrency anomalies can occur at each level, and identify the correct isolation level for a given business requirement. (Bloom L3–L4: Apply/Analyze)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
Four tabs across the top: Read Uncommitted | Read Committed | Repeatable Read | Serializable.

Each tab shows a two-transaction timeline diagram (Transaction A on top lane, Transaction B on bottom lane) with time flowing left to right.

**Read Uncommitted tab:** Transaction B writes balance = $200 (uncommitted). Transaction A reads $200. Transaction B rolls back. Transaction A displays stale $200. Red badge: "Dirty Read occurred."

**Read Committed tab:** Transaction B writes $200 (uncommitted). Transaction A attempts to read — blocked until B commits or rolls back. B commits $200. A reads $200. No dirty read. But if A reads again after B makes another change, A gets a different value. Orange badge: "Non-Repeatable Read possible."

**Repeatable Read tab:** Transaction A reads row at t=1 (value $100). Transaction B updates and commits (value $200). Transaction A reads same row at t=3 — still sees $100 (snapshot from t=1). Green badge: "Repeatable reads guaranteed." But a range query can return new rows if B inserts: Orange badge: "Phantom Reads still possible."

**Serializable tab:** All anomalies prevented. Shows SSI detecting and aborting a conflicting transaction rather than allowing anomalies.

Each tab has a "What business scenario requires this?" sidebar.

**Responsive:** Redraws on window resize.
</details>

## Two-Phase Locking

**Two-Phase Locking (2PL)** is the pessimistic concurrency control protocol that achieves serializability through lock acquisition discipline. The protocol has two phases:

1. **Growing phase:** The transaction acquires all the locks it needs (shared locks for reads, exclusive locks for writes). During this phase it may acquire but not release locks.
2. **Shrinking phase:** After the transaction reaches its lock point (typically when it commits or aborts), it releases all locks. During this phase it may not acquire new locks.

The name "two-phase" refers to these two phases — not to two-phase commit (a different protocol covered in Chapter 12).

**Strict 2PL** (the most common variant) holds all locks until the transaction commits or aborts, eliminating cascading aborts (a transaction reading uncommitted data that is later rolled back). Most relational databases implement Strict 2PL.

The cost of 2PL is contention: long-running transactions hold locks that block other transactions. This is the primary motivation for MVCC, which allows readers to proceed without blocking writers.

!!! mascot-tip "When 2PL Meets 2PL: Deadlocks"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex offers a tip">
    Two-phase locking creates deadlock risk whenever two transactions acquire locks in different orders. Transaction A holds lock on row 1 and wants row 2; Transaction B holds row 2 and wants row 1. The database detects this by scanning the wait-for graph and rolls back the victim. Application code must be written to catch deadlock errors (PostgreSQL error code 40P01) and retry. Order your lock acquisitions consistently to minimize deadlock frequency.

## Optimistic Concurrency Control

**Optimistic Concurrency Control (OCC)** takes the opposite philosophy from 2PL: instead of preventing conflicts by acquiring locks upfront, OCC allows transactions to proceed without locks and detects conflicts at commit time.

An OCC transaction proceeds in three phases:

1. **Read phase:** The transaction reads data and maintains a private read set and write set (buffered changes not yet applied).
2. **Validation phase:** At commit time, the engine checks whether any data in the read set was modified by another committed transaction since the current transaction started. If yes, there is a conflict.
3. **Write phase:** If validation succeeds, the buffered writes are applied and the transaction commits. If validation fails, the transaction is aborted and the application retries.

OCC performs well when conflicts are rare — most transactions complete without any conflict, and the overhead of locking is avoided entirely. OCC performs poorly when conflicts are frequent, because aborted transactions waste work. PostgreSQL's MVCC implementation borrows OCC ideas for its Serializable Snapshot Isolation mode.

## Savepoints and Rollback

A **savepoint** is a named checkpoint within a transaction that allows partial rollback without aborting the entire transaction. After establishing a savepoint with `SAVEPOINT sp1`, subsequent operations can be undone back to `sp1` with `ROLLBACK TO SAVEPOINT sp1`, while preserving all work done before `sp1`.

Savepoints are implemented via the WAL: the undo log records every change, and rolling back to a savepoint replays the undo log from the current position back to the savepoint marker.

A **rollback** (without a savepoint target) undoes all changes since the transaction began and returns the database to its state before the `BEGIN`. Rollbacks are triggered explicitly by the application, automatically on error (depending on database configuration), or by the engine when a deadlock victim is selected.

The **commit protocol** is the sequence of steps the engine takes to make a transaction permanent: flush the transaction's WAL records to disk (fsync), write the commit record to the WAL, then release locks. Only after the commit record is durably written is the transaction considered committed.

#### Diagram: Savepoint and Rollback Flow

<details markdown="1">
<summary>Savepoint and Partial Rollback Visualizer</summary>
Type: MicroSim
**sim-id:** savepoint-rollback-flow<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Demonstrate how savepoints allow partial rollback within a transaction, preserving earlier work while undoing later steps. (Bloom L3: Apply)

**Canvas:** 780px wide × 420px tall. CANVAS_HEIGHT: 420.

**Description:**
A horizontal timeline of operations within a single transaction: BEGIN → Op1 (INSERT order) → SAVEPOINT sp1 → Op2 (INSERT line_item 1) → Op3 (INSERT line_item 2) → SAVEPOINT sp2 → Op4 (UPDATE inventory, fails) → ROLLBACK TO sp1 → Op5 (retry line_items differently) → COMMIT.

Each operation is a node on the timeline. Savepoints are shown as vertical flags. Rolled-back operations (Op2, Op3, Op4) are grayed out with strikethrough after the ROLLBACK TO sp1. Op1 remains in effect (green). Op5 onwards re-execute on the clean state after sp1.

**Interactions:**
- Hovering any operation shows what changed: "Op1: Inserted order row id=1001."
- Clicking ROLLBACK TO sp1 animates the undo: Op2, Op3, Op4 flash red and gray out sequentially.
- A "Database State" panel on the right updates after each step, showing what rows currently exist.

**Full Rollback button:** Shows what happens with ROLLBACK (no savepoint) — all operations undone, database returns to pre-BEGIN state.

**Responsive:** Redraws on window resize.
</details>

## Distributed Sagas: ACID for Microservices

Distributed ACID transactions — where a single transaction spans multiple databases or microservices — require a protocol like Two-Phase Commit (covered in Chapter 12). 2PC adds significant latency and coordination overhead. For microservices architectures where services are independently deployed and own their own databases, 2PC is often impractical.

The **distributed saga** pattern is the primary alternative. A saga is a sequence of local transactions, one per service, where each local transaction updates its own database and publishes an event or message triggering the next step. If any step fails, the saga executes **compensating transactions** — operations that semantically undo the effect of the already-committed local transactions.

Two coordination styles exist:

**Saga Orchestration** uses a central **saga orchestrator** that sends commands to each service and awaits responses. The orchestrator knows the full sequence and compensation plan. If Step 3 fails, the orchestrator explicitly commands Steps 2 and 1 to compensate.

**Saga Choreography** has no central coordinator. Each service listens for events from the previous step, executes its local transaction, and publishes an event for the next step. Compensation is triggered by failure events. There is no single point of control — and no single point of failure.

Sagas do NOT provide isolation. Between steps, the intermediate state is visible to other transactions (the order exists but payment has not yet been confirmed). Applications using sagas must be designed to handle this — using techniques like "pending" states, idempotent operations, and careful UI design that does not expose partial saga state to end users.

#### Diagram: Saga Orchestration vs Choreography

<details markdown="1">
<summary>Saga Orchestration vs. Choreography — Interactive Flow Comparison</summary>
Type: MicroSim
**sim-id:** saga-pattern-comparison<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Compare saga orchestration and choreography patterns, identify how compensating transactions restore consistency after partial failure, and select the appropriate coordination style for a given microservices scenario. (Bloom L4–L5: Analyze/Evaluate)

**Canvas:** 780px wide × 520px tall. CANVAS_HEIGHT: 520.

**Description:**
Two side-by-side panels: "Orchestration" (left) and "Choreography" (right).

**Orchestration panel:** A central Orchestrator node at top. Three service nodes below: Order Service, Payment Service, Inventory Service. Arrows flow from Orchestrator → each service with command labels ("PlaceOrder", "ChargeCard", "ReserveInventory"). Success arrows return to Orchestrator.

**Choreography panel:** No central node. Order Service → publishes "OrderPlaced" → Payment Service listens → publishes "PaymentCharged" → Inventory Service listens → publishes "InventoryReserved". Arrows are event-driven, flowing horizontally.

**Failure injection:** A "Fail Payment" button triggers a failure in the Payment Service in both panels. In the Orchestration panel, the Orchestrator receives a failure, sends "CancelOrder" compensation command back to Order Service (red arrow). In the Choreography panel, Payment Service publishes "PaymentFailed" event, Order Service listens and self-compensates.

**Animation:** Step-by-step playback with 800ms per step. Each step highlights the active service and the message flowing.

**Responsive:** Redraws on window resize.
</details>

## The ATAM Lens: Choosing the Right Isolation Level

Isolation levels are a classic ATAM sensitivity point: the right choice depends entirely on the quality attribute scenarios the system must satisfy.

A utility tree for a payment processing system should include a scenario like: "When two users attempt to purchase the last unit of inventory simultaneously, exactly one transaction must succeed and the other must receive a stock-out error." This scenario requires Serializable isolation — no weaker level prevents the phantom read that would allow both transactions to see the item as available.

A utility tree for an analytics dashboard might include: "Reports may reflect data up to 60 seconds old." This scenario permits Read Committed or even eventually consistent reads, dramatically reducing lock contention and improving report query throughput.

!!! mascot-warning "The Default Isolation Level Trap"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex warns">
    PostgreSQL's default is Read Committed. MySQL's default is Repeatable Read. Neither default is Serializable. Most applications that believe they have "full ACID" actually have Read Committed, which permits non-repeatable reads and phantom reads. For most OLTP workloads, this is fine — but for inventory management, financial accounting, and any scenario where the same data is read twice within a transaction and acted upon, it is not. Verify the isolation level your application actually uses before shipping.

## Key Takeaways

ACID is not a binary property — databases have it or they don't. It is a spectrum of four guarantees, each independently configurable, each with operational costs and benefits.

- **Atomicity** — all-or-nothing; no partial commits; enforced via WAL undo
- **Consistency (ACID)** — constraint satisfaction on every commit; the database never exposes invalid state
- **Isolation** — four levels from Read Uncommitted to Serializable; each prevents a different set of anomalies
- **Durability** — committed data survives crashes; requires fsync to stable storage
- **Dirty read / Non-repeatable read / Phantom read** — the three anomaly types; know which isolation level prevents each
- **Two-Phase Locking** — pessimistic; holds locks until commit; prevents conflicts proactively
- **MVCC / Optimistic CC** — no reader-writer blocking; detects conflicts at commit time
- **Distributed Saga** — ACID alternative for microservices; compensating transactions replace rollback; no isolation between steps

!!! mascot-celebration "Chapter 10 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now have the full ACID toolkit — not as a marketing claim, but as a set of precise, independently configurable guarantees. The next two chapters take these foundations into distributed territory: how do you scale databases horizontally while keeping replication consistent (Chapter 11), and how do you achieve full ACID across a distributed cluster (Chapter 12)? The answers build directly on what you just learned.
