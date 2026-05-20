---
title: Chapter 9 - CAP Theorem and Consensus Protocols
description: The theoretical foundations of distributed databases — CAP theorem, PACELC, the consistency spectrum, BASE semantics, vector clocks, and the Paxos and Raft consensus protocols.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 9: CAP Theorem and Consensus Protocols

## Summary

This chapter introduces the theoretical foundations that govern all distributed database behavior. Students learn the CAP theorem and its practical refinement, the PACELC model, and work through the full spectrum of consistency models from eventual consistency to linearizability. The chapter also covers the consensus protocols — Paxos and Raft — that underpin distributed agreement, laying the groundwork for leader election, replication, and NewSQL transactions covered in later chapters.

## Concepts Covered

This chapter covers the following 21 concepts from the learning graph:

1. CAP Theorem
2. Consistency (CAP)
3. Availability (CAP)
4. Partition Tolerance
5. CP Database System
6. AP Database System
7. PACELC Model
8. Latency-Consistency Tradeoff
9. Eventual Consistency
10. Strong Consistency
11. Read-Your-Writes Consistency
12. Monotonic Read Consistency
13. Causal Consistency
14. Session Consistency
15. Linearizability
16. Serializability
17. BASE Semantics
18. Conflict Resolution
19. Vector Clock
20. Paxos Protocol
21. Raft Consensus Protocol

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)

---

!!! mascot-welcome "Welcome to Chapter 9!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    This chapter is where distributed systems theory meets practical database engineering. The CAP theorem is one of the most cited results in distributed computing — and one of the most misunderstood. By the end of this chapter you will be able to apply CAP and PACELC correctly, reason through the consistency spectrum with precision, and explain how consensus protocols like Raft make distributed databases possible. This vocabulary appears in every architecture review where a distributed database is on the table.

## The CAP Theorem

In 2000, Eric Brewer proposed the conjecture that became known as the **CAP theorem**: a distributed data store can guarantee at most two of three properties simultaneously — **Consistency**, **Availability**, and **Partition Tolerance**. Gilbert and Lynch formally proved it in 2002.

Before the theorem becomes useful, each term needs a precise definition — everyday language meanings will mislead you.

**Consistency (CAP)** means that every read receives the most recent write or an error. Every node in the cluster agrees on the current value of every piece of data. A client reading from any node sees a consistent, up-to-date view. Note: this is *not* the same as the "C" in ACID (which means constraint satisfaction).

**Availability (CAP)** means that every request receives a non-error response — though that response may not reflect the most recent write. The system is always willing to answer, even if its answer is slightly stale.

**Partition Tolerance** means the system continues to operate despite an arbitrary number of messages being dropped or delayed between nodes. A network partition is a real-world failure where some nodes can no longer communicate with others.

The critical insight is that partition tolerance is not optional for any distributed system deployed in production. Networks fail. Messages get dropped. Any system that claims to sacrifice partition tolerance is either running on a single node (not distributed) or claiming something it cannot guarantee. This means the real choice in the CAP theorem is: **when a partition occurs, do you sacrifice Consistency (becoming an AP system) or Availability (becoming a CP system)?**

- A **CP database system** responds to partitions by refusing reads and writes on nodes that cannot confirm they have the latest data — preserving consistency at the cost of availability. HBase, ZooKeeper, and etcd are CP systems.
- An **AP database system** responds to partitions by continuing to serve reads and writes, accepting that different nodes may return different values until the partition heals — preserving availability at the cost of consistency. Cassandra, CouchDB, and DynamoDB (with default settings) are AP systems.

#### Diagram: CAP Theorem Triangle Explorer

<details markdown="1">
<summary>Interactive CAP Theorem Triangle Explorer</summary>
Type: MicroSim
**sim-id:** cap-theorem-triangle<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify where major database systems fall on the CAP triangle and explain the consistency/availability tradeoff during a network partition. (Bloom L1–L2: Remember/Understand)

**Canvas:** 700px wide × 520px tall. CANVAS_HEIGHT: 520.

