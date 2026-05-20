---
title: Chapter 5 - Key-Value Stores
description: Hash-based storage, TTL, cache eviction policies, caching patterns, cache stampede, hot key problem — and Redis and DynamoDB as canonical representatives.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 5: Key-Value Stores

## Summary

This chapter examines the simplest and highest-throughput database paradigm: the key-value store. Students learn how hash-based storage achieves extreme read/write performance, how TTL and eviction policies manage memory, and how caching patterns (read-through, write-through, write-behind) are applied in production systems. The chapter also covers operational failure modes unique to key-value stores — cache stampedes and hot key problems — and examines Redis and DynamoDB as representative products.

## Concepts Covered

This chapter covers the following 12 concepts from the learning graph:

1. Key-Value Data Model
2. Hash Table Storage
3. Time-to-Live
4. Cache Eviction Policy
5. Redis
6. DynamoDB
7. Read-Through Cache
8. Write-Through Cache
9. Write-Behind Cache
10. Cache Stampede
11. Hot Key Problem
12. Key Expiration

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)

---

!!! mascot-welcome "Welcome to Chapter 5!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Key-value stores are where the relational model's generality goes to retire in favor of raw speed. There are no tables, no joins, no schemas, no complex queries — just keys and values. That radical simplicity is the point: by doing one thing extremely well, key-value stores achieve throughput that relational databases cannot match at any price. This chapter teaches you when that tradeoff is exactly right — and the failure modes that bite you when you don't plan for them.

## The Key-Value Data Model

A **key-value data model** is the simplest database abstraction: the database stores a collection of (key, value) pairs and supports three core operations — `PUT(key, value)` to store a pair, `GET(key)` to retrieve a value, and `DELETE(key)` to remove a pair. No query language, no schema, no secondary lookups by anything other than the key.

The key is a string (or binary blob) chosen by the application. The value is opaque to the database — it can be a string, a JSON blob, a serialized object, an image, or any byte sequence. The database does not interpret the value's contents; it stores and retrieves it as a unit.

This simplicity has profound performance implications. Because every operation is addressed by an exact key, the database need not search any structure more complex than a hash table. There is no query planner, no join logic, no lock hierarchy to navigate. The path from request to response is nearly as short as it can be.

Two more terms before proceeding: **namespace** (or keyspace) is the logical grouping of keys, similar to a table in a relational database; **value serialization** is the application's responsibility — the database stores bytes, so the application must encode and decode structured objects.

## Hash Table Storage

The engine behind key-value performance is **hash table storage**. A hash table maps each key to a memory address (or disk location) through a hash function. Given a key, the hash function computes a bucket index in O(1) time — constant regardless of how many keys are stored. The engine reads the value from that bucket in a single memory access.

Three properties of hash functions matter for database use:

- **Uniform distribution:** Keys should spread evenly across buckets to avoid collision clustering.
- **Determinism:** The same key must always produce the same bucket index.
- **Speed:** The hash computation itself must be cheap (sub-microsecond).

Modern key-value stores (Redis, Memcached) keep their entire working set in RAM and use in-memory hash tables. A well-configured Redis instance serving cached data can handle 1 million GET operations per second on a single node — latencies in the 100-microsecond range.

#### Diagram: Hash Table Lookup Animation

<details markdown="1">
<summary>Interactive Hash Table Lookup — Key to Bucket to Value</summary>
Type: MicroSim
**sim-id:** hash-table-lookup<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Demonstrate how a hash function maps a key to a bucket index, enabling O(1) average-case lookup. (Bloom L2: Understand)

**Canvas:** 760px wide × 440px tall. CANVAS_HEIGHT: 440.

**Description:**
Left: A text input labeled "Enter key" (default: "user:42"). A "Hash" button. Center: An animated hash function box showing: `hash("user:42") mod 8 = 3`. Right: An array of 8 buckets (vertical stack, labeled 0–7). Bucket 3 is highlighted gold with a value label.

