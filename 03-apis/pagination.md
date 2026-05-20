# API Pagination Techniques

> **TL;DR** — Returning millions of rows in one response is a bad idea. **Pagination** is how APIs serve large collections in pieces. Three main techniques: **offset/limit** (simple but slow and unstable at scale), **cursor/keyset** (fast and stable — the modern default), and **page tokens** (opaque cursors, à la Google / Relay). Always cap the page size, expose total counts only when cheap, and prefer cursors for anything backed by a real database.

---

## 1. Why Pagination Exists

A list endpoint without pagination is a ticking bomb:
- **Memory** — loading 10M rows blows your server.
- **Bandwidth** — clients don't need them all.
- **Latency** — clients wait minutes for the full payload.
- **DB load** — a single endpoint becomes a hot path.
- **Cost** — egress, CPU, all multiplied.

Pagination converts an unbounded read into a *bounded, repeatable* one. Every list endpoint must paginate.

---

## 2. The Three Main Techniques

### 2.1 Offset / Limit (a.k.a. Page Number)

```
GET /users?offset=0&limit=20
GET /users?offset=20&limit=20
GET /users?offset=40&limit=20
```

or
```
GET /users?page=1&limit=20
```

**SQL underneath:**
```sql
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 40;
```

**Pros**
- Trivial to understand and implement.
- Easy to jump to any page ("page 47").
- UI-friendly — "previous / next / 1 2 3 ... 99".

**Cons (the killers)**
- **Performance**: `OFFSET 1000000` makes the DB walk 1M rows and discard them. Latency grows linearly with depth.
- **Instability**: under concurrent inserts/deletes, items shift across pages. A user paginating sees duplicates or skips records.
- **No natural "since" semantics** — hard to express "give me everything new since I last polled."

**Use when**: small or static lists, admin tools, internal pages, search results capped at < ~1000 items.

### 2.2 Cursor / Keyset Pagination

```
GET /events?limit=50
   → returns 50 items + next_cursor
GET /events?limit=50&after=evt_abc123
   → returns 50 items starting AFTER evt_abc123
```

**SQL underneath:**
```sql
SELECT * FROM events
WHERE (created_at, id) > ('2026-05-19T10:00:00Z', 'evt_abc123')
ORDER BY created_at, id
LIMIT 50;
```

The cursor is whatever uniquely identifies a row in the sort order. For most "newest first" feeds, that's `(timestamp, id)`.

**Pros**
- **O(1)** at any depth — a sorted index makes "next 50 after X" a B-tree seek.
- **Stable** — inserts/deletes don't shift other pages.
- Natural fit for "give me everything since the last sync".
- Friendly to infinite-scroll UIs.

**Cons**
- Can't jump to page N — only forward / backward sequentially.
- Backward pagination needs a separate reversed cursor.
- Implementation slightly more involved.

**Use when**: anything backed by a real DB, especially activity feeds, message streams, search results > a few hundred items, public APIs.

### 2.3 Page Tokens (Opaque Cursors)

A variant of cursor pagination where the cursor is **opaque** (base64-encoded JSON, signed):

```
GET /items?page_token=eyJsYXN0X2lkIjoiYWJjMTIzIiwic29ydCI6Im5hbWUifQ
   → returns items + next_page_token
```

Inside the token you may encode `last_id`, `sort_order`, `filter_hash`, even a signature.

**Pros**
- The client treats the token as a black box. You can change the encoding without a client update.
- Carries page-state safely (sort, filter, version) so different queries don't collide.
- Tamper-proof if signed/encrypted.

**Cons**
- A bit more work server-side.
- Tokens look ugly in URLs.

**Use when**: public APIs, Google-style design, anywhere you want maximum flexibility to change pagination internals later.

This is the canonical pattern in:
- **Google Cloud APIs** (`page_size`, `page_token`).
- **AWS APIs** (`NextToken`).
- **Relay GraphQL connection** (`edges`, `pageInfo`, `endCursor`).

---

## 3. Cursor Pagination — The Right Way to Build It

### Pick the cursor field carefully
The cursor key must be:
- **Unique** — two rows must never share the same cursor.
- **Indexed** — your DB must seek by it cheaply.
- **Monotonic per page** — sorted order matches cursor comparison.

For most timeline-style queries, `(created_at, id)` works. If two rows share a timestamp, the `id` breaks the tie.

```sql
CREATE INDEX events_created_id_idx ON events (created_at, id);
```

