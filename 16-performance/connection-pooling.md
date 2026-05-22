# Connection Pooling & Keep-Alive

> **TL;DR** — Opening a new TCP connection costs **1 RTT just to handshake** (3 RTTs for TLS over plain TCP, 1 RTT for TLS 1.3, 0 RTT for resumed sessions or QUIC). A connection **pool** reuses already-open connections instead of opening a new one per request; **keep-alive** is the HTTP/TCP mechanism that lets the pool exist. Done right, pooling cuts P50 latency by 30–80% and dramatically reduces resource pressure on databases, downstream services, and the network. Done wrong — pool too big, no idle timeout, no health check — it causes the most baffling production outages you'll see, because the connection state lives outside your code. The boring rule that nobody breaks twice: **start small, measure, and remember that the database's `max_connections` is the real ceiling, not yours**.

---

## 1. The big picture

```
Without pool                       With pool
─────────────                      ─────────

req → [open TCP]                   req → [reuse conn] → query → keep conn
       [TLS handshake]                                          (alive)
       [auth handshake]
       query
       [close]

5 RTTs                             1 RTT for the query
```

Every connection-oriented protocol has setup cost — TCP handshake, TLS handshake, often an application-layer auth/login (Postgres startup, MySQL handshake, Redis AUTH, AMQP handshake). Setup can easily exceed the actual query in latency, especially over a network with non-trivial RTT.

A connection pool maintains a set of established connections. Borrow one to do work; return it to the pool when done. The next request reuses it, paying zero setup cost.

```
┌──────────────────────────────────────────────────┐
│ App process                                      │
│  ┌────────────────────────────────────────────┐  │
│  │ Connection pool                            │  │
│  │  [conn] [conn] [conn] [conn] [conn]        │  │
│  └──────────────┬─────────────────────────────┘  │
└─────────────────┼────────────────────────────────┘
                  │
                  ▼
            ┌──────────┐
            │ Database │
            └──────────┘
```

It's that simple in principle. The pain is in the details of sizing, lifecycle, and failure handling.

---

## 2. What "keep-alive" actually means

**Keep-alive** is the underlying mechanism that allows a connection to live longer than one request/response cycle.

### HTTP keep-alive (HTTP/1.1 default)

HTTP/1.0 closed the TCP connection after every response. HTTP/1.1 made keep-alive the default: the client and server agree (via headers or simply by not closing) to leave the TCP connection open for subsequent requests on the same host. The `Connection: keep-alive` header is historical — in HTTP/1.1 it's already implied.

Server tunables:
- **`keepalive_timeout`** (nginx) — how long an idle connection stays open before the server closes it. 60–75s is typical.
- **`keepalive_requests`** — how many requests a single connection serves before forced close (rotates connections for load balancing fairness).

### TCP keep-alive

A different thing despite the shared name. TCP keep-alive is the OS-level mechanism that sends probes on otherwise-idle TCP connections to detect dead peers (NAT timeouts, half-open connections after a crash). Tunables (`tcp_keepalive_time`, `tcp_keepalive_intvl`, `tcp_keepalive_probes`) live in the kernel.

Default Linux: probe after 2 hours of idle. Way too long for production load balancers and NATs. Tune lower (300s) for any service that expects long-lived idle connections.

### HTTP/2 and HTTP/3

HTTP/2 multiplexes many requests over a single TCP connection — pooling is mostly automatic per-host. HTTP/3 runs on QUIC (UDP) and inherits multiplexing plus 0-RTT resumption. Both significantly reduce the importance of per-request pooling on the HTTP side. See [HTTP/1.1 vs HTTP/2 vs HTTP/3 →](../02-networking/http-versions.md).

---

## 3. The setup-cost numbers

Approximate wall-clock cost of opening a fresh connection:

| Setup | Round trips | Within DC (1ms RTT) | Cross-region (60ms RTT) |
|---|---|---|---|
| Plain TCP | 1 (SYN, SYN-ACK, ACK) | 1 ms | 60 ms |
| TLS 1.2 over TCP | 3 (TCP + 2 for TLS) | 3 ms | 180 ms |
| TLS 1.3 over TCP | 2 (TCP + 1 for TLS) | 2 ms | 120 ms |
| TLS 1.3 resumed | 1 | 1 ms | 60 ms |
| QUIC initial | 1 | 1 ms | 60 ms |
| QUIC 0-RTT | 0 | 0 ms | 0 ms |

Add database-specific costs on top:
- **Postgres**: startup message exchange, authentication, parameter status — roughly 1 more round trip plus password hashing.
- **MySQL**: handshake + auth ≈ 1 round trip.
- **Redis**: minimal (AUTH if password protected).
- **MongoDB**: handshake + auth + isMaster.

For a Postgres query that takes 0.5 ms over a 1ms RTT network, opening a new connection per request multiplies latency by 6–10×.

---

## 4. Why the wrong pool size kills you

Most engineers' first instinct is "more connections = more throughput." Almost always wrong for databases.

### The database is the constraint

Postgres `max_connections` defaults to 100. Each connection is a forked backend process consuming ~5–10 MB of RAM, plus kernel resources. Past a few hundred, scheduling overhead and lock contention rise faster than throughput. **Postgres throughput often peaks at concurrency ≈ 2–4× CPU cores**, not at concurrency = number of clients.

If you run 10 app instances with 100-connection pools each, you've configured **1000 connections to Postgres** — three times what it can effectively serve.

### Little's Law

A foundational queueing theory result:

```
L = λ × W
```

- `L` = average concurrency (in-flight requests)
- `λ` = throughput (requests/sec)
- `W` = average response time (sec)

For a database serving 5000 QPS at 5ms average latency, you need `L = 25` concurrent connections to keep up. Going above 25 doesn't help — those extra connections just sit idle or queue.

**Real-world rule of thumb (Brett Wooldridge, HikariCP):** `pool_size = ((core_count × 2) + effective_spindle_count)`. For SSD-backed Postgres on an 8-core box, ~16–20 connections per app instance is often optimal.

That can feel scarily small. Trust the math; benchmark.

### Pool too big — symptoms

- High latency under load, despite low DB CPU.
- DB connection count saturating `max_connections`; new connections rejected.
- Lock contention inside the DB (`pg_locks`, `SHOW ENGINE INNODB STATUS`).
- Increased context-switching overhead.
- Memory bloat on the DB host.

### Pool too small — symptoms

- Requests waiting on `getConnection()` — pool exhaustion timeouts.
- DB CPU underutilized.
- Throughput plateaus below the DB's actual capacity.

The right size is between these. Find it by ramping load and watching both p99 and DB CPU.

---

## 5. PgBouncer, ProxySQL, RDS Proxy — connection multiplexers

Beyond pools-per-app, dedicated connection multiplexers sit between your apps and the database to share a small set of real DB connections across many client connections.

**PgBouncer** (Postgres) — the canonical example:

| Mode | Behavior | Use case |
|---|---|---|
| `session` | One client connection ↔ one server connection for the session | Default, safest |
| `transaction` | Server connection released after each transaction commits | **Sweet spot** for most apps |
| `statement` | Released after each statement | Very high multiplex, breaks transactions and many features |

Transaction-mode PgBouncer is often the magic that lets a Postgres with 100 backends serve 10,000 client connections. It's nearly universal in Postgres production setups.

Equivalents:
- **ProxySQL** for MySQL.
- **RDS Proxy** (AWS managed).
- **Cloud SQL Connector** patterns.
- **HAProxy** in front of read replicas.

