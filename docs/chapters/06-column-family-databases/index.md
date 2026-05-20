---
title: Chapter 6 - Column-Family Databases
description: Wide-column storage, LSM trees, compaction strategies, Bloom filters, and the Cassandra and HBase ecosystems for write-heavy workloads at massive scale.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:49:42
version: 0.08
---

# Chapter 6: Column-Family Databases

## Summary

This chapter covers the wide-column (column-family) database paradigm, built around the LSM tree storage engine that optimizes for write-heavy workloads at massive scale. Students learn how compaction strategies manage the write/read amplification tradeoff, how Bloom filters accelerate negative lookups, and how partition and clustering keys control data distribution and sort order. Apache Cassandra and HBase are examined as the dominant representatives of this paradigm, with emphasis on time-series and IoT use cases.

## Concepts Covered

This chapter covers the following 12 concepts from the learning graph:

1. Column-Family Data Model
2. Wide Row
3. LSM Tree
4. Compaction Strategy
5. Bloom Filter
6. Apache Cassandra
7. Apache HBase
8. Partition Key
9. Clustering Column
10. Write-Optimized Storage
11. Read Amplification
12. Write Amplification

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)
- [Chapter 5: Key-Value Stores](../05-key-value-stores/index.md)

---

!!! mascot-welcome "Welcome to the World of Wide Rows"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello at the entrance to Chapter 6">
    Column-family databases are where "simple" key-value stores grow up and get complicated in useful ways. By the end of this chapter you will understand why a time-series IoT pipeline can ingest a million writes per second without breaking a sweat — and exactly what that convenience costs you at read time. Let's dig in.

## From Key-Value to Column-Family: A Logical Step

In the previous chapter you met key-value stores — the spartan, blazingly fast databases that treat values as opaque blobs. They are excellent at "give me the thing with this key," but they cannot answer "give me all sensor readings for device 4271 between noon and 1 PM sorted by timestamp." For that, you need structure inside the value.

Column-family databases — also called **wide-column databases** — solve exactly that problem. They retain the key-based access model of key-value stores but allow each row to contain an arbitrarily large number of named columns, each with its own value and optional timestamp. The result is a data model that sits midway between a flat key-value store and a full relational table.

The term **column-family data model** refers to the logical organization of a table into groups of related columns. A table might have a "sensor_readings" column family containing columns like `temperature`, `humidity`, and `pressure`, and a "metadata" column family containing `firmware_version` and `location`. Column families are defined at table creation time and are stored separately on disk — but within a column family, columns can vary freely from row to row.

## Wide Rows and Sparse Storage

The defining feature of column-family databases is the **wide row**: a row with potentially thousands of individual columns, where different rows in the same table may have completely different column sets. This stands in sharp contrast to relational databases, where every row in a table has exactly the same columns, including NULL placeholders for missing values.

The sparseness of wide rows matters for storage efficiency. In a relational system, if 99% of rows have no value for a given column, you still allocate space for the NULL in every row. In a column-family system, absent columns consume no space at all. For workloads like time-series sensor data — where each device has its own set of optional readings — this sparsity can translate to a factor-of-ten reduction in storage.

Wide rows also determine how you model queries. In relational design, normalization pushes repeated data into separate tables joined at query time. In column-family design, you denormalize aggressively: you pick a query pattern first and design your rows to answer it in a single scan. This is not laziness — it is deliberate schema-on-query-first design, and it is one of the habits that separates practitioners who succeed with Cassandra from those who build a painful mess.

## LSM Trees: The Engine Behind Write Performance

To understand why column-family databases are so fast for writes, you need to understand the **LSM tree** — the Log-Structured Merge-tree — the storage engine architecture that powers both Apache Cassandra and Apache HBase.

Traditional relational databases use B-Trees for on-disk storage. B-Trees are optimized for reads: any key can be found in O(log n) time with a small number of disk seeks. But writes on a B-Tree involve in-place updates: the engine must find the exact location on disk for each changed page and write it there. For spinning disks, random writes are catastrophically slow. For SSDs, random writes accelerate wear leveling. Neither is ideal for ingest-heavy workloads.

