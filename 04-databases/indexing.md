# Database Indexing (B-Tree, Hash, LSM-Tree)

> **TL;DR** — An **index** is an auxiliary data structure that makes lookups faster than a full table scan, at the cost of extra storage and slower writes. The three workhorse structures are **B-tree** (balanced trees with O(log N) range and equality, the default), **hash** (O(1) equality only), and **LSM-tree** (write-optimized, used by Cassandra/RocksDB/Bigtable). Postgres adds **GIN** (inverted index for JSONB / arrays / full-text), **GiST** (geometry, ranges), **BRIN** (block ranges, tiny indexes for sorted huge tables). Picking the right index — and *which columns* to index — is one of the highest-leverage skills in databases.

---

## 1. Why Indexes Exist

Without an index, a `WHERE email = '...'` against a billion-row table reads every row. **O(N) — disastrous at scale.**

An index stores keys in a sorted/structured form so the engine can jump straight to matching rows. **O(log N)** with B-tree, **O(1) average** with hash.

The cost:
- Extra disk space.
- Extra work on every write (the index must be updated).
- Risk of *over-indexing* — slowing writes, fattening backups, polluting the plan cache.

Indexing is a **trade-off between read speed and write cost**. Index what you query, not everything.

---

## 2. B-Tree (and B+Tree) — The Default

A **B-tree** is a self-balancing tree where every node holds many keys and many child pointers. Keys are kept sorted. The tree stays shallow (a few levels for billions of rows), so a lookup walks only a handful of pages.

```
          [ 50 | 100 | 200 ]
         /        |        \
     [10,30] [60,80,90] [120,150,180,250]
```

- **B+Tree** (the most common variant) stores **all keys in the leaves**, with leaves linked in a list. That's perfect for range scans.

### Operations
- Equality `=`: walk root → leaf, O(log N).
- Range `<`, `>`, `BETWEEN`: walk to first match, then scan linked leaves.
- Prefix `LIKE 'abc%'`: same as range.
- Sorted reads: leaves are already sorted.

### What B-trees are great at
- Most relational workloads — everything except very specialized cases.
- Sorted output (`ORDER BY` matching the index).
- Composite indexes (`(a, b, c)`) — match prefixes `a`, `a+b`, `a+b+c`.

### What they're not
- `LIKE '%abc%'` (suffix or middle match) — the index can't help.
- High-write workloads with random keys — every insert touches a random page (write amplification). LSM-trees do better here.

---

## 3. Hash Indexes — Pure Equality

A **hash index** maps `hash(key) → row pointer`. Equality lookup is O(1) average.

- **Cannot** do range queries (`<`, `>`, `BETWEEN`).
- **Cannot** support `ORDER BY`.
- Used inside engines (PostgreSQL has hash indexes; MySQL InnoDB has an in-memory adaptive hash; Memcached/Redis are essentially hash tables).

Use hash indexes when you only ever look up by exact key. In practice, B-trees handle equality almost as well and offer range queries on top, so explicit hash indexes are niche.

---

## 4. LSM-Trees — Write-Optimized

Used by **RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, Bigtable, MyRocks, MongoDB WiredTiger (somewhat), CockroachDB**.

```
Write path:
  1. WAL (append-only log on disk)              — durability
  2. Memtable (in-memory sorted map)             — hot writes
When memtable fills:
  3. Flush to disk as an immutable SSTable
Background:
  4. Compaction merges SSTables to reduce read amplification
```

```
On disk:
  Level 0:  recent SSTables (overlap allowed)
  Level 1:  larger sorted SSTables (non-overlapping)
  Level 2:  even larger ...
  ...
  Bloom filters per SSTable to skip files that can't contain a key.
```

### Trade-offs
- **Writes are sequential** → enormous throughput.
- **Reads** may touch multiple SSTables → bloom filters mitigate.
- **Compaction** runs in the background — read/write amplification is the cost.
- **Space amplification** during compaction can spike.

Knowing your compaction strategy (Size-Tiered, Leveled, Time-Window) is the LSM-tree DBA's bread and butter. See [Wide-Column Stores](./wide-column-stores.md).

