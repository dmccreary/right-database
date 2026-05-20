---
title: Workload Characterization Explorer
description: Interactive radar chart comparing OLTP, OLAP, and HTAP workloads across 7 dimensions — click a workload card to see its profile, storage engine, canonical examples, and when to choose it.
status: implemented
library: p5.js
bloom_level: 4
bloom_verb: compare
---

# Workload Characterization Explorer

<iframe src="main.html" width="100%" height="582" scrolling="no"></iframe>

## Learning Objective

Analyzing (Bloom's level 4) — students compare the architectural signatures of OLTP, OLAP, and HTAP workloads across seven dimensions and identify which database design choices each workload drives.

!!! warning "Scaffold"
    This MicroSim has been scaffolded from its specification. The interactive
    implementation has not been built yet.

## Learning Objective

Analyzing (Bloom's level 4) — students compare the architectural signatures of OLTP, OLAP, and HTAP workloads across seven dimensions and identify which database design choices each workload drives.

- **Bloom Level:** TBD
- **Bloom Verb:** TBD
- **Library:** p5.js

## Preview

<iframe src="main.html" width="100%" height="600"></iframe>

[Run MicroSim in Fullscreen](main.html){ .md-button .md-button--primary }

## Specification

The full specification below is extracted from
[Chapter 2: Chapter 2 - Database Foundations](../../chapters/02-database-foundations/index.md).

```text
Type: interactive-infographic
**sim-id:** workload-characterization-explorer
**Library:** p5.js
**Status:** Specified

**Learning objective:** Analyzing (Bloom's level 4) — students compare the architectural signatures of OLTP, OLAP, and HTAP workloads across seven dimensions and identify which database design choices each workload drives.

**Canvas:** 900 × 540px, responsive. Left panel (280px): three large clickable workload cards (OLTP, OLAP, HTAP). Right panel (580px): a radar chart showing the selected workload's profile across seven axes, plus a text summary below.

**Workload data:**

Seven radar axes (each scored 1–5):
- Concurrency (simultaneous users)
- Write frequency
- Read frequency
- Result set size
- Query complexity
- Latency sensitivity
- Consistency strictness

OLTP scores: Concurrency=5, Write=4, Read=4, ResultSize=1, Complexity=2, Latency=5, Consistency=5
OLAP scores: Concurrency=1, Write=1, Read=5, ResultSize=5, Complexity=5, Latency=2, Consistency=2
HTAP scores: Concurrency=4, Write=4, Read=5, ResultSize=3, Complexity=4, Latency=4, Consistency=4

**Left panel cards:**
- OLTP card: blue (#4682B4), icon of "⚡ Fast Transactions", subtitle "Point reads/writes at scale"
- OLAP card: purple (#6f42c1), icon of "📊 Deep Analysis", subtitle "Aggregate scans over history"
- HTAP card: orange (#E65100), icon of "🔀 Hybrid", subtitle "Both, from one system"

**Right panel:**
Radar chart with axes labeled. Selected workload polygon filled at 40% opacity in the workload's card color. A second semi-transparent grey polygon shows the "other" workload profiles at low opacity for comparison.

Below the radar: a text block with three subsections:
- "Typical storage engine" (B-tree / Columnar / Both)
- "Canonical database examples" (PostgreSQL, MySQL / Snowflake, BigQuery / TiDB, SingleStore)
- "When to choose this workload pattern" (2-sentence description)

**Interaction:**
- Clicking a workload card highlights it and updates the radar and text panel with animation (300ms ease).
- Default selected on load: OLTP.
- Hovering a radar axis label shows a tooltip with a one-sentence description of that dimension.

**Responsive:** Below 700px, left panel stacks above the radar chart.
```

## Related Resources

- [Chapter 2: Chapter 2 - Database Foundations](../../chapters/02-database-foundations/index.md)
