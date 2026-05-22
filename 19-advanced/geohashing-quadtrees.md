# Geohashing & Quadtrees (Geo-spatial Indexing)

> **TL;DR** — Standard B-tree indexes are one-dimensional. Geographic queries are two-dimensional ("find every restaurant within 2 km of me") and don't reduce cleanly to a single sort order. **Geohashing** encodes lat/lon into a single string by interleaving binary digits — points near each other in space share long common prefixes, so range scans become "find all rows whose geohash starts with `dr5ru`." **Quadtrees** recursively subdivide a 2D plane into four quadrants until each leaf holds a bounded number of points; queries walk the tree, pruning quadrants that don't intersect the search region. Together with **R-trees** (covered separately) these are the data structures every map, ride-sharing, food-delivery, and proximity-search backend rests on. The honest take: **geohash strings are the right answer when you want a B-tree-friendly proxy for 2D location** (DynamoDB partition keys, Redis sorted sets, Cassandra primary keys). **Quadtrees / R-trees are the right answer for in-memory or specialized spatial engines** (PostGIS, S2, H3, modern in-memory indexes). Modern stacks usually combine both — **store a geohash for cheap sharding, query through S2 / H3 / PostGIS for correctness**.

---

## 1. The big picture

A geo query is a special kind of range query: "find every row whose location is *near* this one." Near means within a circle of radius R, or within a polygon, or within a bounding box.

```
  ┌───────────────────────┐
  │ ........+............ │   . = a point in the dataset
  │ .....+......+........ │   + = a hit (within 1km of o)
  │ ........+......+..... │   o = the query point
  │ .........o........... │
  │ ....+...+......+..... │
  │ ........+............ │   bounding box around o:
  │ +........+...+....... │   the query
  └───────────────────────┘
```

Brute force scans every point. At 10M points, that's slow. Geo-spatial indexes prune the search to a small fraction of the data.

Two families of techniques:

- **Space-filling curves and prefix encoding** — map 2D coordinates to 1D strings/numbers that preserve locality. Geohash, Hilbert curve, Z-order curve, **Google S2**, **Uber H3**.
- **Hierarchical spatial trees** — recursively partition the plane. Quadtrees, R-trees, KD-trees.

Both are used together in production. A typical Uber-style backend: bucket drivers by S2 cell ID (a hashed prefix), then within each cell run a finer-grained nearest-neighbor over an R-tree.

---

## 2. Geohash — interleave the bits

**Geohash** (Gustavo Niemeyer, 2008) is the simplest and most influential idea in this space. It turns a (lat, lon) pair into a base-32 string.

The algorithm:

1. Start with lat in `[-90, 90]`, lon in `[-180, 180]`.
2. Interleave bits: even bits encode lon, odd bits encode lat.
3. For each bit, narrow the range by half based on whether the value is in the upper or lower half.
4. Group every 5 bits into a base-32 character.

```
San Francisco: (37.7749, -122.4194)
            →  binary: 01101 00111 01111 00010 01000 11100 11011 00001 ...
            →  base32: 9q8yyk8...

Berlin:        (52.5200, 13.4050)
            →  binary: ...
            →  base32: u33d8z6...
```

The defining property: **points near each other in space share long common prefixes**. `9q8yy` covers a small region around San Francisco; `9q8` covers a larger region containing it.

| Prefix length | Approximate cell size |
|---|---|
| 1 char | ~5000 × 5000 km |
| 2 chars | ~1250 × 625 km |
| 3 chars | ~156 × 156 km |
| 4 chars | ~39 × 19 km |
| 5 chars | ~5 × 5 km |
| 6 chars | ~1.2 × 0.6 km |
| 7 chars | ~150 × 150 m |
| 8 chars | ~38 × 19 m |
| 12 chars | ~4 × 2 cm |

### Geohash queries

