# Embedding-Based Retrieval (ANN, HNSW, FAISS)

> **TL;DR** — **Embedding-based retrieval** turns documents (or images, audio, code) into dense numeric vectors via a learned model, then finds "similar" items by **nearest-neighbor search** in that vector space. Brute-force comparison of a query vector against every document is O(N · d) — fine at thousands, infeasible at billions. **Approximate Nearest Neighbor (ANN)** algorithms — **HNSW**, **IVF**, **PQ**, **ScaNN** — sacrifice a small amount of recall for **logarithmic-or-better** query time. **FAISS** (Facebook AI), **HNSWLib**, **ScaNN** (Google), **DiskANN** (Microsoft) are the dominant libraries. Vector databases (**Pinecone, Weaviate, Milvus, Qdrant, Vespa, pgvector**) package ANN with filtering, hybrid search, and operational concerns. The honest take: **embedding retrieval is now part of every serious search and recommendation stack**, almost always combined with BM25 ("hybrid search"). It enables semantic match, multimodal search, and the retrieval half of RAG. The hard parts aren't the algorithms — they're **picking the right embedding model**, **engineering the index for your workload**, and **dealing with index updates and filtering at scale**.

---

## 1. The big picture

```
Documents          ┌──────────┐
 (text/img/etc)   ─►│  Encoder │── 768-dim vector per doc
                    │  model   │
                    └──────────┘
                          │
                          ▼
                    ┌──────────────────┐
                    │ Vector index      │
                    │  (HNSW/IVF/etc)  │
                    └──────────┬───────┘
                               │
   Query   ────► Encoder ──► query vector ──► top-K nearest ──► ranked results
```

The shift from keyword search:

| Classical (BM25) | Embedding-based |
|---|---|
| Match exact tokens | Match semantic similarity |
| "car" doesn't match "automobile" | "car" matches "automobile" by vector closeness |
| Inverted index | Vector index |
| Lexical | Conceptual / semantic |

Embedding retrieval doesn't replace BM25. It complements it. The standard pattern in 2026 is **hybrid search**: retrieve with BM25 + ANN, fuse the scores, rerank.

---

## 2. Why embeddings work for retrieval

A modern encoder model (BERT-derived, sentence-transformers, OpenAI `text-embedding-3`, Cohere, Google Gecko, BGE, E5) maps text to a vector in some learned space. The model is trained so that **semantically related texts are close in cosine distance**.

```
embed("apache kafka streaming")     ≈ [0.21, -0.35, 0.07, ...]
embed("event streaming with kafka") ≈ [0.18, -0.32, 0.10, ...]   # close
embed("how to bake bread")          ≈ [-0.55, 0.40, -0.21, ...]   # far
```

Cosine similarity (or inner product / Euclidean distance, depending on training) ranks documents by closeness. The amazing fact: this works across paraphrases, multiple languages, and even modalities (text-to-image with CLIP, text-to-code with CodeBERT).

The encoder is the secret sauce. Improvement to retrieval quality usually comes from a **better embedding model**, not a better index. Get this right first.

---

## 3. Brute force — when it's enough

For small N (up to a few million vectors), a naive comparison is fine:

```python
import numpy as np
docs = np.array([...])           # shape (N, d), e.g. (1_000_000, 768)
query = np.array([...])          # shape (d,)
scores = docs @ query            # shape (N,) — inner products
top_k = np.argpartition(-scores, 10)[:10]
```

A million 768-dim vectors against one query takes ~10–50 ms on a modern CPU. For some apps, that's perfectly fine — no fancy index needed.

The cost shape:
- **Compute**: O(N · d). Linear in corpus size.
- **Memory**: O(N · d · 4 bytes) for float32. 1M × 768 = ~3 GB.
- **Latency**: scales linearly with N.

Past 10M vectors, brute force becomes painful. That's where ANN earns its keep.

---

## 4. The ANN families

### HNSW — Hierarchical Navigable Small World

The dominant graph-based ANN algorithm (Malkov & Yashunin, 2018). Builds a multi-layer graph where higher layers contain fewer nodes with long-range connections, and lower layers are denser. Searching is a greedy walk that descends layers, finding closer neighbors at each step.

