# REST API Design Principles

> **TL;DR** — A well-designed REST API maps **resources** to URLs, uses **HTTP methods** as verbs, returns **standard status codes**, paginates with **cursors**, supports **idempotency** on writes, **versions** explicitly, and ships with **OpenAPI** schemas. Good REST is a discipline, not a religion — the goal is APIs that are predictable, evolvable, and pleasant for the people who consume them (which is often "future you").

---

## 1. The Mental Model

REST = **Re**presentational **S**tate **T**ransfer (Roy Fielding, 2000). The core ideas:

- Everything is a **resource** identified by a URL.
- A small set of **verbs** (HTTP methods) acts on resources.
- Responses are **representations** (usually JSON) of resource state.
- **Stateless** — each request carries everything the server needs to answer it.
- **Cacheable** — responses say if and how they can be cached.
- **Uniform interface** — the same patterns everywhere.

> *In practice, almost no public API is "RESTful" by the strictest definition. What people mean by REST today is "JSON over HTTP with resource-shaped URLs and the right verbs." That's fine. Be consistent, not pure.*

---

## 2. Resource-Shaped URLs

Use plural nouns. Hierarchy reflects containment.

```
GET    /users                ← list users
POST   /users                ← create a user
GET    /users/42             ← read one
PUT    /users/42             ← replace
PATCH  /users/42             ← partial update
DELETE /users/42             ← remove

GET    /users/42/orders      ← orders that belong to user 42
GET    /orders/o-77          ← read a specific order directly
GET    /users/42/orders/o-77 ← either works; pick one and stick with it
```

Avoid verbs in URLs:
```
❌ GET /getUser?id=42
❌ POST /createOrder
✅ GET /users/42
✅ POST /orders
```

Lowercase, hyphens for multi-word resources:
```
/api/v1/shipping-addresses
/api/v1/access-tokens
```

---

## 3. HTTP Methods — What Each One Means

| Method | Use | Safe? | Idempotent? |
| --- | --- | --- | --- |
| `GET` | Read a resource | ✅ | ✅ |
| `HEAD` | Like GET but no body | ✅ | ✅ |
| `OPTIONS` | Capabilities / CORS preflight | ✅ | ✅ |
| `POST` | Create or non-idempotent action | ❌ | ❌ (use idempotency keys) |
| `PUT` | Replace (full update) | ❌ | ✅ |
| `PATCH` | Partial update | ❌ | depends |
| `DELETE` | Remove | ❌ | ✅ |

- **Safe** = no server state changes.
- **Idempotent** = repeating the call has the same effect (after the first).

Get these right and clients and middle-boxes (proxies, retries, CDNs) will behave correctly.

---

## 4. Status Codes — The Right Ones, Used Right

### 2xx success
- `200 OK` — most reads / updates.
- `201 Created` — successful POST that created something. Include a `Location:` header pointing to the new resource.
- `202 Accepted` — async; we'll work on it.
- `204 No Content` — success with no body (typical for DELETE).

### 3xx redirection
- `301 Moved Permanently` / `308` — resource has a new URL forever.
- `302 Found` / `307` — temporary redirect.
- `304 Not Modified` — conditional GET hit; reuse cached copy.

### 4xx client error
- `400 Bad Request` — malformed request.
- `401 Unauthorized` — auth missing or invalid (despite the name).
- `403 Forbidden` — authenticated but not allowed.
- `404 Not Found` — resource doesn't exist.
- `405 Method Not Allowed` — wrong verb on the URL.
- `409 Conflict` — concurrent change, duplicate, version mismatch.
- `410 Gone` — used to exist, won't be back.
- `412 Precondition Failed` — conditional request didn't match (ETag).
- `415 Unsupported Media Type` — bad `Content-Type`.
- `422 Unprocessable Entity` — request parsed but semantically wrong (validation).
- `429 Too Many Requests` — rate limited (include `Retry-After`).

### 5xx server error
- `500 Internal Server Error` — generic crash.
- `502 Bad Gateway` — upstream broken.
- `503 Service Unavailable` — temporarily overloaded; clients should retry with backoff.
- `504 Gateway Timeout` — upstream slow.

Use them precisely. Returning `200 OK` with `{ "error": ... }` is *technically* legal and *practically* terrible for client retry logic.

---

## 5. Standard Error Bodies

