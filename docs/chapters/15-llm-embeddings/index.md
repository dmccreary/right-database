---
title: Chapter 15 - LLM-Generated Embeddings
description: How transformer-based LLMs generate embeddings — tokenization, attention, pooling strategies, embedding model selection, production pipeline design, cost at scale, and the re-embedding migration problem.
generated_by: claude skill chapter-content-generator
date: 2026-05-20 10:35:11
version: 0.08
---

# Chapter 15: LLM-Generated Embeddings

## Summary

This chapter covers how large language models generate the vector embeddings that power semantic search, recommendation, and retrieval-augmented generation systems. Students learn the transformer architecture and pooling strategies that produce embeddings, how to select and compare embedding models (OpenAI, Sentence Transformers, self-hosted open-source), how to design production embedding pipelines with batching and caching, and the hidden operational costs of embedding at scale — including the re-embedding migration problem that arises when models are upgraded or replaced.

## Concepts Covered

This chapter covers the following 15 concepts from the learning graph:

1. Large Language Model
2. Transformer Architecture
3. Tokenization
4. Attention Mechanism
5. CLS Token Pooling
6. Mean Pooling
7. Embedding Model Selection
8. OpenAI Embeddings API
9. Sentence Transformers
10. Self-Hosted Embedding Model
11. Embedding Cost at Scale
12. Re-Embedding Migration
13. Multimodal Embedding
14. Embedding Pipeline Architecture
15. Embedding Model Versioning

## Prerequisites

This chapter builds on concepts from:

- [Chapter 2: Database Foundations](../02-database-foundations/index.md)
- [Chapter 14: Vector Search as a Database Feature](../14-vector-search/index.md)

---

!!! mascot-welcome "Welcome to Chapter 15!"
    <img src="../../img/mascot/welcome.png" class="mascot-admonition-img" alt="Dex waves hello">
    Chapter 14 showed you what the database does with embeddings. This chapter shows you where embeddings come from. Understanding the transformer architecture and its pooling strategies gives you the foundation to select the right embedding model for your workload — and to understand why switching models later is not a simple configuration change but a full database migration. The re-embedding problem is one of the most underestimated operational risks in AI-powered systems, and this chapter makes it concrete.

## Large Language Models and the Embedding Intuition

A **large language model (LLM)** is a neural network trained on vast quantities of text to predict the next token in a sequence. The training process forces the model to develop internal representations that capture the statistical regularities of language — and those internal representations turn out to encode semantic meaning in a geometrically useful way.

When an LLM processes a piece of text, intermediate layers of the network produce a vector for each token in the input. These intermediate vectors — not the final token predictions — are what we extract as embeddings. The model has learned, through billions of training examples, to place tokens with similar meanings in similar regions of its internal vector space. By pooling the token-level vectors into a single sentence-level vector (discussed shortly), we get an embedding that captures the meaning of the entire input.

The key insight is that LLMs were not explicitly trained to produce useful embeddings — they were trained to predict text. The embeddings are a byproduct of that training that turns out to be extraordinarily useful for similarity retrieval. Models fine-tuned specifically for embedding tasks (Sentence Transformers, OpenAI text-embedding-3) optimize this byproduct explicitly, producing better embeddings than raw language models.

## Transformer Architecture

The **transformer architecture** (Vaswani et al., 2017, "Attention Is All You Need") is the neural network design underlying virtually every modern LLM. Understanding its key components is necessary for reasoning about embedding quality, input limits, and computational cost.

Three components matter most for embedding use cases:

**Tokenization** is the preprocessing step that converts raw text into discrete tokens before the model sees it. A **tokenizer** splits text into subword units — fragments like "un", "believ", "able" — using an algorithm such as Byte-Pair Encoding (BPE). Each token maps to an integer ID that the model looks up in an embedding table. The model's **context window** (maximum input length) is measured in tokens, not characters or words. OpenAI's text-embedding-3-small accepts up to 8,192 tokens; most Sentence Transformer models accept 128–512 tokens.

Tokenization has a practical consequence: text that exceeds the context window is truncated. A document of 10,000 tokens fed to a model with an 8,192-token limit loses its last ~1,800 tokens. For document embedding, this means long documents must be chunked — split into overlapping segments — before embedding.