To find points near (37.7749, -122.4194):
1. Compute the query's geohash (e.g., `9q8yyk`).
2. Use a prefix scan: `WHERE geohash LIKE '9q8yyk%'`.
3. Inspect each returned point and check actual distance.

Step 3 matters because **not every point within distance R has a geohash prefix matching yours**. Two points can be 10 meters apart but cross a geohash cell boundary, putting them in different prefixes.

### The 8-neighbor trick

To handle cell boundaries: compute the geohashes of all **8 neighboring cells** plus the query's own cell, then scan all 9 prefixes. Almost every geohash library has a `neighbors()` function. After scanning, filter by actual distance to discard the corners of those 9 cells.

This is the standard recipe and it works well at moderate scale. Costs: 9 scans instead of 1.

### Why geohash is so popular

- **Sortable** — a B-tree, Redis sorted set, or DynamoDB sort key handles it natively.
- **Sharding-friendly** — partition by first N characters; geographically nearby data on the same shard.
- **Cache-friendly** — a single string is easy to put in URL paths, cache keys, logs.
- **Human-readable** — geohashes appear in geocaches, Twitter posts, OpenStreetMap notes.

### Why geohash is limited

- **Distortion near the poles and along the equator** — cells aren't squares; they're rectangles that stretch with latitude.
- **Boundary artifacts** — the 8-neighbor trick is a workaround.
- **Not a great proxy for true distance** — geohash similarity ≠ great-circle distance, especially at fine resolutions.

For most "find places near me" use cases, geohash is fine. For routing, ETAs, or anything that depends on accurate distance, use a more sophisticated system.

---

## 3. S2 and H3 — the modern successors

### Google S2

**S2** maps the sphere onto a cube, then onto a Hilbert curve over each face. Cells are nearly equal-area, irregular polygons (small spherical squares at low levels, rectangles at high latitudes for higher levels).

Properties:
- **Hierarchical** — each cell has 4 children, like a quadtree.
- **30 levels** — from 85,000,000 km² (whole face) down to 0.7 cm² at level 30.
- **Cell IDs are 64-bit integers** — sortable, cheap to index.
- **Coverage** — you can approximate any polygon as a union of S2 cells at varying levels. Powerful for "things within this irregular boundary."

Used by Google Maps, Foursquare, Uber (pre-H3), Pokémon Go, and many spatial workloads in BigQuery (which has native S2 functions).

### Uber H3

**H3** (Uber, 2018) divides the sphere into **hexagonal** cells. Hexagons are the only regular polygon that tiles the plane *and* has uniform neighbor distances — every hexagon has exactly 6 equidistant neighbors, unlike squares (4 close + 4 diagonal).

This matters for analytics: counting "drivers within 1 ring" or "events in this hex" produces clean, intuitive aggregations. H3 is what Uber's modern matching, mapping, and dispatch systems are built on.

H3 also has 16 resolutions (0–15) and 64-bit cell IDs. Resolution 9 hexagons are ~150m across — typical for urban data.

### When to use each

| | Geohash | S2 | H3 |
|---|---|---|---|
| Cell shape | Rectangles (distorted) | Quasi-squares | Hexagons (uniform neighbors) |
| Library quality | Many; basic | Excellent (C++, Go, Java) | Excellent (C, Python, JS) |
| Polygon coverage | Manual | Built-in | Built-in |
| Best for | Quick prefix indexing in any DB | Polygon queries, BigQuery | Aggregations, ride-sharing, mapping |
| ID type | String | int64 | int64 |

Default in 2026: **H3 for analytics and aggregations, S2 when you need irregular-polygon coverage, geohash when you just need a sortable string proxy in a generic DB**.

---

## 4. Quadtrees — the recursive subdivision

A **quadtree** stores 2D points by recursively splitting space into four equal quadrants until each leaf holds at most K points (or reaches a max depth).

```
            ┌─────────────┐
            │   root      │
            │   (whole    │
            │    map)     │
            └──┬───┬──┬───┘
              NW  NE  SE  SW
              ┌──────────────┐
              │ NW quadrant  │
              │ (subdivides  │
              │  further)    │
              └──────────────┘
```

