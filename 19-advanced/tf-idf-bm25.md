# TF-IDF & BM25

> **TL;DR** — **TF-IDF** scores how relevant a term is to a document by multiplying **term frequency** (how often the term appears in this doc) by **inverse document frequency** (how rare the term is across the corpus). Common words like "the" get IDF near zero; rare words get high IDF. The idea is 50+ years old (Sparck Jones, 1972) and still works as a baseline ranker. **BM25** (Robertson & Walker, 1994) is the modern refinement: it saturates term frequency (so a 100-mentions of "kafka" isn't 100× more relevant than 1), and normalizes by document length (so long documents don't get a free boost). **BM25 is the default ranker in Elasticsearch / OpenSearch / Lucene / Solr / Vespa**, and the strong baseline that every modern reranker is compared against. The honest take: **for keyword search, BM25 is shockingly hard to beat with simpler ML**. Modern systems usually **combine BM25 retrieval with dense-vector reranking** ("hybrid search") rather than replace it.

---

## 1. The big picture

A search engine asks: *given a query, how relevant is each document?*

The TF-IDF and BM25 family answers this with a small handful of statistical signals:

```
Query: "apache kafka"

Doc A:  "Apache Kafka is a distributed event streaming platform..."
        TF(kafka)=4, length=300 words
Doc B:  "Kafka studied apache (helicopter) maintenance..."
        TF(kafka)=2, length=10000 words
Doc C:  "Modern data infrastructure is everywhere..."
        TF(kafka)=0
```

Even before opening these, the ranker can answer:

- Doc A: high TF for both terms; short doc; rare terms → high score.
- Doc B: term present but diluted; misleading context → lower BM25 score.
- Doc C: doesn't even contain "kafka" → 0.

The ranker isn't "smart" in the LLM sense. It's a statistical bet that **rare words in a query that show up densely in a short document are good signals of relevance**. That bet has held up for 30 years.

---

## 2. TF-IDF, broken down

### Term frequency (TF)

The raw count of how many times term `t` appears in document `d`. Often normalized:

```
tf(t, d) = count(t, d)               # raw
tf(t, d) = log(1 + count(t, d))      # log-scaled (common)
tf(t, d) = count(t, d) / |d|         # length-normalized
```

The intuition: a document mentioning "kafka" five times is more about kafka than one mentioning it once. But not 5× more — diminishing returns. Log-scaling captures that.

### Inverse document frequency (IDF)

How rare is term `t` across all documents?

```
idf(t) = log(N / df(t))              # classic
idf(t) = log((N - df(t) + 0.5) / (df(t) + 0.5) + 1)   # BM25 / Robertson-Sparck Jones
```

Where:
- `N` is total documents in the corpus.
- `df(t)` is the number of documents containing `t`.

A word appearing in every document (`df = N`) gets IDF ≈ 0 — useless for discrimination. A word in 1 out of 1M documents gets high IDF — strong signal.

### TF-IDF score

For a query with terms `t1, t2, ..., tk`:

```
score(d, q) = sum over t in q of  tf(t, d) · idf(t)
```

That's it. Compute per-term contributions, sum.

### Why it works

- Long, common terms ("the", "is", "data") get tiny IDF; they don't dominate scores.
- Distinctive, rare query terms drive the ranking.
- Documents that contain rare query terms rank well **even if other matches are weak**.

TF-IDF is the canonical example of an **information-theoretic** ranker: signal strength is high when terms are surprising.

---

## 3. TF-IDF's weaknesses (and why BM25 exists)

TF-IDF has two well-known problems.

### Linear TF is too aggressive

A document with TF=100 for "kafka" gets 100× the contribution of one with TF=1. But intuitively, after maybe 10 mentions, additional ones don't make the document *more* about kafka — they're filler, repetition, or spam.

### No length normalization

A 50-word doc with TF=5 vs a 5000-word doc with TF=5: TF-IDF treats them equally. But the short one is much more focused on the term.