---

## 5. The Specialized Index Family (Postgres)

Postgres ships with many index types beyond B-tree. Knowing them changes how you model.

### GIN — Generalized Inverted Index
For values that contain *many sub-values*: arrays, JSONB keys, full-text tokens.
```sql
CREATE INDEX users_tags_idx   ON users USING GIN (tags);              -- array
CREATE INDEX users_data_idx   ON users USING GIN (data);              -- jsonb
CREATE INDEX articles_tsv_idx ON articles USING GIN (to_tsvector('english', body));
```
Slow to update, very fast to query. The right tool for JSONB and FTS.

### GiST — Generalized Search Tree
Flexible spatial / range index. PostGIS uses it for geometry. Range types use it too.
```sql
CREATE INDEX places_geo_idx ON places USING GIST (location);   -- PostGIS point
CREATE INDEX events_range_idx ON events USING GIST (during);   -- tstzrange
```

### BRIN — Block Range Index
Tiny index that records min/max per *block* of the heap. Ideal for huge, time-sorted tables.
```sql
CREATE INDEX events_ts_brin ON events USING BRIN (created_at);
```
If the table is mostly sorted (e.g., append-only time-series), BRIN is megabytes vs a B-tree's gigabytes — at a small cost to selectivity.

### SP-GiST
Space-partitioned trees (quad-trees, k-d trees) for very irregular data — IP ranges, phone-number prefixes, geometry.

### Bloom (extension)
Multi-column index based on Bloom filters. Useful when you query many columns with `=` and don't know which.

### HNSW / IVF (pgvector)
Nearest-neighbor indexes over high-dimensional vectors. See [Vector Databases](./vector-databases.md).

---

## 6. Composite, Covering, Partial, Functional

### Composite (multi-column)
```sql
CREATE INDEX orders_user_status_idx ON orders (user_id, status, created_at);
```
Matches:
- `WHERE user_id = ?`
- `WHERE user_id = ? AND status = ?`
- `WHERE user_id = ? AND status = ? AND created_at > ?`

Does **not** efficiently match `WHERE status = ?` alone — the leftmost prefix rule applies.

### Covering / index-only scan
```sql
CREATE INDEX orders_covering_idx
  ON orders (user_id, status) INCLUDE (total, created_at);
```
If the query needs only the indexed + included columns, the engine can answer from the index alone (no heap fetch). Faster, less I/O.

### Partial
```sql
CREATE INDEX orders_pending_idx ON orders (created_at)
WHERE status = 'pending';
```
Index only matching rows. Smaller, faster maintenance, perfect for hot subsets.

### Functional / Expression
```sql
CREATE INDEX users_email_lower_idx ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'ada@example.com';
```
Index the *expression*, not the raw column.

### Unique
```sql
CREATE UNIQUE INDEX users_email_idx ON users (email);
```
Doubles as a constraint enforcement.

---

## 7. The Cost of an Index

- **Storage** — sometimes 10–50% of the table size each.
- **Write amplification** — every insert/update/delete touches every relevant index.
- **Maintenance** — `VACUUM`, `REINDEX`, autovacuum cost.
- **Plan-cache pressure** — more indexes = harder choices for the optimizer (rarely a big deal, but real).

A common pattern: an OLTP table accrues 8 "we might need this someday" indexes; writes degrade 30%. Audit indexes regularly.

---

## 8. How the Optimizer Uses Indexes

The optimizer picks among options:
- **Seq Scan** — read the whole table. Fastest when selectivity is poor or table is small.
- **Index Scan** — walk an index to find row pointers; fetch from heap.
- **Index-Only Scan** — answer from the index alone (when covered).
- **Bitmap Heap Scan** — combine multiple indexes into a bitmap of matching pages; then read.
- **Index Skip Scan** (some engines) — jump past distinct prefix values.

Decisions are driven by **statistics**. If statistics are stale, plans go wrong. Run `ANALYZE` (Postgres often auto-does this) and check `EXPLAIN ANALYZE`.