A region containing many points subdivides; an empty region stays one leaf. The tree is naturally adaptive — dense urban areas get deep, sparse oceans stay shallow.

### Range query

To find all points in a bounding box:

```
def range_query(node, box):
    if not node.bounds.intersects(box):
        return []          # prune
    if node.is_leaf():
        return [p for p in node.points if box.contains(p)]
    return (range_query(node.NW, box)
          + range_query(node.NE, box)
          + range_query(node.SE, box)
          + range_query(node.SW, box))
```

Average complexity: O(log N + K) where K is the number of hits. Much better than O(N) brute force.

### Nearest-neighbor query

Search the quadrant containing the query point first; check if any other quadrant could contain a closer point (using best-so-far distance); prune the ones that can't.

### Variants

- **Point quadtree** — one point per leaf split.
- **Region quadtree** — splits whether or not points exist; cells are uniform.
- **PR (point-region) quadtree** — like above, but splits only when a cell has >1 point.
- **Compressed quadtree** — collapse chains of single-child nodes to save memory.
- **MX quadtree** — power-of-2 cell sizes; used in image processing.

For most use cases, a PR quadtree with a leaf threshold of 4–16 points is the practical default.

### Quadtree vs geohash — they're related

A quadtree's path-from-root-to-leaf is a sequence of `{NW, NE, SE, SW}` choices, which is exactly a base-4 encoding of position — and geohash is a base-32 encoding (5 bits at a time). **Geohash is a Morton-order (Z-order) curve over space; quadtrees walk the same hierarchy, with explicit pointers**.

The choice is implementation-level: a geohash gives you a B-tree-friendly string; a quadtree gives you in-memory pointers and easier dynamic insertion. Same idea, different machine model.

---

## 5. K-Dimensional Trees (KD-trees)

A **KD-tree** generalizes binary search trees to K dimensions. Each level alternates which dimension it splits on: level 0 splits on X, level 1 on Y, level 2 on X again, etc.

```
Splits at root:    X = 30
   Left subtree     Right subtree
   (X < 30)         (X >= 30)
```

KD-trees are great for **nearest-neighbor search on low-dimensional, static data**. They struggle when data is dynamic (rebalancing is expensive) and when dimensionality grows beyond ~10 (the curse of dimensionality — every leaf must be inspected).

For 2D geo data, quadtrees and R-trees are usually preferred. KD-trees still shine for ML feature spaces, computational geometry, and academic scenarios.

---

## 6. Worked example — proximity search at Uber-ish scale

Scenario: 5 million active drivers worldwide. Rider opens app at (37.7749, -122.4194). Find 50 nearest drivers in under 100 ms.

### Approach 1 — geohash + table scan

```sql
-- approximate
SELECT driver_id,
       ST_Distance(geom, ST_MakePoint(-122.4194, 37.7749)) AS d
FROM drivers
WHERE geohash_5 IN ('9q8yy', '9q8yu', '9q8yv', '9q8yz', '9q8yt',
                    '9q8yk', '9q8ym', '9q8yj', '9q8yn')  -- 9 cells
ORDER BY d ASC
LIMIT 50;
```

Cheap, generic, fits in any relational DB. Works well at city scale; weakens near cell boundaries.

### Approach 2 — H3 cells with ring expansion

```python
import h3
home = h3.geo_to_h3(37.7749, -122.4194, resolution=9)
neighborhood = h3.k_ring(home, k=2)  # ~5 km radius

candidates = drivers.filter(driver.h3_cell.isin(neighborhood))
return sorted(candidates, key=distance_to_rider)[:50]
```

Fewer cells to scan; uniform neighborhood semantics. The standard Uber pattern.

### Approach 3 — in-memory R-tree or quadtree