**The attention mechanism** allows the transformer to relate every token in the input to every other token when computing each token's representation. For each token, the model computes a weighted sum of all other tokens' representations, where the weights (attention scores) measure how relevant each other token is. This allows the model to capture long-range dependencies — the meaning of "bank" in "I went to the bank to deposit money" is shaped by "deposit" and "money" even if they are many tokens away.

The computational cost of attention is O(n²) in the sequence length n, which is why context windows are bounded. Doubling the context length quadruples the computation.

**The encoder pass** produces a contextualized vector for every token in the input. In a 512-token input processed by a 768-dimensional model, the encoder produces a 512 × 768 matrix of intermediate vectors. The embedding is derived from this matrix through a pooling step.

#### Diagram: Tokenization Visualizer

<details markdown="1">
<summary>Interactive Tokenization Visualizer</summary>
Type: MicroSim
**sim-id:** tokenization-visualizer<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Identify how a tokenizer splits text into subword tokens and explain why token count (not character count) determines what fits in a model's context window. (Bloom L2: Understand)

**Canvas:** 760px wide × 400px tall. CANVAS_HEIGHT: 400.

**Description:**
A large text input area (default text: "The quick brown fox jumped over the lazy database administrator"). A "Tokenize" button triggers the visualization.

**Output:** Each token appears as a colored chip below the input, with the token text inside and its integer ID below. Tokens from the same word are adjacent but may have different background colors indicating subword splits (e.g., "admin" and "istrator" as two chips).

**Token count display:** "17 tokens | Context window: 8,192 tokens | 0.2% used"

**Context window fill bar:** A horizontal progress bar showing what fraction of the 8,192-token window is consumed.

**Long document demo:** A "Load Long Document" button fills the input with a 600-word article. The tokenizer shows ~750 tokens and a warning: "This document would need to be chunked for a 512-token model."

**Chunking visualization:** A "Show Chunks" toggle splits the token chips into groups of 128 with 20-token overlaps, each chunk shown in a different background shade.

**Responsive:** Redraws on window resize.
</details>

## Pooling Strategies: From Token Vectors to Sentence Vectors

The encoder produces one vector per token. To get a single embedding for the entire input, the token vectors must be reduced to one. This reduction is called **pooling**.

Two pooling strategies dominate the embedding model landscape.

### CLS Token Pooling

BERT-style models add a special `[CLS]` (classification) token at the beginning of every input. The model is trained so that the `[CLS]` token's final vector captures a summary of the entire input's meaning. **CLS token pooling** simply uses the `[CLS]` token's vector as the sentence embedding and discards all other token vectors.

CLS pooling is computationally efficient — no averaging step required. Its quality depends on how well the model was trained to concentrate meaning into the `[CLS]` token. For models specifically fine-tuned for sentence similarity (like BERT fine-tuned on natural language inference), CLS pooling works well. For raw language models not fine-tuned for this purpose, CLS pooling produces mediocre embeddings.

### Mean Pooling

**Mean pooling** averages all token vectors (excluding padding tokens) to produce the sentence embedding. If the encoder produces token vectors \( \mathbf{v}_1, \mathbf{v}_2, \ldots, \mathbf{v}_n \), the embedding is:

\[ \mathbf{e} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{v}_i \]

Mean pooling generally produces higher-quality sentence embeddings than CLS pooling for models not specifically trained for CLS-based representation. The Sentence Transformers library defaults to mean pooling. OpenAI's text-embedding models use a proprietary pooling strategy that behaves similarly to mean pooling.

| Strategy | How it works | Best for |
|----------|-------------|----------|
| CLS Pooling | Use [CLS] token vector | BERT-family models fine-tuned for sentence similarity |
| Mean Pooling | Average all token vectors | General-purpose; Sentence Transformers default |
| Max Pooling | Take element-wise maximum | Occasionally used; rarely optimal |
| Weighted Mean | Weight tokens by attention scores | Some specialized models |

## Embedding Model Selection

With dozens of embedding models available from multiple providers, model selection is a non-trivial architectural decision. The right model depends on the workload. Before comparing models, define the key selection criteria:

- **Dimensionality:** Higher dimensions = more expressiveness, more storage, slower ANN index operations. OpenAI text-embedding-3-large: 3072 dimensions. Sentence-BERT (all-MiniLM-L6-v2): 384 dimensions.
- **Max token input length:** Longer context windows handle longer documents without chunking. Most models: 128–512 tokens. Long-context models (jina-embeddings-v2, OpenAI text-embedding-3): 8,192 tokens.
- **Multilingual support:** Models trained on multilingual corpora can embed text in multiple languages into a shared space. SBERT's multilingual models and Cohere's multilingual embeddings are common choices.
- **Cost per token:** API-based models charge per token. At billions of documents, cost differences of $0.001 per 1K tokens compound dramatically.
- **Latency:** API calls introduce network latency (20–100ms). Self-hosted models running on GPU can process batches of 64 documents in 10–50ms total.

### OpenAI Embeddings API

The **OpenAI Embeddings API** provides access to OpenAI's text-embedding models via a REST API. The current generation models are:

- **text-embedding-3-small:** 1536 dimensions (reducible to as few as 512 via Matryoshka training). $0.02 per 1M tokens. Best cost-performance tradeoff for most applications.
- **text-embedding-3-large:** 3072 dimensions (reducible to 256). $0.13 per 1M tokens. Higher accuracy on MTEB benchmarks, at 6.5× the cost.

OpenAI's models return unit-normalized vectors, making dot product the preferred similarity metric.

### Sentence Transformers

**Sentence Transformers** is an open-source Python library (built on Hugging Face Transformers) that provides pre-trained and fine-tunable models specifically optimized for sentence-level embedding. Models range from tiny (all-MiniLM-L6-v2: 384 dimensions, 22M parameters, runs on CPU) to large (all-mpnet-base-v2: 768 dimensions, 110M parameters). Sentence Transformers models are free to use, can be run on your own hardware, and can be fine-tuned on domain-specific data.

### Self-Hosted Embedding Models

A **self-hosted embedding model** runs on infrastructure you control — a GPU server, a Kubernetes cluster, or a cloud VM. Self-hosting eliminates per-token API costs, eliminates network latency (embedding happens in-process or in a local service), and keeps data private (no text leaves your infrastructure).

The operational cost is managing the infrastructure: GPU provisioning, model serving framework (Triton Inference Server, vLLM, FastAPI), batching logic, and availability guarantees. At high volume, self-hosting typically becomes cheaper than API pricing above roughly 1 billion tokens per month.

#### Diagram: Embedding Model Comparison Matrix

<details markdown="1">
<summary>Interactive Embedding Model Selection Matrix</summary>
Type: MicroSim
**sim-id:** embedding-model-comparison<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Select an appropriate embedding model for a given use case by comparing dimensionality, context length, cost, latency, and multilingual support. (Bloom L5: Evaluate)

**Canvas:** 780px wide × 500px tall. CANVAS_HEIGHT: 500.

**Description:**
A sortable, interactive comparison table with 6 rows (embedding models) and 6 columns:
- Model: text-embedding-3-small, text-embedding-3-large, all-MiniLM-L6, all-mpnet-base-v2, cohere-embed-v3, jina-embeddings-v2
- Dimensions: 1536, 3072, 384, 768, 1024, 768
- Max Tokens: 8192, 8192, 256, 514, 512, 8192
- Cost ($/1M tokens): $0.02, $0.13, Free, Free, $0.10, Free
- Hosting: API, API, Self/API, Self/API, API, Self/API
- Multilingual: No, No, No, No, Yes, Yes

**Interactions:**
- Clicking a column header sorts the table by that column.
- Hovering a row highlights it and shows a sidebar: "Best for: English-only text at moderate scale, cost-sensitive, < 8K tokens per document."
- A "My Requirements" panel: sliders for "Max cost per 1M tokens", "Min dimensions", "Need multilingual". Models that don't meet requirements fade out.
- A "Compare Two Models" mode lets user select any two rows to see a side-by-side quality tradeoff chart (MTEB score vs cost).

**Responsive:** Redraws on window resize.
</details>

## Embedding Pipeline Architecture