BM25 fixes both.

---

## 4. BM25 — the workhorse

The **BM25** scoring function (Stephen Robertson and Karen Spärck Jones, 1994):

```
score(d, q) = Σ over t in q:
    IDF(t) ·  (tf(t,d) · (k1 + 1))
              ─────────────────────────────────────────
              tf(t,d) + k1 · (1 − b + b · |d| / avgdl)
```

Three tunables:
- `k1` ∈ [1.2, 2.0] — how quickly TF saturates. Lower = saturates earlier.
- `b` ∈ [0, 1] — how strongly to penalize long documents. 0 = no penalty, 1 = strong penalty.
- `avgdl` — average document length in the corpus.

The shape (for a fixed term, varying TF):

```
score
  │      ┌───── BM25 (saturating)
  │     /
  │    /
  │   /
  │  /
  │ /
  │/───────── TF-IDF (linear)
  └──────────────────────► term frequency
```

BM25 hands out big gains for the first few occurrences of a term, then plateaus. Spamming "kafka kafka kafka..." 100 times doesn't help. Length normalization (`b ≈ 0.75`) dampens scores for long documents that mention everything once.

### Why the IDF form is different

BM25 uses a smoothed IDF based on the Robertson-Sparck Jones model:

```
IDF(t) = log( (N − df(t) + 0.5) / (df(t) + 0.5) + 1 )
```

The `+0.5` smoothing avoids log of zero and stabilizes scores for very rare or very common terms. Some BM25 variants also clip negative IDF values (which can occur for terms in more than half of all docs).

### Tuning k1 and b

The defaults `k1 = 1.2`, `b = 0.75` work well for general text. Empirical sweet spots:

- **Web search**: k1 ≈ 1.5, b ≈ 0.75.
- **Scientific papers**: k1 ≈ 0.5, b ≈ 0.5 (less length penalty; more vocabulary diversity).
- **Product / e-commerce**: k1 ≈ 1.2, b ≈ 0.0–0.3 (titles and short descriptions; length penalty hurts).
- **Code search**: k1 ≈ 2, b ≈ 0.3.

Tuning matters. A 5-15% NDCG bump from k1/b sweeping is common.

---

## 5. BM25 variants

### BM25F (fielded)

Documents have multiple fields with different importance: title vs body vs tags. **BM25F** computes a length-normalized score per field, weights them, sums.

```
title gets weight 3.0
body gets weight 1.0
tags get weight 2.0
```

Standard in Elasticsearch's `multi_match` queries with per-field boosts. The right default for product search.

### BM25+

Adds a small constant `δ` to each term contribution to ensure that a document containing the term gets a guaranteed positive score, no matter the length normalization.

### BM25-adpt / BM11 / BM15

Variants tweaking the length normalization curve. Mostly academic.

### DFR, LM-Dirichlet, LM-JM

Other classical rankers (Divergence From Randomness, Language Model with Dirichlet smoothing, Jelinek-Mercer). Compete with BM25 in narrow domains; BM25 is the dominant choice for general work.

---

## 6. The retrieval / ranking pipeline

In production, BM25 is the **retrieval** layer in a multi-stage pipeline:

```
   Query
     │
     ▼
  ┌─────────────────────┐
  │  BM25 retrieval     │   millions of docs → ~1000 candidates
  │  (Elasticsearch /   │
  │  Lucene / Vespa)    │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Vector / dense     │   semantic rerank — ANN over embeddings
  │  rerank (HNSW)      │   ~1000 → ~100
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  ML reranker        │   gradient-boosted or transformer
  │  (LTR / cross-       │   ~100 → top 10
  │   encoder)          │
  └──────────┬──────────┘
             │
             ▼
        Top results
```

Each stage refines a smaller, more accurate set. BM25's speed (milliseconds across millions of docs) is what makes the whole pipeline tractable. Vector and ML stages are too expensive to apply to the entire corpus.