If the optimizer refuses your index, common causes:
- The index doesn't match the query (wrong leading columns, function applied to the column).
- Statistics suggest the seq scan is cheaper.
- The column type doesn't match the literal (silent cast disables index).
- The index isn't valid (still building, marked invalid).

---

## 9. EXPLAIN — Your One Required Tool

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... FROM ... WHERE ...;
```
Read it carefully:
- **Seq Scan vs Index Scan vs Bitmap Heap Scan.**
- **Estimated rows vs actual rows.** Big disagreement = stale stats or skewed data.
- **Buffers**: how many pages were read from cache vs disk.
- **Filter vs Index Cond**: a `Filter` after the scan means the index isn't doing the filtering.

Spend the 5 minutes reading the plan. The fix is usually one line.

---

## 10. Anti-Patterns

- **Indexing every column.** Writes suffer; bloat grows.
- **Wrong column order** in a composite index — putting the highly selective column second when the first is rarely filtered.
- **Function on the column** without a matching functional index: `WHERE LOWER(email) = ?` won't use `email`'s index.
- **Type mismatch**: `WHERE id = '42'` against an `int` column → silent cast → no index use.
- **Indexes on tiny tables**: not worth the maintenance.
- **`LIKE '%foo%'`** with hope: B-tree can't help; use trigram (`pg_trgm`) or FTS.
- **Foreign keys without an index** on the child side → cascade deletes lock-storm.
- **Reindex during peak traffic** — schedule online, use `CREATE INDEX CONCURRENTLY`.
- **Duplicate indexes**: `(a)` and `(a, b)` both exist — the lone `(a)` is redundant.
- **Indexes on volatile counters** that update every request — write hotspot.

---

## 11. Index Strategy Recipe

1. **List the top-N queries** by volume and latency.
2. For each, write the **ideal index** by hand from the `WHERE`, `ORDER BY`, and `JOIN` columns.
3. Combine indexes that share leading columns.
4. Drop indexes that are unused (Postgres: `pg_stat_user_indexes`; MySQL: `sys.schema_unused_indexes`).
5. Re-`ANALYZE`. Re-run `EXPLAIN`.

That cycle answers >90% of "the DB is slow" tickets.

---

## 12. Index in NoSQL Land

### DynamoDB
- **Primary index** = `(partition key, sort key)`.
- **LSI** (Local Secondary Index) — different sort key, same partition. Cheap, strongly consistent.
- **GSI** (Global Secondary Index) — different partition + sort key. Eventually consistent, costs extra capacity.

### MongoDB
- B-tree-style on any field, single or compound.
- Multi-key for arrays, text for full-text, 2dsphere for geo, partial, TTL, wildcard.

### Cassandra
- Primary key = partition + clustering. Per-partition sorted.
- Secondary indexes exist but are **local per node** and don't scale well — prefer **materialized views** or denormalized tables for query patterns.

### Elasticsearch / OpenSearch
- Inverted index per field, plus **doc values** (column-store sidecar) for aggregations and sorting.

### Redis
- The data structures *are* the index. Use sorted sets for leaderboards, hashes for fast field lookup, etc.

---

## 13. Common Performance Diagnoses by Index

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Slow `=` lookup | No B-tree on column | Add it |
| Slow range / sort | Index column order wrong | Reorder index |
| Slow `LIKE '%x%'` | B-tree can't help | trigram / FTS / search engine |
| Slow JOIN | No index on FK | Add it |
| Slow `ORDER BY` | Index sort doesn't match query | Match index sort or add it |
| Bloated tables | Heavy updates + no autovacuum | Tune autovacuum, REINDEX |
| Tail latency on writes | Many indexes per write | Drop unused, batch writes |
| Some queries fast, others slow | Plan switches based on parameter | Plan-cache investigation, `EXPLAIN` per param |

---

## 14. Index in NewSQL (Spanner, CockroachDB, TiDB)

- Same B-tree-like primary keys, but **sharded** (split into ranges).
- Sequential keys (`AUTO_INCREMENT`) create **hot ranges** — one shard absorbs all writes. **Use UUIDs / hash-sharded keys** instead, or explicit `HASH SHARDED` syntax in CockroachDB.
- Composite, partial, expression, and inverted indexes also exist.
- Secondary indexes are real B-trees, replicated independently.

The hot-range problem is the biggest difference vs single-leader Postgres — a fact every NewSQL beginner re-learns at 2 AM.

---

## 15. Cheat Card

```
B-TREE      default. equality + range + sort. composite by leading cols.
HASH        equality only. niche.
LSM-TREE    write-heavy stores (Cassandra, Rocks, Bigtable). bloom filters help reads.