An **embedding pipeline** is the end-to-end system that converts raw source data into vectors stored in the database. Before designing the pipeline, understand the two temporal modes:

**Batch embedding** processes all existing documents upfront, typically before the system is live. A batch job reads documents from a source (database, object storage, data lake), calls the embedding model (API or self-hosted), and writes (document_id, vector) pairs to the database. Batch pipelines optimize for throughput — sending large batches to the embedding model to maximize GPU utilization or API rate limits.

**Real-time embedding** processes new documents as they arrive. When a user creates a new product listing, the system immediately embeds it and writes the vector to the database so it is searchable within seconds. Real-time pipelines optimize for latency — the embedding must complete before the user-facing write is acknowledged, or be processed asynchronously with a brief window where new items are not yet searchable.

The canonical embedding pipeline stages are:

1. **Source:** Raw documents arrive from a database, event stream, or API.
2. **Preprocessing:** Clean and normalize text (strip HTML, normalize whitespace, truncate or chunk to model's token limit).
3. **Embedding:** Send preprocessed text to the embedding model; receive vectors. Batch size: 16–512 texts per API call for throughput.
4. **Storage:** Write vectors alongside source record IDs to the database (e.g., UPDATE products SET embedding = $vector WHERE id = $id in PostgreSQL with pgvector).
5. **Index update:** The vector index (HNSW, IVF) is updated automatically on write (for HNSW) or rebuilt periodically (for IVF).

#### Diagram: Embedding Pipeline Architecture

<details markdown="1">
<summary>Interactive Embedding Pipeline — Batch and Real-Time Modes</summary>
Type: MicroSim
**sim-id:** embedding-pipeline-architecture<br/>
**Library:** p5.js<br/>
**Status:** Specified

**Learning Objective:** Design an embedding pipeline architecture for a given source system, selecting appropriate batch vs. real-time processing, chunking strategy, and storage approach. (Bloom L6: Create)

**Canvas:** 780px wide × 460px tall. CANVAS_HEIGHT: 460.

**Description:**
A horizontal pipeline flow with 5 stages (rounded rectangles): Source → Preprocess → Embed → Store → Index. Two toggle modes at top: "Batch Mode" and "Real-Time Mode."

**Batch Mode:** Animated arrows show bulk documents flowing from Source (a database cylinder). Preprocess stage shows a counter "Chunking: 12 docs → 47 chunks (512 tokens each)." Embed stage shows batches of 32 going to an API icon; a throughput meter shows "~200 docs/sec." Store shows bulk INSERT to database. Index shows "HNSW index auto-updated on write."

**Real-Time Mode:** A single document flows in from a Kafka icon. Preprocess: "1 doc → 3 chunks." Embed: single API call, ~80ms latency. Store: single INSERT. A "Searchable after: ~100ms" badge.

**Hovering any stage** opens a configuration panel: "Embed stage: Batch size = 32. API: text-embedding-3-small. Rate limit: 3000 req/min. Retry on 429: exponential backoff."

**Cost calculator panel:** Enter document count and avg tokens per doc → shows estimated API cost at text-embedding-3-small pricing.

**Responsive:** Redraws on window resize.
</details>

## Embedding Cost at Scale and the Re-Embedding Migration Problem

**Embedding cost at scale** is a budget line that engineering teams routinely underestimate. At 100 million documents averaging 500 tokens each, the initial embedding cost using text-embedding-3-small is:

\[ 100\text{M docs} \times 500 \text{ tokens} \div 1{,}000 \times \$0.02 = \$1{,}000{,}000 \]

That is $1 million for the initial embed — before ongoing re-embedding of updated documents. The cost calculation must include:

- Initial batch embedding of the full corpus
- Real-time embedding of new and updated documents
- Re-embedding of the full corpus when the model changes (see below)

**Embedding model versioning** is the practice of tagging every stored vector with the model and version that produced it. Without version tags, it is impossible to identify which stored vectors are stale after a model upgrade.

The **re-embedding migration problem** is what happens when you change your embedding model. Because embeddings from different models live in different geometric spaces, a vector from model A is not comparable to a vector from model B. A query embedded with model B will not return correct nearest neighbors when compared against vectors stored from model A. The only solution is to re-embed the entire corpus with model B.

At 100 million documents, re-embedding is a major operation: it takes hours to days, costs significant API fees, requires running both old and new indexes in parallel during the cutover, and must be coordinated with the vector index rebuild. OpenAI's deprecation of text-embedding-ada-002 in favor of text-embedding-3 required teams to execute exactly this migration.

!!! mascot-warning "Model Versioning Is Not Optional"
    <img src="../../img/mascot/warning.png" class="mascot-admonition-img" alt="Dex warns">
    Every stored embedding vector must be tagged with the model ID and version that produced it. This is not optional. When your embedding provider releases a new model version (or deprecates an old one), you need to know exactly which documents were embedded with which model, so you can plan and execute the re-embedding migration systematically. Teams that skip this tagging discover their oversight only when results degrade mysteriously after a model upgrade — at which point identifying the affected vectors requires a full corpus scan.

## Multimodal Embeddings

**Multimodal embeddings** place multiple modalities — text, images, audio, video — into the same high-dimensional space. A multimodal embedding model (like OpenAI's CLIP or Meta's ImageBind) can embed a text query "a dog running in a park" and find images containing that scene, without any explicit text labels on the images.

For database architects, multimodal embeddings expand what "similar items" can mean: a product search that finds visually similar items from an image upload, a music recommendation system that finds songs similar to a hummed melody, or a video retrieval system that finds clips matching a natural language description. The database implications are the same as for text embeddings — store the vector alongside the source record, build an HNSW or IVF index — but the embedding model and pipeline are multimodal.

## The ATAM Lens: Embedding Architecture Decisions

!!! mascot-thinking "The Build vs. Buy Question"
    <img src="../../img/mascot/thinking.png" class="mascot-admonition-img" alt="Dex thinks">
    OpenAI API vs. self-hosted Sentence Transformers is a classic ATAM tradeoff point: the API is faster to ship, eliminates GPU management, and benefits from OpenAI's model improvements — but it introduces vendor dependency, per-token cost, network latency, and data privacy exposure. Self-hosting eliminates those costs and risks but adds GPU operations burden. Which side wins depends entirely on your system's specific quality attribute scenarios for cost, latency, privacy, and operational complexity. Build that utility tree.

In an ATAM analysis, embedding architecture produces these sensitivity points:

- **Dimensionality vs. storage cost:** 3072-dimensional vectors require 8× more storage than 384-dimensional vectors. At tens of millions of records, this is the difference between a manageable vector column and a database storage tier upgrade.
- **Model quality vs. re-embedding frequency:** Better models improve retrieval quality but are released more frequently. Organizations with very large corpora and tight budgets may rationally choose a slightly inferior but stable model to avoid the re-embedding migration cost of frequent upgrades.

## Key Takeaways

LLM-generated embeddings are the fuel for vector search — and the choice of embedding model, pipeline architecture, and versioning strategy are as consequential as the vector index choice.

- **Transformer architecture** — attention-based neural network; processes all tokens in parallel; produces contextualized token vectors
- **Tokenization** — text → subword tokens; context window limits measured in tokens; long docs need chunking
- **CLS pooling** — use [CLS] token as sentence vector; good for BERT fine-tuned on similarity
- **Mean pooling** — average all token vectors; generally better for general-purpose embeddings
- **Model selection criteria** — dimensionality, context length, cost/token, latency, multilingual support
- **Batch vs. real-time pipeline** — batch for throughput; real-time for freshness
- **Embedding cost** — tokens × price × volume; re-embedding adds a multiplication factor at model upgrade time
- **Re-embedding migration** — changing models requires re-embedding the entire corpus; plan and budget for it
- **Model versioning** — tag every stored vector with its model ID; non-negotiable for operational sanity

!!! mascot-celebration "Chapter 15 Complete!"
    <img src="../../img/mascot/celebration.png" class="mascot-admonition-img" alt="Dex celebrates">
    You now understand the full stack: transformer → tokenizer → encoder → pooling → vector → ANN index → similarity search. The final chapter brings all of this together. Chapter 16 is the capstone: you will apply ATAM to a realistic database selection scenario, build a polyglot persistence architecture, and produce the kind of documented decision artifact that distinguishes a senior engineer from a junior one.
