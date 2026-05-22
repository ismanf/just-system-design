# R-Trees

> **TL;DR** — An **R-tree** is a balanced tree where every node stores **minimum bounding rectangles (MBRs)** instead of point values. Each MBR encloses all the geometry in the subtree below it. Range and nearest-neighbor queries walk the tree, descending into subtrees whose MBRs intersect the query and pruning the rest. Invented by Antonin Guttman in 1984 to do for spatial data what B-trees did for sorted data, R-trees became the foundation of spatial indexing in **PostGIS** (via GiST), **Oracle Spatial**, **MySQL Spatial**, **Lucene**, **SQLite (R*Tree module)**, and most GIS engines. They handle **arbitrary geometries — points, lines, polygons** — not just points, which is why they outlast simpler structures for real geographic workloads. The honest take: **you'll rarely write an R-tree yourself, but understanding the MBR-and-prune mental model lets you reason about query plans in PostGIS, choose between R-tree variants, and avoid pathological data layouts**. The interesting modern variants — **R*-tree**, **Hilbert R-tree**, **bulk-loaded STR R-tree** — are about *how nodes get filled* during inserts and bulk loads, not the basic shape.

---

## 1. The big picture

A B-tree splits 1D values: "left subtree has keys < 30, right has keys >= 30." That doesn't generalize to 2D points — no single value divides a plane.

R-tree's answer: **each tree node holds a minimum bounding rectangle that encloses every geometry in its subtree**. Internal nodes are "this rectangle contains everything below me." Leaves hold the actual records (or pointers to them).

```
                        ┌───────────────────────────────┐
                        │  Root MBR: whole map          │
                        └────────────┬──────────────────┘
                  ┌──────────────────┼──────────────────┐
                  ▼                  ▼                  ▼
            ┌──────────┐       ┌──────────┐       ┌──────────┐
            │ MBR_A    │       │ MBR_B    │       │ MBR_C    │
            │ (region) │       │ (region) │       │ (region) │
            └────┬─────┘       └────┬─────┘       └────┬─────┘
                 │                  │                  │
       ┌─────────┼────┐    ┌────────┼─────┐    ┌──────┼──────┐
       ▼              ▼    ▼              ▼    ▼             ▼
   restaurant_A   shop_B  park_C       road_D   bridge_E   building_F
```

A query "what's in this box?" starts at the root, checks which child MBRs intersect the box, and recurses only into those. Subtrees outside the box are pruned without inspecting their contents — same idea as a B-tree's "skip the half that can't contain my key."

---

## 2. Why R-trees won the spatial-DB war

Several systems competed in the 80s and 90s: KD-trees, point quadtrees, BSP trees, grid files. R-trees won because they:

- **Handle any geometry**, not just points. Lines, polygons, irregular shapes — anything with a bounding rectangle.
- **Stay balanced** under arbitrary insertions, like a B-tree.
- **Map well to disk** — each node fits in a page; tree height is small.
- **Compose naturally with B-trees** in real DB engines — PostGIS's GiST is a generic balanced tree that uses R-tree-style MBR keys.
- **Tolerate updates** — inserts and deletes work; bulk loaders pack densely when data is static.

That combination is rare among spatial structures. R-tree variants dominate every major spatial database in 2026.

---

## 3. MBRs — the core abstraction

A **minimum bounding rectangle** is the smallest axis-aligned rectangle that contains a geometry.

```
   ┌─────────────────┐
   │   ●             │
   │      ●  ●       │
   │  ●         ●    │
   │     ●           │
   └─────────────────┘
   MBR enclosing 5 points
```

For a complex polygon, the MBR is much coarser than the polygon itself. That's intentional — the tree is for **pruning**, not exact tests. After the tree narrows candidates, the engine runs the precise geometry check (point-in-polygon, distance, intersection) on the small surviving set.

This is the **filter then refine** pattern that PostGIS, Oracle Spatial, and every other engine follows:

1. **Filter** — tree returns candidates whose MBRs intersect the query (cheap).
2. **Refine** — exact geometry test on candidates (expensive, run on few).

---

## 4. Building an R-tree — inserts and splits

The basic Guttman algorithm:

### Insert

1. Start at the root.
2. At each level, pick the child whose MBR needs the **least enlargement** to include the new object.
3. Recurse to a leaf; add the object.
4. If the leaf overflows (more than M entries), **split** it.
5. Propagate the split upward, splitting parents if needed.

