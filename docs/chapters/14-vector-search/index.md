---
title: Chapter 14 - Vector Search as a Database Feature
description: Vector embeddings, similarity metrics, ANN indexes (HNSW, IVF, flat), pgvector, semantic search, hybrid search — and how vector search integrates into any database paradigm.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 14: Vector Search as a Database Feature

## Summary

This chapter introduces vector search as a cross-cutting database capability — not a separate paradigm, but a feature that can be added to relational, document, key-value, and graph databases to enable similarity retrieval across any item or property. Students learn the mathematics of vector similarity (cosine, dot product, Euclidean distance), how approximate nearest-neighbor indexes (HNSW, IVF, flat) trade recall for speed, and how native vector search extensions such as pgvector integrate into existing databases. Semantic search and hybrid search architectures are examined as the primary use cases.

## Concepts Covered

This chapter covers the following 14 concepts from the learning graph:

1. Vector Embedding
2. Embedding Dimensionality
3. Cosine Similarity
4. Dot Product Similarity
5. Euclidean Distance
6. Approximate Nearest Neighbor
7. HNSW Index
8. IVF Index
9. Flat Vector Index
10. pgvector Extension
11. Semantic Search
12. Hybrid Search
13. Native Vector Search Feature
14. ANN Recall vs Speed

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)
- [Chapter 3: Relational Databases](../03-relational-databases/index.md)

---

!!! mascot-welcome "Welcome to Chapter 14!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Vector search is one of the most consequential additions to database technology in the past decade — and it is not a new database paradigm. It is a feature. The insight this chapter delivers is that any database can become a semantic search engine by adding a vector index alongside its existing indexes. The relational database you already operate can find documents similar to a query, products similar to one a user liked, or code similar to a bug report — without migrating to a separate system. That is a powerful architectural option, and this chapter gives you the technical foundation to use it well.

## What Is a Vector Embedding?

A **vector embedding** is a dense array of floating-point numbers that represents an item — a document, an image, a product, a piece of code, a user profile — as a point in a high-dimensional geometric space. Items that are semantically similar are placed close together in this space; items that are semantically different are placed far apart.

The crucial property of embeddings is that geometric proximity corresponds to semantic similarity. Two product descriptions that describe the same kind of item will have embeddings that are close together, even if they share no keywords. A query "comfortable running shoes" will be close to "lightweight athletic footwear" in embedding space, enabling retrieval by meaning rather than by keyword matching.

Before examining similarity metrics, define two terms: **dimensionality** refers to the number of values in the embedding vector (the number of dimensions in the geometric space); **embedding model** is the machine learning model that converts an item (text, image, etc.) into its embedding vector. Chapter 15 covers embedding models in depth; this chapter focuses on what the database does with the vectors once they exist.

**Embedding dimensionality** ranges from 256 to 3072 dimensions in common embedding models. Higher dimensionality generally captures more semantic nuance at the cost of more storage and slower index operations. A 1536-dimensional embedding requires 6KB of storage per vector (1536 × 4 bytes for float32). At 10 million documents, that is 60GB of vector data — a significant consideration in database design.

#### Diagram: 2D Embedding Space Visualizer

<details markdown="1">
<summary>Interactive 2D Embedding Space — Semantic Proximity Explorer</summary>
Type: MicroSim
**sim-id:** embedding-space-2d<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Explain how vector embeddings place semantically similar items close together in geometric space, enabling similarity retrieval. (Bloom L2: Understand)

**Canvas:** 720px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
A 2D scatter plot (representing a 2D projection of a high-dimensional embedding space). Pre-populated with 30 labeled points grouped in semantic clusters:
- Cluster 1 (top-left, blue): "running shoes", "athletic footwear", "sneakers", "trail runners"
- Cluster 2 (top-right, orange): "laptop", "notebook computer", "MacBook", "ultrabook"
- Cluster 3 (bottom-left, green): "pasta", "spaghetti", "carbonara", "Italian cuisine"
- Cluster 4 (bottom-right, purple): "jazz music", "trumpet", "Miles Davis", "bebop"

A search input at the top lets the user type a query term. Clicking "Find Similar" animates a gold star at an approximate position in the space (based on semantic category) and draws dashed circles indicating "nearest neighbor" radius. Points within the circle are highlighted with a "Match: 94% similar" badge. Points outside fade slightly.

**Interactions:**
- Hovering any point shows its label and a fake cosine similarity score to the query.
- A "Show Distance" toggle draws lines from the query point to all other points, colored by similarity (green = close, red = far).
- A note at bottom: "This 2D view is a simplified projection. Real embeddings have 768–3072 dimensions."

**Responsive:** Redraws on window resize.
</details>

## Vector Similarity Metrics

