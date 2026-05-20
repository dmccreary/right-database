---
title: Chapter 13 - High Availability Architecture
description: Engineering database systems for five-nines uptime through failure domain analysis, active-active topologies, automated failover, and chaos engineering.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 13: High Availability Architecture

## Summary

This chapter covers the engineering discipline of designing database systems that achieve five-nines (99.999%) availability — approximately 5.26 minutes of allowable downtime per year. Students learn to decompose SLA targets across dependent services, identify and eliminate single points of failure, and design active-active and active-passive topologies with automated failover. Chaos engineering is introduced as the discipline of proactively discovering failure modes before they surface in production. The chapter closes with geographic redundancy and multi-region deployment as the final line of defense against regional outages.

## Concepts Covered

This chapter covers the following 15 concepts from the learning graph:

1. High Availability
2. Five-Nines Availability
3. SLA Decomposition
4. Failure Domain
5. Single Point of Failure
6. Active-Active Clustering
7. Active-Passive Clustering
8. Failover
9. Heartbeat Monitoring
10. Chaos Engineering
11. Mean Time Between Failures
12. Mean Time to Recovery
13. Geographic Redundancy
14. Multi-Region Deployment
15. Circuit Breaker Pattern

## Prerequisites

This chapter builds on concepts from:

- [Chapter 9: CAP Theorem and Consensus Protocols](../09-cap-theorem-consensus/index.md)
- [Chapter 11: Distributed Scaling and Replication](../11-distributed-scaling/index.md)

---

!!! mascot-welcome "In Search of Five Nines"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex the robot welcoming students">
    "Five nines" sounds like a poker hand but it is actually a promise: your database will be unavailable for no more than 5.26 minutes per year. Dex has seen many organizations quote this number in sales decks and then discover, at 3 AM on Black Friday, that they had not actually designed for it. This chapter teaches you how to design for it — not just promise it.

## What High Availability Actually Means

**High availability (HA)** is a system design goal: minimize unplanned downtime, expressed as a percentage of time the system is functioning and reachable. It is important to distinguish *unplanned* downtime (failures) from *planned* downtime (maintenance windows). SLAs typically cover only unplanned downtime, though modern practices aim to eliminate even planned downtime through rolling upgrades and blue/green deployments.

Availability percentages are conventionally stated in terms of "nines":

| Availability | Annual Downtime | Monthly Downtime | Typical Use Case |
|---|---|---|---|
| 99% (two nines) | 87.6 hours | 7.3 hours | Internal tools, batch systems |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes | Standard commercial SaaS |
| 99.99% (four nines) | 52.6 minutes | 4.4 minutes | Financial applications, e-commerce |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds | Telecommunications, critical infrastructure |
| 99.9999% (six nines) | 31.5 seconds | 2.6 seconds | Rarely achieved at system level |

The jump from four nines to five nines is not a small engineering improvement — it is a fundamental architectural shift. Four-nines availability allows roughly one hour of unplanned downtime per year; at that level, a 15-minute database failover is a valid (if painful) option. At five nines, a 15-minute failover is your entire annual downtime budget for 170 years. Five-nines systems must recover in seconds, not minutes.

#### Diagram: Downtime Budget Calculator

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** downtime-budget-calculator
**Library:** p5.js
**Status:** Specified

Show a slider from 99.0% to 99.9999% availability (log scale with stops at each "nines" level). As the user moves the slider, update three displays:

- Annual downtime allowed (in hours, minutes, or seconds — automatically select the appropriate unit)
- Monthly downtime allowed
- A horizontal bar chart showing the downtime budget visually against a 1-year timeline (the bar representing allowed downtime shrinks dramatically as nines increase)

Below the bar chart, show a "Failover Comparison" section listing three common failover mechanisms and whether they fit in the downtime budget:
- Manual failover: ~15 minutes — shows green/red depending on budget
- Automated failover (Raft/Paxos): ~30 seconds — shows green/red
- Active-active with DNS cutover: ~5 seconds — shows green/red