### Encode the cursor
A safe encoding:
```json
{"created_at": "2026-05-19T10:00:00Z", "id": "evt_abc123", "v": 1}
```
Base64 it. Optionally sign it. Don't expose the raw DB primary key as a string — too easy to leak DB internals.

### The query
```sql
SELECT * FROM events
WHERE (created_at, id) > ($cursor_ts, $cursor_id)
ORDER BY created_at, id
LIMIT $limit + 1;        -- ask for one extra to detect "has more"
```

### Response shape
```json
{
  "data": [ ... 50 items ... ],
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi4uLiJ9",
  "has_more": true
}
```

If `has_more = false`, the client stops.

### Bidirectional
If you need *previous* page:
```
GET /events?limit=50&before=evt_xyz
```
Same query reversed:
```sql
WHERE (created_at, id) < ($cursor_ts, $cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT $limit;
```
Then reverse the resulting array before returning.

---

## 4. The Relay GraphQL Connection Pattern

A standard pagination shape, useful even for REST:

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

Query:
```graphql
query {
  users(first: 20, after: "abc") {
    edges {
      node { id name }
      cursor
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

Strengths:
- Cursor lives on each edge — you can branch the pagination from any point.
- `pageInfo` cleanly tells the client to stop.
- A single conventional shape across all paginated collections.

Worth copying in REST APIs too, minus the GraphQL syntax.

---

## 5. Page Size — Always Cap It

Common defaults: 20 default, 100 max. Stripe uses 10/100, GitHub 30/100, Google 100/1000.

```
GET /items?limit=99999
   → either silently capped at 100, or returns 400 Bad Request
```

Document the cap. Return the actual `limit` used in the response so the client knows.

---

## 6. Total Counts — Be Careful

Clients often want "X of Y results". On a large table, `COUNT(*)` is expensive. Options:

- **Don't return totals** unless the client explicitly asks (`?include_total=true`) — Twitter / GitHub style.
- **Return approximate totals** (e.g., from `pg_class.reltuples` or `EXPLAIN`). Faster, less precise.
- **Return totals only for small filtered results** (`WHERE` clauses limit cardinality).
- **Cache totals** per-filter for short windows.

Don't make every list endpoint pay for a `COUNT(*)`.

---

## 7. Stable Sorting Is Non-Negotiable

Cursors only work if rows have a **total order**. Always:

- Sort by your **primary index column**, OR
- Sort by a column + the row ID as tiebreaker.

```sql
ORDER BY created_at DESC, id DESC
```

Otherwise items with the same `created_at` may shuffle and the cursor will skip or duplicate them.

---

## 8. Pagination With Filtering

When users apply filters (search, tags), the cursor must **commit to the filter** — encode the filter or its hash inside the cursor.

```json
{ "filter_hash": "f8a91e...", "cursor": { "id": "evt_xyz" } }
```

If the filter changes between requests, treat the cursor as stale and restart from the top. Or reject the request with `400 Bad Request, "filter changed, restart"`.

---

## 9. Pagination With Real-Time Inserts

New rows being inserted while paginating is normal. Cursor pagination handles this gracefully — new rows appear at the *top*, not in the middle of pages.

For activity feeds, the API often exposes:

```
GET /events                       ← newest at top, cursor backward into history
GET /events?since=evt_abc         ← new events since I last looked
```

`since=` paginates **forward in time** for catching up; `after=` paginates **backward in time** for browsing history. Different cursors, same idea.

---

## 10. The Trade-Off Matrix

| | Offset | Cursor | Page Token (opaque) |
| --- | --- | --- | --- |
| Performance at depth | ❌ Linear | ✅ Constant | ✅ Constant |
| Stability under inserts | ❌ | ✅ | ✅ |
| "Go to page 47" | ✅ | ❌ | ❌ |
| Bookmarkable mid-list | ✅ (if data static) | ⚠️ via cursor | ⚠️ via token |
| Implementation cost | Trivial | Modest | Modest |
| Future-proofing | Hard to change | Easy if encoded as token | Easiest |
| Best for | Internal / static | Most APIs | Public / Google-style |

---

## 11. Worked Examples

### Twitter / X-style activity feed
```
GET /tweets?since_id=...        ← newer tweets
GET /tweets?max_id=...           ← older tweets
```
Twitter used numeric IDs that were k-sortable (Snowflake), so the `id` doubled as a time-cursor.

### Stripe (offset-ish but capped to 100)
```
GET /v1/charges?limit=100&starting_after=ch_xxx
```
Cursor-based. Stripe's pagination uses `starting_after` / `ending_before` keyed on object ID.

### GitHub (RFC 5988 Link headers)
```
HTTP/1.1 200 OK
Link: <.../events?page=2>; rel="next",
      <.../events?page=8>; rel="last"