The LSM tree takes a fundamentally different approach. Every write goes first to an in-memory structure called a **memtable** — a sorted write buffer that accepts updates at memory speed. When the memtable fills up (typically after tens of megabytes), it is flushed to disk as an immutable, sorted file called an **SSTable** (Sorted String Table). The critical insight is that the flush is a sequential write — the fastest operation possible on any storage medium, spinning or solid-state. There are no random seeks. You simply stream a sorted blob to disk.

The tradeoff is that data from a single logical key may now be scattered across multiple SSTables — one per flush cycle. Reading that key requires checking the current memtable and then searching through potentially many SSTable files. This is the origin of **read amplification**.

#### Diagram: LSM Tree Write Path Animation

<details markdown="1">
<summary>Interactive LSM tree write path: memtable, SSTable flush, and compaction</summary>

**sim-id:** lsm-tree-write-path
**Library:** p5.js
**Status:** Specified

A horizontally scrollable p5.js canvas (900×500 px) showing the LSM tree write path in three animated phases.

**Visual layout (left to right):**
- Left panel: "Write Buffer" box labeled "Memtable (in memory)" — a sorted vertical list of key-value pairs that grows downward as new writes arrive
- Center panel: Three stacked boxes labeled "L0 SSTables" — each is an immutable sorted file
- Right panel: Two taller boxes labeled "L1 SSTables" and one very tall box labeled "L2 SSTables"
- Arrows show: (1) keys flowing from left into Memtable, (2) a flush arrow from Memtable to L0 when full, (3) compaction arrows from L0→L1 and L1→L2

**Controls:**
- "Add Write" button: drops a new key into the Memtable list, causing it to sort itself
- "Flush Memtable" button: animates the Memtable contents streaming rightward into a new L0 SSTable file
- "Run Compaction" button: merges and sorts two L0 SSTables into a single L1 SSTable, deleting the originals
- Speed slider (1× to 5×) controlling animation rate

**Interactive behavior:**
- Clicking any SSTable box shows a tooltip: file name, key range (min-key … max-key), record count, size in KB
- Clicking a key in the Memtable highlights which SSTable(s) also contain that key (read path visualization)
- When Memtable reaches 8 items, it flashes amber and the Flush button pulses

