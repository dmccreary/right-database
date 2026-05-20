---
title: Chapter 12 - Distributed ACID and NewSQL
description: How modern NewSQL databases achieve full ACID compliance across distributed clusters using two-phase commit, saga patterns, and Paxos/Raft consensus.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 12: Distributed ACID and NewSQL

## Summary

This chapter examines how modern NewSQL databases achieve full ACID compliance across distributed clusters — solving the problem that traditional relational databases could not scale and traditional NoSQL databases sacrificed transactions. Students learn two-phase commit and its failure modes, the saga pattern for long-running distributed transactions, and how NewSQL products such as Google Spanner, CockroachDB, and YugabyteDB use Paxos/Raft consensus to deliver global transactions with horizontal scalability. The ZAB protocol (used by Apache ZooKeeper) is also covered as a practical consensus variant.

## Concepts Covered

This chapter covers the following 14 concepts from the learning graph:

1. Two-Phase Commit
2. Three-Phase Commit
3. Distributed Transaction Coordinator
4. Saga Orchestration
5. Saga Choreography
6. Compensating Transaction
7. NewSQL Database
8. Google Spanner
9. CockroachDB
10. YugabyteDB
11. TrueTime API
12. Global Transaction ID
13. Cross-Shard Transaction
14. ZAB Protocol

## Prerequisites

This chapter builds on concepts from:

- [Chapter 9: CAP Theorem and Consensus Protocols](../09-cap-theorem-consensus/index.md)
- [Chapter 10: ACID Transactions](../10-acid-transactions/index.md)
- [Chapter 11: Distributed Scaling and Replication](../11-distributed-scaling/index.md)

---

!!! mascot-welcome "The Promise and the Problem"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex the robot welcoming students">
    For a decade, the NoSQL movement told us a story: if you want scale, you must sacrifice transactions. Give up ACID. Embrace eventual consistency. Stop joining tables. Dex is pleased to report that the story had a sequel. NewSQL databases arrived and said: "Actually, you can have both." This chapter explains how — and why it costs more than the brochure suggests.

## The Core Tension: Transactions vs. Distribution

ACID transactions (covered in Chapter 10) are elegant and precise in a single-node world. The database engine can lock rows, buffer writes in a journal, and commit or roll back with complete confidence that no other process will observe a partial state.

Distribute data across multiple nodes and the problem compounds. A **cross-shard transaction** — a transaction that reads or writes data living on more than one shard — requires coordination across machines that communicate over a network. Networks are unreliable. Nodes crash. Clocks drift. How do you guarantee atomicity — either all nodes commit the transaction or none do — when any participant could fail mid-operation?

This is the problem that **Two-Phase Commit** was designed to solve.

## Two-Phase Commit: The Classic Protocol

**Two-Phase Commit (2PC)** is a distributed algorithm that ensures all participants in a transaction either commit or abort together. It involves a **Distributed Transaction Coordinator** — typically a dedicated node or process that orchestrates the protocol across all participating nodes.

Phase 1 is the **voting phase** (also called the prepare phase):

1. The coordinator sends a `PREPARE` message to each participant.
2. Each participant checks whether it can commit: it acquires locks, writes a prepare record to its write-ahead log, and replies either `VOTE-COMMIT` or `VOTE-ABORT`.
3. If any participant votes to abort (because of a constraint violation, a lock timeout, or any other reason), the coordinator will abort the transaction.

Phase 2 is the **commit phase**:

1. If all participants voted commit, the coordinator sends `COMMIT` to all participants. If any participant voted abort, the coordinator sends `ABORT` to all participants.
2. Each participant applies or rolls back the transaction, releases its locks, and acknowledges.

2PC guarantees atomicity: no node commits unless all nodes agreed. The protocol is used in every major relational database that supports distributed transactions — including Oracle, SQL Server, and PostgreSQL's pg_dist_transaction (via Citus).

#### Diagram: Two-Phase Commit Flow

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** two-phase-commit-flow
**Library:** p5.js
**Status:** Specified

Show a vertical swim-lane diagram with 4 columns: Coordinator, Participant A, Participant B, Participant C.

Provide four scenario buttons at the top: "Happy Path," "Participant Crash in Phase 1," "Coordinator Crash After Prepare," "Participant Timeout in Phase 2."

For Happy Path: animate message arrows flowing down (Prepare →), then voting arrows flowing up (Vote Commit ←), then commit arrows flowing down (Commit →), then acknowledgment arrows flowing up (ACK ←). Color each phase distinctly (blue for Phase 1, green for Phase 2).