**Description:**
Draw an equilateral triangle with vertices labeled Consistency (top), Availability (bottom-left), and Partition Tolerance (bottom-right). Shade the three edges with subtle gradients. Position database product logos or name badges near their CAP position:
- CP edge (Consistency + Partition Tolerance): ZooKeeper, HBase, etcd, MongoDB (default)
- AP edge (Availability + Partition Tolerance): Cassandra, CouchDB, DynamoDB (eventual)
- CA edge (Consistency + Availability, no partition tolerance): traditional single-node RDBMS (labeled "not distributed")

**Interactions:**
- Hovering a database name shows a tooltip: "Cassandra: AP system. During a partition, Cassandra continues serving reads and writes but nodes may temporarily diverge. Consistency is restored after partition heals."
- A "Simulate Partition" button animates a red lightning bolt across the center of the triangle. The CP region grays out (databases in this region show "Unavailable"). The AP region stays lit (databases show "Serving stale data").
- A "Heal Partition" button removes the partition and shows AP databases converging (animated sync arrows between nodes).

**Responsive:** Redraws on window resize.
</details>

## The PACELC Model

The CAP theorem is correct but incomplete as a practical design tool. It only tells you what happens *during* a partition — a relatively rare event. Most of the time, a distributed database runs without partitions, and you still face tradeoffs. The **PACELC model** (Daniel Abadi, 2012) extends CAP to cover the normal case.

PACELC states: if there is a Partition (P), the system must choose between Availability (A) and Consistency (C); **else** (E), during normal operation, the system must choose between Latency (L) and Consistency (C).

The latency-consistency tradeoff during normal operation is, in practice, more consequential than the partition behavior. Every read or write operation in a distributed system involves a choice: do we wait for acknowledgment from multiple nodes (stronger consistency, higher latency) or acknowledge after one node (lower latency, weaker consistency)?

The **latency-consistency tradeoff** is quantified by the number of replicas that must acknowledge before a read or write succeeds (covered in detail in Chapter 11 under quorum reads/writes). For now, the conceptual point is that consistency costs coordination, and coordination costs latency.

| Database | Partition behavior | Normal-operation behavior |
|----------|--------------------|--------------------------|
| DynamoDB | PA (prefers availability) | EL (prefers low latency) |
| Cassandra | PA | EL |
| MongoDB | PC (prefers consistency) | EC (prefers consistency) |
| Google Spanner | PC | EC |
| MySQL (single-node) | N/A (no partition) | EC |

## The Consistency Spectrum

CAP's binary CP/AP distinction conceals a rich spectrum of consistency models that practitioners use to tune the consistency-latency tradeoff. These models are ordered from weakest (fastest) to strongest (most coordination required).

Before walking the spectrum, define the actors: a **client** sends reads and writes; **replicas** are copies of the data on different nodes; **replication** is the mechanism that propagates writes from the primary to replicas.

**Eventual Consistency** is the weakest model: replicas will converge to the same value *eventually* — after all writes have propagated — but at any point in time different replicas may return different values. DNS is the canonical non-database example. Cassandra and DynamoDB default to eventual consistency.

**Read-Your-Writes Consistency** guarantees that after a client performs a write, all subsequent reads by *that same client* reflect that write. Other clients may still see stale data. This is the minimum practically useful consistency level for most user-facing applications — a user who changes their profile picture should see the new picture immediately, even if others see it later.

**Monotonic Read Consistency** guarantees that once a client has seen a value, it will never subsequently see an older value. Reads do not go backward in time. This prevents the disorienting experience where refreshing a page shows an older version of data than you saw a moment ago.

**Causal Consistency** guarantees that operations that are causally related (write B happened because of read A) appear in causal order to all processes. Concurrent (causally unrelated) operations may appear in different orders to different observers. This is the strongest model achievable without global coordination.

**Session Consistency** combines read-your-writes and monotonic reads within a single client session. It is the typical guarantee provided by sticky-session load balancers routing a client to the same replica.

**Strong Consistency** guarantees that every read returns the most recent write. This requires coordination across replicas before every write is acknowledged.

**Linearizability** is the strongest consistency model: every operation appears to take effect atomically at some instant between its invocation and completion. Operations have a total ordering consistent with real time. Google Spanner's use of TrueTime achieves linearizability globally.