```
Layer 3:  o ──────────────────────── o
            \                       /
Layer 2:    o ──── o ───── o ───── o
              \   |   \    |     /
Layer 1:     o ─ o ─ o ── o ─ o ─ o
              \  |  \  |  \  | / |
Layer 0:    o-o-o-o-o-o-o-o-o-o-o-o   (all vectors)
```

Properties:
- **Query time**: O(log N) for high recall.
- **Recall**: 0.95+ at well-tuned settings.
- **Build time**: moderate (slower than IVF for huge corpora).
- **RAM footprint**: graph adds ~50–100 bytes/vector overhead.
- **Tunable**: `M` (graph degree), `efConstruction`, `efSearch`.

HNSW is the default in **Lucene 9+, Elasticsearch, OpenSearch, Vespa, Qdrant, Weaviate, pgvector** (since 0.5.0), and dozens more. For text retrieval, it's almost certainly what you want.

### IVF — Inverted File Index

Partition vectors into K clusters (k-means). At query time, find the few clusters closest to the query, search only those.

```
                ┌─────────┐
                │ centroid │
                │  c1      │
                │  ────    │
                │  • • •   │  ← vectors in cluster 1
                └─────────┘
                ┌─────────┐
                │ centroid │
                │  c2      │
                │  ────    │
                │  • • •   │
                └─────────┘
                ...

  Query → find closest K centroids → search within them.
```

Properties:
- **Query time**: roughly O(N · sqrt(K) / K) ≈ O(N / sqrt(K)).
- **Recall**: tunable via `nprobe` (how many clusters to search).
- **Build time**: dominated by k-means; once trained, inserts are cheap.
- **RAM**: less than HNSW for the same N.

IVF dominates **very large** indexes (100M+ vectors), especially when combined with PQ for compression.

### PQ — Product Quantization

A compression scheme. Each d-dimensional vector is split into M subvectors; each subvector is quantized to one of 256 codebook entries (8 bits). A 768-dim float32 vector (3072 bytes) compresses to M=96 bytes — **32× compression** with minimal recall loss.

Used inside IVF (`IVFPQ`) and as a standalone "OPQ" optimizer. Critical for billion-scale indexes that can't fit raw vectors in RAM.

### ScaNN

Google's ANN (Asymmetric Hashing with Quantization). Heavy use of SIMD and learned partitioning. Slightly higher recall/QPS than HNSW in many benchmarks, but harder to operate.

### DiskANN

Microsoft's disk-resident ANN. Sits on SSD. Lower RAM cost; higher latency than RAM HNSW. Used for huge corpora where the full vector set doesn't fit in memory.

### LSH — Locality-Sensitive Hashing

The pre-deep-learning classic. Hash similar vectors to the same buckets with high probability. Mostly historical; HNSW and IVF beat it in practice.

### FLAT — exact search

No index — brute force. The "ground truth" comparison.

---

## 5. The libraries and engines

### FAISS

Facebook's C++ library. The reference implementation. Implements HNSW, IVF, PQ, ScaNN-ish, GPU support, and dozens of variants. Embedded into many vector DBs. If you need raw control, FAISS is the toolbox.

### HNSWLib

A tight, fast standalone HNSW implementation in C++. Used by **Weaviate**, **OpenSearch**, many others.

### ScaNN

Google's ANN library, in TensorFlow. Strong benchmarks; used inside Google products.

### Annoy

Spotify's open-source ANN (forests of random projection trees). Older; less competitive than HNSW for most workloads. Still seen in older recommendation stacks.

### NMSLIB

Reference implementation of HNSW. Largely superseded by HNSWLib and library-integrated versions.

### Vector databases (full-service)

- **Pinecone** — managed, serverless, mature.
- **Weaviate** — open source + managed; built-in hybrid search.
- **Milvus** — open source, large-scale; built on FAISS / Knowhere.
- **Qdrant** — open source + managed; strong filtering.
- **Vespa** — open source, batteries-included search + ranking.
- **Chroma** — lightweight, popular in early prototypes.
- **Vald** — Yahoo Japan's distributed engine.
- **LanceDB** — embedded, Arrow-based.

### General databases with vector support

- **Postgres + pgvector** — HNSW since 0.5. Realistic at <100M vectors.
- **Elasticsearch / OpenSearch** — HNSW with hybrid query syntax.
- **Redis Stack** — vector type with HNSW.
- **MongoDB Atlas Vector Search**.
- **ClickHouse** — vector indexes for analytical workloads.
- **DuckDB** — vector extension.
- **MySQL** — vector type coming (rolling).