Three similarity metrics are used to measure how close two vectors are. Before examining them, establish the notation: vector **a** and vector **b** are each arrays of n floating-point numbers, where n is the embedding dimensionality.

### Cosine Similarity

**Cosine similarity** measures the angle between two vectors, ignoring their magnitude. It is defined as the dot product of the vectors divided by the product of their magnitudes:

\[ \text{cosine\_similarity}(\mathbf{a}, \mathbf{b}) = \frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\| \|\mathbf{b}\|} \]

The result ranges from -1 (opposite directions) to +1 (identical directions), with 0 indicating orthogonality (no similarity). For text embeddings, cosine similarity is the most common choice because it is magnitude-invariant — a short document and a long document about the same topic can have identical cosine similarity to a query, regardless of their different raw magnitudes.

### Dot Product Similarity

**Dot product similarity** is the raw dot product \( \mathbf{a} \cdot \mathbf{b} = \sum_{i=1}^{n} a_i b_i \) without normalization. It is faster to compute than cosine similarity (no normalization step) and is preferred when embedding vectors are unit-normalized (magnitude = 1), in which case dot product and cosine similarity are equivalent. OpenAI's embedding API returns unit-normalized vectors, making dot product the recommended metric for those embeddings.

### Euclidean Distance

**Euclidean distance** is the straight-line distance between two points in the embedding space:

\[ d(\mathbf{a}, \mathbf{b}) = \sqrt{\sum_{i=1}^{n} (a_i - b_i)^2} \]

Unlike cosine and dot product (higher = more similar), Euclidean distance produces lower values for more similar vectors. It is magnitude-sensitive — vectors with the same direction but different lengths are not considered identical. Euclidean distance is preferred for embedding models where magnitude encodes meaningful information.

| Metric | Range | Preferred when |
|--------|-------|----------------|
| Cosine Similarity | -1 to +1 | Text embeddings, magnitude doesn't matter |
| Dot Product | Unbounded | Unit-normalized vectors (faster computation) |
| Euclidean Distance | 0 to ∞ | Magnitude is meaningful; image embeddings |

#### Diagram: Similarity Metric Calculator

<details markdown="1">
<summary>Interactive Vector Similarity Metric Calculator</summary>
Type: MicroSim
**sim-id:** similarity-metric-calculator<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Calculate cosine similarity, dot product, and Euclidean distance for two vectors and explain which metric is appropriate for different embedding types. (Bloom L3: Apply)

**Canvas:** 760px wide × 460px tall. CANVAS_HEIGHT: 460.

**Description:**
Two 3-dimensional vectors displayed as editable sliders (3 sliders each, range -2 to +2): Vector A (blue) and Vector B (orange). A geometric visualization on the left shows both vectors as arrows from the origin in 3D (rendered as 2D projection with perspective).

**Computed values panel (right side):**
- Cosine Similarity: computed live as sliders move. Formula shown.
- Dot Product: computed live.
- Euclidean Distance: computed live.

When both vectors point in the same direction (sliders set identically), cosine = 1.0, highlighted green. When vectors are perpendicular, cosine ≈ 0. When opposite, cosine = -1.0.

A "Unit Normalize" button normalizes both vectors to magnitude 1 and shows that dot product now equals cosine similarity.

A "Reset to Text Example" button sets vectors to typical word embedding values (A = [0.8, 0.2, -0.3] representing "king", B = [0.7, 0.3, -0.4] representing "queen") and shows high cosine similarity (≈ 0.99).

**Responsive:** Redraws on window resize.
</details>

## The Nearest-Neighbor Problem and Approximate Solutions

Given a query vector, **nearest-neighbor search** finds the k vectors in a dataset that are most similar (closest) to the query. The naive approach — compute the similarity between the query and every vector in the dataset — is called **exact nearest-neighbor** or **flat** search. It guarantees perfect recall (finding the true k nearest neighbors) but is O(n × d) per query, where n is the number of vectors and d is the dimensionality. At 10 million 1536-dimensional vectors, a single flat search requires 15 billion multiply-add operations.

**Approximate Nearest Neighbor (ANN)** algorithms trade a small amount of recall accuracy for dramatically faster query times by searching only a subset of the vector space. The fundamental tradeoff: higher speed means a higher probability of missing a true nearest neighbor. This tradeoff is the central design decision in vector index selection.

**ANN recall** is the fraction of true nearest neighbors returned in the approximate results. An ANN index with recall@10 = 0.95 means that on average, 9.5 of the true 10 nearest neighbors appear in the results. The missing 0.5 are close vectors that the ANN algorithm did not examine.

### The Flat Vector Index

A **flat vector index** performs exact nearest-neighbor search: every vector is compared to the query. Recall is 100% by definition. Flat indexes are appropriate when the dataset is small (< 1 million vectors), when recall must be perfect (medical imaging, legal document retrieval), or as a baseline for measuring ANN accuracy.