**Animation flow when "Hash" is clicked:**
1. Key text animates into hash function box (0.3s).
2. Hash function shows computation step-by-step (0.5s per step).
3. Arrow shoots from hash box to computed bucket (0.4s).
4. Bucket glows green and shows stored value (0.3s).

**Collision demonstration:** "Show Collision" button inserts a second key hashing to the same bucket. Bucket expands to show a linked list of two entries. Tooltip: "Collision: two keys map to the same bucket — chaining stores both."

**Resize slider:** "Table size" slider (4–16 buckets) re-computes bucket assignments, showing how larger tables reduce collision probability.

**Responsive:** Redraws on window resize.
</details>

## Time-to-Live and Key Expiration

One of the most important features of key-value stores for caching use cases is **Time-to-Live (TTL)**: each key can be given an expiration timestamp, after which the key is automatically deleted. In Redis: `SET session:abc123 "{...}" EX 3600` stores the value and schedules deletion after 3,600 seconds.

TTL solves the cache staleness problem: cached data that reflects a past system state is automatically evicted after a configurable interval. The application sets TTL based on how quickly the underlying data changes and how much staleness is tolerable.

**Key expiration** is implemented through two complementary mechanisms:

- **Lazy expiration:** When a key is accessed, the engine checks its expiration timestamp. If expired, the key is deleted and "not found" is returned. Zero background overhead, but allows expired keys to consume memory until accessed.
- **Active expiration:** A background thread periodically samples a random subset of keys with TTLs and deletes those that have expired. Prevents memory accumulation for keys never accessed again.

## Cache Eviction Policies

Key-value stores used for caching operate within a fixed memory budget. When memory is full and new keys need to be stored, the engine must evict (delete) existing keys to make room. The **cache eviction policy** determines which keys are chosen for eviction.

Eviction is distinct from expiration: expiration removes keys after a time limit; eviction removes keys to free memory regardless of time.

| Policy | Behavior | Best For |
|--------|----------|----------|
| LRU (Least Recently Used) | Evict the key last accessed furthest in the past | General caching — temporal locality |
| LFU (Least Frequently Used) | Evict the key accessed the fewest times | Skewed access patterns — protect popular keys |
| volatile-ttl | Evict the key closest to its TTL expiration | Workloads where all keys have explicit TTLs |
| allkeys-random | Evict a random key | Uniform access distributions |
| noeviction | Reject writes when full | Non-cache use cases (persistence required) |

Redis implements approximate LRU using random sampling rather than a true LRU linked list — trading a small accuracy cost for significant memory savings.

#### Diagram: Cache Eviction Policy Simulator

<details markdown="1">
<summary>Cache Eviction Policy Simulator</summary>
Type: MicroSim
**sim-id:** cache-eviction-simulator<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Apply LRU and LFU eviction policies to a sequence of cache operations and predict which key will be evicted when memory is full. (Bloom L3: Apply)

**Canvas:** 780px wide × 480px tall. CANVAS_HEIGHT: 480.

**Description:**
A fixed-size cache display (6 slots as horizontal bars) with a "Cache Access Log" on the right.

**Controls:**
- Policy dropdown: LRU | LFU | Random | TTL-based
- "Next Access" button simulates the next operation from a pre-defined 20-step sequence
- "Full Sequence" runs all 20 steps at 500ms intervals

Each cache slot shows: key name, value preview, access count (LFU), last-accessed time (LRU), TTL countdown (TTL-based). The slot at the bottom is the next evicted (red border). When eviction occurs, the slot flashes red and disappears; the new key animates in from the top.

**Statistics panel:** Hit rate (%), evictions so far, current memory usage.

**Access pattern:** Key "A" is accessed frequently early then not at all — highlighting how LFU protects it while LRU eventually evicts it.

**Responsive:** Redraws on window resize.
</details>

## Caching Patterns

Key-value stores almost always sit between an application and a slower primary database, acting as a high-speed cache layer. Three patterns define how the cache and primary database are kept in sync.

### Read-Through Cache

In the **read-through cache** pattern, the application always reads from the cache. On a cache hit, the value is returned immediately. On a cache miss, the cache itself fetches the value from the primary database, stores it with a TTL, and returns it to the application.

