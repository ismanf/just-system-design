# PageRank Algorithm

> **TL;DR** — **PageRank** is a graph algorithm that computes the importance of each node based on the importance of nodes that point to it. The defining intuition: **a link from a popular page is worth more than a link from an obscure one, and importance flows through the graph**. Sergey Brin and Larry Page invented it for Google's web search in 1996; it's been generalized into a workhorse algorithm used in **search ranking, fraud detection, recommendation systems, citation analysis, influence detection in social networks, and graph machine learning embeddings**. Mathematically it's the **stationary distribution of a random walk** on a directed graph with a teleport probability — a power-iteration eigenvector computation. The honest take: **PageRank by itself is rarely the final ranker anymore**, but it remains a fundamental building block for graph reasoning, and its variants (personalized PageRank, SimRank, TrustRank, HITS) are essential tools whenever the question is "how important is X in this network?"

---

## 1. The big picture

```
Page A ─────► Page B ─────► Page C
   │                          ▲
   ▼                          │
Page D ◄──────────────────────┘
   │
   ▼
Page E
```

Question: how do we rank these pages by importance?

PageRank's answer: simulate a random surfer who:
- At each step, with probability `d` (typically 0.85), clicks a random outgoing link.
- With probability `1 − d`, teleports to a uniformly random page.

After enough steps, the proportion of time spent on each page converges to a stable distribution — that's the PageRank.

The recursive intuition: a page is important if important pages link to it. A page is unimportant if few or only unimportant pages link to it. Mathematically:

```
PR(p) = (1 - d) / N + d · sum over q in incoming(p) of  PR(q) / out_degree(q)
```

Where:
- `N` is the total number of pages.
- `d` is the damping factor (~0.85). It prevents the algorithm from getting stuck in cycles and dead ends.
- `(1 − d) / N` is the "teleport" probability — every page gets a baseline.
- The sum spreads each in-neighbor's rank across its out-links equally.

You solve this iteratively. Start with `PR = 1/N` for every page. Apply the formula. Repeat. After ~30–100 iterations, values stabilize.

---

## 2. Why this changed search

Pre-1998 search engines (AltaVista, Yahoo, Lycos) ranked by **on-page signals**: how often a term appeared on a page, where it appeared, density. Spammers exploited that — keyword stuffing, hidden text, fake metadata.

PageRank introduced a **graph-structural signal**: the *web itself* votes on what's important via its links. Manipulating PageRank requires getting other pages to link to you. That was harder than keyword stuffing, and led to vastly better search results.

Google's early advantage was substantially this signal, combined with strong tokenization and BM25-like text matching. Inside a year of its release, Google had captured the search market.

PageRank also generalized far beyond web search. It works on **any directed graph** where you want to measure node importance from the graph's structure.

---

## 3. Random-walk and matrix view

Mathematically, PageRank is the **stationary distribution** of a Markov chain.

Define the transition matrix `M` where `M[i][j]` = probability of going from page `j` to page `i`. For a uniform random click: `M[i][j] = 1/out_degree(j)` if there's a link from `j` to `i`, else 0.

With the teleport adjustment ("Google matrix"):

```
G = d · M + (1 - d) · (1/N) · J
```

Where `J` is the all-ones matrix.

PageRank vector `r` satisfies:

```
r = G · r
```

i.e., `r` is the eigenvector of `G` for eigenvalue 1 (the stationary distribution).

Solving methods:

- **Power iteration** — repeatedly multiply: `r_{k+1} = G · r_k`. Converges in tens of iterations. Easy to parallelize and implement in MapReduce or Spark. The original Google approach.
- **Direct solve** — `(I - dM) · r = (1-d)/N · 1`. For small graphs, solve the linear system.
- **Monte Carlo random walks** — simulate many short random walks and count visits. Useful for **personalized PageRank** with localized computation.
- **Push-style algorithms** — propagate residuals from "high-rank" nodes; faster for sparse graphs.

For the web (~30 billion pages historically), power iteration over a sparse representation is how Google originally did it. On modern hardware, frameworks like **GraphX** (Spark), **Giraph**, **GraphFrames**, or specialized GPU graph engines run PageRank routinely on tens of billions of edges.

---

## 4. Handling dangling nodes and cycles

The math has corner cases that have to be patched:

### Dangling nodes (no outgoing links)

A page with no outgoing links would absorb rank forever. Two standard fixes:

