# Inverted Indexes

> **TL;DR** — An **inverted index** maps each term to a list of documents that contain it — the inverse of a "forward" index that maps each document to its words. This single inversion is what makes **full-text search** practical: a query like `apache flink` retrieves the **postings lists** for "apache" and "flink", intersects them, and ranks the surviving hits in milliseconds — even over billions of documents. The structure is the heart of **Elasticsearch / OpenSearch / Solr / Lucene** (which builds inverted indexes plus a small zoo of supporting structures), and it's also the substrate for **log search engines** (Loki, ClickHouse skip indexes), **secondary indexes in databases**, **scientific literature search**, and **product catalog search**. The honest take: **you'll almost never write an inverted index by hand**, but understanding **how it's built, ranked, sharded, and updated** lets you make sense of search relevance, why "reindex" is a verb, and the cost shapes of every search system.

---

## 1. The big picture

Forward index — what's in each document:

```
doc1 → [apache, kafka, streaming]
doc2 → [apache, spark, batch]
doc3 → [streaming, latency]
```

Inverted index — which documents contain each term:

```
apache    → [doc1, doc2]
batch     → [doc2]
kafka     → [doc1]
latency   → [doc3]
spark     → [doc2]
streaming → [doc1, doc3]
```

A query for `apache streaming`:

1. Look up `apache` → [doc1, doc2].
2. Look up `streaming` → [doc1, doc3].
3. Intersect for AND, union for OR. AND = [doc1].
4. Rank with BM25 / TF-IDF (see [TF-IDF & BM25 →](./tf-idf-bm25.md)).
5. Return ranked top N.

That's the whole idea. Everything else — tokenization, stemming, positional info, score calibration, sharding — is implementation detail around this core.

---

## 2. Anatomy of an inverted index

A real index has more than just term → docs:

```
TERM      DOC_FREQ    POSTINGS LIST
apache    2           [(doc1, tf=1, positions=[3]),
                       (doc2, tf=2, positions=[1, 7])]
streaming 2           [(doc1, tf=1, positions=[8]),
                       (doc3, tf=1, positions=[2])]
```

Each **posting** typically stores:
- **Document ID** (sorted; small integer or compressed delta).
- **Term frequency (TF)** — how often the term appears in this doc.
- **Positions** — where exactly (for phrase queries: "data engineering" must be adjacent).
- **Field references** — title vs body vs tags.
- **Payloads** — optional per-occurrence data (e.g., is this term bold?).

Postings lists for common terms ("the", "data") can be millions long. They live on disk; query engines stream and merge them.

### Term dictionary

The list of all unique terms in the corpus. Often stored as an **FST** (finite-state transducer) for compact memory use and fast lookup. The dictionary keeps pointers from each term to its postings list on disk.

### Auxiliary structures

- **Term vectors** — per-document inverted lists (used for "more like this" and rerankers).
- **Norms** — per-document length / boost values used in ranking.
- **Skip lists** in postings for fast jumps (see [Skip Lists →](./skip-lists.md)).
- **Doc values / column store** — per-field, per-document values stored in column orientation. Used for sorting and aggregations alongside the inverted index.

Lucene is a great example because it makes all of this visible; Elasticsearch / OpenSearch / Solr are wrappers atop Lucene.

---

## 3. Building the index — the analysis pipeline

Before a term goes into the index, the input text passes through a pipeline:

```
Input:    "Running Apache Flink on Kubernetes — 2026!"
          │
   ┌──────┴────────┐
   ▼               ▼
 char filter   strip HTML, normalize Unicode (NFC), lowercase
   │
   ▼
 tokenizer    split into ["running", "apache", "flink", "on",
                          "kubernetes", "2026"]
   │
   ▼
 token filters
   - lowercase            → ["running", "apache", "flink", ...]
   - stop words           → ["running", "apache", "flink", "kubernetes", "2026"]
   - stemming / lemma     → ["run", "apach", "flink", "kubernet", "2026"]
   - synonym expansion    → adds "k8s" alongside "kubernetes"
   - ASCII folding        → "café" → "cafe"
```

Each step affects what queries match. **The query goes through the same pipeline** so it can find what's indexed. If a query types "running" and the index has "run," the analyzer must stem both.

Common configurations:

- **Standard analyzer** — lowercase + Unicode tokenizer + stop words (English/multilingual).
- **Keyword analyzer** — no tokenization. The whole value is one term (used for IDs, tags, status enums).
- **N-gram / edge n-gram** — for autocomplete and partial-word match.
- **Language-specific** analyzers — stemmer for English (Snowball, Porter), Mandarin segmentation, Arabic root extraction.