Keep all active drivers in an in-memory spatial index (e.g., `boost::geometry::index::rtree`, Java's JTS, Python `rtree` package, Go's `github.com/dhconnelly/rtreego`). Insert / update on each driver heartbeat. Query in O(log N + K).

Throughput: hundreds of thousands of queries per second per node. Used by dispatch services that pre-fetch active driver state into RAM.

### What real systems combine

- Geohash / H3 cell IDs as a database **partition key** so each shard handles a region.
- An in-memory **R-tree or quadtree** per shard for fast nearest-neighbor.
- PostGIS or Elasticsearch with **GeoJSON** as the durable index of record.

You rarely pick one. You build a tier.

---

## 7. PostGIS and Elasticsearch — the workhorses

For most teams that don't run their own spatial infrastructure:

### PostGIS

A Postgres extension that adds geometry types, spatial functions, and **GiST** / **SP-GiST** / **BRIN** indexes. The R-tree-equivalent index is built into GiST. PostGIS is the gold standard for transactional spatial workloads.

```sql
CREATE EXTENSION postgis;
CREATE TABLE drivers (id BIGINT PRIMARY KEY, location GEOGRAPHY(POINT, 4326));
CREATE INDEX drivers_loc_gix ON drivers USING GIST (location);

SELECT id
FROM drivers
WHERE ST_DWithin(location, ST_MakePoint(-122.4194, 37.7749)::geography, 5000)
ORDER BY location <-> ST_MakePoint(-122.4194, 37.7749)::geography
LIMIT 50;
```

The `<->` operator is "distance," and the GiST index supports **k-nearest-neighbor (KNN)** queries directly. PostGIS does the geohash-equivalent indexing under the hood.

### Elasticsearch / OpenSearch

Has `geo_point` and `geo_shape` field types, with **geohash precision tree** and **Bkd-tree** (a successor to KD-tree) indexes. Queries like `geo_distance`, `geo_bounding_box`, `geo_polygon`. The right choice when geo is part of a broader search experience.

### Redis

Redis's `GEO*` commands (GEOADD, GEORADIUS, GEOSEARCH) use a geohash internally on top of a sorted set. Fine for live state at modest scale; one of the fastest "where is everyone right now" backends if it fits in RAM.

### Specialized engines

- **Tile38** — Redis-compatible geo-fencing server.
- **Druid / Pinot / ClickHouse** — analytics over geo data with H3 / S2 helpers.
- **PostGIS + pg_partman** — partitioned spatial tables at very large scale.

---

## 8. Common Mistakes / Anti-Patterns

- **Storing lat/lon as separate `FLOAT` columns and querying with `WHERE lat BETWEEN ... AND lon BETWEEN ...`.** No spatial index used; full table scan.
- **Geohash without the 8-neighbor expansion.** Misses points near cell boundaries.
- **Geohash precision too coarse** for the query distance (returns everything) or too fine (returns nothing).
- **Treating geohash distance as real distance.** Two adjacent geohashes can be 5 km apart in one direction and 500 m in another.
- **Computing distances in lat/lon as if they were Cartesian.** Use Haversine, Vincenty, or PostGIS' built-ins. The error gets huge away from the equator.
- **Building a custom geo index when PostGIS would do.** PostGIS is mature, fast, and free.
- **Using Elasticsearch as the source of truth for live driver/rider locations.** ES indexes update on a refresh interval (seconds); for sub-second freshness, use Redis or an in-memory index.
- **Indexing every point at maximum precision in S2 / H3.** Cell counts explode. Pick a resolution suited to your query distances.
- **Quadtree implementations that don't bound depth.** A pathological cluster of points at the same location recurses infinitely.
- **No reindexing strategy for moving objects.** R-tree / quadtree updates need care; some implementations are slow on heavy update workloads.
- **Polygon queries against geohash prefixes.** Doesn't work cleanly; switch to S2 / H3 / PostGIS for polygon coverage.
- **Geohash as a primary key in a write-heavy hot-spot region** (e.g., everyone in Manhattan). Hot partition. Combine with another high-cardinality key or use a hash prefix.