For most teams, the right path in 2026: **start with pgvector or your search engine's built-in vector type; graduate to a dedicated vector DB at large scale**.

---

## 6. Hybrid search — combining BM25 and vectors

A typical hybrid search:

```
Query → BM25 retrieval  (top 200 by lexical score)
Query → ANN retrieval   (top 200 by semantic distance)
        │                   │
        └─────── fuse ──────┘
                  │
                  ▼
           Top 100 candidates
                  │
                  ▼
        Cross-encoder reranker (LLM-style)
                  │
                  ▼
              Top 10 results
```

Fusion methods:

- **Reciprocal Rank Fusion (RRF)** — simple, robust. For each ranking, score = `Σ 1/(k + rank)`. Combine. Often works as well as more complex methods.
- **Weighted score combination** — calibrate BM25 and cosine scores onto a common scale (min-max normalize, sigmoid), weight, sum.
- **Learned fusion** — ML model trained on click data.

The empirical takeaway: **hybrid almost always beats either alone**. BM25 catches exact terms (codes, names, brand strings); vectors catch synonyms and paraphrases. The combination's recall is higher than either by itself.

---

## 7. Filtering — the hard problem

Real queries aren't "find similar." They're "find similar to this query **AND** in stock **AND** under $50 **AND** in Europe."

Filtering interacts badly with ANN. Two strategies:

### Pre-filtering

Apply filters first; ANN search only on the surviving set. Works well when filters are selective (small surviving set), poorly when filters are loose (most data passes).

For very selective filters, pre-filter then brute-force is often best.

### Post-filtering

ANN first; drop hits that don't match filters. Works well when filters are loose. Risk: ANN returns 100 candidates, filter drops 80 → you have 20 results when you wanted 100. Solve by over-fetching (`top_k * 5`) and dropping.

### Hybrid / native filtering

Modern engines (Pinecone, Qdrant, Weaviate, Milvus) implement **filter-aware ANN search**: the graph walk skips nodes that don't match filters. Quality depends on the implementation and filter selectivity.

This is one of the differentiators between vector DBs — how well they handle "ANN with predicates."

---

## 8. Embedding model choice

The quality of retrieval is mostly bounded by the encoder. Common choices:

| Model | Use | Notes |
|---|---|---|
| **OpenAI `text-embedding-3-small / -large`** | General text | 1536 / 3072 dim; strong default |
| **Cohere `embed-v3`** | General text | Multilingual; commercial |
| **`sentence-transformers/all-MiniLM-L6-v2`** | Lightweight | 384 dim; fast, open-source default |
| **BGE (BAAI)** | General + Chinese | Strong on MTEB benchmark |
| **E5 (Microsoft)** | Multilingual general | Good for non-English |
| **CLIP, OpenCLIP** | Image-text | Multimodal |
| **CodeBERT, CodeT5, GTE-code** | Code retrieval | Specialized |
| **Voyage / Jina / Mistral embed** | General | Various trade-offs |

Choosing:
- For pure English text, OpenAI embed-3 or BGE-large are top.
- For multilingual, Cohere multilingual or BGE-multilingual.
- For multimodal, CLIP family.
- For domain-specific (medical, legal, code), consider domain-tuned models or fine-tune a base model.

**Always evaluate on your own data.** A benchmark winner may underperform on your corpus. The MTEB benchmark is a good shortlist filter; final choice should be data-driven.

---

## 9. Indexing engineering

Production vector indexes face concerns brute-force CSV demos don't:

### Incremental updates

HNSW supports insertions; deletions are tricky (often soft-delete + periodic rebuild). IVF requires re-clustering periodically as data drifts.

### Sharding

100M+ vectors per shard becomes painful for HNSW. Shard by document ID and fan out queries.

### Recall / latency tuning

Each ANN algorithm has knobs (`efSearch`, `nprobe`) that trade recall for speed. A typical sweep finds the knee — usually 0.95 recall at acceptable latency.

### Memory pressure

A 100M × 768 × float32 index is ~300 GB raw. With graph overhead, ~400 GB. Either compress (PQ, INT8 quantization) or shard.

### Throughput vs latency

A single-query test gives latency; sustained QPS may collapse to a fraction. Saturate the index before promising p99.