This is the **hybrid search** pattern adopted at Google, Bing, Amazon, Algolia, Vespa, Pinecone, Weaviate. BM25 doesn't go away — it provides the lexical scaffolding semantic models build on.

See [Inverted Indexes →](./inverted-index.md), [Embedding-Based Retrieval →](./embedding-retrieval.md).

---

## 7. Worked example — three documents

Corpus of 3 documents (toy):

```
d1: "apache kafka is a streaming platform"  (6 words)
d2: "apache spark and apache kafka"          (5 words)
d3: "modern data engineering with snowflake" (5 words)
```

Query: `apache kafka`.

Stats:
- N = 3
- df("apache") = 2 → IDF ≈ log((3 - 2 + 0.5) / (2 + 0.5) + 1) ≈ log(1.6) ≈ 0.47
- df("kafka") = 2 → IDF ≈ 0.47
- avgdl = (6 + 5 + 5) / 3 = 5.33
- k1 = 1.2, b = 0.75

For d1 (TF apache=1, kafka=1, |d|=6):

```
For each term: BM25 component = IDF · (1·2.2) / (1 + 1.2·(1 - 0.75 + 0.75·6/5.33))
             = 0.47 · 2.2 / (1 + 1.2·(0.25 + 0.84))
             = 0.47 · 2.2 / (1 + 1.31)
             = 0.47 · 0.95
             = 0.45 per term
Total d1 = 0.45 + 0.45 = 0.90
```

For d2 (TF apache=2, kafka=1, |d|=5):

```
apache: 0.47 · (2·2.2) / (2 + 1.2·(0.25 + 0.75·5/5.33))
     = 0.47 · 4.4 / (2 + 1.2·(0.25 + 0.70))
     = 0.47 · 4.4 / (2 + 1.14)
     = 0.47 · 1.40
     = 0.66

kafka: 0.47 · (1·2.2) / (1 + 1.2·(0.25 + 0.70))
     = 0.47 · 2.2 / 2.14
     = 0.48

Total d2 = 0.66 + 0.48 = 1.14
```

d3 has neither term → score = 0.

Ranking: **d2 > d1 > d3**. d2 wins despite being shorter because apache appears twice. TF saturation prevents d2 from running away.

---

## 8. Why "modern" hasn't killed BM25

Several reasons BM25 keeps winning the cost/benefit comparison:

- **Speed.** Per-query cost is dominated by inverted-list traversal; BM25 scoring is a handful of arithmetic ops per posting. Milliseconds across millions of docs on a single node.
- **No training data needed.** Works out of the box on any corpus, in any language (with proper tokenization).
- **Robust baselines.** In TREC and BEIR benchmarks, BM25 is a strong baseline that many neural rerankers struggle to dominate on **out-of-domain** data.
- **Explainability.** Why did this rank #1? Look at the contributions per term. Cannot be done easily for a black-box transformer.
- **Lexical precision.** Matches exact terms — useful for codes, IDs, product names where semantic similarity isn't enough.

In 2026, **most production search systems retrieve candidates with BM25, then rerank with a learned model**. The combination is consistently better than either alone — vectors miss exact matches (codes, names, rare words), BM25 misses semantic similarity ("car" vs "automobile"). Together they cover both.

---

## 9. Practical tips

### Tokenization matters more than the formula

Garbage in, garbage out. A good analyzer (lowercase + Unicode normalization + stop words + stemming or lemmatization) often beats a BM25 tuning sweep. Match index-time and query-time analyzers exactly.

### Stop words handling

Removing "the", "is" speeds queries and reduces noise. But "to be or not to be" loses everything if you remove all stop words. Modern engines often keep stop words and let IDF push them down naturally.

### Phrase queries

BM25 alone doesn't capture phrase order. For "data engineer" (phrase), use the engine's position-aware query types (`match_phrase`, `phrase_match`). They blend BM25 with positional bonuses.