---

## 9. Cheat Card

```
PURPOSE   Index 2D location data so "near me" queries are fast.

THREE FAMILIES
  Geohash / S2 / H3      1D encoding of 2D position
                         B-tree / hash friendly; sortable; partitionable
  Quadtree              recursive 4-way split of space; in-memory pointer tree
  R-tree                bounding-rectangle tree; see [R-Trees →](./r-trees.md)
  KD-tree               binary tree, alternates dims; good for low-D static data

GEOHASH IN BRIEF
  Interleave lat/lon bits, base-32 encode
  Prefix length → cell size (5 chars ≈ 5 km, 7 chars ≈ 150 m)
  Query: prefix scan + 8 neighbors + distance filter

S2 vs H3
  S2     quasi-square cells; polygon coverage; BigQuery native
  H3     hexagons; uniform neighbors; Uber-grade analytics

QUADTREE IN BRIEF
  Recursively split a region into 4 until leaf holds K points
  Range/NN queries prune by bounding-box intersection
  Z-order = geohash's curve = quadtree path

PRODUCTION STACK PATTERN
  H3/S2/geohash for sharding + cheap prefix queries
  PostGIS / R-tree / quadtree for distance and polygon queries
  Redis / in-memory index for live state
  Elasticsearch when search + geo combine

PITFALLS
  Geohash without 8 neighbors → boundary misses
  Treating geohash distance as real distance
  Lat/lon as Cartesian (must use Haversine / great circle)
  No spatial index → full table scan
  Custom geo when PostGIS / H3 already solve it
  Hot partitions in dense regions (Manhattan, Tokyo)

RULE   Encode for partitioning (H3/S2/geohash). Index for
       querying (PostGIS / R-tree / in-mem). Filter by real
       distance at the end. Don't roll your own.
```

---

## 10. Resources

### Books and papers
- "Geohash" — Niemeyer, 2008 (Wikipedia is sufficient): <https://en.wikipedia.org/wiki/Geohash>
- *Geographic Information Systems and Science* — Longley et al.
- "Quadtrees" — Finkel & Bentley, 1974 (the original paper).
- S2 design docs — <https://s2geometry.io/about/overview>
- H3 documentation and design papers — <https://h3geo.org/>

### Documentation
- **PostGIS** — <https://postgis.net/docs/>
- **Elasticsearch Geo Queries** — <https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-queries.html>
- **Redis Geo Commands** — <https://redis.io/commands/?group=geo>
- **Uber H3** — <https://h3geo.org>
- **Google S2** — <https://s2geometry.io>

### Articles
- "Geohashes and you" — geohash.org explainers.
- "H3: Uber's Hexagonal Hierarchical Geospatial Indexing" — Uber Engineering: <https://www.uber.com/blog/h3/>
- "Geo-distributed Cassandra" — DataStax blog.
- "Building a real-time geo index" — Foursquare / Yelp engineering blogs.

### Videos
- *H3: Geospatial Indexing System* — Uber Engineering YouTube.
- *Spatial Indexing in PostGIS* — Paul Ramsey talks.
- ByteByteGo — "Geospatial Indexing Explained."

### Tools
- **PostGIS**, **Spatialite** (SQLite spatial extension).
- **H3** libraries (C, Python, JS, Java, Go).
- **S2** libraries (C++, Go, Java).
- **`geohash`** libraries everywhere.
- **`rtree`** packages (Python, Go, Java JTS).
- **GeoPandas**, **Shapely**, **Turf.js**.

### Adjacent reading
- [R-Trees →](./r-trees.md)
- [Trie Data Structure for Autocomplete →](./trie.md)
- [Database Indexing →](../04-databases/indexing.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Hot Partition Problem →](../10-scalability/hot-partitions.md)
- [Consistent Hashing →](../04-databases/consistent-hashing.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [R-Trees →](./r-trees.md)