Add a text label that updates: "At this availability level, you can afford approximately N failover events per year of type X."
</details>

## SLA Decomposition: The Math That Hurts

Organizations frequently set a top-level SLA (say, 99.99% for a customer-facing API) without calculating what that implies for each component in the dependency chain. This omission is costly.

**SLA decomposition** is the practice of breaking the top-level SLA into per-component availability budgets. The key mathematical insight is that **serial dependencies multiply downtime**. If service A depends on service B, and both have independent failure probabilities, the combined availability is:

```
A_combined = A_a × A_b
```

A system with three serial dependencies, each at 99.99% availability, delivers only:

```
0.9999 × 0.9999 × 0.9999 = 0.9997 ≈ 99.97% combined availability
```

That combined 99.97% allows 2.6 hours of downtime per year — even though every individual component met four nines. If you need the system to deliver 99.99%, each component must be at roughly 99.9997%. Every additional dependency in your dependency chain makes achieving your top-level SLA harder.

This is why architects obsessively minimize critical-path dependencies and use asynchronous communication patterns wherever possible. A dependency that is not in the critical path does not contribute to downtime when it fails.

## Failure Domains: Thinking in Blast Radii

A **failure domain** is a set of components that share a common failure cause. Components in the same failure domain fail together when that cause occurs. Common failure domains include:

- **Same physical server** — power supply failure, kernel panic, disk failure
- **Same rack** — Top-of-rack switch failure, rack power distribution unit (PDU) failure
- **Same Availability Zone (AZ)** — cooling failure, power substation failure
- **Same cloud region** — regional network outage, natural disaster
- **Same cloud provider** — global provider incidents (rare but documented)

HA architecture is fundamentally about ensuring that no single failure domain contains enough of your critical components to take the system down. A three-node database cluster where all three nodes are in the same rack is not highly available — a single rack failure takes all three nodes offline simultaneously.

A **single point of failure (SPOF)** is the degenerate case: a component whose failure alone brings down the entire system. Common database SPOFs include:

- A single primary database node with no automatic failover
- A single load balancer with no standby
- A single network switch on the path between application servers and the database
- A shared storage volume (SAN) attached to multiple "redundant" nodes

Finding and eliminating SPOFs is the foundational exercise of HA design. Walk your architecture diagram and ask: "If this component fails right now, does the system stay up?" For every "no," you have found a SPOF.

!!! mascot-warning "The Hidden SPOF"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex the robot raising a warning flag">
    The most dangerous SPOFs are the invisible ones. The load balancer is redundant — but both load balancers share a single configuration management server. The database has two nodes in different AZs — but both use the same Route 53 hosted zone, which has its own availability. SPOFs hide in the gaps between explicitly designed components. Threat model your architecture like an adversary would: what does an attacker need to take down to cause maximum impact?

## Active-Active and Active-Passive Clustering

Once SPOFs are eliminated, the next architectural decision is how redundant nodes handle traffic.

**Active-passive clustering** maintains one active node serving all traffic and one or more passive (standby) nodes that receive replication but serve no client traffic. On failure of the active node, the passive node is promoted to active — a process called **failover**.

Active-passive is simpler to implement and easier to reason about because there is always exactly one authoritative source of state. Its weaknesses are: the passive node is wasted capacity during normal operation, and failover introduces a brief period of unavailability while the passive node is promoted.

**Active-active clustering** runs all nodes simultaneously serving traffic. A load balancer distributes requests across all active nodes. Every node can accept reads and writes (in eventually consistent systems) or writes are distributed to any node and replicated (in systems with multi-leader or leaderless replication). Active-active provides both higher capacity and higher availability: the loss of one node immediately reduces capacity but does not cause an outage.