### Multi-field search (BM25F)

Almost always the right shape. Boost titles 3×, tags 2×, body 1×. Per-field length normalization keeps comparisons fair.

### Per-field analyzers

Title might use a stricter analyzer (no synonyms); body might use a synonym filter; tags might be `keyword` (no tokenization). One size doesn't fit all.

### Query expansion

Synonyms, abbreviations, related terms ("k8s" → "kubernetes"). Either at index time (more disk, faster query) or query time (slower query, smaller index).

### Negative tuning

If users keep typing "best laptop 2025" and you keep returning 2018 reviews, time decay or recency boosts help. Add via `function_score`, not by hacking BM25.

### A/B test

The only real measure of relevance is human judgment plus business metrics. Build a golden set; track NDCG / MRR / click-through; rank changes against the baseline.

---

## 10. Beyond text — TF-IDF and BM25 in other domains

The shape generalizes:

- **Music recommendation** — "track frequency" × "rarity of the track" across user playlists.
- **Product recommendation** — "item frequency in cart" × "item rarity overall."
- **Log search** — Splunk uses BM25-like scoring for ranking log lines for a query.
- **Code search** — BM25 over token frequencies in repositories. GitHub Blackbird uses variants.
- **Anomaly detection** — rare features in an event get high IDF-like scores; this is a building block of NLP-flavored anomaly systems.

Wherever **"how rare is this signal, and how concentrated is it in this entity?"** is the right question, TF-IDF / BM25 thinking applies.

---

## 11. Common Mistakes / Anti-Patterns

- **Plain TF-IDF without saturation.** A single highly-spammed term dominates the ranking.
- **No length normalization (`b = 0`).** Long catch-all pages win unfairly.
- **Same analyzer for vastly different fields.** "Apple" the company vs "apple" the fruit need different handling for query understanding.
- **No empirical tuning** of k1, b. Defaults are OK; sweeping often gives easy wins.
- **Skipping BM25 because "we use vectors now."** Lexical recall on names, codes, IDs is hard for vectors; hybrid wins.
- **Building a vector-only search and being surprised when "BMW M3" doesn't return the BMW M3.**
- **No relevance evaluation.** Gut-feel changes ship; nothing improves; subtle regressions stack up.
- **Ignoring per-field weights** for a doc with title + body + tags.
- **Using BM25 with bad tokenization.** Stemming "kafkas" to "kafka" matters. Half the wins live in the analyzer.
- **Tuning on the wrong metric.** A/B test the metric the business cares about (clicks, conversion), not just NDCG.
- **Mixing different IDF computations across shards.** Lucene approximates global IDF from per-shard df; for very small indexes, you can see anomalies. Increase shard size or use the explicit "global" IDF mode.
- **Reranking with a deep model on the entire candidate set.** Expensive. Always retrieve with BM25 first, then rerank a small top-K.

---

## 12. Cheat Card

```
PURPOSE   Score document relevance to a query using term-frequency
          and term-rarity statistics — fast, robust, explainable.

TF-IDF
  tf(t,d) · idf(t)  summed over query terms
  Classic, easy to compute, no saturation, no length norm

BM25 (default for modern engines)
  score = Σ IDF(t) · (tf · (k1+1)) / (tf + k1 · (1 - b + b · |d|/avgdl))
  k1 ≈ 1.2–2.0     saturation rate (lower = saturates earlier)
  b  ≈ 0.75        length-normalization strength
  Smoothed IDF avoids div-by-zero / negatives

BM25F (multi-field)
  Per-field length normalization + field weights
  Titles 3×, tags 2×, body 1× as a starting point

PIPELINE
  BM25 retrieval (millions → 1000)
  ANN / dense vector rerank (1000 → 100)
  ML reranker (100 → top 10)

WHEN BM25 STILL WINS
  Out-of-domain data
  Exact match on names / codes / rare words
  Need explainable scoring
  Low operational cost

WHEN TO ADD VECTORS
  Synonyms ("car" ↔ "automobile")
  Long natural-language questions
  Cross-lingual queries
  Semantic similarity beyond surface form

TUNING TIPS
  Match index-time and query-time analyzers
  Tune k1 and b on a held-out set
  Multi-field with per-field weights (BM25F)
  Recency / popularity boosts via function score
  Synonym lists for known equivalences

PITFALLS
  Plain TF-IDF (linear) on long docs
  Vector-only search missing exact-match cases
  No relevance evaluation set
  Same analyzer for all fields
  Reranking the entire corpus, not top-K

RULE   BM25 is the floor of modern search. Vectors and ML
       rerankers raise the ceiling. Use both — hybrid wins.
```