### Split — the hard part

When a node has M+1 entries, we split it into two. The challenge: produce two child MBRs that are *small* and *don't overlap much*. Different algorithms make this trade-off differently:

- **Linear split** (Guttman) — pick two "seed" entries far apart; assign remaining entries to whichever group's MBR enlarges least. Fast, mediocre quality.
- **Quadratic split** (Guttman) — pick seeds maximizing wasted area; smarter assignment. Slower, better.
- **R*-tree split** (Beckmann et al. 1990) — minimizes overlap + area + margin via more expensive heuristics. Slowest, **best query performance**.

Modern engines mostly use R*-tree or a close variant. The extra insertion cost is repaid many times over in faster queries.

### Forced reinsert (R*-tree's trick)

R*-tree adds: when a node overflows, instead of splitting immediately, **remove a fraction of its entries and reinsert them**. They often land in better-suited nodes, improving the tree's overall shape. Slightly more insert work; significantly better queries.

---

## 5. Queries — range, intersection, nearest neighbor

### Range query

"Find all entries whose MBR intersects this query rectangle."

```
def range_query(node, query_rect):
    results = []
    for entry in node.entries:
        if entry.mbr.intersects(query_rect):
            if node.is_leaf:
                results.append(entry.data)
            else:
                results.extend(range_query(entry.child, query_rect))
    return results
```

Complexity: O(log N + K) where K is hits, **assuming the tree has low overlap**. High overlap (poorly-built trees) degrades toward O(N).

### Nearest neighbor

Best-first search with a priority queue:

```
queue = [(0, root)]
while queue:
    dist, node = heappop(queue)
    if node.is_leaf_entry:
        return node.data   # closest!
    for entry in node.entries:
        heappush(queue, (min_dist(query_point, entry.mbr), entry))
```

`min_dist` is the minimum possible distance from the query point to any geometry in the subtree (distance to the MBR). The first leaf entry popped is the answer.

This is Hjaltason and Samet's "incremental nearest neighbor" algorithm, fast and exact for both points and arbitrary geometries.

### Point-in-polygon, distance-within

The tree filters to candidate geometries; the engine then runs exact tests (`ST_Contains`, `ST_DWithin`, `ST_Intersects`). The tree's job is pruning, not answering.

---

## 6. R-tree variants — the ones to know

| Variant | Idea | When to use |
|---|---|---|
| **Original R-tree** (Guttman, 1984) | Linear / quadratic split | Educational; rarely best in production |
| **R+-tree** | No overlap between siblings (split shared geometries) | Static data; reads dominate |
| **R*-tree** (Beckmann et al., 1990) | Smarter split + forced reinsert | The default for general workloads |
| **Hilbert R-tree** | Order entries by Hilbert curve before grouping | Better for skewed data; faster bulk insertion |
| **STR-packed R-tree** | Sort-Tile-Recursive bulk loader for static data | Bulk-loading large datasets; nearly perfectly packed |
| **Priority R-tree** | Worst-case-optimal query bounds | Theoretical; uncommon in production |
| **R-tree with z-order / S2** | Cluster entries by space-filling curve | Often used in distributed indexes |

In PostGIS, GiST builds an R*-tree variant by default. For bulk-loaded static data (a precomputed gazetteer, a building footprint database), an STR pack produces an unbeatable index.

---

## 7. R-tree in real systems

### PostGIS / GiST

The default spatial index in PostgreSQL/PostGIS is implemented via the **GiST** (Generalized Search Tree) extension framework, with R-tree-style split logic. Queries use the `&&` operator (bounding box overlap) under the hood; PostGIS rewrites distance and intersection queries to use it for filtering.

```sql
CREATE INDEX bldg_gix ON buildings USING GIST (geom);

EXPLAIN ANALYZE
SELECT name FROM buildings
WHERE ST_DWithin(geom, ST_MakePoint(-122.42, 37.77)::geography, 500);
-- uses bldg_gix to filter, then runs ST_DWithin on candidates
```

### SQLite R*Tree module

A built-in virtual table that maintains an in-database R*-tree:

```sql
CREATE VIRTUAL TABLE roads_rtree USING rtree(
  id, min_x, max_x, min_y, max_y
);
SELECT id FROM roads_rtree
WHERE min_x <= -122 AND max_x >= -122.5
  AND min_y >= 37.7 AND max_y <= 37.8;
```