### HNSW: Hierarchical Navigable Small World

**HNSW (Hierarchical Navigable Small World)** is the most widely deployed ANN algorithm for high-recall, low-latency workloads. It builds a multi-layer graph where each vector is a node and edges connect nearby vectors. The graph is "navigable" in the small-world sense: any two nodes are reachable via a short path, like the six-degrees-of-separation phenomenon in social networks.

To answer a query, HNSW starts at a random entry point in the top (coarsest) layer and greedily navigates toward the query vector — at each step moving to the neighbor most similar to the query. It descends through layers, using coarser layers for long-distance navigation and finer layers for local refinement. The search terminates when no neighbor is closer than the current node.

HNSW's key parameters:

- **M** — maximum number of edges per node. Higher M = better recall, more memory, slower index build.
- **ef_construction** — the size of the candidate list during index build. Higher = better graph quality, slower build.
- **ef** (search-time parameter) — the size of the candidate list during search. Higher ef = higher recall, higher latency.

HNSW achieves excellent recall (95–99%) with query latencies in the 1–10ms range at million-scale datasets.

### IVF: Inverted File Index

**IVF (Inverted File Index)** clusters vectors into k groups (centroids) using k-means. The index stores vectors by their centroid assignment. To answer a query, IVF:

1. Computes similarity between the query and all k centroids.
2. Selects the `nprobe` most similar centroids.
3. Performs flat search within only those `nprobe` clusters.

The key parameter is `nprobe`: higher nprobe searches more clusters (higher recall, higher latency); lower nprobe is faster but misses vectors in unexamined clusters.

IVF is more memory-efficient than HNSW (no graph structure) and faster to build, but typically achieves lower recall at the same query latency. It is commonly combined with quantization (IVF-PQ) to compress vectors and reduce memory further.

#### Diagram: HNSW vs IVF ANN Index Explorer

<details markdown="1">
<summary>Interactive HNSW vs IVF Index — Recall vs Speed Tradeoff</summary>
Type: MicroSim
**sim-id:** ann-index-comparison<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Evaluate the recall-speed tradeoff for HNSW and IVF ANN indexes and select the appropriate index type for a given accuracy and latency requirement. (Bloom L5: Evaluate)

**Canvas:** 780px wide × 480px tall. CANVAS_HEIGHT: 480.

**Description:**
A 2D scatter plot with axes: X = Query Latency (ms, log scale: 1–1000ms), Y = Recall@10 (0%–100%).

Pre-plotted points representing different configurations:
- Flat (exact): Recall=100%, Latency=500ms (far right, top)
- HNSW ef=16: Recall=92%, Latency=2ms (far left, upper)
- HNSW ef=64: Recall=97%, Latency=5ms
- HNSW ef=200: Recall=99%, Latency=12ms
- IVF nprobe=4: Recall=85%, Latency=1ms
- IVF nprobe=32: Recall=94%, Latency=8ms
- IVF nprobe=128: Recall=98%, Latency=30ms

Two separate curves are drawn through the HNSW points (blue) and IVF points (orange), showing the recall-latency Pareto frontier for each algorithm.

**Interactions:**
- Hovering any point shows: "HNSW ef=64: 97% recall, 5ms p50 latency. At 1M vectors, 1536 dimensions."
- Two draggable sliders (ef for HNSW, nprobe for IVF) animate the highlighted point along each curve.
- A "My Requirement" crosshair can be dragged onto the chart: "I need ≥95% recall and ≤10ms latency." It highlights all configurations that satisfy both constraints (green zone).

**Responsive:** Redraws on window resize.
</details>

## pgvector: Vector Search in PostgreSQL

**pgvector** is an open-source PostgreSQL extension that adds a native vector data type and ANN index support to PostgreSQL. It is the clearest example of **native vector search feature** integration: rather than routing similarity queries to a separate vector database, the application sends them to the same PostgreSQL instance it already uses.

After installing pgvector, a table with a vector column looks like this:

```sql
CREATE TABLE products (
    id         SERIAL PRIMARY KEY,
    name       TEXT,
    category   TEXT,
    embedding  VECTOR(1536)  -- 1536-dimensional embedding
);

CREATE INDEX ON products USING hnsw (embedding vector_cosine_ops);
```

A nearest-neighbor query that finds the 5 products most similar to a query embedding — combined with a traditional SQL filter:

```sql
SELECT id, name, 1 - (embedding <=> $1) AS similarity
FROM   products
WHERE  category = 'Electronics'
ORDER BY embedding <=> $1
LIMIT  5;
```

The `<=>` operator computes cosine distance (1 - cosine similarity). pgvector supports three index types: HNSW (recommended for production), IVF (ivfflat), and flat (no index, exact search).

