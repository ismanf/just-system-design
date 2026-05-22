# Trie Data Structure for Autocomplete

> **TL;DR** — A **trie** (pronounced "try" or "tree") is a tree where each node represents one character of a key, and every path from root to a marked node spells a complete word. Looking up a key takes O(L) time where L is the key length — independent of the dictionary size. That makes tries the canonical data structure for **autocomplete**, **prefix search**, **typeahead**, **IP routing tables**, **dictionary spell-checks**, and **regex / pattern matching**. Real production autocomplete systems use trie *ideas* combined with **frequency / popularity weighting**, **compressed trie variants** (radix trees, Patricia tries, ternary search trees), **fuzzy matching** (Damerau-Levenshtein, BK-trees), and **personalization layers**. The honest take: **a textbook trie is the right mental model, but production autocomplete is a search problem — search engines (Elasticsearch's completion suggester, custom Lucene FST-based stores) usually beat hand-rolled tries above modest scale**.

---

## 1. The big picture

```
       (root)
       /  |  \
      c   t   p
      |   |   |
      a   o   e
      |   |   |
      t*  *   n
      |       |
      s*      *
              |
              c
              |
              i
              |
              l*

  Contains: "cat", "cats", "to", "pen", "pencil"
  *  = end-of-word marker
```

Each edge is a character. Each node may be:
- An **internal node** (path through to longer words).
- A **terminal node** (`*`) — the path so far is a complete word.

To search for `pencil`:
1. From root, follow edge `p`.
2. From there, edge `e`.
3. Then `n`, `c`, `i`, `l`.
4. Check that the final node is terminal.

`O(L)` lookup regardless of how many words are in the trie.

To find **all words starting with `pe`**:
1. Walk to the node at `pe`.
2. DFS / BFS everything below it, collecting terminal nodes.

That's the autocomplete primitive. Add ranking, fuzziness, and personalization, and you have a real product.

---

## 2. Why tries fit autocomplete

The query pattern is "I have a prefix; give me complete words that start with it." A trie is *precisely* shaped for that:

- **Sub-linear in dictionary size**. Adding a million more words doesn't make a 5-character prefix lookup slower.
- **Branch-by-branch storage**. Words sharing a prefix share path memory.
- **Streaming-friendly**. After each keystroke, you have already navigated to the prefix node — the next keystroke only requires one extra step.
- **Bounded latency**. Worst-case lookup time depends on the longest word in the dictionary, not the dictionary size.

Compare to a sorted array with binary search: O(L · log N) lookups (the log N is binary-search comparisons; each comparison costs O(L) character compares). Or a hash map: O(L) per word — but no prefix query.

For autocomplete, the trie wins on the operation that matters most.

---

## 3. The basic trie — implementation sketch

A canonical Python implementation:

```python
class TrieNode:
    __slots__ = ("children", "terminal", "data")
    def __init__(self):
        self.children = {}       # char → TrieNode
        self.terminal = False    # is this the end of a word?
        self.data = None         # frequency, payload, etc.

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word, data=None):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.terminal = True
        node.data = data

    def search(self, word):
        node = self._walk(word)
        return node.data if node and node.terminal else None

    def starts_with(self, prefix):
        node = self._walk(prefix)
        if not node:
            return []
        return list(self._collect(node, prefix))

    def _walk(self, s):
        node = self.root
        for ch in s:
            node = node.children.get(ch)
            if not node:
                return None
        return node

    def _collect(self, node, prefix):
        if node.terminal:
            yield (prefix, node.data)
        for ch, child in node.children.items():
            yield from self._collect(child, prefix + ch)
```

That's the textbook implementation. For 100k–1M words it works. Beyond that you'll want to think harder about memory and performance.

---

## 4. Variants — when the textbook trie hits limits

### Compressed trie / radix tree

Plain tries store one character per node. For dictionaries with long unique suffixes, that's wasteful — chains of single-child nodes pad the tree. A **compressed trie** (also called a **radix tree** or **Patricia trie**) collapses chains of single-child nodes into single edges labeled with the entire substring.

```
plain:        c → a → t → s*
compressed:   "cats"*
```

10–50× less memory for sparse dictionaries. Standard in Linux kernel routing tables (IPv4 routing → "longest prefix match" → Patricia trie).

### Ternary search tree (TST)

Each node has three children — `lt`, `eq`, `gt` — and stores one character. A more cache-friendly layout than a plain trie for dense character sets. Used in some classic auto-complete libraries; faster than a hashmap-based trie for English-like alphabets.

### Double-array trie

Stores transitions in two parallel integer arrays with clever indexing. Extremely memory-compact. Used in some IME (Japanese / Chinese input) systems. Tricky to maintain dynamically.

### FST (Finite-State Transducer)

A directed graph that merges both prefixes and **suffixes**. Lucene's auto-complete and dictionary stores use FSTs. Tiny on disk (often a few bytes per word), fast to traverse, optimal for read-mostly workloads. The de facto data structure for production text auto-complete in Java/Elasticsearch land.

### Suffix tree / suffix array

For "find all positions in this text where the pattern appears" rather than "complete this prefix." Different problem, related machinery. Out of scope here.

For most autocomplete: start with a plain trie if you control everything; use Elasticsearch / FST-backed engine in real systems.

---

## 5. Ranking — autocomplete is search, not just prefix match

Returning *all* words matching a prefix is rarely what users want. They want the **top N most likely** completions. Three ranking inputs:

1. **Popularity / frequency** — how often is this completion clicked? Stored on the terminal node.
2. **Recency / freshness** — newly trending queries should rank up.
3. **Personalization** — user history, location, language, device.

A typical ranked autocomplete:

```python
def top_n(prefix, n=10):
    node = self._walk(prefix)
    if not node:
        return []
    # collect candidates with priority queue
    heap = []
    for word, data in self._collect(node, prefix):
        score = data.popularity * decay(data.last_seen)
        heapq.heappush(heap, (-score, word))
    return [heapq.heappop(heap)[1] for _ in range(min(n, len(heap)))]
```

For huge trees, you don't want to walk the entire subtree on every keystroke. Two optimizations:

- **Precompute top-K per node**. Each node stores its own top-K completions; lookup is O(L + K) total.
- **Best-first search**. Use a priority queue ordered by an *upper-bound* on what each subtree could contribute. Prune when no improvement is possible.

Production systems use both. Google's autocomplete is essentially "best-first search on a huge weighted trie/FST, personalized per user, with continual updates."

---

## 6. Fuzzy autocomplete — typos and approximate matches

Users mistype. "tomato" should match "tomtao" / "tomatoe" / "tomatoo" / "tomatto." Two approaches:

### Damerau-Levenshtein over the trie

Walk the trie with a small edit budget (e.g., 1 or 2 edits). Standard dynamic programming over (trie node, prefix length) pairs, pruned by the budget. The trie shape makes this much faster than scanning the dictionary, because invalid subtrees are pruned early.

### BK-trees

A **Burkhard-Keller tree** is a separate metric-tree structure for fuzzy matching by edit distance. Useful when you don't need prefix semantics — pure "near this query" matches.

### Phonetic encoding

For names, use Soundex, Double Metaphone, or NYSIIS — encode the word phonetically, then exact-match in the trie of phonetic codes. "Smith" / "Smyth" / "Smithe" collide.

### Modern approaches

Embedding-based suggestion ranking (small transformer or fastText embeddings) is increasingly common in production. The trie/FST acts as a candidate generator; an embedding-based reranker scores the top 100 candidates. See [Embedding-Based Retrieval →](./embedding-retrieval.md).

---

## 7. Memory — the trie's biggest weakness

A naive trie uses one node per character with a dict-of-children. For 1M English words averaging 8 characters, that's ~8M nodes, each with overhead — easily 1–2 GB in Python.

Mitigations:

- **Compressed trie / radix tree** — collapse single-child chains.
- **Use arrays instead of dicts** for children when the alphabet is small and dense (a–z).
- **Bit-pack flags** — `terminal`, `weight` packed into a single integer.
- **Marisa-trie / FST** — serializable, compact on-disk formats.
- **Lazy node loading** — for huge tries, load only the active prefix's region.

For server-side autocomplete with millions of entries, FSTs (Lucene / Marisa / SCDK) are the right answer. For client-side autocomplete (in-browser, in-mobile-app), a small compressed trie with ~10k entries fits well.

---

## 8. Worked example — multi-tier autocomplete

A typical production architecture:

```
                ┌─────────────────┐
                │   Client UI     │
                │  (debounce 200ms)│
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐         ┌──────────────┐
                │ Frontend / CDN  │  cache  │  Top queries │
                │  edge cache     │ ◄────── │  per region  │
                └────────┬────────┘         └──────────────┘
                         │ miss
                         ▼
                ┌─────────────────┐         ┌──────────────┐
                │ Suggest service │ ────►   │  Trie / FST  │
                │  (per-language) │ ◄────── │  in memory   │
                └────────┬────────┘         └──────────────┘
                         │
                         ▼
                ┌─────────────────┐
                │  Reranker (ML)  │
                │  personalize    │
                └─────────────────┘
```

- **Debounce** the input client-side (300 ms).
- **CDN cache** the most popular prefixes regionally (Cloudflare Workers + KV).
- **Trie / FST** in memory on the suggest service for instant top-N.
- **Reranker** applies personalization, query freshness, ML signals.

That's a multi-team build. Most teams start with **Elasticsearch's completion suggester** (which is an FST under the hood) and add personalization later. **PostgreSQL's `pg_trgm` extension** is a workable starting point for small dictionaries.

---

## 9. Tries beyond autocomplete

Tries are general enough that they show up in many places:

- **IP routing tables** — longest prefix match. Linux uses a level-compressed trie (LC-trie); modern hardware uses TCAMs.
- **URL routing** in web frameworks — match `/api/v1/users/:id` patterns. Many frameworks (gin, Echo, FastAPI under the hood) use radix tries.
- **Spell checkers** — fuzzy matching on a dictionary trie.
- **Regex / glob matching** — Aho-Corasick (a trie + failure links) for multi-pattern search.
- **Dictionary attacks / password hashing** — analyze password lists.
- **DNA / bioinformatics** — suffix tries for sequence analysis.
- **JSON / log key indexing** — Cribl, Splunk use trie-like structures for field discovery.

Recognize the shape and you save yourself reinventing.

---

## 10. Common Mistakes / Anti-Patterns

- **Returning every match for a prefix.** Always limit to top N.
- **No ranking.** "ab" returns alphabetical order. Useless. Rank by popularity at minimum.
- **Recomputing the entire subtree on each keystroke.** Cache the prefix node; recompute incrementally.
- **No debounce on the client.** Hammering the server with one request per keystroke.
- **No CDN cache for top prefixes.** "ne" → "netflix" is identical for millions of users; cache it.
- **Hash map per node in a memory-tight environment.** Use arrays or compressed tries.
- **Custom trie when Elasticsearch / FST would do.** For server-side production autocomplete, FST-backed engines are mature and free.
- **Fuzzy matching budget too generous.** Damerau-Levenshtein with edit distance 3 against a 1M-word trie is slow. Cap at 1 (sometimes 2) for live UI.
- **No normalization.** "Café" vs "Cafe" — normalize before insert and query (NFC + lowercase + diacritic strip).
- **Per-user trie in memory.** Doesn't scale. Personalization belongs in the ranker, not the trie.
- **Ignoring CJK / IME / segmentation.** Chinese / Japanese / Korean autocomplete needs language-specific tokenization, not character-level.
- **Trie that never expires entries.** Trending queries lose freshness; queries that no one searches anymore stay forever. Decay weights or rotate.
- **Loading the entire trie at server startup with no warmup, blocking traffic for minutes.** Build at deploy time; mmap from disk.
- **Returning suggestions that link to dead URLs.** Validate completions are still valid in the catalog (item still in stock, page still exists).
- **No per-language separation.** One trie containing English + Mandarin + Cyrillic produces nonsense rankings.

---

## 11. Cheat Card

```
PURPOSE   Prefix-keyed dictionary that answers "what words start
          with X?" in O(L) — independent of dictionary size.

CORE
  Root → children by character → terminal nodes mark words
  Lookup: walk character by character
  Prefix listing: walk to prefix, DFS below it
  Each terminal can carry data (frequency, payload)

VARIANTS
  Plain trie       textbook, fine for small N
  Radix / Patricia compress single-child chains; routing tables
  Ternary search   3-way; cache-friendly for English-like alphabets
  FST              merged prefixes + suffixes; Lucene/Elasticsearch
  Marisa / Double  ultra-compact, mostly-static dictionaries
  Suffix tree      "find substring" problem; different beast

AUTOCOMPLETE RANKING
  Popularity / frequency (most important)
  Recency / freshness decay
  Personalization (user, location, language)
  Precompute top-K per node OR best-first with pruning

FUZZY MATCH
  Damerau-Levenshtein over trie, cap edit budget at 1 (or 2)
  BK-tree for pure metric search
  Phonetic encoding (Soundex, Metaphone) for names
  Embedding reranker on top-K candidates

ARCHITECTURE
  Client debounce 200-300 ms
  CDN / edge cache for hot prefixes
  In-memory FST / trie per language
  ML reranker for personalization

WHEN A TRIE LOSES
  Production at large scale → FST in Lucene/Elasticsearch
  Substring search → suffix array / FM-index
  Vector similarity → ANN (HNSW)

PITFALLS
  No ranking, no top-K limit
  Per-keystroke server hit (no debounce / cache)
  Hash map node in tight RAM
  Edit distance budget too high
  No language separation
  Personalization stored inside the trie

RULE   Use a trie's mental model for any prefix problem. In
       production, reach for Elasticsearch / Lucene FST first.
       Always rank, always limit, always debounce.
```

---

## 12. Resources

### Books
- *Algorithms* (4th ed.) — Sedgewick & Wayne. Chapter on tries and TSTs.
- *Introduction to Information Retrieval* — Manning, Raghavan, Schütze (free online). Auto-complete and search structures.
- *Data Structures and Network Algorithms* — Tarjan.

### Documentation
- **Elasticsearch Completion Suggester** — <https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester>
- **Lucene FSTs** — <https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/util/fst/package-summary.html>
- **Linux LC-trie (routing)** — kernel source `net/ipv4/fib_trie.c`
- **`pygtrie`** — pure Python tries: <https://github.com/google/pygtrie>
- **`marisa-trie`** — compact static tries.

### Articles
- "How autocomplete works" — Algolia engineering blog.
- "Building autocomplete at scale" — Twitter, LinkedIn, Pinterest engineering posts.
- "FSTs at Twitter" — Twitter Search engineering.
- "Inside Lucene's FST" — Mike McCandless blog.

### Videos
- *Tries and Suffix Trees* — Stanford CS166 / MIT 6.851 lectures.
- *Building autocomplete with Elasticsearch* — Elastic webinars.
- ByteByteGo — "Autocomplete System Design."

### Tools
- **Elasticsearch / OpenSearch** completion suggester (FST under the hood).
- **`marisa-trie`**, **`pygtrie`**, **`datrie`** (Python).
- **`tst`**, **`go-radix`** (Go).
- **`Lucene FSTs`** (Java).
- **`tantivy`** (Rust, Lucene-like).
- **`pg_trgm`** (Postgres extension) for simple typo-tolerant matching.

### Adjacent reading
- [Geohashing & Quadtrees →](./geohashing-quadtrees.md)
- [R-Trees →](./r-trees.md)
- [Inverted Indexes →](./inverted-index.md)
- [TF-IDF & BM25 →](./tf-idf-bm25.md)
- [Embedding-Based Retrieval →](./embedding-retrieval.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Design Typeahead / Autocomplete →](../18-case-studies/typeahead.md)
- [Design a Search Autocomplete →](../18-case-studies/search-autocomplete.md)

---

*Previous:* [← R-Trees](./r-trees.md)  |  *Next:* [Skip Lists →](./skip-lists.md)