Wonderfully simple; great for embedded apps (route planners, vehicle telematics).

### Oracle Spatial, MySQL Spatial, SQL Server

All ship R-tree variants under their geometry index types. APIs differ; mental model identical.

### Lucene / Elasticsearch

Modern Lucene moved from grid-based / quadtree indexes to **BKD-tree** (a Bkd dynamic KD-tree variant). Pragmatically: similar query semantics, different internals. For 2D, GIS-heavy work, dedicated PostGIS still tends to win.

### Cesium, Mapbox, Tile servers

Render tilesets often pre-index features in static R-trees for the tile-generation pipeline (STR-packed for read-only access). Standard infrastructure.

---

## 8. R-tree limitations

- **Overlap of sibling MBRs** is the dominant cost. High overlap = multiple subtrees to visit per query.
- **Update-heavy workloads** can leave the tree poorly shaped over time. Periodic re-bulk-load or compaction helps.
- **High-dimensional data** (>10D) hits the curse of dimensionality — every MBR covers nearly everything, no pruning. Use ANN structures (HNSW, IVF, see [Embedding-Based Retrieval →](./embedding-retrieval.md)) instead.
- **Non-rectangular pruning** isn't natural. Polygon queries still use MBRs for the filter step.

For 2D geographic workloads, none of this is a problem. R-trees are essentially perfect for that case.

---

## 9. STR bulk loading

When you have a static dataset (a country's roads, a country's buildings, a historical OD-pair matrix), don't insert one-by-one — **bulk-load**.

The **Sort-Tile-Recursive** (STR) algorithm:

1. Sort all N geometries by their centroid X.
2. Split into S vertical strips, where S = `ceil(sqrt(N / page_size))`.
3. Within each strip, sort by centroid Y.
4. Pack into leaves of size `page_size`.
5. Recursively build internal levels the same way.

Result: a near-perfectly packed R-tree, with minimal overlap and great query performance. Used by PostGIS' `gist__intbig_ops` for static data, and standalone tools.

For ~100M static geometries, STR-load takes minutes once; queries are then 2–4× faster than equivalent insert-built trees. Always prefer it for static data.

---

## 10. Common Mistakes / Anti-Patterns

- **Storing geometry as two columns (`min_x`, `max_x`, etc.) without a spatial index.** Range queries do full scans.
- **Indexing all points at maximum precision** when many queries are coarse. Hierarchical indexes (S2, H3) help; pure R-tree depth grows linearly with data.
- **Per-row inserts during a bulk load.** Use STR or temp-load + reindex.
- **Highly overlapping MBRs from skewed inserts.** Periodic VACUUM + REINDEX on PostGIS; or rebuild with STR.
- **Treating the R-tree as if it gave exact answers.** It gives candidates. The exact geometry check still runs.
- **Forgetting to use the right operator.** In PostGIS, `&&` triggers the index; some functions don't.
- **Mixing different SRIDs (spatial reference systems).** Index won't be usable across mismatched SRIDs.
- **Indexing huge polygons (continents, oceans) alongside tiny ones (buildings)** in the same tree. Big MBRs intersect almost every query → poor pruning. Consider splitting by feature class.
- **Updating a row's geometry causes index churn.** For mobile objects (Uber drivers), an in-memory R-tree with bulk re-build often beats per-update DB churn.
- **R-tree on >10 dimensions.** Use ANN (HNSW, IVF). See [Embedding-Based Retrieval →](./embedding-retrieval.md).
- **Building a custom R-tree.** Use PostGIS, JTS, Boost.Geometry, GeoTools — they're far better than what you'll roll.
- **Comparing R-tree query timings without considering cache hit rate.** First query is slow (cold pages); steady-state is fast.

---

## 11. Cheat Card