**Advantage:** Cache population is automatic and lazy — only requested data is cached. **Disadvantage:** The first request for any key incurs a cold miss with full primary database latency. Cache warming (pre-loading popular keys on startup) mitigates this.

### Write-Through Cache

In the **write-through cache** pattern, every write goes to the cache first, and the cache synchronously writes through to the primary database before acknowledging success. Both cache and primary database are always in sync.

**Advantage:** No cache staleness. **Disadvantage:** Write latency equals primary database write latency — no speedup for writes.

### Write-Behind Cache

In the **write-behind cache** (write-back) pattern, writes go to the cache and the cache acknowledges success immediately. The cache asynchronously flushes updates to the primary database in the background.

**Advantage:** Dramatically faster writes (sub-millisecond). **Disadvantage:** If the cache fails before flushing, unflushed writes are lost. This pattern requires cache persistence and the application to tolerate potential data loss.

!!! mascot-tip "Cache Invalidation: The Hard Problem"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex offers a tip">
    Phil Karlton's quip — "There are only two hard things in Computer Science: cache invalidation and naming things" — is funny because it's accurate. When the primary database is updated by a path that doesn't go through the cache (a bulk SQL update, a migration script, a different service), the cache holds stale data until TTL expires or an explicit invalidation is triggered. Design your invalidation strategy before you build your caching layer, not after.

## Cache Stampede and Hot Key Problems

Two operational failure modes are unique to key-value caches at scale.

### Cache Stampede

A **cache stampede** (thundering herd) occurs when a popular cached key expires and hundreds of concurrent requests simultaneously experience a cache miss — all querying the primary database at once. The resulting load spike can overwhelm the primary database and cascade into an outage.

**Mitigation strategies:**

- **Probabilistic early expiration:** Re-generate the cache entry before it actually expires, based on a probability function that increases as expiration approaches. Distributes regeneration over time.
- **Mutex lock on miss:** One request acquires a lock and regenerates the value; others return a slightly stale value or wait briefly. Serializes regeneration without starving callers.
- **Staggered TTLs:** Add random jitter to TTL values so a batch of simultaneously-cached entries don't all expire at the same instant.

### Hot Key Problem

A **hot key problem** occurs when a disproportionate fraction of requests target a single key. In a distributed key-value store, each key is assigned to a specific shard. If one key receives 100,000 requests per second, all routing to the same shard, that shard becomes the bottleneck regardless of cluster size.

**Mitigation strategies:**

- **Key replication with random suffix:** Store `popular_key:1` through `popular_key:N`; randomly select a replica on each read. Writes update all N copies.
- **Local in-process caching:** A small in-memory cache within each application server instance serves extreme hot keys without any network call.
- **Read replicas:** Route hot-key reads to dedicated replicas.

#### Diagram: Cache Failure Mode Simulator

<details markdown="1">
<summary>Cache Stampede and Hot Key Failure Mode Simulator</summary>
Type: MicroSim
**sim-id:** cache-failure-modes<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify cache stampede and hot key failure modes and select appropriate mitigation strategies. (Bloom L4–L5: Analyze/Evaluate)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
Two side-by-side panels: "Cache Stampede" (left) and "Hot Key" (right).

**Cache Stampede panel:** Timeline with Cache (top bar, TTL countdown), 10 Application Server circles (middle), and Primary DB (bottom bar). When TTL hits 0, all 10 servers simultaneously fire red arrows at the DB. DB bar turns red: "DB Overload." Load spike bar chart shows 10x spike.

"Mitigate: Jitter TTL" button re-runs: 10 servers fire at staggered times; DB load spreads out; DB bar stays green.

**Hot Key panel:** 3 shards as columns. Each second, request dots fall from top: 80% land on Shard 1 (turns red: "OVERLOADED"). Shards 2–3 are nearly idle (green).

"Mitigate: Key Replicas" button re-runs: request dots split across popular_key:1, :2, :3 — all shards stay green.