For "Coordinator Crash After Prepare": show Prepare messages sent, then the Coordinator column goes dark (red X). Show Participants A, B, C stuck in a "waiting" state, displayed as spinning clocks. Show a timer counting up. Label: "Participants are blocked — they cannot commit or abort without the coordinator." Show a "Recovery" button that brings a new coordinator online to complete the commit.

For each scenario, show a timeline at the bottom and a label indicating whether the transaction ultimately committed or aborted.
</details>

### The Blocking Problem

2PC has a well-known Achilles' heel: it is a **blocking protocol**. During Phase 2, participants that voted to commit hold their locks while waiting for the coordinator's decision. If the coordinator crashes after sending PREPARE but before sending COMMIT or ABORT, the participants are stuck. They have voted to commit but cannot know whether to proceed. They must hold their locks indefinitely — blocking all concurrent transactions that touch the same rows — until the coordinator recovers.

In practice, coordinators recover quickly (minutes in most deployments), but the theoretical blocking window is unbounded. This failure mode is why 2PC is called a "blocking" protocol and why engineers with long memories are wary of it.

### Three-Phase Commit: Adding a Safety Net

**Three-Phase Commit (3PC)** attempts to address the blocking problem by inserting an additional phase between the vote and the final commit:

1. **Voting phase** — same as 2PC.
2. **Pre-commit phase** — the coordinator sends a `PRE-COMMIT` message if all participants voted commit. Participants acknowledge. The coordinator now knows that all participants know a commit is coming.
3. **Commit phase** — the coordinator sends `COMMIT`. Participants apply and acknowledge.

The key insight: after receiving PRE-COMMIT, a participant that cannot reach the coordinator can safely commit on its own (it knows all peers also voted commit). This eliminates the blocking window. The tradeoff is an additional round of network communication and the assumption that network partitions are fail-stop — a property that real networks do not guarantee. For these reasons, 3PC is rarely implemented in practice.

## The Saga Pattern: Long-Running Transactions Without Locking

Two-Phase Commit works well for short transactions (milliseconds to seconds) but is unsuitable for business processes that span multiple services and take minutes or hours. Holding locks across a hotel booking, a payment processing step, and an airline reservation API call for 30 seconds is a non-starter.

The **saga pattern** decomposes a long-running business transaction into a sequence of local transactions, each with a corresponding **compensating transaction** that semantically undoes it. If step N fails, the saga executes the compensating transaction for step N-1, then N-2, and so on, unwinding the completed work.

A compensating transaction is not a rollback in the traditional database sense. The individual local transactions have already committed. A compensating transaction is a new write operation that undoes the business effect of a prior step — for example, issuing a refund to cancel a payment, or releasing a held hotel room.

There are two architectural approaches to saga coordination:

**Saga Orchestration** uses a central orchestrator — a dedicated process or service — that explicitly directs each step of the saga and decides what compensating actions to take on failure. The orchestrator knows the full workflow and tracks its state in a persistent store. This approach is easy to debug (the orchestrator's state log tells you exactly where things went wrong) but creates a centralized dependency.

**Saga Choreography** uses no central orchestrator. Instead, each service listens for events from the previous step and emits events for the next step. If a step fails, the service emits a failure event, and earlier services react by executing their compensating transactions. This is more loosely coupled but harder to debug — the saga's state is distributed across multiple services' event logs.

#### Diagram: Saga Orchestration vs Choreography

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** saga-orchestration-vs-choreography
**Library:** p5.js
**Status:** Specified

Show a toggle switch labeled "Orchestration / Choreography" at the top.

**Orchestration view:** Show a central "Saga Orchestrator" box in gold. Draw three service boxes: Order Service, Payment Service, Inventory Service. Show arrows from Orchestrator → Order Service (Create Order), then → Payment Service (Charge Card), then → Inventory Service (Reserve Items). Include a "Inject Failure" button that fails the Inventory step, triggering animated compensation arrows in red flowing back: Inventory (skipped), then Payment Service ← Orchestrator (Refund Card), then Order Service ← Orchestrator (Cancel Order). Show each step's status: Completed (green), Compensated (orange), Failed (red).

**Choreography view:** Remove the Orchestrator box. Show the same three services connected by an event bus (horizontal line). Events flow rightward as labeled arrows: "OrderCreated" → Payment Service, "PaymentCharged" → Inventory Service. "Inject Failure" emits "InventoryFailed" event flowing leftward; Payment Service receives it and emits "PaymentRefunded" flowing back to Order Service.

Show a summary: "Orchestration: easier to debug, central dependency. Choreography: loosely coupled, harder to trace."
</details>

!!! mascot-thinking "When Compensation Is Not Possible"
    <img src="../../img/mascot/thinking.png" class="mascot-admonition-img" alt="Dex the robot in a thoughtful pose">
    Some business operations cannot be compensated. You cannot un-send an email, un-fire a missile, or un-ring a doorbell. When designing sagas, identify these **pivot transactions** — operations with irreversible real-world effects — and place them as late in the saga as possible. Everything before the pivot can be compensated; everything after it must succeed.

## NewSQL: The Third Way

The NoSQL movement of the 2010s traded ACID for scale. The NewSQL movement of the 2010s said: we want both.

A **NewSQL database** is a relational database that delivers SQL query compatibility and ACID transactions while also providing horizontal scalability across distributed clusters. NewSQL databases achieve this through distributed consensus protocols (Paxos or Raft) that replace the single-node transaction manager of traditional RDBMS systems.

The three most prominent NewSQL databases are:

| Database | Consensus | SQL Compatibility | Open Source | Notable Feature |
|---|---|---|---|---|
| Google Spanner | Paxos | ANSI SQL, Google SPQL | No (GCP only) | TrueTime global ordering |
| CockroachDB | Raft | PostgreSQL wire protocol | Yes (BSL license) | Geo-partitioning, survivability goals |
| YugabyteDB | Raft | PostgreSQL + Cassandra APIs | Yes (Apache 2.0) | Dual API: YSQL + YCQL |

### Google Spanner

**Google Spanner** is Google's globally distributed relational database, introduced in a landmark 2012 paper and available externally through Google Cloud Spanner. Spanner achieves external consistency — a guarantee stronger than serializable isolation — across data centers on multiple continents.

The mechanism behind this achievement is the **TrueTime API**, a Google-internal service that exposes the current time not as a single timestamp but as an interval `[earliest, latest]` bounded by measured clock uncertainty. Spanner's commit protocol waits for the uncertainty interval to pass before considering a transaction committed — a technique called **commit wait**. This ensures that any transaction that commits later in real time will also have a higher timestamp, enabling global serializable ordering even across data centers with no shared clock.

Each Spanner **spanserver** (shard) runs an independent Paxos group. A **Global Transaction ID** uniquely identifies each transaction across all participating Paxos groups. Cross-shard transactions use a 2PC-like protocol with Paxos groups acting as participants — but unlike traditional 2PC, the coordinator is itself replicated via Paxos, eliminating the single-coordinator failure mode.

#### Diagram: TrueTime Commit Wait

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** truetime-commit-wait
**Library:** p5.js
**Status:** Specified

Show a horizontal timeline. A transaction T1 arrives, acquires a TrueTime timestamp showing an interval [T-ε, T+ε] (uncertainty window). Show a shaded region on the timeline representing this interval.

Animate a "commit wait" period: the transaction is marked "waiting" until the real-world clock has advanced past T+ε. Then the transaction commits and its commit timestamp is marked as a vertical line on the timeline.

Show a second transaction T2 arriving after T1's commit. Its TrueTime interval starts after T1's commit line. Label: "T2 will always receive a timestamp > T1's commit timestamp."

Show a slider for "Clock Uncertainty (ε): 1ms – 7ms" — as uncertainty increases, the commit-wait period visibly extends. Add a label: "Typical Spanner ε = 1-7ms. Commit wait adds this latency to every transaction write."

Add a counter showing effective write latency = base latency + 2ε.
</details>

### CockroachDB

**CockroachDB** is an open-source NewSQL database built by Cockroach Labs, designed to survive data center failures (hence the name — cockroaches are famously difficult to kill). It uses Raft consensus per range (roughly equivalent to a shard), presents a PostgreSQL-compatible wire protocol, and supports serializable isolation by default.

CockroachDB's approach to global transaction ordering uses hybrid logical clocks (HLCs) rather than TrueTime. HLCs combine a physical timestamp with a logical counter, ensuring that causal relationships between events are preserved even when physical clocks are unsynchronized. Transactions that span multiple Raft ranges go through a coordinator that manages the Raft-based 2PC protocol across the involved ranges.

A distinctive CockroachDB feature is **survivability goals**: operators can declare that data must survive a node failure, a rack failure, or a data center failure, and CockroachDB will enforce the replication topology accordingly.

### YugabyteDB

**YugabyteDB** is an open-source NewSQL database with a distinctive dual-API design: YSQL (PostgreSQL-compatible) and YCQL (Cassandra-compatible). This makes it an attractive migration target for teams moving from either ecosystem. Like CockroachDB, it uses Raft consensus per shard (called "tablets") and supports serializable isolation.

YugabyteDB separates storage from query processing through its DocDB storage layer, which is based on RocksDB. This architecture allows it to serve both relational and wide-column workloads from the same storage engine, which is how the dual API becomes practical.

## The ZAB Protocol

No treatment of distributed coordination would be complete without the **ZAB protocol** (ZooKeeper Atomic Broadcast), the consensus algorithm that powers Apache ZooKeeper. ZAB is a Paxos variant designed specifically for the primary-backup model — it ensures that writes are totally ordered and that all nodes agree on the same history, even after leader failures.

ZAB operates in two modes:

**Recovery mode** runs at startup or after a leader failure. The new leader broadcasts its complete transaction history to all followers. A follower only begins serving requests after it has received and acknowledged the full history from the new leader. This ensures no committed transaction is ever lost.

**Broadcast mode** is the normal operating mode. The leader receives writes, assigns them monotonically increasing **zxids** (ZooKeeper transaction IDs), and broadcasts them to followers. A write is committed once a quorum of followers have acknowledged it. All reads in ZooKeeper are served locally by the follower without broadcasting — making reads fast but potentially slightly stale.

The key difference between ZAB and Raft is design intent. Raft was designed for pedagogical clarity and general use. ZAB was designed specifically for the ZooKeeper use case: high-read, low-write, total-order broadcast. Both deliver correct consensus; the implementation details differ.

!!! mascot-tip "Cross-Shard Transactions Are Expensive"
    <img:src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex the robot giving a helpful tip">
    Every cross-shard transaction adds at least one additional network round-trip compared to a single-shard transaction. In practice, design your shard key so that your most common transactions touch only one shard. The 80/20 rule applies: if 80% of your transactions are single-shard, the 20% cross-shard penalty is manageable. If 80% are cross-shard, you have chosen the wrong shard key.

## Choosing a Distributed Transaction Strategy

The choice between 2PC, sagas, and NewSQL depends on your consistency requirements and operational tolerance:

| Scenario | Recommended Approach | Reason |
|---|---|---|
| Financial ledger requiring strict ACID across 3 services | NewSQL (CockroachDB/Spanner) | True distributed ACID without saga complexity |
| Long-running booking workflow (hotel + flight + car) | Saga with Orchestration | Steps span services; compensation is defined |
| Microservices with loose coupling requirements | Saga with Choreography | No central coordinator; services stay independent |
| Legacy PostgreSQL system needing distributed writes | Citus + 2PC | Familiar tooling; manage blocking risk operationally |
| Global app requiring < 10ms reads everywhere | Google Spanner | TrueTime enables strong consistency at global scale |

The dirty secret of distributed transactions is that the best strategy is often **avoid them**. If you can design your data model so that nearly all transactions are local to a single shard, you eliminate the coordination problem entirely. Domain-driven design's concept of **aggregate boundaries** — where one aggregate owns its data and changes within it are always local — is directly applicable here.

## Key Takeaways

- **Two-Phase Commit** guarantees atomicity across distributed participants but is a blocking protocol — a coordinator failure can leave participants stuck holding locks.
- **Three-Phase Commit** adds a pre-commit phase to reduce blocking but introduces additional network overhead and relies on fail-stop assumptions rarely true in practice.
- **Sagas** decompose long-running transactions into local steps with compensating transactions; orchestration provides centralized control, choreography provides loose coupling.
- A **compensating transaction** is not a rollback — it is a new business operation that semantically undoes a prior committed step.
- **NewSQL databases** (Spanner, CockroachDB, YugabyteDB) deliver horizontal scalability and distributed ACID through Paxos/Raft consensus, eliminating the need for application-level saga logic in many cases.
- **Google Spanner's TrueTime API** uses bounded clock uncertainty and commit wait to achieve external consistency across global data centers.
- **CockroachDB** uses Raft and hybrid logical clocks; it is PostgreSQL-compatible and open source.
- **ZAB** (ZooKeeper Atomic Broadcast) is a Paxos variant optimized for ZooKeeper's high-read, low-write, total-order-broadcast use case.
- The best distributed transaction strategy is often designing shard keys so that common transactions are single-shard — avoiding the coordination problem entirely.

!!! mascot-celebration "Distributed ACID: Achieved"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex the robot celebrating">
    You have survived two-phase commit, saga compensation, TrueTime, Raft, ZAB, and the subtle art of blaming the coordinator. In Chapter 13, Dex turns from transaction correctness to system survival: high availability, five-nines uptime, and the chaos engineering mindset that keeps production from becoming a horror story.