### Multi-tenant

Per-tenant filters destroy ANN performance if implemented naively. Some vector DBs support tenant-aware indexes; others require per-tenant indexes.

### Updates with eventual consistency

Indexes update on a refresh cycle (like Elasticsearch). A new document is searchable seconds-to-minutes after insert. Critical for some use cases (chat-history retrieval), tolerable for others (catalog search).

---

## 10. RAG and retrieval — where this gets used

The **retrieval** half of **RAG (Retrieval-Augmented Generation)** is almost entirely embedding-based now:

```
User question
   │
   ▼
Encoder ──► query vector ──► ANN over chunked docs ──► top K passages
                                                         │
                                                         ▼
                                              LLM with passages in context
                                                         │
                                                         ▼
                                                    Generated answer
```

Almost every "chat with your documents" / customer support bot / internal knowledge assistant is built on this loop. The pieces:

- **Chunking strategy** — how you split documents matters more than the model. 200–800 token chunks, with overlap, typically.
- **Hybrid retrieval** — BM25 + vector beats either alone.
- **Reranker** — cross-encoder model on top K. Big quality boost.
- **Caching** — repeated queries → cached candidates.

This stack is now standard. Every cloud has a managed version (Azure AI Search, Vertex AI Search, Bedrock Knowledge Bases). The interesting engineering is in **data preparation**, **evaluation**, and **hallucination control** — not the ANN library.

---

## 11. Common Mistakes / Anti-Patterns

- **Vector-only search.** Misses exact-match cases (product codes, names, IDs). Use hybrid.
- **No reranker.** Top 100 ANN results are often relevant-but-not-best-ranked. Cross-encoder rerank fixes this.
- **Wrong distance metric.** Cosine vs inner product vs Euclidean — must match how the model was trained. Most sentence-transformers use cosine; OpenAI uses cosine; some use raw inner product.
- **Comparing scores across models or runs.** Cosines from one model don't transfer to another.
- **Bad chunking.** Too big → loss of precision. Too small → loss of context. Sweep.
- **Storing 3072-dim float32 vectors uncompressed for 100M docs.** RAM explodes. Use PQ or INT8.
- **No filtering plan.** First filter request reveals that ANN + filter is much slower than the demo.
- **No evaluation harness.** "Search is better" by vibes. Build a gold set: queries with expected relevant docs.
- **Recomputing embeddings on every query for the corpus.** Index once, query embeddings cheap.
- **Updating the corpus without re-embedding.** Old vectors drift from current model.
- **Mixing model versions in one index.** Document A embedded with v1, B with v2 — cosines are meaningless across them.
- **Cold-starting the vector model.** First request slow as model loads. Pre-warm or use a hosted model.
- **Choosing the cheapest embedding API and saving 30% while losing 20% recall.** Compute is rarely the constraint; quality is.
- **No metric reset on dictionary changes.** When you re-embed everything with a new model, archive the old gold set runs.
- **Treating ANN as exact.** It's approximate. Allow over-fetch for filtering and fallback for hard cases.
- **Building one giant index when filters could partition.** Per-tenant or per-language indexes often outperform a unified one with filters.

---

## 12. Cheat Card