For **product search**, you'll often mix several analyzers per field: one stemmed for natural-language matches, one keyword for exact matches, one n-gram for typo tolerance.

---

## 4. Ranking — BM25, TF-IDF, and beyond

A match list alone isn't useful — you need to **rank** results.

The default ranker for most modern search engines is **BM25** (Best Match 25), a refined version of TF-IDF that handles long documents better. See [TF-IDF & BM25 →](./tf-idf-bm25.md).

```
score(doc, query) = sum over terms t in query:
  IDF(t) · (TF(t, doc) · (k1+1)) / (TF(t, doc) + k1 · (1 - b + b · |doc|/avgdl))
```

Where:
- `TF(t, doc)` — frequency of term in doc.
- `IDF(t)` — `log((N - df + 0.5) / (df + 0.5) + 1)`.
- `|doc|` — doc length; `avgdl` — average doc length in the corpus.
- `k1` ≈ 1.2–2.0, `b` ≈ 0.75 — standard tunables.

BM25 strikes a balance between rewarding repeated terms and penalizing length. It's the workhorse of Elasticsearch, OpenSearch, Solr, and almost every classical search engine in 2026.

Beyond BM25:

- **Boosts** — per-field weights ("title matches > body matches").
- **Function scoring** — recency decay, popularity multipliers.
- **Learning to rank (LTR)** — ML reranker on top BM25 results.
- **Vector search** — embedding-based reranking for semantic relevance. See [Embedding-Based Retrieval →](./embedding-retrieval.md).

A modern production search ranker is almost always a **hybrid**: BM25 retrieves candidates fast, ML or vector models rerank the top 100–1000 for relevance.

---

## 5. Updating an index — the segments model

A huge subtle topic.

Inverted indexes are **mostly append-only** for performance. Adding a document is cheap; **modifying** a document is "mark it deleted, write a new one." Bulk-updating millions of docs is "rewrite a segment in the background."

Lucene's design (mirrored by Elasticsearch / OpenSearch / Solr):

- An index is composed of multiple **segments**, each a self-contained mini-index.
- Writes go to an in-memory buffer plus a transaction log (WAL).
- Periodically (every `refresh_interval`, default 1s) the buffer is **flushed** to a new on-disk segment, making writes visible to search.
- A periodic **merge** combines small segments into bigger ones, dropping tombstones for deleted/updated docs.
- A `commit` makes changes durable (fsync) and prunes the WAL.

The implication: **writes are nearly real-time visible (around 1s by default), but full durability with the WAL is what survives crashes.** Tune `refresh_interval` higher (e.g., 30s) to gain write throughput; lower (`-1` for explicit refresh) to control visibility.

### Reindex — the rebuild

Schema changes (new analyzer, new field type, mapping change) often can't be applied retroactively to existing segments. The standard answer: **reindex** — read all docs, write them to a fresh index with the new mapping, alias-swap.

Reindex is normal, planned, and expensive on large indexes. Pipeline tools make it routine in mature search teams.

---

## 6. Sharding and replication

Search engines distribute the index across machines:

- **Shards** — each shard is its own complete inverted index over a subset of docs.
- **Replicas** — copies of shards for HA and read scaling.
- **Routing** — a doc's shard is determined by a hash of its ID (or a custom routing key).
- **Coordinator node** — handles a query by fanning out to all shards, collecting partial results, and merging.

Query latency is dominated by the **slowest shard** (the tail). A single hot shard ruins p99. See [Tail Latency & p99 →](../16-performance/tail-latency.md).

Tuning notes:

- **Too many shards** → fan-out cost; query coordinator becomes the bottleneck.
- **Too few shards** → can't scale horizontally; large shards run out of resources.
- **Common starting point**: shard size ~30–50 GB. Number of shards = `data_size / 30–50 GB`.
- **Replicas**: at least 1 (often 2) for HA.

Multi-tenancy: avoid one shard per tenant if tenants are uneven. Use routing or per-tier strategies. See [Multi-Tenant SaaS Architecture →](./multi-tenant-saas.md).

---

## 7. Query types

Beyond the basic AND/OR:

| Query | What it does |
|---|---|
| `match` | Tokenize query and find matches (BM25 ranked) |
| `multi_match` | Same, across multiple fields with per-field weights |
| `term` | Exact match on a single tokenized term (or keyword) |
| `terms` | OR over multiple exact terms |
| `phrase` | Tokens adjacent in this order (uses positions) |
| `prefix` / `wildcard` | "starts with" / glob — slow on large indexes |
| `fuzzy` | Edit distance match (Damerau-Levenshtein) |
| `range` | For numeric / date fields (uses BKD trees in Lucene) |
| `bool` | Combine with must / should / must_not / filter |
| `function_score` | Custom score functions, decay, weighting |
| `geo_*` | Geo queries — see [Geohashing & Quadtrees →](./geohashing-quadtrees.md) |
| `knn` | Vector / embedding similarity (Lucene 9.x+) |

For each, the engine plans an execution: which postings lists to read, in what order, with what skip optimizations. The query planner is a real piece of engineering.

---

## 8. Inverted indexes beyond search engines

Once you recognize the shape, you see it everywhere:

### Database secondary indexes

A non-primary-key index is effectively an inverted index from "values of this column" to "rows that have them." Postgres GIN indexes are explicitly inverted indexes for `tsvector` (full-text), JSONB containment, and array fields.

### Log search

**Loki** indexes log labels (not log content) and uses content scanning over those label-filtered ranges. **ClickHouse** has **skip indexes** (bloom filter + min/max + token bloom) that act like coarse inverted indexes on text columns. **Splunk** uses traditional inverted indexes.

### Tag / facet search

E-commerce filter sidebars ("show all blue dresses size M $50–$100") run dozens of small inverted-index queries combined with bitset operations.

### Code search

**GitHub code search** (Blackbird, 2023) is essentially a massive inverted index keyed on n-grams over source code, with relevance based on usage patterns. **Sourcegraph** uses indexed code search with similar primitives.

### Reverse map for analytics

A "find all events with this customer_id" query is an inverted-index lookup. Druid, Pinot, and ClickHouse all use inverted-index-like structures for dimension filtering.

The pattern: **whenever you query by "things containing this attribute" rather than "row with this ID," some form of inverted index is involved.**

---

## 9. Worked example — small Lucene-style index

```python
from collections import defaultdict

class TinyIndex:
    def __init__(self, analyzer):
        self.analyzer = analyzer
        self.docs = {}                # doc_id → text
        self.postings = defaultdict(list)  # term → list of (doc_id, tf, positions)
        self.doc_lengths = {}

    def add(self, doc_id, text):
        tokens = self.analyzer(text)
        self.docs[doc_id] = text
        self.doc_lengths[doc_id] = len(tokens)
        tf = defaultdict(list)
        for pos, tok in enumerate(tokens):
            tf[tok].append(pos)
        for tok, positions in tf.items():
            self.postings[tok].append((doc_id, len(positions), positions))

    def search(self, query):
        terms = self.analyzer(query)
        if not terms:
            return []
        # AND: intersect postings
        candidate_sets = [set(p[0] for p in self.postings.get(t, [])) for t in terms]
        candidates = set.intersection(*candidate_sets) if candidate_sets else set()
        return list(candidates)
```

Add BM25 scoring and you have a tiny but real search engine. Add segments, refresh, merge, sharding, replication, an analysis pipeline, and a query DSL, and you have Elasticsearch.

---

## 10. Common Mistakes / Anti-Patterns

- **Using a relational DB's `LIKE '%foo%'` for "search."** Full table scan, no relevance, no scaling. Use a real search index.
- **Indexing huge fields (whole books, log payloads) without thinking about norms / size.** Slow scoring, big segments.
- **Mismatched analyzer for index vs query.** Query "running" doesn't match indexed "run."
- **Treating Elasticsearch as a primary database.** It's an index, not a system of record. Reindex from a durable source.
- **Not setting `refresh_interval` for bulk loads.** Default 1s refresh thrashes; bulk loads should use `-1` and explicit refresh at the end.
- **One massive shard or thousands of tiny ones.** Both lose. Aim for 30–50 GB shards.
- **Mixing tenants on one shard with very different cardinalities.** Big tenants dominate fan-out; small tenants pay for big tenants' problems.
- **No reindex plan.** Mapping changes accumulate; eventually you need a hot reindex with alias swap.
- **No relevance evaluation.** "Search is good" by gut feel. Build a gold set of queries with expected results; track NDCG over time.
- **Wildcard / regex queries on huge indexes.** Linear in the term dictionary — very slow.
- **Storing PII unencrypted in the index when the source DB encrypts it.** Compliance violation.
- **`_source`-disabled with no way to rebuild snippets.** Snippets, highlight, "more like this" all need the source.
- **Trusting vector search alone for relevance.** Hybrid with BM25 keyword retrieval is almost always better.
- **No tail-latency budget.** Hot shard → p99 disaster. See [Tail Latency →](../16-performance/tail-latency.md).
- **Reindex while a hot deploy is happening.** Combine with normal change control.