**Responsive:** Redraws on window resize.
</details>

## Redis and DynamoDB: The Canonical Representatives

### Redis

**Redis** (Remote Dictionary Server) is an in-memory key-value store that extends the basic model with rich data structures: strings, lists, sets, sorted sets (score-based ordering), hashes (nested maps), bitmaps, HyperLogLogs, and streams. This makes Redis suitable for use cases beyond simple caching: session storage, pub/sub messaging, leaderboards (sorted sets), rate limiting (atomic increment), and real-time analytics.

Redis persistence options range from none (pure cache) to RDB snapshots (periodic full-dataset dump), AOF (append-only log of every write), and combined RDB+AOF. Redis Cluster provides horizontal sharding using hash slots (16,384 slots across nodes). Redis Sentinel provides automatic failover.

### DynamoDB

**DynamoDB** (Amazon Web Services) is a fully managed, serverless key-value and document database. Unlike Redis, DynamoDB persists all data to SSDs across multiple availability zones by default. Its data model uses a **partition key** (primary hash key) and an optional **sort key** (enabling range queries within a partition). DynamoDB's throughput is provisioned in Read/Write Capacity Units or configured for on-demand scaling.

| Feature | Redis | DynamoDB |
|---------|-------|----------|
| Primary storage | In-memory (optional persistence) | Persistent SSD (multi-AZ) |
| Data structures | Rich (lists, sets, sorted sets, streams) | Key-value + document items |
| Consistency | Strong (single node) / eventual (replicas) | Eventual (default) / strong (opt-in) |
| Scaling | Redis Cluster (manual sharding) | Auto-scaling, serverless |
| Primary use cases | Cache, sessions, pub/sub, leaderboards | Serverless backends, IoT, gaming |

!!! mascot-warning "Redis Is Not a Database of Record"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex warns">
    Redis with no persistence enabled loses all data on restart. Redis with AOF persistence can lose up to one second of writes depending on fsync policy. For data that cannot be lost — user accounts, orders, financial transactions — Redis is a cache layer in front of a durable database, not a replacement. DynamoDB is durable by design; Redis is durable only by careful configuration.

## The ATAM Lens: Key-Value Tradeoffs

In an ATAM utility tree, key-value stores appear as strong candidates when a quality attribute scenario values performance and throughput over query expressiveness or schema structure. The canonical scenario: "The system must serve 500,000 user session reads per second with p99 latency under 5ms."

**Sensitivity point — consistency vs. throughput:** Eventual consistency (Redis Cluster read replicas, DynamoDB eventually consistent reads) doubles or triples effective read throughput at the cost of potentially serving slightly stale data. For session tokens and user preferences, this is often acceptable; for inventory counts or financial balances, it is not.

**Tradeoff point — the query expressiveness wall:** Every key-value store operation is addressed by an exact key. There is no `WHERE status = 'active'` scan, no multi-column filter, no full-text search. Systems that evolve to need richer queries must add an index (a secondary database maintained in sync), accept full-dataset scans, or migrate to a different paradigm. Design the access patterns before choosing a key-value store, not after.

## Key Takeaways

Key-value stores achieve extraordinary throughput by doing less: less structure, less query expressiveness, less schema enforcement — and in exchange, sub-millisecond latencies at millions of operations per second.

The concepts from this chapter that recur throughout the book:

- **Hash table storage** — the O(1) mechanism behind key-value speed
- **TTL / key expiration** — automatic cache invalidation by time
- **Eviction policies (LRU, LFU)** — decision logic when memory is full
- **Caching patterns** — read-through, write-through, write-behind
- **Cache stampede / hot key** — the failure modes that kill caching layers in production
- **Redis / DynamoDB** — the two canonical representatives with complementary strengths

!!! mascot-celebration "Chapter 5 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now understand the fastest database paradigm in common use — and its principal failure modes. The next chapter scales this thinking to a related paradigm that adds one critical dimension: instead of a single value per key, column-family databases store thousands of columns per key, organized for write-heavy workloads at extreme scale. Same spirit, very different internal architecture.