- **Treat dangling node out-links as uniform**: pretend they link to every page with probability `1/N`.
- **Renormalize each iteration**: redistribute the rank "lost" through dangling nodes uniformly.

### Spider traps (cycles with no escape)

A small cluster of pages linking only to each other would accumulate all the rank. The teleport probability `1 − d` solves this — the random surfer can jump to any page at each step.

### Pages with no incoming links

They still get the baseline `(1 − d) / N`. Important: every page has some PageRank.

The constant `d = 0.85` is empirical. Brin and Page chose it because it balances "follow the graph" with "explore everywhere." Lower `d` flattens the distribution (more uniform); higher `d` amplifies the graph signal but converges slower and is more sensitive to noise.

---

## 5. Worked example — five-page graph

Graph: A → B, A → C, B → C, C → A, D → C.

```
N = 4  (ignore E)
d = 0.85
Initialize PR = [0.25, 0.25, 0.25, 0.25]

Iteration 1:
  PR(A) = 0.0375 + 0.85 · (PR(C)/1)              = 0.0375 + 0.2125 = 0.25
  PR(B) = 0.0375 + 0.85 · (PR(A)/2)              = 0.0375 + 0.1063 = 0.1438
  PR(C) = 0.0375 + 0.85 · (PR(A)/2 + PR(B)/1 + PR(D)/1)
                = 0.0375 + 0.85 · (0.125+0.25+0.25)
                = 0.0375 + 0.5313 = 0.5688
  PR(D) = 0.0375 + 0                              = 0.0375
```

Then renormalize so `sum(PR) = 1` (or `N`, depending on the convention), and iterate again. After ~30 iterations, values converge.

Intuition: C has many in-links (and one is from itself via the cycle A→C, C→A, B→C). It ends up dominant. D has no in-links and stays at the baseline.

This tiny example is enough to see why PageRank is computationally an eigenvector problem — the values *interact* until they reach equilibrium.

---

## 6. PageRank variants and relatives

### Personalized PageRank (PPR)

Replace the uniform teleport vector with a non-uniform one biased toward a specific set of "seed" nodes. The result ranks pages by *importance relative to those seeds*. Used for:

- **Search personalization** — boost results closer to a user's interests / history.
- **Recommendation systems** — find products similar to ones a user liked.
- **Twitter's "Who to Follow"** — personalized over the social graph.
- **Pinterest Pixie** — random walks over the pin graph for recommendations.

PPR is the most operationally useful variant. Many production systems use it without ever calling it "PageRank."

### TrustRank

Bias the teleport vector toward known *trusted* pages, propagate trust through the graph. Used historically for spam detection — pages connected only to spammy clusters get low TrustRank.

### HITS (Hyperlink-Induced Topic Search)

Jon Kleinberg's contemporary alternative (1998). Each page has two scores:
- **Hub score** — links to many good authorities.
- **Authority score** — linked to by many good hubs.

Iteratively reinforce. HITS is **query-dependent** (computed on a subgraph for the query), while PageRank is query-independent (global). Mostly historical now, but the dual-score idea reappears in modern graph learning.

### SimRank

Measures structural similarity between nodes. "Two nodes are similar if their neighbors are similar." Used for clustering, recommendation.

### EigenTrust

Reputation in P2P networks. Same eigenvector intuition applied to peer reliability.

### Markov chain centrality, eigenvector centrality

The general family. PageRank is a specific instance with the teleport adjustment.

---

## 7. Modern uses

### Search ranking (then and now)

Google still uses PageRank-like signals among hundreds of ranking factors. The original "PageRank score per URL" view has been replaced by far more sophisticated learning-to-rank models, but the underlying graph signals (BackRub-era link analysis) endure.

### Fraud detection

In financial networks, suspicious accounts often form clusters that link to each other. PPR seeded from known fraud cases highlights the cluster. Used at PayPal, Stripe, Chime.

### Bot detection

Twitter / X uses graph signals to detect bot accounts. Bots tend to follow many users and be followed by few real users; PageRank-style scores flag them.

### Recommendation

Pinterest's Pixie system runs personalized random walks over the pin/board graph to surface recommendations. Each visit is a "vote" for the destination pin. Conceptually a Monte Carlo PPR.

### Citation networks

In academia, PageRank-like measures (Eigenfactor, h-index variants) rank papers by how often other influential papers cite them. Standard in bibliometrics.

### Knowledge graphs