Caveats with transaction-mode PgBouncer:
- Prepared statements get tricky (need PgBouncer 1.21+ with `prepared_statement_support`, or use protocol-level prepared statements carefully).
- Session-scoped settings (`SET LOCAL` is fine; `SET` without local won't work).
- Advisory locks and `LISTEN/NOTIFY` need session mode.

Read the docs; test your code paths against PgBouncer in CI.

---

## 6. Connection lifecycle — the failure modes

A pooled connection is a stateful object that can break in many ways while sitting idle.

### Idle connections die

- NAT tables expire (commonly 5–15 min).
- Load balancers close idle TCP connections (AWS NLB: 350s, ALB: 60s by default, GCP LB: 600s, often configurable but not unlimited).
- Database server-side timeouts (`idle_in_transaction_session_timeout`, `wait_timeout`).
- Firewalls drop "stale" connections silently.
- The DB process restarts; client's connection is bytes pointing at nothing.

If your pool hands out a dead connection, the first query on it fails. Symptoms: intermittent `connection reset by peer`, `EOFException`, `SSL_read failed`, weird errors after deploys.

### The remedies

- **Validation on borrow (`testOnBorrow`)** — light query (`SELECT 1`) before handing out. Catches dead connections; small per-borrow cost.
- **Idle test (`testWhileIdle`)** — periodic background validation. Higher detection rate, no per-borrow cost.
- **Max lifetime** — close connections after, say, 30 minutes regardless of activity. Forces freshness; rotates around LB timeouts. **HikariCP defaults to 30 min.**
- **Idle timeout** — close connections idle for X minutes; pool shrinks under low load.
- **Aggressive TCP keepalive** at the OS level — kernel probes detect dead peers within minutes.

A good pool configuration handles all of these. Misconfiguring `max_lifetime` is a top cause of "we have a deploy + 5 minutes later, errors stop" patterns.

### Server-side connection limits

- **Postgres**: `max_connections` (hard limit), `idle_in_transaction_session_timeout` (drops abandoned transactions), `statement_timeout` (kills runaway queries).
- **MySQL**: `max_connections`, `wait_timeout`, `interactive_timeout`.
- **Redis**: `maxclients` (default 10000).

Always set DB-side timeouts. The client's `query_timeout` doesn't help if the network blip drops the response — only the server can kill an actually-running statement.

---

## 7. Pool configurations — what to actually set

A representative HikariCP (JVM) configuration for a service hitting Postgres:

```properties
maximumPoolSize = 20            # the famous magic number
minimumIdle = 20                # same as max (no shrink under steady load)
connectionTimeout = 5000        # 5s waiting to borrow → fail fast
idleTimeout = 600000            # 10 min
maxLifetime = 1800000           # 30 min (must be < server-side max)
keepaliveTime = 30000           # 30s — keeps NATs and LBs happy
validationTimeout = 3000
leakDetectionThreshold = 60000  # warn if a borrow lasts >60s
```

What every value does:

- **`maximumPoolSize`** — hard cap on connections. Sized for the database, not the app.
- **`connectionTimeout`** — how long to wait for a free connection. **Fail fast** instead of building unbounded queues.
- **`maxLifetime`** — recycle to avoid being closed at random by an LB.
- **`leakDetectionThreshold`** — flags a connection borrowed and never returned (a missing `close()` somewhere).

Cross-language equivalents exist: `psycopg-pool`, `asyncpg.create_pool`, Go's `database/sql` with `SetMaxOpenConns` / `SetMaxIdleConns` / `SetConnMaxLifetime`, Node `pg` / `mysql2`, Ruby's `ActiveRecord` pool, .NET's `SqlConnection` pool. Same knobs, different names.

---

## 8. Pools for non-database resources

The pattern applies to anything with non-trivial setup cost:

| Resource | Pool needed? | Notes |
|---|---|---|
| Database connections | **Yes** | The primary case. |
| HTTP clients to downstream services | **Yes** | Use the language's HTTP client with proper keep-alive. |
| gRPC channels | **Yes** | Channels multiplex many calls; usually one per peer is enough. |
| Redis connections | **Yes** | Single-threaded server; usually a small pool per app instance. |
| Message broker producers (Kafka, RabbitMQ) | Usually one producer per process, reused. |
| SSH connections | Yes for automation tools. |
| GPU contexts (CUDA) | Yes for inference servers. |

HTTP clients are the most-forgotten pool. In many languages the default client opens a new TCP connection per request *unless you reuse the same client object*:

```go
// Wrong: new transport every request → no pooling
http.Get(url)

// Right: reuse a single client/transport
var client = &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
    },
    Timeout: 5 * time.Second,
}
client.Get(url)
```

In Python, prefer **`requests.Session`** or `httpx.Client` over `requests.get` per call. In Node, **`undici` `Pool`** or `http.Agent` with `keepAlive: true`. Per-request new connections during a load test was a 10× perf miss in real production codebases.

---

## 9. The serverless wrinkle

Serverless functions (AWS Lambda, Cloud Run, Vercel) have a problem: many ephemeral instances, each with its own pool, multiplied by concurrent invocations. A burst from 0 to 1000 concurrent Lambdas with a 10-connection pool each is 10,000 connections to your DB. Postgres falls over.

Solutions:

- **Managed proxies** — AWS RDS Proxy, Cloud SQL Auth Proxy, Neon's HTTP edge.
- **HTTP-based DB protocols** — Neon serverless driver, Cloudflare D1, PlanetScale's HTTP API — designed for ephemeral compute.
- **PgBouncer in front** — transaction-mode pooler sized for the DB, not the function fleet.
- **Per-function pool size = 1**. With high concurrency limits, this is sometimes the right answer.

The lesson: traditional pooling assumes long-lived processes. Serverless breaks that assumption. Plan for it.

---

## 10. Common Mistakes / Anti-Patterns

- **Pool way too big "to be safe."** First load spike saturates the DB.
- **Per-request connection (no pool).** P50 explodes; DB gets churned by handshake load.
- **No validation on borrow with long-lived pools.** Dead connections handed out after LB or NAT timeouts.
- **No `maxLifetime`.** Connections live forever; eventually one breaks silently and the pool keeps handing it out.
- **`maxLifetime` longer than the LB/NAT timeout.** Same problem.
- **Forgetting connection rotation behind a load balancer.** All app instances stick to one DB host because they opened connections during a brief window.
- **HTTP client recreated per request.** No keep-alive reuse; latency 5× higher than needed.
- **Pool-per-tenant in a multi-tenant app.** Connection count explodes by N tenants × M apps.
- **Pool sized identically in staging and prod.** Staging is fine; prod traffic blows past `max_connections`.
- **Sharing one pool across unrelated workloads.** Heavy reporting query starves the API.
- **No timeouts on `acquire`.** Workers wait forever for a connection; the upstream client gives up; you keep computing.
- **Treating PgBouncer transaction mode as a drop-in.** Prepared statements, `SET`, advisory locks all have quirks. Test them.
- **`SELECT pg_advisory_lock()` from a transaction-mode connection.** Released as soon as the transaction ends, which is not what you want.
- **No idle timeout.** Pool sits at max size forever, no shrink under low load.
- **No metrics on pool state.** You don't know when you're at exhaustion until paged.
- **Connection leaks from a missing `close()` in an error path.** Pool drains over hours; service degrades.
- **Trying to "make it work without PgBouncer" past a few thousand QPS.** Almost always wrong for Postgres at scale.

---

## 11. Metrics every pool should expose

If you can't see these, you don't really run a pool — you run a mystery:

- **In-use connections** (currently borrowed)
- **Idle connections** (available)
- **Waiting threads/tasks** (waiting on `acquire`)
- **Acquire wait time histogram** (p50, p99)
- **Pool exhaustion events** (count of `connectionTimeout` rejections)
- **Validation failures** (dead-connection detections)
- **Connection lifetime distribution**
- **Errors during borrow / return**

Plot these alongside DB CPU, DB connection count, and request latency. Most pool issues become obvious once visualized.

---

## 12. Cheat Card

```
PURPOSE   Reuse connections instead of paying TCP+TLS+auth setup
          on every request. Bound resource use on the DB side.

OPEN COST  TCP 1 RTT · TLS 1.2 +2 RTT · TLS 1.3 +1 RTT · QUIC 0-RTT

KNOBS
  maxPoolSize         hard cap. size for the DB, not the app
  minIdle             warm pool size at low load
  connectionTimeout   fail-fast on acquire (1–5s)
  idleTimeout         shrink under low load
  maxLifetime         force rotation (< LB/NAT idle limit)
  keepaliveTime       prevent silent NAT timeouts
  testOnBorrow / Idle catch dead connections

SIZING RULE
  pool ≈ ((cores × 2) + effective_spindles)
  L = λ × W (Little's Law) — measure, don't guess

PG MULTIPLEXING
  PgBouncer transaction mode  →  10k clients → 100 backends
  Watch for: prepared statements, SET, advisory locks

KEEP-ALIVE LEVELS
  HTTP keep-alive    HTTP/1.1 default; multiplex via HTTP/2/3
  TCP keep-alive     OS probes; tune lower than 2h default
  DB keep-alive      DB-specific timeouts must be set

HTTP CLIENTS
  ALWAYS reuse one client/transport
  Per-host idle connection limits
  Timeout on every call

SERVERLESS
  Use RDS Proxy / Cloud SQL Proxy / Neon HTTP / PgBouncer
  Per-function pool size 1 with high concurrency caps

PITFALLS
  Pool too big → DB saturated, latency up
  No maxLifetime → dead connections handed out
  No validation → first query after LB timeout fails
  HTTP client recreated per request
  No metrics on pool state
  Connection leaks from error paths
  Sharing one pool across reporting + API

RULE   Start small. Measure under load. Trust the math (Little's
       Law). The DB's max_connections is the real ceiling.
```

---

## 13. Resources

### Documentation
- **HikariCP wiki** — About Pool Sizing: <https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing>
- **PgBouncer** — <https://www.pgbouncer.org/>
- **ProxySQL** — <https://proxysql.com/>
- **AWS RDS Proxy** — <https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html>
- **Postgres connection management** — <https://www.postgresql.org/docs/current/runtime-config-connection.html>

### Articles
- "About Pool Sizing" — Brett Wooldridge (the canonical "smaller than you think" piece).
- "Little's Law and Connection Pools" — various engineering blogs.
- "Connection pooling with PgBouncer" — Heroku Postgres docs.
- "The case against connection pooling in serverless" — Neon / Cloudflare engineering posts.
- "TCP keepalive HOWTO" — Linux kernel docs.

### Videos
- *Connection pooling deep dive* — Postgres Conference talks.
- *Architecting connection pools* — various AWS re:Invent talks.
- ByteByteGo — "Connection Pooling Explained."

### Tools
- **HikariCP** (JVM), **psycopg-pool / asyncpg.Pool** (Python), **database/sql** (Go), **node-postgres pg.Pool** (Node), **`undici Pool`** (Node HTTP).
- **PgBouncer**, **Pgpool-II**, **ProxySQL**, **RDS Proxy**, **Cloud SQL Auth Proxy**.
- **`pg_stat_activity`, `pg_locks`** — visibility into DB-side connections.
- **HAProxy** — connection routing and limits for almost any TCP service.

### Adjacent reading
- [Threading, Async I/O, Event Loops →](./threading-async.md)
- [Concurrency vs Parallelism →](./concurrency-parallelism.md)
- [Profiling & Benchmarking →](./profiling.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Connection Pooling (Databases) →](../04-databases/connection-pooling.md)
- [Backpressure →](../10-scalability/backpressure.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 →](../02-networking/http-versions.md)
- [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md)

---

*Previous:* [← Threading, Async I/O, Event Loops](./threading-async.md)  |  *Next:* [Compression →](./compression.md)
