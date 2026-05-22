# N+1 Query Problem

> **TL;DR** — The **N+1 query problem** is when a piece of code issues **one query** to fetch a list of N items, then **one extra query per item** to fetch related data — for a total of **N+1 round trips** instead of 1 or 2. It is the single most common database performance bug in business applications. It often looks innocuous in code (`for user in users: print(user.orders.count())`) and only reveals itself under load, in long traces, or when N grows. The cure is almost always one of three things: **eager loading**, **a join**, or **a batch fetch (DataLoader-style)**. The lesson behind the lesson: **every iteration in your application code that triggers I/O is suspect**. Look for it in ORMs, GraphQL resolvers, REST endpoints that return lists, and template engines that lazy-load fields during render.

---

## 1. The big picture

A list page that should be one query:

```python
# 1 query: fetch authors
authors = Author.objects.all()    # SELECT * FROM authors LIMIT 50

for a in authors:
    # N more queries: one per author
    print(a.name, a.books.count()) # SELECT COUNT(*) FROM books WHERE author_id = ?
```

50 authors → **51 queries**. Each round-trip costs ~1ms on a fast LAN, more across regions, more under load. A 50ms page becomes a 500ms page. Apply it to the homepage on a popular site, and you've recreated the most common database overload pattern in the industry.

The same shape appears everywhere:
- **REST endpoint** returning a list of objects with nested relationships.
- **GraphQL resolver** that walks `posts → author → posts → author → ...`.
- **Server-side template** iterating a list and calling helper methods that hit the DB.
- **ORM lazy loading**: accessing `user.profile` issues a query if not preloaded.
- **Even microservices**: a service that returns a list of orders, calls user-service once per order to enrich.

The pattern is general: **a loop, and inside it, a network/I/O call**.

---

## 2. Where N+1 actually comes from

### ORM lazy loading
Every major ORM has a "load related objects when you access them" mode. It's a convenience that quietly fans out queries.

- **Django**: `author.books.all()` is lazy.
- **SQLAlchemy**: `relationship(..., lazy='select')` triggers a query on access.
- **Hibernate / JPA**: `FetchType.LAZY` proxies, fires on first dereference.
- **ActiveRecord**: `user.posts` lazy by default.

### GraphQL resolvers
Schema:
```graphql
type Author { id: ID, name: String, books: [Book] }
type Book   { id: ID, title: String, author: Author }
type Query  { authors: [Author] }
```

Resolvers for `Author.books` get called once *per author* — N+1 unless you add a DataLoader.

### Microservice fan-out
```
GET /orders
  → returns 50 orders
    for each order:
      GET /users/:id        ← 50 cross-service calls
```

Same pattern at a coarser layer.

### Templates and views
```html
{% for user in users %}
  {{ user.profile.avatar_url }}   ← lazy hit per user
{% endfor %}
```

### Caching that misses uniformly
Even with a cache, the first request after a flush hits N+1 underneath.

---

## 3. The math — why N+1 hurts so much

For a single request:

```
total_time = base_query_time + N × (per_query_overhead + per_query_work)
```

`per_query_overhead` includes round-trip latency, connection acquisition, parse/plan time (small but not free), and the cost of context switching back to your app. Even with the actual SQL taking 100 µs each, **the round-trips dominate**:

```
1 query returning 50 rows  ≈  1 RTT  +  1 SQL  = ~2 ms
51 queries                  ≈  51 RTTs + 51 SQL = ~80–500 ms
```

At scale, N+1 inflates DB connection usage (each request holds a pool slot longer), increases the chance of pool exhaustion, and creates tail latency that synthetic load tests miss — only the slow-tail user with N=100 sees the worst of it.

This is also why N+1 is so often the answer to "the homepage is slow." It scales linearly with whatever list you show.

---

## 4. The cures

There are essentially three:

### 4.1 Eager loading (preload / prefetch / fetchJoin)

Tell the ORM up front: "I want these relationships." The ORM emits one or two queries instead of N+1.

**Django**:
```python
# Bad:    51 queries
authors = Author.objects.all()
# Good:    2 queries (one for authors, one IN-fetch for books)
authors = Author.objects.prefetch_related('books')
```