Wikipedia, Wikidata, Freebase use PageRank-derived scores to rank entities. Google's Knowledge Graph uses graph signals heavily.

### GraphML / GNNs

Modern Graph Neural Networks generalize the "aggregate from neighbors, iterate" pattern. PageRank-style propagation is the simplest GNN; deep GNNs (GCN, GraphSAGE) add learnable transformations. PageRank can also bootstrap embeddings (DeepWalk uses random walks, the same idea).

---

## 8. Computing PageRank at scale

### Sparse matrix-vector multiply

Each iteration is one **SpMV** (sparse matrix × vector) operation. On modern CPUs, billions of edges per second per core.

### Distributed power iteration

- **Pregel / Giraph / GraphX / GraphFrames** — vertex-centric programming model ("think like a vertex"). Each vertex sums incoming messages and sends out new messages.
- **MapReduce / Spark** — map: each edge sends `PR(src) / out_degree(src)` to dst. Reduce: sum at each dst, apply damping. Loop.
- **GraphChi / Galois** — single-machine large-graph engines.

Convergence after 20–50 iterations is typical for the web; smaller, denser graphs converge faster.

### Push-based personalized PageRank

For PPR on huge graphs, computing the full vector is wasteful when you only care about top-K from one seed. **Forward push** (Andersen, Chung, Lang) and **backward push** (Lofgren et al.) compute approximate PPR locally — much cheaper than full power iteration.

### Approximation by Monte Carlo

Run K random walks of length L from each query seed; count visit frequencies. With K=1000, L=10, you get usable PPR for one seed in milliseconds on a graph with billions of edges (data pre-loaded). Pinterest's Pixie and Twitter's WTF are built this way.

---

## 9. Limitations and modern view

PageRank has known weaknesses:

- **Manipulable** — link farms, paid links, comment spam. Google has spent 25 years fighting this with penalties and ranking adjustments.
- **Topic-blind** — vanilla PageRank doesn't know if a page is about cooking or finance.
- **Static** — recomputed periodically; doesn't react quickly to new pages.
- **Treats all links equally** — a link in a footer counts the same as one in the body.
- **Over-rewards old pages** — long-established pages accumulate links, new pages start at zero.

Modern Google ranking blends:
- **PageRank-like graph signals** for trust and authority.
- **BM25** and language-model-based text relevance. See [TF-IDF & BM25 →](./tf-idf-bm25.md), [Inverted Indexes →](./inverted-index.md).
- **Click-through data, dwell time, user behavior signals**.
- **Neural rerankers** (BERT-based, MUM, RankBrain).
- **Embedding-based retrieval** for semantic match. See [Embedding-Based Retrieval →](./embedding-retrieval.md).
- **Many quality / freshness / spam / E-E-A-T heuristics**.

The lesson: in 2026, **PageRank is one signal of many**. Useful, foundational, never the whole story.

---

## 10. Common Mistakes / Anti-Patterns

- **Treating PageRank as the only ranker.** Combine with text, click, embedding signals.
- **Not handling dangling nodes.** Rank leaks; sum drifts; convergence is wrong.
- **Wrong damping factor.** Lower or higher than 0.85 without justification.
- **Forgetting to normalize each iteration.** Sum diverges or shrinks; comparisons across versions are meaningless.
- **Computing global PageRank when personalized is the right question.** Global says "important to everyone." Personalized says "important to this user."
- **Ignoring edge weights when they matter.** Weighted PageRank exists; use it for things like citation counts or transaction amounts.
- **Recomputing for every query.** Compute and cache; refresh periodically.
- **Treating PageRank as a black box for ML features.** Inspect, sanity-check on small subgraphs, validate against intuition.
- **Trusting PageRank for fraud without monitoring for gaming.** Adversaries adapt.
- **No convergence check.** Iterate until residual < threshold; otherwise you stop too early or compute too long.
- **Hash-keyed adjacency lists in Python at 10B edges.** Use a real graph engine.
- **Ignoring graph sparsity.** PageRank is fast on sparse graphs. Don't materialize the full N×N matrix.

---

## 11. Cheat Card