---

## 13. Resources

### Papers
- "A Statistical Interpretation of Term Specificity and Its Application in Retrieval" — Karen Spärck Jones, 1972 (the IDF paper).
- "Some Simple Effective Approximations to the 2-Poisson Model for Probabilistic Weighted Retrieval" — Robertson & Walker, 1994.
- "Okapi at TREC-3" — Robertson, Walker, Beaulieu, Gatford, Payne (1994). BM25 named.
- "The Probabilistic Relevance Framework: BM25 and Beyond" — Robertson & Zaragoza, 2009 (free PDF).
- "Beyond BM25: Learning to Rank" — Liu, 2009 (book-length tutorial).

### Books
- *Introduction to Information Retrieval* — Manning, Raghavan, Schütze (free online). Chapter 11 covers BM25 in detail.
- *Search Engines: Information Retrieval in Practice* — Croft, Metzler, Strohman.
- *Relevant Search* — Doug Turnbull, John Berryman. Hands-on Elasticsearch tuning.

### Documentation
- **Elasticsearch BM25** — <https://www.elastic.co/guide/en/elasticsearch/reference/current/similarity.html>
- **Lucene similarity** — <https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/search/similarities/BM25Similarity.html>
- **Vespa ranking** — <https://docs.vespa.ai/en/ranking.html>
- **OpenSearch ranking** — <https://opensearch.org/docs/latest/search-plugins/>

### Articles
- "BM25 — The Next Generation of Lucene Relevance" — Doug Turnbull et al.
- "Hybrid search: BM25 + vectors" — Vespa / Elastic / Pinecone blogs.
- "How we tuned search at Algolia / Etsy / Pinterest" — engineering blogs.
- "BEIR: A Heterogeneous Benchmark for Zero-Shot Evaluation of Information Retrieval Models" — Thakur et al., 2021.

### Videos
- *Information Retrieval lectures* — Stanford CS276 / CMU 11-642.
- *BM25 explained* — multiple Elastic / OpenSearch conference talks.
- ByteByteGo — "BM25 Explained."

### Tools
- **Elasticsearch / OpenSearch / Solr / Vespa** — BM25 built in.
- **Lucene / Tantivy** — embed-able BM25 implementations.
- **Pyserini** — Python research toolkit with BM25 (built on Lucene).
- **`rank_bm25`** — pip-installable Python lib for quick experiments.

### Adjacent reading
- [Inverted Indexes →](./inverted-index.md)
- [Trie Data Structure for Autocomplete →](./trie.md)
- [PageRank Algorithm →](./pagerank.md)
- [Embedding-Based Retrieval (ANN, HNSW, FAISS) →](./embedding-retrieval.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Vector Databases →](../04-databases/vector-databases.md)
- [Design Google Search / Web Crawler →](../18-case-studies/search-engine.md)
- [Design Recommendation System →](../18-case-studies/recommendation-system.md)

---

*Previous:* [← PageRank Algorithm](./pagerank.md)  |  *Next:* [Embedding-Based Retrieval →](./embedding-retrieval.md)