Pick one error envelope and use it everywhere. A solid pattern (loosely based on Google's API design guide):

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user with id 42.",
    "details": [
      { "field": "id", "issue": "not_found" }
    ],
    "request_id": "req_01H..."
  }
}
```

Include:
- A **stable machine-readable code** (`USER_NOT_FOUND`). Clients branch on this.
- A **human-readable message**. Logs and toast notifications use it.
- **Details / fields** for validation errors.
- A **request ID** for support and tracing.

Don't change the shape between endpoints. Don't put HTML in error messages.

---

## 6. Resource Representations

### JSON conventions
- `snake_case` or `camelCase` — pick one, never both.
- Time as **ISO 8601 strings** with timezone (`2026-05-19T14:30:00Z`). Or Unix epoch seconds — but pick one.
- Money as **integer minor units** (`amount: 4200, currency: "USD"`) — never floats.
- IDs as strings (`"u_42"`) even when numeric; lets you change format later.
- Booleans are `true`/`false`, not `0`/`1` or `"yes"`/`"no"`.
- Empty list is `[]`, not `null`.

### Nesting
Choose between "expand-by-default" and "flat-with-IDs". Expanding everything bloats payloads; never expanding causes N+1 client calls. A pragmatic middle:

```http
GET /orders/o-1?expand=customer,items
```

Stripe's API popularized this pattern: explicit `?expand=` query param, default to lean.

---

## 7. Versioning

You will change your API. Plan for it.

Common strategies:

| Strategy | Looks like | Pros | Cons |
| --- | --- | --- | --- |
| URL path | `/v1/users` | Obvious, easy to route | Big-bang version bumps |
| Header | `Accept: application/vnd.acme.v2+json` | Cleaner URLs | Hidden from logs / curl |
| Query param | `?v=2` | Easy | Pollutes URLs |
| Date-based | `Stripe-Version: 2024-04-10` | Per-feature granularity | Requires a translation layer |

Prefer **URL path versioning** for most APIs. Date-based (Stripe-style) is brilliant for products that evolve rapidly but expensive to maintain (you need request/response transformers per version).

See: [API Versioning Strategies →](./versioning.md).

---

## 8. Pagination, Filtering, Sorting

### Pagination
- **Cursor-based** is the default — stable under concurrent inserts:
  ```
  GET /events?limit=50&after=evt_abc123
  ```
- **Offset-based** (`?page=3`) is fine for small static lists but breaks at scale (DBs hate large `OFFSET`).
- Always cap `limit`. Document it.

See: [API Pagination Techniques →](./pagination.md).

### Filtering
- Simple equality:
  ```
  GET /users?status=active&country=DE
  ```
- For complex filters use a structured query language as a single param:
  ```
  GET /events?filter=type:checkout.completed AND created_at>=2026-05-01
  ```

### Sorting
```
GET /users?sort=-created_at,name
```
Prefix `-` for descending. Document allowed sort fields.

---

## 9. Idempotency

Network retries are inevitable. Without idempotency, a single button click can charge a card twice.

- `GET`, `PUT`, `DELETE` are naturally idempotent.
- `POST` is not — fix it with an **idempotency key**:
  ```
  POST /payments
  Idempotency-Key: f4f6c2e1-3b58-4d11-a8e3-...
  ```
- Server stores the result keyed by the idempotency key. Replays return the *same* response without re-processing.

See: [Idempotency →](./idempotency.md).

---

## 10. Caching & Conditional Requests

Use HTTP caching primitives — they're free.

### Cache-Control
```
Cache-Control: public, max-age=300
Cache-Control: no-store          ← never cache (auth tokens, etc.)
Cache-Control: private, max-age=60  ← browser only, not CDN
```

### Conditional GETs
- Server returns `ETag` (a hash) or `Last-Modified` on responses.
- Client sends `If-None-Match` / `If-Modified-Since` on next request.
- Server returns **304 Not Modified** if nothing changed → no body, fast.

### Conditional updates (optimistic concurrency)
```
PATCH /users/42
If-Match: "v17"
```
Server checks `If-Match` against current version. If mismatch → `412 Precondition Failed`. Prevents lost updates.

---

## 11. Authentication & Authorization

For machine-to-machine: **bearer tokens** in `Authorization: Bearer ...`.
For user-facing: **OAuth 2.0 / OIDC** flows producing tokens.
For first-party SPAs on the same root domain: **cookies** with `SameSite=Lax`, plus CSRF tokens for mutations.

Don't:
- Put credentials in URLs (`?api_key=...`).
- Roll your own crypto.
- Use plain HTTP for anything ever.

See: [OAuth 2.0 & OIDC →](../12-security/oauth-oidc.md) · [JWT →](../12-security/jwt.md).

---

## 12. Rate Limiting

Every public API needs it.

Standard response:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 1716072600
```

Granularities: per-IP, per-API-key, per-account, per-endpoint, per-method. The right combination depends on your abuse model.

See: [Rate Limiting →](./rate-limiting.md).

---

## 13. Documentation & OpenAPI

If your API isn't documented, it doesn't exist for new clients.

- Use **OpenAPI 3.x** (or its Stripe-style fork). It's the industry standard.
- Generate docs from the spec (Redoc, Stoplight, SwaggerUI, Mintlify, Scalar).
- Generate **clients** from the spec for popular languages.
- Keep examples in sync with reality. Out-of-date examples are worse than no examples.