`select_related` does a `JOIN`; `prefetch_related` does a separate IN-list query — different trade-offs:

| Strategy | Queries | When |
|---|---|---|
| `select_related` / `fetchJoin` | 1 | One-to-one / many-to-one (no row explosion) |
| `prefetch_related` / `subselect` | 2 | One-to-many / many-to-many |

**SQLAlchemy**: `joinedload(Author.books)`, `selectinload(Author.books)`.
**ActiveRecord**: `Author.includes(:books)`.
**Hibernate**: `JOIN FETCH` in JPQL, or `@EntityGraph`.

### 4.2 Plain SQL JOIN with aggregation

When you want a single query and you control it:

```sql
SELECT a.id, a.name, COUNT(b.id) AS book_count
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
GROUP BY a.id, a.name;
```

One round trip, one result set. Best when:
- The relationship is small per row (no million-row blow-up).
- The data fits the shape you want directly.
- You don't need to hydrate full ORM objects.

When NOT to JOIN: large one-to-many relationships create **row explosion** — 100 authors × 50 books each = 5000 rows shipped per row. Two queries with IN-lists are cheaper than one giant JOIN.

### 4.3 Batch fetch with DataLoader

For GraphQL and any code where you can't easily eager-load (you don't know in advance what's needed), use **DataLoader** (the original is Facebook's; every language has a port).

```javascript
const userLoader = new DataLoader(async (ids) => {
  const users = await db('users').whereIn('id', ids);
  // Return in the same order as ids
  return ids.map(id => users.find(u => u.id === id));
});

// Resolver code:
function authorOf(book, _, ctx) {
  return ctx.userLoader.load(book.author_id);   // batched, cached per request
}
```

DataLoader batches loads within a single tick of the event loop: if 50 resolvers call `load(id)` for 50 different IDs, you get **one** SQL query for all 50, plus per-request caching so the same ID isn't fetched twice. This is the de facto solution to GraphQL's natural N+1 tendency.

Equivalents:
- **graphql-batchloader**, **graphql-java DataLoader** — JVM.
- **graphene-django** `prefetch_related` integration.
- **Hot Chocolate `IDataLoader`** — .NET.
- **dataloader-rs** — Rust.

### 4.4 Cache the loop, not the loaded thing

If the related data rarely changes, cache it. The N+1 becomes N+1 cache hits, which is much faster — though still inferior to a single SQL query.

This is a partial fix. Use it when the data is read-heavy and infrequently updated.

---

## 5. JOIN explosion — the other failure mode

The naive cure ("just JOIN everything") creates its own problem. Consider:

```sql
SELECT a.*, b.*, r.*
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
LEFT JOIN reviews r ON r.book_id = b.id
WHERE a.id = 42;
```

If author 42 has 50 books with 100 reviews each, this query returns **5000 rows** — most of them duplicating the author's fields. The DB shipped megabytes when you wanted kilobytes.

The right shape is usually:
1. One query for the author.
2. One query for the books (`WHERE author_id = 42`).
3. One query for the reviews (`WHERE book_id IN (...)`).

This is exactly what `prefetch_related` / `selectinload` / DataLoader does. Three queries, not 5001 rows.

**Rule of thumb**: JOIN only when the cardinality stays bounded. For "list a parent with many children, each having many grandchildren," prefer multiple IN-list queries.

---

## 6. Detection — finding N+1 before it ships

### Tools and habits

- **ORM "explain" mode**:
  - Django Debug Toolbar shows every query per request.
  - SQLAlchemy `echo=True` for dev.
  - ActiveRecord `Bullet` gem warns automatically.
  - Hibernate statistics + `n_plus_one_query_detection_enabled`.
- **APM**: Datadog, New Relic, Honeycomb, Sentry — span counts per request. A request with 100 spans of the same DB call is N+1.
- **Tracing**: OpenTelemetry per-DB-call spans. Visualize on flame graph; "wall of stacked DB calls" is N+1.
- **Slow query log + count by template**: Postgres `pg_stat_statements` groups by query shape. A query with 1M calls and 100 ms total time is suspicious.
- **Tests**: assert that a request runs N queries, not N+50.