```
URL-driven; client follows links. Good for hypermedia clients.

### Google Cloud / AWS (opaque tokens)
```
GET /v1/items?page_size=50&page_token=<opaque>
```
Maximum flexibility; tokens may rotate behind the scenes.

### Cursor pagination over composite key (general SQL)
```sql
SELECT * FROM orders
WHERE (status, created_at, id) > ($cursor_status, $cursor_ts, $cursor_id)
ORDER BY status, created_at, id
LIMIT 50;
```
Composite cursors handle multi-column sorts.

---

## 12. Common Mistakes

- **Pure offset pagination** at scale — kills DB performance and shifts rows under load.
- **No upper cap on `limit`** — clients request 100k and OOM the server.
- **Returning `total` on every request** — every page costs a full `COUNT(*)`.
- **Cursor that's just the row ID** when sort is by another column — sort order doesn't match cursor → broken.
- **No tiebreaker** when sorting by a non-unique column.
- **Forgetting `has_more`** — clients can't tell when to stop.
- **Cursor leaking internal IDs** — couples API surface to DB schema.
- **Mutable cursor schema** — breaks old clients when you "fix" pagination.
- **Same cursor format for different sort orders** — `?cursor=abc&sort=name` then `?cursor=abc&sort=-created_at` returns nonsense. Encode the sort into the cursor.

---

## 13. API Surface — Practical Patterns

A pragmatic, modern shape:

```http
GET /v1/orders?limit=50&cursor=<opaque>
```
Response:
```json
{
  "data": [...],
  "next_cursor": "...",
  "has_more": true,
  "limit": 50
}
```

Or, REST-ier, with `Link` headers:
```http
HTTP/1.1 200 OK
Link: </v1/orders?limit=50&cursor=xyz>; rel="next"
```

Either works. Pick one across your whole API and document it once.

---

## 14. Cheat Card

```
ALWAYS PAGINATE list endpoints. ALWAYS cap limit.

THREE TECHNIQUES
  Offset   - simple, jump-to-page, slow at depth, unstable. Internal/static only.
  Cursor   - fast, stable, infinite scroll. Default for most APIs.
  Token    - opaque cursor; future-proof; Google/AWS style.

CURSOR KEY = unique + indexed + monotonic
            usually (created_at, id) with id as tiebreaker.

GREATER-THAN comparison in SQL:
  WHERE (created_at, id) > ($cur_ts, $cur_id)
  ORDER BY created_at, id LIMIT $n+1

SHAPE
  { data, next_cursor, has_more, limit }

TOTALS
  Don't compute unless asked. They're often expensive.

PITFALLS
  No tiebreaker, OFFSET in the millions, mutable cursor format,
  cursor that ignores the sort/filter combo, no limit cap.

UI HINTS
  Infinite scroll → cursor.
  Page 1/2/3      → offset (but only for small static lists).
  Catch-up sync   → ?since=<cursor>.
```

---

## 15. Resources

### Articles
- "Pagination in REST APIs" — Stripe blog (cursor pagination explained): <https://stripe.com/blog/pagination>
- "Twitter's API pagination" — Twitter dev docs (since_id / max_id).
- "Faster pagination in MySQL" — Markus Winand (use-the-index-luke.com): <https://use-the-index-luke.com/no-offset>
- "Pagination Done the PostgreSQL Way" — Markus Winand: <https://use-the-index-luke.com/>
- "Relay Cursor Connections Spec": <https://relay.dev/graphql/connections.htm>
- "Google API Design Guide — List Pagination": <https://cloud.google.com/apis/design/design_patterns#list_pagination>

### Books
- *Designing Data-Intensive Applications* — Kleppmann; relevant index/scan discussion.
- *SQL Performance Explained* — Markus Winand; the canonical "OFFSET considered harmful" resource.

### Videos
- ByteByteGo: "Cursor vs Offset Pagination" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser: "OFFSET considered harmful": <https://www.youtube.com/@hnasr>

### Adjacent reading
- [REST API Design Principles](./rest-design.md)
- [GraphQL Fundamentals](./graphql.md) (Relay connection spec)
- [Database Indexing](../04-databases/indexing.md)
- [Hot Partition Problem](../10-scalability/hot-partitions.md)

---

*Previous:* [← API Versioning Strategies](./versioning.md)  |  *Next:* [Idempotency →](./idempotency.md)