**Serializability** (distinct from linearizability) is the transaction-level equivalent: concurrent transactions produce results as if they had executed serially in some order. Serializability does not require that order to match real time; linearizability does.

#### Diagram: Consistency Model Spectrum

<details markdown="1">
<summary>Interactive Consistency Model Spectrum Explorer</summary>
Type: MicroSim
**sim-id:** consistency-spectrum<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Explain the consistency spectrum from eventual to linearizable, identify which anomalies each level prevents, and recognize which databases operate at each level. (Bloom L2–L4: Understand/Analyze)

**Canvas:** 780px wide × 480px tall. CANVAS_HEIGHT: 480.

**Description:**
A horizontal spectrum bar from left ("Weakest / Fastest") to right ("Strongest / Slowest"). Seven labeled markers on the spectrum: Eventual → Read-Your-Writes → Monotonic Read → Causal → Session → Strong → Linearizable.

A draggable slider lets the user position themselves on the spectrum. When the slider moves to a position:

- The "Anomalies Prevented" panel below lists which anomalies are ruled out at this level and above (e.g., at Monotonic Read: "Stale re-reads eliminated. Dirty reads still possible at this level.")
- The "Example Databases" panel lists databases operating at this level: Eventual → Cassandra/DynamoDB defaults. Linearizable → Google Spanner, FoundationDB.
- The "Coordination Required" panel shows a simple diagram of how many nodes must acknowledge before a write completes (1 node at eventual; all replicas + global timestamp at linearizable).

A "Show Anomaly" button animates a concrete scenario illustrating the anomaly that the current level does NOT prevent (e.g., at Eventual: two clients read different values of the same key from different replicas simultaneously).

**Responsive:** Redraws on window resize.
</details>

## BASE Semantics

**BASE** is the acronym that captures the consistency philosophy of AP distributed systems — deliberately contrasted with ACID:

- **B**asically Available — the system guarantees availability in the CAP sense: every request gets a response.
- **S**oft state — the state of the system may change over time even without new inputs, as replicas converge.
- **E**ventually consistent — the system will become consistent over time, given no new updates.

BASE does not mean "no consistency guarantees." It means the system trades strong consistency for availability and performance, and the application must be designed to tolerate and handle temporary inconsistencies. Shopping cart data (where adding the same item twice is recoverable), social media likes counts (where ±1 error is imperceptible), and session tokens (where a stale token causes a login retry) are all reasonable BASE candidates. Financial balances and inventory counts are not.

## Conflict Resolution

When an AP system allows concurrent writes to the same key on different replicas during a partition, those writes may conflict. **Conflict resolution** is the strategy for determining which write wins when the partition heals and replicas attempt to converge.

Common strategies:

- **Last-Write-Wins (LWW):** The write with the latest timestamp wins. Simple, but requires synchronized clocks — and clock drift in distributed systems is unavoidable. LWW can silently discard legitimate writes.
- **Application-level merge:** The application receives all conflicting versions and decides how to merge them. Amazon Dynamo's shopping cart is the canonical example: conflicting cart versions are shown to the user who resolves them at checkout.
- **CRDT (Conflict-free Replicated Data Type):** A data structure designed so that all concurrent updates can be merged deterministically without conflicts — counters, sets, maps with specific merge semantics. CRDTs are the gold standard for eventually consistent systems but constrain what data structures can be used.

## Vector Clocks

To determine whether two events are concurrent or causally ordered — necessary for correct conflict resolution — distributed systems use **vector clocks**. A vector clock is an array of counters, one per node in the system. Each node increments its own counter on every write and attaches the full vector to each message.

Before comparing two vector clocks, understand the rule: event A **happens-before** event B if every counter in A's vector is ≤ the corresponding counter in B's vector (and at least one is strictly less). If neither A happens-before B nor B happens-before A, the events are **concurrent** — and require conflict resolution.

Vector clocks detect causality without requiring synchronized wall clocks, making them essential for eventually consistent systems.