```python
# Django test
with self.assertNumQueries(2):
    list(Author.objects.prefetch_related('books'))
```

Add these assertions to important endpoints. They catch regressions when someone introduces a lazy attribute that wasn't there before.

### Red flags in code review

- A loop that accesses a `.something` on each item where `something` is a related object.
- A list-returning endpoint with no `.includes` / `.prefetch_related` / DataLoader nearby.
- A GraphQL resolver that does direct DB access (not a loader).
- A microservice that loops over a list and calls another service per item.
- A template that calls `obj.helper_that_hits_db()` inside a loop.

---

## 7. The microservice version

N+1 doesn't end at the database. Replace "ORM call" with "HTTP/RPC call":

```python
# Bad
orders = order_service.list_orders(user_id)
for o in orders:
    o.user = user_service.get_user(o.user_id)   # N RPCs
```

Same cure shapes:
- **Batch endpoints**: `user_service.get_users(ids=[...])`.
- **DataLoader-style batching** across an event loop tick.
- **Joined responses**: a single composite endpoint (BFF) that does the joins internally.
- **gRPC streaming**: server streams batched responses.

A microservice architecture without batch endpoints will recreate N+1 at the network layer — usually slower than the DB version (10+ ms per call instead of 1).

---

## 8. Worked example — N+1 to "1.5 queries"

A blog homepage shows 20 posts with author names and tag lists.

**Naive**:
```python
posts = Post.objects.order_by('-created_at')[:20]   # 1 query
for p in posts:
    p.author.name                                    # 20 queries
    [t.name for t in p.tags.all()]                  # 20 queries
                                                     # = 41 queries
```

**Eager-loaded**:
```python
posts = (
    Post.objects
    .order_by('-created_at')
    .select_related('author')         # JOIN authors (1-to-1 with each post)
    .prefetch_related('tags')         # IN-fetch tags (many-to-many)
    [:20]
)
# 2 queries, regardless of N
```

Two queries, ~5ms total instead of 41×~2ms = 80 ms. The fix is one line of code.

---

## 9. ORM, raw SQL, and the productivity / discipline trade

ORMs make queries cheap to write and expensive to think about. Almost every N+1 is "the ORM was helpful by default."

A working strategy:

- **Treat ORM laziness as the enemy**. Default to eager loading. Many teams configure their ORM to log when lazy loading fires in production.
- **Use the ORM for CRUD; drop to raw SQL for list pages and reports.** That's the boundary where ORMs cost more than they save.
- **Profile every list-returning endpoint** at least once during development. APM traces, query counters, Bullet-style warnings.
- **GraphQL ⇒ DataLoader, always.** Don't write a GraphQL backend without it.

There's no shame in raw SQL. There's plenty of shame in 200 queries to render a page.

---

## 10. Common Mistakes / Anti-Patterns

- **Lazy loading on by default.** Every nested access becomes a DB call. Configure for eager-by-default and override per case.
- **`select_related`/`includes` on everything.** Over-prefetch creates wasted I/O and row explosion. Eager-load what the view actually uses.
- **JOIN-ing a one-to-many with thousands of children.** Row explosion. Use two queries (IN-list).
- **DataLoader not scoped per-request.** Caches outlive the request, leaking data across users.
- **DataLoader returning results in the wrong order.** Subtle bug; always sort outputs to match input IDs.
- **Microservice "list" endpoints with no batch counterpart.** N+1 by other means.
- **`prefetch_related` chained but rendering still touches uncovered relationships.** Fix by tracing or assertion.
- **Counting children with `.count()` in a loop.** That's a `COUNT(*)` per parent. Use `Count` in the parent query: `annotate(book_count=Count('books'))`.
- **No `assertNumQueries`-style tests.** Regressions ship silently.
- **`pg_stat_statements` ignored.** "List queries" with millions of calls hiding in plain sight.
- **GraphQL with raw resolvers and "we'll fix N+1 later."** Later doesn't come.
- **Adding caches around N+1 instead of fixing it.** Now you have N+1 cache calls and a cache invalidation problem.
- **Pagination that triggers N+1 per page.** Combine pagination with eager loading; otherwise each page is its own fan-out.
- **`for user in users: send_email(user)` triggering a sync HTTP call per user.** N+1 at the email-provider boundary; use batched APIs or a queue.