```
PURPOSE   Find similar items (text, image, audio, code) via
          nearest-neighbor search in a learned vector space.

PIPELINE
  Encoder (model) → vector
  Index (HNSW / IVF / PQ) → fast nearest-neighbor
  Fuse with BM25 → hybrid
  Cross-encoder reranker → final ranking

ANN ALGORITHMS
  HNSW       graph-based; default for most workloads
  IVF + PQ   cluster + quantize; best at 100M+ scale
  ScaNN      Google's high-perf
  DiskANN    SSD-resident; lower RAM
  FLAT       brute force; ground truth

LIBRARIES
  FAISS / HNSWLib / ScaNN / Annoy
  Vector DBs: Pinecone, Weaviate, Milvus, Qdrant, Vespa, Chroma
  Embedded: pgvector, Elasticsearch, OpenSearch, Redis Stack

DISTANCE METRICS
  Cosine        normalized inner product (most common)
  Inner product when model embeddings aren't normalized
  Euclidean (L2) for some models
  Must match how the encoder was trained

EMBEDDING MODEL CHOICE
  OpenAI text-embed-3, Cohere embed-v3, BGE, E5
  Sentence-transformers (open-source, lightweight)
  CLIP family for multimodal
  Evaluate on your data; MTEB is a shortlist filter

HYBRID SEARCH
  BM25 + ANN, fuse via RRF or weighted score
  Almost always beats either alone
  Add cross-encoder reranker on top-K for big wins

FILTERING
  Pre-filter when selective; post-filter when not
  Filter-aware ANN (Qdrant, Weaviate) for combined cases
  Over-fetch to backfill post-filter drops

INDEX KNOBS
  HNSW: M, efConstruction, efSearch
  IVF:  nlist, nprobe
  PQ:   M, nbits (compression)

WHEN TO USE
  Semantic search, paraphrase tolerance
  Multimodal (image+text, etc.)
  RAG retrieval
  Recommendation, deduplication, clustering

PITFALLS
  Vector-only (no BM25) for exact-match queries
  No reranker on top-K
  Wrong distance metric for the model
  Mixed model versions in one index
  No evaluation set
  Filter on a non-filter-aware index → bad latency
  Uncompressed float32 vectors at huge scale
  Bad chunking for RAG

RULE   Embeddings + ANN are the retrieval layer; reranking is
       the ranking layer. Hybrid with BM25. Evaluate everything
       on your own data.
```

---

## 13. Resources

### Papers
- "Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs" — Malkov & Yashunin, 2018 (HNSW).
- "Billion-scale similarity search with GPUs" — Johnson, Douze, Jégou, 2017 (FAISS).
- "Product Quantization for Nearest Neighbor Search" — Jégou et al., 2011.
- "Accelerating Large-Scale Inference with Anisotropic Vector Quantization" — Guo et al., 2020 (ScaNN).
- "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" — Khattab & Zaharia, 2020.
- "BEIR: A Heterogeneous Benchmark for Zero-Shot Evaluation of IR Models" — Thakur et al., 2021.

### Documentation
- **FAISS** — <https://github.com/facebookresearch/faiss>
- **HNSWLib** — <https://github.com/nmslib/hnswlib>
- **ScaNN** — <https://github.com/google-research/google-research/tree/master/scann>
- **pgvector** — <https://github.com/pgvector/pgvector>
- **Milvus** — <https://milvus.io>
- **Pinecone** — <https://docs.pinecone.io>
- **Weaviate** — <https://weaviate.io/developers/weaviate>
- **Qdrant** — <https://qdrant.tech/documentation/>
- **Vespa** — <https://docs.vespa.ai/en/nearest-neighbor-search.html>

### Articles
- "ANN benchmarks" — <https://ann-benchmarks.com>
- "MTEB leaderboard" — <https://huggingface.co/spaces/mteb/leaderboard>
- "Hybrid search at Vespa / Elastic / Pinecone" — engineering blogs.
- "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node" — Microsoft Research.
- "Best practices for RAG" — Databricks, Pinecone, LangChain blogs.

### Videos
- *FAISS deep dive* — Facebook AI Research talks.
- *Vector Search Explained* — Vespa, Weaviate webinars.
- ByteByteGo — "Vector Database Explained."

### Tools
- **FAISS / HNSWLib / ScaNN / Annoy / NMSLIB** — embeddable libraries.
- **Pinecone / Weaviate / Milvus / Qdrant / Vespa / Chroma / LanceDB** — vector DBs.
- **pgvector, Elasticsearch, OpenSearch, Redis Stack** — vector in general DBs.
- **sentence-transformers, OpenAI / Cohere / Voyage / Jina / Mistral embed APIs** — encoders.
- **LangChain / LlamaIndex / Haystack** — RAG frameworks.

### Adjacent reading
- [Inverted Indexes →](./inverted-index.md)
- [TF-IDF & BM25 →](./tf-idf-bm25.md)
- [Trie Data Structure for Autocomplete →](./trie.md)
- [R-Trees →](./r-trees.md)
- [Vector Databases (Pinecone, Weaviate, pgvector) →](../04-databases/vector-databases.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Design Recommendation System →](../18-case-studies/recommendation-system.md)
- [Design Google Search →](../18-case-studies/search-engine.md)

---

*Previous:* [← TF-IDF & BM25](./tf-idf-bm25.md)  |  *Next:* [Real-Time Analytics →](./real-time-analytics.md)