POSTGRES SPECIALS
  GIN   inverted: JSONB keys, arrays, full-text
  GiST  ranges, geometry (PostGIS)
  BRIN  block-range; tiny indexes for huge sorted tables (time-series)
  SP-GiST  partitioned trees (IPs, prefixes)
  HNSW/IVF (pgvector) — ANN over embeddings

VARIANTS
  composite    (a, b, c)   leftmost prefix matches
  covering     INCLUDE     answer from index alone
  partial      WHERE …     index a hot subset
  expression   lower(email) match expression
  unique       enforces constraint

RULES
  index what you query — not "just in case".
  watch composite column order (selectivity first).
  beware functions and casts hiding the column.
  avoid `LIKE '%x%'` on B-trees (use trigram / FTS).
  FKs need indexes on the child side.
  always `EXPLAIN ANALYZE`.

NoSQL EQUIVALENTS
  Dynamo: LSI / GSI         Cassandra: materialized views / denorm tables
  Mongo: many index types    Redis: data structures = indexes
  Elastic: inverted + doc values

OPS
  CREATE INDEX CONCURRENTLY in PG to avoid table locks.
  drop unused indexes — they slow writes.
  ANALYZE keeps stats fresh; bad stats → bad plans.
```

---

## 16. Resources

### Books
- *SQL Performance Explained* — Markus Winand. The single best book on indexing.
- *Use the Index, Luke!* — Markus Winand (free online): <https://use-the-index-luke.com/>
- *Database Internals* — Alex Petrov. B-trees, LSM-trees, all the data structures.
- *PostgreSQL Internals* — Hironobu Suzuki (free): <https://www.interdb.jp/pg/>
- *Designing Data-Intensive Applications* — Kleppmann; Ch. 3 (storage engines).

### Documentation
- **Postgres index types** — <https://www.postgresql.org/docs/current/indexes-types.html>
- **MySQL index reference** — <https://dev.mysql.com/doc/refman/en/optimization-indexes.html>
- **Oracle index guide** — <https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-INDEX.html>
- **RocksDB Wiki** — LSM-tree behavior in depth.

### Articles
- "Index Internals" — Andy Pavlo CMU lecture slides (15-445 / 15-721): <https://15445.courses.cs.cmu.edu/>
- "BRIN, GIN, GiST: When to use what" — many Postgres blog posts.
- "Hot ranges and you" — CockroachDB blog.
- "LSM-tree compaction strategies" — ScyllaDB blog.

### Videos
- Hussein Nasser: indexing deep dives — <https://www.youtube.com/@hnasr>
- Markus Winand conference talks on YouTube.
- ByteByteGo: "How database indexes work" — <https://www.youtube.com/@ByteByteGo>

### Tools
- `EXPLAIN ANALYZE` (Postgres) / `EXPLAIN FORMAT=JSON` (MySQL).
- `pg_stat_user_indexes`, `pg_stat_statements`.
- `pgbadger`, `pganalyze`, `pgmustard`.
- **HypoPG** — try out hypothetical indexes without creating them.

### Adjacent reading
- [Relational Databases Deep Dive](./relational-databases.md)
- [MVCC](./mvcc.md)
- [Concurrency Control](./concurrency-control.md)
- [Wide-Column Stores](./wide-column-stores.md) (LSM-trees)
- [Search Engines](./search-engines.md) (inverted indexes)

---

*Previous:* [← ACID vs BASE](./acid-vs-base.md)  |  *Next:* [Database Normalization & Denormalization →](./normalization.md)