```yaml
openapi: 3.1.0
info:
  title: Acme API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: Get a user
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        '200':
          description: A user
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

---

## 14. Webhooks for the Other Direction

REST is "you ask, I answer." Push events come from **webhooks** — your service POSTs to a customer URL when something happens.

See: [Webhooks →](../02-networking/webhooks.md).

---

## 15. HATEOAS (and why most APIs ignore it)

Fielding's REST includes "Hypermedia as the Engine of Application State" — responses link to other actions:

```json
{
  "id": "o-1",
  "state": "pending",
  "_links": {
    "self":   { "href": "/orders/o-1" },
    "cancel": { "href": "/orders/o-1/cancel", "method": "POST" },
    "items":  { "href": "/orders/o-1/items" }
  }
}
```

Beautiful in theory. In practice, almost no one uses it: clients are hand-coded against fixed URLs and prefer it that way. Worth knowing the term; rarely worth implementing.

---

## 16. A Reference Endpoint, Fully Specified

```
POST /v1/payments
Content-Type: application/json
Authorization: Bearer sk_live_...
Idempotency-Key: 6a7c...

{
  "amount": 4200,
  "currency": "USD",
  "source": "tok_visa",
  "description": "Order o-1"
}
```

Possible responses:
- `201 Created` + the new payment object + `Location: /v1/payments/pay_123`.
- `400 Bad Request` — malformed JSON.
- `401 Unauthorized` — bad / missing token.
- `402 Payment Required` — card declined.
- `409 Conflict` — same idempotency key with different body.
- `422 Unprocessable Entity` — amount must be > 0.
- `429 Too Many Requests` — rate limit.
- `503 Service Unavailable` — try again with backoff.

Each status has a clear, deterministic meaning. Clients can branch reliably.

---

## 17. Top 12 Design Rules

1. Resources are nouns; methods are verbs.
2. Use the right status code, every time.
3. One error envelope across the whole API.
4. Cursor pagination by default; cap `limit`.
5. Idempotency keys on every mutating POST.
6. Version explicitly; never break v1.
7. Money as integer minor units; time as ISO 8601 + UTC.
8. Stable machine-readable error codes.
9. Document with OpenAPI; generate clients & docs from it.
10. Authenticate with bearer tokens or proper OAuth flows.
11. Rate-limit with `429 + Retry-After + RateLimit-*` headers.
12. Don't break clients silently — communicate deprecations.

---

## 18. Common Mistakes

- Verbs in URLs (`/createUser`).
- Returning `200 OK` for errors.
- Using `PUT` for partial updates (use `PATCH`).
- Random snake/camel/Pascal case mixing.
- Floating-point money.
- `null` for empty lists.
- Offset pagination that breaks under inserts.
- No rate limiting, then surprised by abuse.
- No versioning, then trapped by every change.
- Treating documentation as a one-time task.
- Inventing your own auth scheme.
- Returning DB column names as JSON field names (couples API to schema).

---

## 19. Cheat Card

```
URLS         /v1/<resource>/<id>/<sub-resource>  ← lowercase, plural, no verbs
METHODS      GET read   POST create   PUT replace   PATCH partial   DELETE remove
STATUS       2xx ok    4xx client    5xx server     409 conflict   429 rate-limited
ERRORS       { error: { code, message, details, request_id } }
PAGINATION   cursor: ?limit=50&after=ID    cap limit
IDEMPOTENCY  Idempotency-Key header on every mutating POST
VERSIONING   path: /v1/...                 deprecate, don't break
CACHING      ETag / If-None-Match → 304    Cache-Control on every GET
TIME         ISO 8601 + Z (UTC)
MONEY        integer minor units + currency code
AUTH         Authorization: Bearer …       OAuth for users
DOCS         OpenAPI 3.x → Redoc / Scalar
RATE-LIMIT   429 + Retry-After + RateLimit-* headers
```

---

## 20. Resources

### Books
- *RESTful Web APIs* — Leonard Richardson, Mike Amundsen.
- *API Design Patterns* — JJ Geewax.
- *Designing Web APIs* — Brenda Jin, Saurabh Sahni, Amir Shevat (O'Reilly).

### Online
- **Stripe API** — best-in-class example, read it cover to cover: <https://stripe.com/docs/api>
- **GitHub REST API**: <https://docs.github.com/en/rest>
- **Google API Design Guide**: <https://cloud.google.com/apis/design>
- **Microsoft REST API Guidelines**: <https://github.com/microsoft/api-guidelines>
- **Heroku HTTP API Design Guide**: <https://github.com/interagent/http-api-design>
- **Zalando RESTful API Guidelines**: <https://opensource.zalando.com/restful-api-guidelines/>

### OpenAPI tooling
- **Redoc**, **Scalar**, **Mintlify**, **Stoplight Studio** — docs.
- **openapi-generator** — clients in 50+ languages.
- **Spectral** — lint OpenAPI specs.
- **Buf for OpenAPI** equivalents emerging.

### Videos
- ByteByteGo: "REST API Best Practices" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser API design series — <https://www.youtube.com/@hnasr>

### Adjacent reading
- [API Versioning →](./versioning.md)
- [API Pagination →](./pagination.md)
- [Idempotency →](./idempotency.md)
- [Rate Limiting →](./rate-limiting.md)
- [API Gateway →](./api-gateway.md)
- [REST vs GraphQL vs gRPC →](../02-networking/api-styles.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [GraphQL Fundamentals →](./graphql.md)