Active-active is more complex. It requires the application to tolerate serving requests from any node, requires conflict resolution if writes are distributed, and requires session affinity if the application holds server-side state. Database systems that support active-active natively include Cassandra (leaderless), YugabyteDB (multi-leader YCQL), and CockroachDB (all nodes accept reads and writes through Raft).

#### Diagram: Active-Active vs Active-Passive Failover

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** active-passive-vs-active-active
**Library:** p5.js
**Status:** Specified

Show two panels side by side, labeled "Active-Passive" and "Active-Active."

**Active-Passive panel:** Show a load balancer at top feeding into two nodes: Node 1 (green, "Active") and Node 2 (gray, "Passive"). Animate client request arrows flowing only to Node 1. Node 2 shows a replication arrow from Node 1. Provide a "Kill Node 1" button. Animate: Node 1 turns red (failed). A "Failover: 28 seconds" timer counts up. Node 2 turns green and labeled "Active." Client arrows redirect to Node 2. Show a gap in the request flow timeline during failover.

**Active-Active panel:** Show a load balancer feeding into two nodes, both green ("Active"). Animate client arrows splitting between both nodes. Provide a "Kill Node 1" button. Animate: Node 1 turns red. Client arrows instantly route entirely to Node 2 with no gap. Show "Capacity reduced: 50%" label. A "Recovery: Node rejoins" button restores Node 1 and traffic spreads again.

Show scorecard: Capacity utilization (50% vs 100%), Failover time (28 sec vs 0 sec), Complexity (Low vs High).
</details>

## Heartbeat Monitoring and Failover

**Heartbeat monitoring** is the mechanism by which nodes in a cluster detect each other's failure. Each node sends periodic "I am alive" signals — heartbeats — to its peers or to a central monitoring agent. When a heartbeat is missed for a configurable number of intervals, the monitoring system declares the node failed and initiates failover.

The challenge is tuning heartbeat intervals and failure thresholds. Too sensitive (short intervals, low miss threshold), and a transient network hiccup triggers an unnecessary failover — causing brief downtime in an attempt to prevent downtime. Too insensitive (long intervals, high miss threshold), and a genuine failure takes too long to detect.

A common configuration for a database cluster is a heartbeat interval of 1–2 seconds with a failure threshold of 3 consecutive missed heartbeats — detecting failure in 3–6 seconds. This is fast enough for automated failover to fit within a four-nines budget.

**Failover** is the process of promoting a standby to active after a failure is detected. Key steps in an automated failover sequence:

1. Failure detected by heartbeat monitoring
2. Leader election determines the new primary (Raft or similar algorithm)
3. New primary is promoted; old primary is fenced (STONITH or equivalent) to prevent split-brain
4. DNS or load balancer is updated to route traffic to the new primary
5. Application connections are re-established (connection pool reconnect logic)
6. Replication streams are re-established from the new primary to remaining replicas

Each step takes time. The sum of all steps determines the actual failover window — what appears in the SLA as Recovery Time Objective (RTO).

## MTBF, MTTR, and the Availability Formula

Two metrics drive availability calculations:

**Mean Time Between Failures (MTBF)** is the average time a system runs before experiencing a failure. A higher MTBF means failures happen less frequently — the system is more reliable.

**Mean Time to Recovery (MTTR)** is the average time to restore service after a failure. A lower MTTR means failures are resolved faster.

The relationship between these metrics and availability is:

```
Availability = MTBF / (MTBF + MTTR)
```

This formula has a critical implication: **MTTR matters more than MTBF for high-availability systems**. Consider two scenarios:

- System A: MTBF = 1000 hours, MTTR = 1 hour → Availability = 99.9%
- System B: MTBF = 100 hours (10x more failures!), MTTR = 0.1 hours → Availability = 99.9%

System B fails ten times as often but recovers ten times faster — identical availability. Investing in fast recovery (automated failover, runbooks, chaos engineering) often yields better availability ROI than investing in hardware reliability.

#### Diagram: MTBF and MTTR Availability Calculator