```
PURPOSE   Index 2D (or N-D, low N) geometries — points, lines,
          polygons — so range, nearest-neighbor, and intersection
          queries prune most of the data.

CORE
  Node holds N entries
  Each entry = (MBR, child) for internal node
  Each entry = (MBR, data) for leaf
  All children's MBRs are inside parent's MBR
  Balanced (like B-tree)

QUERY
  Range:   walk tree, recurse into subtrees whose MBR intersects query
  KNN:     priority queue by min_dist(query, MBR); first leaf = nearest
  Always:  tree filters, exact geometry test refines

VARIANTS TO KNOW
  R-tree              original; basic
  R*-tree             smarter split + forced reinsert (default in PostGIS)
  Hilbert R-tree      space-filling-curve ordering
  STR-packed R-tree   bulk-loader for static data — use when possible
  R+-tree             zero overlap (static, read-heavy)

BUILD MODES
  Per-insert      regular DDL; can drift over time
  Bulk STR-load   sort, tile, recurse — best for static data
  Periodic rebuild on update-heavy workloads

WHERE YOU MEET R-TREES
  PostGIS / GiST                 (default spatial index)
  SQLite R*Tree module
  Oracle Spatial / MySQL / MSSQL
  GeoTools / JTS / Boost.Geometry / Shapely (via libspatialindex)
  Tile servers and offline GIS pipelines

WHEN R-TREE LOSES
  >10 dimensions       → ANN / HNSW / IVF
  Massive overlap of big and tiny features → split index
  Pure point-only data → geohash / S2 / H3 may be simpler

PITFALLS
  Geometry as float columns, no index → full scan
  Per-row inserts on a static dataset (use STR)
  Mixing huge and tiny geometries in one tree
  Updating geometry frequently in a single shared R-tree
  Trusting MBR result as final (always refine)
  Mismatched SRIDs → index unused

RULE   Filter with MBR, refine with geometry. Use PostGIS' R-tree
       (GiST) by default. STR-pack static datasets. Don't roll
       your own.
```

---

## 12. Resources

### Papers
- "R-Trees: A Dynamic Index Structure for Spatial Searching" — Antonin Guttman, 1984. The original.
- "The R*-tree: An Efficient and Robust Access Method for Points and Rectangles" — Beckmann, Kriegel, Schneider, Seeger, 1990.
- "STR: A Simple and Efficient Algorithm for R-tree Packing" — Leutenegger, Lopez, Edgington, 1997.
- "Hilbert R-tree: An Improved R-tree Using Fractals" — Kamel, Faloutsos, 1994.

### Books
- *Foundations of Multidimensional and Metric Data Structures* — Hanan Samet. The encyclopedic reference.
- *Geographic Information Systems and Science* — Longley et al.

### Documentation
- **PostGIS GiST** — <https://postgis.net/docs/using_postgis_dbmanagement.html#indexing>
- **SQLite R*Tree** — <https://www.sqlite.org/rtree.html>
- **Boost.Geometry R-tree** — <https://www.boost.org/doc/libs/release/libs/geometry/doc/html/geometry/spatial_indexes.html>
- **JTS Topology Suite** — <https://github.com/locationtech/jts>

### Articles
- "How R-trees work" — Norbert Beckmann's interviews / blog posts.
- "Spatial indexing in PostgreSQL/PostGIS" — Paul Ramsey (PostGIS lead): <https://postgis.net/workshops/>
- "Indexing the Universe" — Foursquare engineering on H3 + R-tree combo.

### Videos
- *Spatial Indexing Deep Dive* — Paul Ramsey at PostgresOpen.
- *Hanan Samet on spatial data structures* — long-form talks.
- ByteByteGo — "R-tree Explained."

### Tools
- **PostGIS / GiST** — the default.
- **libspatialindex** — C++ R*-tree library (Python `rtree` wraps it).
- **JTS Topology Suite** (Java) / **GEOS** (C++) — used by everything in Java/Python GIS.
- **Boost.Geometry**.
- **Tippecanoe** — Mapbox's tool for building vector tiles; uses R-tree-style pre-indexing.

### Adjacent reading
- [Geohashing & Quadtrees →](./geohashing-quadtrees.md)
- [Trie Data Structure for Autocomplete →](./trie.md)
- [Embedding-Based Retrieval (ANN, HNSW, FAISS) →](./embedding-retrieval.md)
- [Database Indexing (B-Tree, Hash, LSM-Tree) →](../04-databases/indexing.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Search Engines (Elasticsearch, OpenSearch, Solr) →](../04-databases/search-engines.md)
- [Vector Databases →](../04-databases/vector-databases.md)

---

*Previous:* [← Geohashing & Quadtrees](./geohashing-quadtrees.md)  |  *Next:* [Trie Data Structure for Autocomplete →](./trie.md)