```
PURPOSE   Rank nodes in a directed graph by their importance,
          measured as the stationary distribution of a damped
          random walk.

CORE FORMULA
  PR(p) = (1-d)/N + d · Σ over q→p   PR(q) / out_degree(q)
  d ≈ 0.85
  Solved by power iteration (~30 rounds)

COMPUTATION
  Sparse matrix-vector multiply, iterated
  Pregel / Giraph / GraphX / GraphFrames at scale
  Push-style or Monte Carlo for personalized PPR

VARIANTS
  Personalized PageRank   recommendations, search personalization
  TrustRank               spam / trust propagation
  HITS                    hub + authority dual scores
  SimRank                 structural similarity
  EigenTrust              P2P reputation

USE CASES
  Web search ranking (one of many signals)
  Recommendations (Pinterest Pixie, Twitter WTF)
  Fraud / bot detection on transaction or follow graphs
  Citation / influence analysis
  Knowledge graph entity ranking
  Bootstrap for graph embeddings (DeepWalk, GNN init)

EDGE CASES
  Dangling nodes → uniform out-distribution or renormalize
  Spider traps → fixed by teleport (1-d) component
  No incoming → baseline (1-d)/N

LIMITATIONS
  Manipulable by link spam
  Topic-blind
  Static / slow to refresh
  Treats all links equally
  One signal — not a full ranker

PITFALLS
  Ignoring dangling nodes
  No normalization between iterations
  Wrong damping factor "by intuition"
  Computing global when personalized is needed
  No convergence check
  Materializing full N×N matrix on big graphs

RULE   PageRank is a graph eigenvector. Use it as a signal,
       not as an oracle. Personalized PPR is often what you
       actually want.
```

---

## 12. Resources

### Papers
- "The Anatomy of a Large-Scale Hypertextual Web Search Engine" — Brin & Page, 1998.
- "The PageRank Citation Ranking: Bringing Order to the Web" — Page, Brin, Motwani, Winograd, 1999.
- "Authoritative Sources in a Hyperlinked Environment" — Kleinberg, 1998 (HITS).
- "Combating Web Spam with TrustRank" — Gyongyi, Garcia-Molina, Pedersen, 2004.
- "Local Graph Partitioning using PageRank Vectors" — Andersen, Chung, Lang, 2006 (forward push).
- "Pixie: A System for Recommending 3+ Billion Items to 200+ Million Users in Real-Time" — Pinterest engineering.

### Books
- *Mining of Massive Datasets* — Leskovec, Rajaraman, Ullman (free online). Excellent chapter on PageRank.
- *Networks, Crowds, and Markets* — Easley & Kleinberg (free online).
- *Graph Algorithms in the Language of Linear Algebra* — Kepner & Gilbert.

### Documentation
- **Apache Spark GraphX** — <https://spark.apache.org/graphx/>
- **Apache Giraph** — <https://giraph.apache.org>
- **Neo4j graph algorithms** — <https://neo4j.com/docs/graph-data-science/current/>
- **NetworkX PageRank** — <https://networkx.org/documentation/stable/reference/algorithms/link_analysis.html>

### Articles
- "Inside Pinterest's recommendation engine: Pixie" — Pinterest engineering blog.
- "How Twitter's WTF works" — Twitter engineering historical.
- "PageRank explained" — Stanford CS245 / CS246 lecture notes.
- "Personalized PageRank in production" — multiple engineering posts.

### Videos
- *Andrew Ng / Stanford CS246* — PageRank lecture.
- *Jure Leskovec, CS224W* — Network Analysis course, free online.
- ByteByteGo — "PageRank Algorithm Explained."

### Tools
- **NetworkX** (Python) — easy for small graphs.
- **igraph** (Python/R/C) — fast for medium graphs.
- **Apache Spark GraphX / GraphFrames** — distributed.
- **Apache Giraph / Pregel-style engines**.
- **Neo4j**, **TigerGraph**, **Memgraph**, **JanusGraph** — graph databases with built-in PageRank.
- **cuGraph / Gunrock** — GPU graph computation.

### Adjacent reading
- [Inverted Indexes →](./inverted-index.md)
- [TF-IDF & BM25 →](./tf-idf-bm25.md)
- [Embedding-Based Retrieval (ANN, HNSW, FAISS) →](./embedding-retrieval.md)
- [Graph Databases (Neo4j, Neptune) →](../04-databases/graph-databases.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Design Recommendation System →](../18-case-studies/recommendation-system.md)
- [Design Google Search / Web Crawler →](../18-case-studies/search-engine.md)

---

*Previous:* [← Inverted Indexes](./inverted-index.md)  |  *Next:* [TF-IDF & BM25 →](./tf-idf-bm25.md)