---

## 11. Cheat Card

```
PURPOSE   Map terms to documents so full-text search is O(query),
          not O(corpus).

CORE STRUCTURE
  Term dictionary       all unique terms (often an FST)
  Postings list/term    [(doc, tf, positions), ...]
  Doc store / source    original document text
  Norms / doc lengths   for BM25 scoring
  Auxiliary             skip lists, doc values, term vectors

PIPELINE
  Text → char filter → tokenizer → token filters → index
  QUERY goes through the same pipeline (must match!)

BUILD MODEL (LUCENE)
  Segments (immutable mini-indexes) merged by background compaction
  Refresh interval = visibility latency (default 1s)
  Commit = durability boundary (WAL applied)
  Reindex = full rebuild for mapping changes

RANKING
  BM25 default (k1≈1.2–2, b≈0.75)
  Multi-field boosts
  Function scoring (recency, popularity)
  Learning to rank (LTR) reranker on top
  Hybrid with vector search for semantic relevance

SHARDING
  Shard ~30–50 GB
  Replica ≥1 for HA
  Coordinator fan-out → tail-latency sensitive

QUERY TYPES
  match / multi_match / phrase / prefix / fuzzy
  term / terms (exact)
  bool / function_score
  range (BKD)
  knn (vector)

USES BEYOND SEARCH
  DB secondary indexes (Postgres GIN, etc.)
  Log search (Loki, ClickHouse skip indexes, Splunk)
  Faceted product search
  Code search (GitHub Blackbird, Sourcegraph)
  Analytics dimension filtering (Druid, Pinot)

PITFALLS
  LIKE '%foo%' as "search"
  Mismatched analyzer index vs query
  Using ES as a system of record
  refresh_interval=1s during bulk loads
  Wrong shard size (too big or too small)
  Wildcard / regex on huge indexes
  No relevance metric tracked
  No reindex plan for schema changes

RULE   Real search needs a real search engine. Mind the analyzer.
       Plan for reindex. BM25 + hybrid vectors beat either alone.
```

---

## 12. Resources

### Books
- *Introduction to Information Retrieval* — Manning, Raghavan, Schütze (free online). The standard textbook.
- *Search Engines: Information Retrieval in Practice* — Croft, Metzler, Strohman.
- *Lucene in Action* — Hatcher & Gospodnetic (older but foundational).
- *Elasticsearch: The Definitive Guide* — Gormley & Tong.

### Documentation
- **Apache Lucene** — <https://lucene.apache.org>
- **Elasticsearch** — <https://www.elastic.co/guide/index.html>
- **OpenSearch** — <https://opensearch.org/docs/>
- **Apache Solr** — <https://solr.apache.org>
- **Tantivy** (Rust Lucene-like) — <https://github.com/quickwit-oss/tantivy>
- **Postgres GIN indexes** — <https://www.postgresql.org/docs/current/gin.html>
- **PGroonga** — <https://pgroonga.github.io/>

### Articles
- "BM25: The Next Generation of Lucene Relevance" — Doug Turnbull et al.
- "Building GitHub code search" — GitHub engineering blog (Blackbird).
- "Hybrid search: BM25 + vectors" — Vespa / Elastic / Pinecone blogs.
- "Inverted indexes inside ClickHouse" — Altinity / ClickHouse posts.

### Videos
- *Inside Lucene* — Mike McCandless talks.
- *Building search at scale* — Algolia, Elastic, Vespa conference talks.
- ByteByteGo — "Inverted Index Explained."

### Tools
- **Elasticsearch / OpenSearch / Solr** — Lucene-based.
- **Tantivy / Quickwit** — modern Rust Lucene-like.
- **Vespa** — search + ranking + vector.
- **Typesense / Meilisearch** — opinionated, lightweight.
- **Postgres GIN / PGroonga** — when search is a small part of a Postgres app.
- **ClickHouse / Druid / Pinot** — log + analytics with inverted-index-ish structures.

### Adjacent reading
- [Trie Data Structure for Autocomplete →](./trie.md)
- [Skip Lists →](./skip-lists.md)
- [TF-IDF & BM25 →](./tf-idf-bm25.md)
- [PageRank Algorithm →](./pagerank.md)
- [Embedding-Based Retrieval (ANN, HNSW, FAISS) →](./embedding-retrieval.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Database Indexing →](../04-databases/indexing.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)

---

*Previous:* [← Skip Lists](./skip-lists.md)  |  *Next:* [PageRank Algorithm →](./pagerank.md)