**Learning objective:** Applying (Bloom's) — students manipulate the write pipeline and observe how data moves from memory to disk, connecting the LSM architecture to its performance characteristics.

**Colors:** Memtable (#4682B4 blue), L0 SSTables (#E65100 orange), L1 SSTables (#28a745 green), L2 SSTables (#6f42c1 purple). White text on all boxes.

Responsive: canvas resizes to container width on window resize event.
</details>

## Bloom Filters: Probabilistic Negative Lookups

Reading from an LSM tree is expensive because a single key might live in any of several SSTable files. Checking each file sequentially — opening it, binary searching for the key, finding nothing — is wasteful. The solution is the **Bloom filter**.

A Bloom filter is a probabilistic data structure that answers a single question: "Has this key *probably* been written to this SSTable?" The filter is a compact bit array. When an SSTable is written, each key is hashed through several hash functions, and the corresponding bits are set. To test whether a key exists in an SSTable, the same hash functions are applied. If any of the bits are unset, the key is *definitely absent* — skip this file. If all bits are set, the key *probably exists* — worth checking.

The critical property is the one-sided error guarantee: a Bloom filter produces no false negatives (it never says "absent" when the key is present) but does produce false positives (it occasionally says "probably present" when the key is absent). Tuning the false-positive rate — typically to 1% or lower — is a matter of choosing how many bits per key to allocate. A 10-bit-per-key Bloom filter delivers roughly 0.8% false positives. For most LSM workloads, the memory cost is trivially small compared to the disk I/O it saves.

!!! mascot-tip "Bloom Filters in Practice"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex points to a useful tip">
    In Apache Cassandra, Bloom filters are loaded into off-heap memory at startup and kept resident for the lifetime of the SSTable. If your Cassandra node is running short on off-heap memory, do not blindly reduce Bloom filter size — you will pay for every saved byte with extra disk reads on your most latency-sensitive paths.

#### Diagram: Bloom Filter Interactive Demo

<details markdown="1">
<summary>Bloom filter "probably in set" vs "definitely not in set" interactive visualizer</summary>

**sim-id:** bloom-filter-demo
**Library:** p5.js
**Status:** Specified

A p5.js canvas (800×450 px) with a split layout.

**Left panel — "Filter State":**
- A horizontal row of 32 bit cells, each either 0 (light gray) or 1 (dark blue)
- Above the row, a label shows "Bit Array (32 bits, 3 hash functions)"
- Three colored horizontal arrow indicators showing which bits each hash function sets for the most recently tested key

**Right panel — "Result":**
- A large colored badge: "DEFINITELY NOT IN SET" (green) or "PROBABLY IN SET" (amber)
- Below: "False positive rate: X.X%" calculated dynamically from the number of set bits

**Controls:**
- Text input "Add key to filter:" with an "Add" button — hashes the key string, sets appropriate bits, turns them blue, animates the three hash arrows
- Text input "Test key:" with a "Test" button — runs the lookup, lights up the three hash positions, shows the result badge
- "Reset" button — clears all bits

**Pre-loaded state:** Keys "alice", "bob", "charlie" are already in the filter when the demo loads, giving students a non-trivial starting state.

**Learning objective:** Understanding (Bloom's) — students observe that testing a key *not* in the set can still return "probably present" (a false positive), cementing the one-sided error guarantee.

Responsive: canvas resizes on window resize.
</details>

## Compaction Strategies: Managing SSTable Debt

The flush-to-SSTable approach produces ever-growing piles of small immutable files. **Compaction** is the background process that merges, sorts, and consolidates SSTables, evicting obsolete values and tombstones (deletion markers). Without compaction, read performance would degrade continuously as the engine must search more and more files.

Different workloads call for different compaction strategies. Three major strategies exist in Apache Cassandra:

**Size-Tiered Compaction Strategy (STCS)** is the default. When several SSTables of similar size accumulate, they are merged into one larger file. This is write-efficient — compaction writes are minimized — but at steady state the data may be spread across several large tiers, increasing read amplification.

**Leveled Compaction Strategy (LCS)** maintains a hierarchy of levels (L0, L1, L2…) where each level is 10× the size of the previous. SSTables within a level have non-overlapping key ranges, so a point read requires checking at most one SSTable per level. LCS produces significantly lower read amplification but writes more data during compaction — the classic **write amplification** cost.

**Time-Window Compaction Strategy (TWCS)** is purpose-built for time-series data. It groups SSTables by time window (e.g., one-hour buckets) and compacts each window independently. Since time-series data is rarely updated after ingestion, TWCS avoids rewriting old data and keeps compaction focused on recent windows. This is the recommended strategy for IoT sensor pipelines.

The following table summarizes the three compaction strategies and their primary tradeoffs:

| Strategy | Best For | Read Amplification | Write Amplification | Space Amplification |
|---|---|---|---|---|
| Size-Tiered (STCS) | Write-heavy workloads | Higher | Lower | Higher |
| Leveled (LCS) | Read-heavy workloads | Lower | Higher | Lower |
| Time-Window (TWCS) | Time-series / append-only | Low within window | Lowest | Low |

!!! mascot-thinking "Which Compaction Strategy Should You Choose?"
    <img src="../../img/mascot/thinking.png" class="mascot-admonition-img" alt="Dex thinks carefully about the decision">
    The honest answer is: start with TWCS if your data is time-series, LCS if you read more than you write, and STCS only if you are write-bound and the operator team understands the space amplification risk. Changing compaction strategy on a live cluster is painless in Cassandra — but changing it retroactively on terabytes of existing SSTables requires a full compaction cycle that will saturate your disks for hours. Plan ahead.

#### Diagram: Compaction Strategy Comparison

<details markdown="1">
<summary>Interactive compaction strategy comparison: STCS vs LCS vs TWCS space and read cost</summary>

**sim-id:** compaction-strategy-comparison
**Library:** p5.js
**Status:** Specified

A p5.js canvas (900×520 px) with three side-by-side columns, one per compaction strategy.

**Each column shows:**
- A vertical stack of SSTable file rectangles, sized proportionally to their on-disk size
- Color coding: recent files (blue), older compacted files (green), tombstone-heavy files (amber)
- A "Read Cost" indicator (1–5 bar chart) showing how many SSTables a point read must visit
- A "Space Used" indicator showing total disk space relative to raw data size

**Timeline scrubber:** A horizontal time slider at the bottom (0–72 hours of writes). Dragging it shows how the SSTable landscape evolves over time under each strategy.

**Interactive behavior:**
- Clicking any SSTable rectangle shows a tooltip: time range, key range, tombstone percentage, size in MB
- The three columns update in sync as the time slider moves
- "Simulate Point Read" button: highlights exactly which SSTables in each column would be checked for a given key

**Learning objective:** Analyzing (Bloom's) — students compare how the three strategies evolve under sustained write load, observing the space/read/write amplification tradeoffs directly.

Responsive: canvas resizes on window resize. Minimum width 600 px.
</details>

## Apache Cassandra: Masterless, Tunable Consistency

**Apache Cassandra** is the dominant wide-column database. Developed at Facebook and open-sourced in 2008, Cassandra was designed from the ground up for geographic distribution, zero single points of failure, and tunable consistency. Understanding Cassandra means understanding three of its foundational design choices: the ring topology, the partition key, and tunable consistency.

Cassandra organizes its nodes in a **ring** — a logical circle where each node owns a range of tokens (hash values). When a write arrives, Cassandra hashes the row's **partition key** — the column or set of columns that determines which physical node stores the row — to produce a token. That token falls into one node's range, making it the primary replica. Two or more additional nodes in the ring (determined by the replication factor) store copies.

The partition key is the most important modeling decision in Cassandra. Rows with the same partition key are always co-located on the same node(s) and stored contiguously on disk. This means that queries that filter by partition key are fast single-node lookups. Queries that must scan across partitions (the dreaded `ALLOW FILTERING`) cause full-cluster scans and should be rare or avoided.

Within a partition, rows are sorted by one or more **clustering columns** — the secondary sort keys that determine the physical order of rows on disk. If your partition key is `device_id` and your clustering column is `event_timestamp`, then all events for a given device are stored in timestamp order, making time-range queries over a single device blazingly fast.

Cassandra is an **AP system** in CAP theorem terms: it chooses availability over strict consistency during a network partition. It compensates with **tunable consistency** — each read and write operation can specify how many replicas must acknowledge the operation before it is considered successful. `QUORUM` requires a majority of replicas, giving "strong eventual consistency." `ONE` accepts a single replica acknowledgment for maximum throughput.

#### Diagram: Cassandra Ring with Partition Key Hashing

<details markdown="1">
<summary>Interactive Cassandra ring: partition key hashing, token ranges, and replica placement</summary>

**sim-id:** cassandra-ring-explorer
**Library:** p5.js
**Status:** Specified

A p5.js canvas (800×600 px) with a circular ring in the center.

**Ring visualization:**
- Six nodes arranged evenly around a circle, labeled N1–N6
- Each node shows its token range (e.g., "0–167" for a 1000-token ring divided among 6 nodes)
- Each node is colored by its replica role: primary (#4682B4 blue), replica-1 (#28a745 green), replica-2 (#E65100 orange), uninvolved (#ccc gray)

**Controls:**
- Text input "Partition Key:" with a "Hash It" button
- Replication Factor selector (1, 2, 3) — controls how many nodes are colored
- Consistency Level selector (ONE, QUORUM, ALL) — highlights which nodes must respond

**Interactive behavior:**
- Entering a partition key and clicking "Hash It" animates a small token value appearing, the ring spinning to show which node owns it, and the replica nodes lighting up
- Clicking any node shows a tooltip: node ID, token range, current write load (simulated %), estimated latency
- A "Simulate Node Failure" checkbox grays out N4 and re-routes traffic to the next node in the ring

**Learning objective:** Applying (Bloom's) — students experiment with partition key hashing to see how data distributes across nodes, and observe how replica factor and consistency level interact.

Responsive: canvas scales to container width on resize.
</details>

## Apache HBase: HDFS-Backed Strong Consistency

Where Cassandra sacrifices strict consistency for masterless availability, **Apache HBase** takes the opposite position: it provides strong consistency using a leader-based architecture backed by the Hadoop Distributed File System (HDFS).

HBase runs on top of HDFS, the fault-tolerant distributed file system from the Hadoop ecosystem. HDFS handles data replication and durability; HBase adds a row-based access layer on top. Coordination is managed by **Apache ZooKeeper**, which handles leader election, distributed locking, and health monitoring.

Because HBase has a single RegionServer leader per region (contiguous key range), it can guarantee that reads always see the most recent committed write — strong consistency without version conflicts. This makes HBase the preferred wide-column store for applications that require strong consistency guarantees, such as financial ledgers and audit trails.

The tradeoff is operational complexity. Running HBase requires operating HDFS, ZooKeeper, and HBase itself as three separate systems. The cluster is considerably more complex to manage than a Cassandra cluster, which is self-contained and masterless. HBase also tends to perform worse than Cassandra at write throughput because all writes go through a single RegionServer per region, whereas Cassandra's leaderless model distributes writes across all replicas.

The following table compares Cassandra and HBase across the dimensions most relevant to architectural selection:

| Attribute | Apache Cassandra | Apache HBase |
|---|---|---|
| Consistency Model | Tunable (AP default) | Strong (CP) |
| Architecture | Masterless ring | Leader per region (ZK-based) |
| Write Throughput | Very high | High |
| Read Consistency | Eventual to strong | Always strong |
| Dependencies | Cassandra only | HDFS + ZooKeeper + HBase |
| Best For | IoT, time-series, high write | Financial records, audit trails |
| CAP Classification | AP | CP |

## Write Amplification and Read Amplification: The Core Tradeoff

Two related costs appear repeatedly in LSM tree databases, and every column-family practitioner needs to understand them precisely.

**Write amplification** is the ratio of bytes written to physical storage to bytes that the application logically wrote. A single 1 KB application write might eventually result in 10 KB of physical writes after compaction. This happens because compaction merges and re-sorts existing SSTables, rewriting data that has not changed at the application level. Leveled compaction (LCS) is the worst offender for write amplification because it actively compacts across levels to keep read costs low.

**Read amplification** is the number of disk reads required to answer a single logical read operation. In an LSM tree, a point read must check the memtable, then L0 SSTables (which may have overlapping key ranges), then one SSTable per level below that. Without Bloom filters, read amplification in a heavily written system can reach 10–20× or higher. With Bloom filters, the vast majority of "not found in this file" checks become cheap in-memory bit tests, reducing effective read amplification dramatically.

The relationship between these costs defines the fundamental tradeoff of write-optimized storage. You can reduce write amplification by compacting less aggressively — but then you accumulate more SSTables and pay higher read costs. You can reduce read amplification by compacting more aggressively — but then you spend more I/O rewriting data. There is no free lunch; the tradeoff is real, and the right point on the curve depends on your read-to-write ratio.

!!! mascot-warning "Tombstones Are Silent Read Killers"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex holds up a warning sign">
    In Cassandra and HBase, deletes are not immediate — they write a **tombstone marker** that masks the deleted row. Tombstones accumulate until compaction evicts them. If your application deletes rows frequently and compaction is not keeping up, reads must scan through vast numbers of tombstones before finding live data. Cassandra will even throw `TombstoneOverwhelmingException` and abort queries that encounter more than 100,000 tombstones in a single read. Monitor your tombstone counts. Keep your `gc_grace_seconds` tuned. And if you delete a lot, consider TWCS or TTL-based expiry instead of explicit deletes.

#### Diagram: Read Path SSTable Inspection with Bloom Filter Short-Circuits

<details markdown="1">
<summary>Interactive read path: L0→L1→L2 SSTable checks with Bloom filter skip visualization</summary>

**sim-id:** lsm-read-path
**Library:** p5.js
**Status:** Specified

A p5.js canvas (900×480 px) showing the LSM read path step-by-step.

**Visual layout (top to bottom):**
- Row 1: "Memtable" — a sorted list of keys currently in memory
- Row 2: "L0 SSTables" — three overlapping SSTable boxes (key ranges can overlap at L0)
- Row 3: "L1 SSTables" — four non-overlapping SSTable boxes
- Row 4: "L2 SSTables" — one large SSTable box

Each SSTable box has a small Bloom filter icon (a bit array thumbnail) in its corner.

**Controls:**
- Text input "Lookup Key:" with "Search" button
- Toggle "Bloom Filters: ON/OFF"

**Animation sequence on Search:**
1. Memtable lights up, shows "miss" or "hit" (key found in memtable)
2. If miss: L0 SSTable 1 Bloom filter evaluates — if it says "definitely absent" → SSTable grays out with a green checkmark "Skipped (Bloom)"
3. Remaining SSTables are checked sequentially; each either shows "Skip (Bloom)" or "Check disk (binary search)" with a pause animation
4. When the key is found, the SSTable highlights in blue; a summary shows "Checked N SSTables, skipped M via Bloom filter"

**When Bloom Filters OFF:** All SSTables are checked with a disk-read animation, dramatically increasing the step count.

**Learning objective:** Analyzing (Bloom's) — students directly observe how Bloom filters reduce read amplification, and can quantify the savings by toggling the feature.

Responsive: scales to container width on window resize.
</details>

## Key Takeaways

- The **column-family data model** organizes data into sparse, wide rows where columns vary per row — ideal for time-series, IoT, and event-log workloads where data shapes differ across entities.
- **LSM trees** achieve high write throughput by converting random writes into sequential disk flushes. The memtable absorbs writes in memory; SSTables are immutable sorted files on disk.
- **Bloom filters** are probabilistic bit arrays that eliminate unnecessary SSTable reads by definitively answering "this key is not here." They are essential for keeping read costs manageable in LSM systems.
- **Compaction** is mandatory debt repayment: STCS favors writes, LCS favors reads, and TWCS is purpose-built for time-series append-only data.
- **Apache Cassandra** is a masterless AP system with tunable consistency; its **partition key** determines data placement, and its **clustering column** determines sort order within a partition.
- **Apache HBase** is a leader-based CP system backed by HDFS and ZooKeeper, offering strong consistency at the cost of operational complexity.
- **Write amplification** (compaction rewriting existing data) and **read amplification** (checking multiple SSTables per read) are the unavoidable dual costs of the LSM design. Compaction strategy selection is the primary lever for shifting the balance.

!!! mascot-celebration "Chapter 6 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates with raised arms">
    You now understand why column-family databases are the engine of choice for sensor pipelines, audit logs, and time-series analytics at scale. LSM trees, Bloom filters, compaction strategies, partition keys, clustering columns — these are not just vocabulary words. They are the levers your architects will pull when a system needs to absorb a million writes per second without falling over. Chapter 7 turns the page to document databases, where schema flexibility is the hero and JOIN is a dirty word.