---

## 11. Cheat Card

```
PURPOSE   Avoid the "1 + N" round-trip pattern: 1 query for a
          list, then 1 per item to load related data.

SIGNATURE
  Loop in app code → I/O inside the loop → many trips
  Common in ORMs, GraphQL, microservice fan-outs, templates

FIXES (PICK ONE PER CASE)
  Eager load     Django prefetch_related / select_related
                 SQLAlchemy selectinload / joinedload
                 ActiveRecord includes
                 Hibernate JOIN FETCH / @EntityGraph
  Batch fetch    DataLoader (GraphQL gold standard)
                 IN-list query per related table
  Single SQL     LEFT JOIN + GROUP BY when cardinality is small
  Aggregate inline   annotate(count=Count('children')) etc.

WHEN TO PREFER
  1-to-1, M-to-1            → JOIN (select_related)
  1-to-M, M-to-M            → IN-list (prefetch_related / selectinload / DataLoader)
  Big cardinality children  → never one giant JOIN — IN-list

DETECTION
  Django Debug Toolbar / Bullet gem / Hibernate stats
  APM: count DB spans per request
  Slow query log + pg_stat_statements grouped by template
  Test: assertNumQueries(<=N) on important endpoints

MICROSERVICE VARIANT
  Same shape with HTTP/RPC; cure with batch endpoints
  or per-request DataLoader

PITFALLS
  Lazy loading on by default in production
  Caching N+1 instead of fixing it
  Over-eager prefetch → row explosion
  DataLoader cache outlives the request
  count() in a loop instead of annotate()
  GraphQL resolvers hitting DB directly with no loader

RULE   If your loop touches related data, the related data
       should already be loaded — by JOIN, prefetch, or batch.
```

---

## 12. Resources

### Articles
- "GraphQL & DataLoader" — original Facebook engineering write-up.
- "What is the N+1 problem?" — Hibernate / SQLAlchemy / Django docs all have great explanations.
- "Bullet — kill your N+1 queries" — Ruby/Rails gem README.
- "Markus Winand on indexing and query patterns" — <https://use-the-index-luke.com>
- "Online migrations" and "API ergonomics" — Stripe engineering blog (good adjacent reading).

### Documentation
- **Django ORM** — `select_related` / `prefetch_related`: <https://docs.djangoproject.com/en/stable/ref/models/querysets/>
- **SQLAlchemy** — Relationship Loading Techniques: <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html>
- **ActiveRecord** — Eager Loading Associations: <https://guides.rubyonrails.org/active_record_querying.html>
- **Hibernate** — `JOIN FETCH`, `@EntityGraph`.
- **GraphQL DataLoader** — <https://github.com/graphql/dataloader>
- **Bullet gem** — <https://github.com/flyerhzm/bullet>

### Videos
- *N+1 Query Problem in 5 minutes* — many short tutorials.
- *GraphQL batching with DataLoader* — Apollo / GraphQL Foundation talks.
- ByteByteGo — "N+1 Query Problem Explained."

### Tools
- **Django Debug Toolbar**, **silk**.
- **Bullet** (Rails).
- **Hibernate Statistics**, **JFR**.
- **pg_stat_statements**, **pganalyze**.
- **Datadog APM**, **New Relic**, **Honeycomb**, **Sentry Performance**.
- **OpenTelemetry** spans on every DB call.

### Adjacent reading
- [Profiling & Benchmarking →](./profiling.md)
- [Connection Pooling & Keep-Alive →](./connection-pooling.md)
- [Batching & Debouncing →](./batching-debouncing.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Database Indexing →](../04-databases/indexing.md)
- [REST vs GraphQL vs gRPC →](../02-networking/api-styles.md)
- [GraphQL Fundamentals →](../03-apis/graphql.md)
- [BFF — Backend for Frontend →](../03-apis/bff.md)
- [Distributed Tracing →](../13-observability/tracing.md)

---

*Previous:* [← Serialization Formats](./serialization.md)  |  *Next:* [Batching & Debouncing →](./batching-debouncing.md)