!!! mascot-thinking "Vector Clocks vs. Timestamps"
    <img src="../../img/mascot/thinking.png" class="mascot-admonition-img" alt="Dex thinks">
    It is tempting to use wall-clock timestamps for causality tracking — they're simpler. The problem: distributed clocks drift. Two nodes may disagree on the current time by milliseconds or even seconds. At write rates of thousands per second, milliseconds of clock skew produce incorrect causal ordering. Vector clocks avoid this by tracking logical causality rather than physical time. Google Spanner's TrueTime solves the clock problem with GPS and atomic clocks, bounded uncertainty, and commit-wait — but that's a proprietary, hardware-dependent solution.

## Paxos and Raft: Consensus Protocols

All distributed databases face a fundamental challenge: how do multiple nodes agree on a single value (the next committed log entry, the current leader, a transaction's commit decision) even when some nodes are slow, crashed, or lying? This is the **consensus problem**, and it is provably impossible to solve perfectly in a purely asynchronous network (the FLP impossibility result). Practical systems solve it by adding timing assumptions.

### Paxos

**Paxos** (Leslie Lamport, 1989) was the first widely studied consensus protocol and remains the theoretical foundation for most production systems. Paxos operates in two phases:

**Phase 1 — Prepare/Promise:**
1. A node that wants to propose a value (the **proposer**) sends a `Prepare(n)` message to a majority of nodes (the **acceptors**), where `n` is a ballot number higher than any it has used before.
2. Each acceptor that has not already promised a higher ballot responds with `Promise(n)`, pledging not to accept any proposal with a ballot number lower than `n`. If the acceptor has already accepted a value, it includes that value in its promise.

**Phase 2 — Accept/Accepted:**
3. If the proposer receives promises from a majority, it sends `Accept(n, v)` where `v` is the value it wants to commit (or the highest-ballot value received in any promise).
4. Acceptors that haven't promised a higher ballot respond with `Accepted(n, v)`.
5. If a majority respond, the value is **chosen** (consensus is reached).

Paxos is notoriously difficult to implement correctly, especially for multi-round consensus (Multi-Paxos). Production systems that use Paxos (Google Chubby, Apache ZooKeeper's early versions) typically build significant scaffolding around the core algorithm.

### Raft

**Raft** (Ongaro and Ousterhout, 2014) was designed explicitly to be more understandable than Paxos while providing equivalent safety guarantees. Raft decomposes consensus into three sub-problems: **leader election**, **log replication**, and **safety**.

In Raft, one node at a time is the **leader**. All writes go through the leader. The leader appends entries to its log and replicates them to followers. An entry is **committed** once the leader has received acknowledgments from a majority of nodes (including itself). Committed entries are guaranteed never to be overwritten.

**Leader election:** Nodes start as followers. If a follower doesn't hear from a leader within an **election timeout** (random, 150–300ms typically), it becomes a **candidate**, increments its **term** counter, and requests votes from other nodes. A candidate that receives votes from a majority becomes the new leader for that term.

Raft's simplicity has made it the consensus protocol of choice for new distributed systems: etcd (Kubernetes), CockroachDB, TiKV (TiDB), YugabyteDB, and InfluxDB all use Raft.

#### Diagram: Raft Leader Election Animation

<details markdown="1">
<summary>Interactive Raft Leader Election Simulator</summary>
Type: MicroSim
**sim-id:** raft-leader-election<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Demonstrate the Raft leader election process — election timeout, candidate vote request, majority vote, and new leader establishment. (Bloom L2–L3: Understand/Apply)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
Five circular nodes arranged in a pentagon. Each node displays its role (Follower/Candidate/Leader) and current term number. One node is highlighted as Leader (gold border). A heartbeat timer countdown is visible on each follower (resets when heartbeat arrives).

**Normal operation:** The leader pulses a heartbeat to all followers every 150ms. Follower timers reset on receipt. All nodes labeled "Follower" except the leader.

**Leader failure:** A "Kill Leader" button removes the leader (node grays out). Remaining followers' timers count down. The first to time out transitions to Candidate (blue border), increments its term, and sends "RequestVote" arrows to all other nodes. Nodes that haven't voted this term send "VoteGranted" back. Once the candidate has votes from 2 of 4 remaining nodes (majority of 5), it becomes the new Leader (gold border). Heartbeats resume.

**Split vote scenario:** A "Slow Network" toggle adds 50% chance of vote messages being dropped, causing split votes and multiple election rounds.

**Term counter:** A global term counter at the top increments each election.

**Responsive:** Redraws on window resize.
</details>

#### Diagram: Paxos Phase Message Flow

<details markdown="1">
<summary>Interactive Paxos Phase 1 and Phase 2 Message Flow</summary>
Type: MicroSim
**sim-id:** paxos-message-flow<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify the Prepare/Promise and Accept/Accepted message flows of Paxos and explain what a majority quorum guarantees. (Bloom L2: Understand)

**Canvas:** 780px wide × 460px tall. CANVAS_HEIGHT: 460.

**Description:**
Three horizontal swim lanes: Proposer (top), Acceptors (middle — three nodes A1, A2, A3), and a "Consensus Reached" indicator (bottom).

A "Run Paxos" button animates the full protocol:
- Phase 1: Proposer sends Prepare(n=1) arrows to A1, A2, A3. A1 and A2 reply Promise(1, null). A3 is slow (arrow grayed out / delayed).
- Phase 2: Proposer sends Accept(1, "value=42") to A1, A2. Both reply Accepted(1, 42). Majority reached. Consensus indicator turns green: "Value 42 chosen."

A "Competing Proposer" button introduces a second proposer with a higher ballot number (n=2), showing how it invalidates the first proposer's promises and starts fresh.

Each message arrow is labeled and shows timing. A sidebar explains what each phase accomplishes.

**Responsive:** Redraws on window resize.
</details>

## The ATAM Lens: Consistency vs. Availability in Practice

The CAP/PACELC vocabulary is the language of distributed database ATAM analysis. When a utility tree contains a quality attribute scenario like "The system must remain writable during a regional network partition," it is specifying AP behavior. When it says "Account balances must always reflect the most recent transaction," it is specifying CP behavior — and likely linearizability.

!!! mascot-warning "CAP Is Not a Free Lunch"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex warns">
    Choosing AP does not mean "no consistency." It means "eventual consistency with conflict resolution." You still need to design your conflict resolution strategy, handle stale reads in the application, and set appropriate TTLs or read-repair policies. Teams that choose Cassandra for availability and then discover they needed read-your-writes consistency have learned this lesson the hard way. Know exactly which consistency model your application requires before choosing a database — not after.

**Sensitivity points for ATAM:**

- **Consistency model vs. geographic distribution:** Linearizability across data centers separated by 100ms round-trip latency means every write waits 100ms minimum. Strong consistency + global distribution is expensive. Most global systems use causal or session consistency with linearizability only for critical operations (payments, inventory).
- **Consensus protocol vs. write throughput:** Leader-based consensus (Raft, Paxos) serializes writes through one node. Throughput is bounded by leader capacity. Leaderless systems (Dynamo-style quorum writes) distribute write load at the cost of conflict resolution complexity.

## Key Takeaways

The theoretical results in this chapter are not academic curiosities — they are the mathematical constraints that every distributed database must navigate. Knowing them makes you a more precise architect.

- **CAP theorem** — under network partition, choose consistency (CP) or availability (AP)
- **PACELC** — adds the normal-operation latency vs. consistency tradeoff to CAP
- **Consistency spectrum** — eventual → read-your-writes → monotonic → causal → session → strong → linearizable
- **BASE** — the availability-first philosophy; designed for graceful inconsistency, not absence of consistency
- **Vector clocks** — track causality without synchronized clocks; enable correct conflict detection
- **Paxos** — the theoretical foundation of distributed consensus; powerful but complex
- **Raft** — the practical, understandable consensus protocol; used in etcd, CockroachDB, and most modern distributed systems

!!! mascot-celebration "Chapter 9 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now speak the language of distributed systems theory — and more importantly, you can apply it. When someone says their database is "eventually consistent," you can ask: does that mean read-your-writes? Monotonic reads? Is there a conflict resolution strategy? When someone proposes a globally distributed system with linearizable reads, you can estimate the latency floor from speed-of-light constraints alone. That precision is what separates architectural reasoning from intuition.
