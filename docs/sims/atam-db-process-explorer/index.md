---
title: ATAM Database Selection Process Explorer
description: Interactive diagram of the ATAM database selection process — click any node to learn its role in transforming requirements and architectural approaches into documented analysis outputs.
status: implemented
library: p5.js
bloom_level: 2
bloom_verb: explain
---

# ATAM Database Selection Process Explorer

<iframe src="main.html" width="100%" height="562" scrolling="no"></iframe>

## Learning Objective

Understanding (Bloom's level 2) — students explain the role of each element in the ATAM database selection process and describe how requirements and architectural approaches flow through the process to produce analysis outputs.

- **Bloom Level:** 2 — Understand
- **Bloom Verb:** Explain
- **Library:** p5.js

## Preview

<iframe src="main.html" width="100%" height="600"></iframe>

[Run MicroSim in Fullscreen](main.html){ .md-button .md-button--primary }

## Specification

The full specification below is extracted from
[Chapter 1: Chapter 1 - The ATAM Method](../../chapters/01-atam-method/index.md).

```text
Type: MicroSim
**sim-id:** atam-db-process-explorer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning objective:** Understanding (Bloom's level 2) — students explain the role of each element in the ATAM database selection process and describe how requirements and architectural approaches flow through the process to produce analysis outputs.

**Canvas:** 900 × 520px, responsive to window resize events. The diagram reproduces the flow from ATAM-db-process.png as interactive nodes. An info panel (300px) slides in from the right when a node is clicked.

**Node layout (two rows + output column):**

Row 1 (requirements path, y=120, color #f0a500 orange boxes with blue border):
- Node A: "Business Drivers" (x=100)
- Node B: "Quality Attributes" (x=320)
- Node C: "User Stories" (x=540)
- Node Analysis: "Analysis" (x=760, shape: ellipse, color #b0c8e8)

Row 2 (design path, y=250, color #ffff99 yellow boxes with blue border):
- Node D: "Architecture Plan" (x=100)
- Node E: "Architectural Approaches" (x=320) — contains a small icon grid of 6 DB type thumbnails (text labels: Relational, Analytical, Key-Value, Column-Family, Graph, Document)
- Node F: "Architectural Decisions" (x=540)

Output column (right side, x=760):
- Node T: "Tradeoffs" (y=300, color #aaaaaa gray)
- Node S: "Sensitivity Points" (y=350, color #aaaaaa)
- Node NR: "Non-Risks" (y=400, color #88cc88 green)
- Node R: "Risks" (y=450, color #e8a0a0 pink)

Bottom row:
- Node RT: "Risk Themes" (x=100, y=400, color #cc0000 red, white text)

Arrows:
- A → B → C → Analysis (horizontal, row 1)
- D → E → F → Analysis (horizontal, row 2)
- Analysis → T, S, NR, R (right column outputs)
- R → RT (labeled "Distilled info")
- RT → A (labeled "Impacts", left side vertical feedback arrow)

**Info panel content for each node:**

Node A (Business Drivers):
"Business drivers are the organizational goals and constraints that determine what quality attributes matter most for this system. They translate strategic intent into architectural requirements. Examples: regulatory compliance, time-to-market pressure, scale requirements, cost constraints."

Node B (Quality Attributes):
"Quality attributes (or 'ilities') are the non-functional characteristics by which architectural fitness is measured: availability, consistency, scalability, performance, operability, security. Each business driver maps to one or more quality attributes that the architecture must satisfy."

Node C (User Stories):
"In the ATAM context, user stories represent quality attribute scenarios — concrete, testable expressions of what the system must do under specified conditions. Each leaf of the utility tree is a quality attribute scenario with a measurable response criterion."

Node D (Architecture Plan):
"The architecture plan documents the existing or proposed system architecture using multiple views: component diagram, deployment topology, data flow, and sequence views for key scenarios. This is what the evaluation team analyzes."

Node E (Architectural Approaches):
"For database selection, the architectural approaches are the candidate database paradigms and their specific configurations: Relational (PostgreSQL, MySQL), Analytical (Snowflake, BigQuery), Key-Value (Redis, DynamoDB), Column-Family (Cassandra, HBase), Graph (Neo4j, TigerGraph), and Document (MongoDB, Firestore). Each approach is evaluated against the prioritized utility tree scenarios."

Node F (Architectural Decisions):
"Architectural decisions are the specific choices made about how to implement the architecture — which database type, which replication topology, which consistency model, which sharding strategy. ATAM analysis produces documented justification for each significant decision."

Node Analysis (Analysis):
"Analysis is the core ATAM activity: probing each architectural approach against each prioritized quality attribute scenario to identify sensitivity points, tradeoff points, risks, and non-risks. It is structured, evidence-based, and produces documented findings."

Node T (Tradeoffs):
"Tradeoff points are properties of the architecture that are sensitivity points for multiple quality attributes simultaneously — satisfying one quality attribute necessarily degrades another. Example: increasing replication factor improves availability but increases write latency."

Node S (Sensitivity Points):
"Sensitivity points are properties of one or more components that are critical to achieving a specific quality attribute. A small change in the property produces a large change in the quality attribute. Example: the quorum size parameter is a sensitivity point for both consistency and availability."

Node NR (Non-Risks):
"Non-risks are architectural decisions confirmed to positively contribute to the system's quality attribute scenarios. Documenting non-risks is as important as documenting risks — they justify parts of the architecture that are correct and prevent unnecessary second-guessing."

Node R (Risks):
"Architectural risks are potentially unsatisfied quality attribute scenarios. They identify places where the architecture may fail to meet requirements, not certain failures. Each risk should include a description, the affected scenario, the likelihood assessment, and the recommended mitigation."

Node RT (Risk Themes):
"Risk themes are patterns of risks that share a common root cause or collectively point to a systemic architectural weakness. Identifying risk themes allows teams to address root causes rather than symptoms. Risk themes feed back to inform business driver prioritization."

**Interaction:**
- Click any node to open the info panel on the right.
- Clicking a node also highlights its incoming and outgoing arrows in bright blue; all other arrows remain dark gray.
- The info panel has a close (×) button.
- Hovering any node shows a tooltip with its short name.

**Color scheme:** Orange boxes = requirements path; yellow boxes = design path; gray/green/pink/red = analysis outputs; blue ellipse = analysis process.

**Responsive behavior:** On resize, all elements scale proportionally. Node labels truncate with ellipsis if canvas width drops below 700px. Below 500px, layout switches to a vertical single-column list of nodes, each expandable.
```

## Related Resources

- [Chapter 1: Chapter 1 - The ATAM Method](../../chapters/01-atam-method/index.md)