<details markdown="1">
<summary>MicroSim Specification</summary>

**sim-id:** mtbf-mttr-availability
**Library:** p5.js
**Status:** Specified

Show two sliders: MTBF (1–10,000 hours) and MTTR (0.1–24 hours), both on log scales.

Display:
- Computed availability: `MTBF / (MTBF + MTTR)` in percentage, updated live
- Number of "nines" (one decimal place: 99.9% = three nines)
- Annual downtime in the appropriate unit

Show a 2D heat map plot where the X-axis is MTBF (log scale) and Y-axis is MTTR (log scale). The color encodes availability from red (low) to green (high). Mark the current slider position as a white dot. Show availability contour lines at 99%, 99.9%, 99.99%, 99.999%.

Include two labeled example points: "Average enterprise DB cluster" (MTBF=2000h, MTTR=2h) and "High-HA cluster" (MTBF=1000h, MTTR=0.1h). Show how moving the white dot toward low MTTR is more effective at crossing contour lines than increasing MTBF.
</details>

## Chaos Engineering: Breaking Things on Purpose

**Chaos engineering** is the discipline of deliberately injecting failures into a production (or production-equivalent) system to discover weaknesses before users encounter them. The term was coined by Netflix, which open-sourced the **Chaos Monkey** tool — a service that randomly terminates virtual machine instances in production during business hours.

The philosophy behind chaos engineering is that complex distributed systems will fail in ways that are impossible to predict from code review or staging environment testing. The only way to know your system handles a failure gracefully is to cause that failure in a controlled way and observe what happens. The alternative — discovering failure modes during an actual incident — is much more expensive.

A chaos engineering experiment follows a structured process:

1. **Define steady state** — what does normal behavior look like? (e.g., p99 latency < 200ms, error rate < 0.1%)
2. **Hypothesize** — "We believe that if a database replica fails, failover will complete in < 30 seconds without a steady-state violation."
3. **Inject the failure** — terminate the replica process, block network traffic, fill the disk, inject CPU contention.
4. **Observe** — measure latency, error rates, and failover time against the steady-state baseline.
5. **Improve** — if the hypothesis was violated, fix the underlying weakness. If not, expand the blast radius.

Common chaos experiments for database systems include:

- Killing the primary database node and measuring failover time
- Blocking replication between primary and replica to test replication lag alerting
- Filling the primary disk to test disk-full behavior
- Introducing 200ms artificial network latency between application servers and the database
- Killing the connection pool manager and testing reconnection behavior

!!! mascot-encourage "Start Small, Go Big"
    <img src="../../img/mascot/encouraging.png" class="mascot-admonition-img" alt="Dex the robot encouraging the reader">
    Your first chaos experiment should be the simplest possible thing: restart one database replica during off-peak hours and confirm that your monitoring alerts fire and your application does not throw errors. Then try restarting the primary. Then try blocking network traffic. The discipline builds confidence incrementally. Nobody goes straight to "terminate the primary in all three regions simultaneously" — well, nobody who still has a job.

## Geographic Redundancy and Multi-Region Deployment

Single-region deployments are vulnerable to events that affect an entire data center or availability zone — power grid failures, natural disasters, regional network outages, cooling system failures. For systems requiring five-nines availability, geographic redundancy is not optional.

**Geographic redundancy** means replicating data and compute across physically separated facilities — different buildings, cities, or countries. The two levels of geographic redundancy are:

**Multi-AZ deployment**: Active database nodes span two or three availability zones within a single cloud region. AZs are physically separate data centers within a region, typically 10–100 km apart, with independent power and networking. Most managed database services (AWS RDS Multi-AZ, Google Cloud SQL HA) implement multi-AZ as their standard HA offering. Multi-AZ protects against individual data center failures; it does not protect against region-wide outages.