The key architectural implication: by adding pgvector to an existing PostgreSQL database, engineers gain vector search without adding a new system to operate, monitor, back up, and replicate. The tradeoff is that pgvector's HNSW implementation has lower recall at very large scales (100M+ vectors) compared to purpose-built vector databases — a sensitivity point that belongs in any ATAM analysis where vector search is required at massive scale.

## Semantic Search and Hybrid Search

**Semantic search** finds results by meaning rather than by keyword matching. A keyword search for "comfortable shoes" finds only documents containing those exact words. A semantic search using vector embeddings finds documents semantically related to the query — "supportive footwear," "cushioned sneakers," "podiatrist-recommended" — even if they share no keywords with the query.

Semantic search alone has a limitation: it ranks by embedding similarity, which can miss exact matches that are highly relevant but not close in embedding space (proper nouns, product codes, exact phrases).

**Hybrid search** combines vector similarity search with traditional keyword or filter search to get the benefits of both:

1. Run a vector similarity search for the top-k similar documents.
2. Run a keyword/BM25 search for the top-k relevant documents.
3. Merge and re-rank the two result lists using **Reciprocal Rank Fusion (RRF)** or a learned re-ranker.

Hybrid search consistently outperforms either approach alone on benchmark retrieval tasks. Most production retrieval systems (e-commerce search, RAG pipelines, document retrieval) use hybrid search.

!!! mascot-tip "pgvector vs. Standalone Vector DB"
    <img src="../../img/mascot/tip.png" class="mascot-admonition-img" alt="Dex offers a tip">
    For datasets under 5–10 million vectors with modest query rates, pgvector in PostgreSQL is operationally simpler and often sufficient. For datasets over 50 million vectors or query rates requiring sub-millisecond p99 latency, purpose-built vector databases (Pinecone, Weaviate, Qdrant) offer better throughput and more index tuning options. The decision belongs in your ATAM utility tree under a "vector search latency" quality attribute scenario — not in the database vendor's marketing material.

## The ATAM Lens: Integrating Vector Search into Database Selection

Vector search creates a new quality attribute dimension in database selection: **semantic similarity retrieval**. When a system needs to find items similar to a query based on meaning rather than exact match, the ATAM analysis must include a scenario like: "The system must return the 10 most semantically similar products to a user's browsing history within 50ms at p99, across a catalog of 20 million products."

**Sensitivity points:**

- **Index type vs. recall vs. latency:** HNSW provides the best recall-per-millisecond but requires significant memory (approximately 1.5× the raw vector storage). IVF is more memory-efficient but requires careful nprobe tuning. The sensitivity is that recall accuracy directly affects search quality — users notice when the top results are irrelevant.
- **Native vs. standalone:** Adding pgvector to PostgreSQL avoids operational complexity (one fewer system) but limits maximum scale. A standalone vector database adds operational complexity but handles billion-scale with specialized hardware optimizations. The tradeoff point is scale — and the threshold moves as pgvector matures.

**Tradeoff point — hybrid search complexity vs. retrieval quality:** Building and operating a hybrid search system (dual indexes, result fusion, optional re-ranker) is significantly more complex than pure keyword or pure vector search. The quality improvement is measurable but adds a new operational surface area. The architectural decision of whether that complexity is warranted belongs in the utility tree, not in engineering intuition.

## Key Takeaways

Vector search is not a new database paradigm — it is a feature that any database can expose alongside its existing indexes. The decision of where to implement it (natively in your primary database vs. a dedicated vector database) is an architectural tradeoff that belongs in your ATAM utility tree.

- **Vector embedding** — a dense float array representing an item; semantic proximity = geometric proximity
- **Embedding dimensionality** — 256–3072 dimensions; higher = more nuance, more storage, slower index
- **Cosine similarity** — the angle between vectors; magnitude-invariant; preferred for text
- **Dot product** — faster than cosine for unit-normalized vectors
- **Euclidean distance** — straight-line distance; magnitude-sensitive
- **ANN (approximate nearest neighbor)** — trades recall for speed; enables million-scale vector search in milliseconds
- **HNSW** — graph-based ANN; best recall-per-millisecond; high memory usage
- **IVF** — cluster-based ANN; memory-efficient; requires careful nprobe tuning
- **Flat** — exact search; perfect recall; too slow for large datasets
- **pgvector** — adds HNSW/IVF to PostgreSQL; the path of least operational resistance for moderate scale
- **Hybrid search** — combines vector and keyword retrieval; outperforms either alone

!!! mascot-celebration "Chapter 14 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now understand why vector search is reshaping how engineers think about database features. The next chapter goes one level deeper: where do those embedding vectors come from? Chapter 15 opens the transformer architecture and shows you how large language models convert raw text into the dense float arrays that vector search depends on — and what the operational costs look like when you do that at production scale.