**Multi-region deployment**: Active database nodes span two or more geographically separated cloud regions (e.g., us-east-1 and eu-west-1). This protects against regional outages — historically rare but real events (AWS us-east-1 and ap-southeast-2 have both had significant region-wide incidents). Multi-region introduces latency: writes must be replicated across the geographic distance (typically 40–150ms between US and EU regions), which impacts consistency and write latency.

## Circuit Breaker Pattern

No discussion of HA is complete without the **circuit breaker pattern** — a resilience mechanism at the application layer that prevents cascading failures.

When service A calls service B (e.g., when an application server calls the database), and service B begins returning errors or timing out, service A faces a choice: keep retrying (potentially overwhelming the already-struggling service B) or fail fast. The circuit breaker pattern provides a third option: track the failure rate and, when it exceeds a threshold, "open the circuit" — stop calling service B entirely and return a cached result or fallback error immediately.

A circuit breaker has three states:

- **Closed** (normal): requests pass through. Failures are counted.
- **Open** (tripped): a failure threshold has been exceeded. All requests fail immediately without calling the downstream service. A timer runs.
- **Half-open** (testing recovery): the timer expires. One probe request is allowed through. If it succeeds, the circuit closes. If it fails, the circuit reopens.

For database connections, circuit breakers prevent a slow database from causing application threads to pile up waiting for timeouts, eventually exhausting the connection pool and taking down the application layer. Libraries like Netflix Hystrix, Resilience4j (Java), and Polly (.NET) implement circuit breaker patterns with configurable thresholds and fallback strategies.

The circuit breaker pattern pairs naturally with **read replicas** and **caching**: when the circuit opens on the primary database, the application can fall back to a read replica for read operations or serve cached data rather than failing completely.

## Designing for HA: A Practical Checklist

Translating the concepts in this chapter into practice means answering the following questions for every database deployment:

- What is the target availability SLA, and what per-component budget does that imply?
- What failure domains exist, and are redundant nodes distributed across different failure domains?
- What are all the single points of failure in the current architecture?
- Is failover automated? What is the measured failover time under the chosen topology?
- What heartbeat interval and failure threshold parameters balance sensitivity against false positives?
- Are MTBF and MTTR measured? Is the team investing in MTTR reduction?
- Is a chaos engineering program in place? When was the last time failover was tested?
- For multi-region deployments: what is the write replication latency, and does it violate consistency requirements?
- Is a circuit breaker implemented on every path from application to database?
- Is the operational runbook documented and tested (not just written)?

## Key Takeaways

- **Five-nines availability** allows only 5.26 minutes of downtime per year; achieving it requires architecture-level design, not just good hardware.
- **SLA decomposition** reveals that serial dependencies multiply downtime probability — every additional dependency makes achieving a top-level SLA harder.
- **Failure domains** define blast radii; redundant components must span different failure domains (rack, AZ, region) to be effective.
- **SPOFs** must be explicitly identified and eliminated; they frequently hide in the gaps between explicitly designed components.
- **Active-active clustering** eliminates failover time at the cost of complexity; **active-passive** is simpler but has a brief failover window.
- **Heartbeat monitoring** triggers failover; tuning heartbeat sensitivity requires balancing false positives against detection speed.
- **MTTR drives availability more than MTBF** — invest in fast recovery over hardware reliability for better availability ROI.
- **Chaos engineering** is the only reliable way to validate HA claims — design what you believe, test what you designed.
- **Geographic redundancy** across AZs handles data center failures; **multi-region deployment** protects against regional outages at the cost of cross-region latency.
- The **circuit breaker pattern** prevents cascading failures when the database degrades; it is the application-layer complement to database-level HA.

!!! mascot-celebration "Five Nines Earned"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex the robot celebrating">
    You now know what it takes to keep a database running when hardware fails, networks partition, data centers flood, and developers accidentally run DROP TABLE in production. Chapter 14 shifts gears entirely: away from keeping data alive, toward a radically different way of finding it — through the geometry of meaning, using vector search.
