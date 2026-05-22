# Ambassador & Adapter Patterns

> **TL;DR** — The **Ambassador** and **Adapter** patterns are two close cousins of the [Sidecar →](./sidecar.md) pattern, each with a specific shape. **Ambassador**: a proxy deployed beside the app that handles **outbound** calls to a remote service — retries, circuit breaking, TLS, service discovery, sharding — so the app makes a simple local call. **Adapter**: a sidecar that **translates between the app's interface and an external monitoring/management interface** — e.g., turning app-specific logs into Prometheus metrics, or speaking a vendor-specific health protocol on behalf of the app. Both keep the app's code small and language-agnostic; both have the same operational trade-offs as sidecars. They predate service mesh (Netflix Eureka + Hystrix, 2012) and remain useful when service mesh is overkill or for specific outbound-only flows (legacy systems, third-party APIs, sharded databases).

---

## 1. The Three Pod-Local Helper Patterns

Microsoft / Bilgin Ibryam (*Kubernetes Patterns*) and other writers usually list three closely-related patterns:

| Pattern | Direction | Job |
| --- | --- | --- |
| **Sidecar** | Augments app generally | Add cross-cutting capability beside the app |
| **Ambassador** | **Outbound** from the app | Proxy outbound calls — retries, TLS, discovery, sharding |
| **Adapter** | **Inbound** from a management plane | Translate the app's outputs/inputs to a standardized interface |

They share the same deployment shape (same pod / same host as the app, shared lifecycle). They differ in **what they do**:

```
                      ┌─────────────────┐
   inbound  ─────────►│      APP        │─────────► outbound
                      └─────────────────┘
                            ▲      │
                            │      ▼
        ┌──────────────────────┐  ┌──────────────────────┐
        │  Adapter (sidecar)   │  │ Ambassador (sidecar) │
        │ standardizes signals │  │ proxies outbound     │
        │ for management plane │  │ to remote services   │
        └──────────────────────┘  └──────────────────────┘
                  │                            │
                  ▼                            ▼
           Prometheus,                  External service,
           healthcheck system,          shard, third-party API
           logging collector
```

The pure sidecar is a general augmentation; ambassador and adapter are sidecars with a specific direction and role.

---

## 2. Ambassador — Proxy for Outbound Calls

The app makes a simple local call (`http://localhost:8001/charge`). The ambassador handles:

- **Service discovery** — find the real backend.
- **Load balancing.**
- **Retries + timeouts + circuit breaking.**
- **TLS termination / mTLS** outbound.
- **Authentication** to the remote.
- **Sharding** — which DB shard or region.
- **Rate limiting** against the remote.
- **Observability** — metrics, tracing for outbound calls.
- **Caching** of remote responses.

App code shrinks dramatically because all the network-resilience concerns live in the ambassador.

```
App ──HTTP─► localhost:8001 ──► Ambassador ─mTLS+retry+LB─► remote service
                                    │
                                metrics/traces/logs
```

### Origin

The pattern formalized in Netflix's **Prana** sidecar (~2014) — a Java sidecar that exposed Eureka discovery, Hystrix circuit breaking, and Zuul routing to non-JVM services (Python, Node) at Netflix. The point: bring "the Netflix infra library" to teams who didn't want to embed Java libraries everywhere.

Modern equivalents: service-mesh sidecars (Envoy in Istio, linkerd-proxy in Linkerd) do exactly this for service-to-service traffic. So in a service-mesh world, the ambassador is *already there* — you don't typically build a custom one.

### When you still build a custom ambassador

- **External / third-party APIs** (Stripe, Twilio, Salesforce) — service mesh doesn't cover these. A custom ambassador handles auth, retries, rate limits, telemetry.
- **Sharded databases / caches** — an ambassador routes by tenant/key to the right shard.
- **Legacy protocol bridging** — app speaks HTTP; ambassador speaks SOAP/FTP/MQ to the legacy system.
- **Connection pooling for a specific backend** — PgBouncer as an ambassador to Postgres.
- **No service mesh available** — for example, K8s sidecar-less environments or VMs.

### Worked example — Stripe ambassador

```
App (Python, no Stripe SDK)
   │
   │ POST localhost:9000/v1/charges
   ▼
Stripe Ambassador (sidecar, written in Go once for the org)
   - authenticates with Stripe API key
   - retries on 5xx and network errors with exponential backoff + jitter
   - idempotency-key bookkeeping per request
   - rate-limits at 90% of Stripe's account ceiling
   - emits Prometheus metrics: stripe_charge_latency, stripe_errors_total
   - traces with W3C headers passed through
   │
   │ POST https://api.stripe.com/v1/charges
   ▼
Stripe
```

Every team's app calls the local ambassador. The org maintains *one* Stripe client. Upgrading is a sidecar version bump.

---

## 3. Adapter — Standardize for Management

The Adapter translates the **app's outputs** to a standardized interface expected by an external system, or vice versa.

Typical jobs:

- **Translate logs** to a standard schema and ship.
- **Expose Prometheus `/metrics`** by scraping the app's native format.
- **Bridge custom health protocols** to a standard probe.
- **Add tracing context** to incoming requests if the app doesn't.
- **Convert an internal RPC into a public HTTP API**.
- **Provide a JMX-like / SNMP-like management interface** on top of an app that doesn't speak it natively.

Example: a legacy app writes proprietary log files. A Fluent Bit adapter sidecar tails them, parses them, normalizes them to the org's JSON schema, and ships to the central aggregator. The app didn't change.

Another: a custom binary doesn't expose Prometheus. An adapter exposes `/metrics` that scrapes the binary's `/healthcheck.txt` and translates.

### Conceptual relationship to interface adapters

The Adapter pattern in this context inherits the name from the classical Gang-of-Four **Adapter pattern** — adapting one interface to another. The difference: in-process Adapter is a class; this Adapter is an out-of-process sidecar. Same idea applied at the deployment level.

---

## 4. Ambassador vs Service Mesh vs API Gateway

These are often confused. Differences:

| | Ambassador (sidecar) | Service Mesh sidecar | API Gateway |
| --- | --- | --- | --- |
| Scope | Per-app outbound concerns | All service-to-service traffic in the mesh | North-south (external client → internal) |
| Per-pod? | Yes | Yes (per pod) | No (central or per-cluster) |
| Knows the app | Often (specific to one backend) | Generic | Generic |
| Use for | External APIs, sharded backends, legacy bridges | mTLS, mesh policies, mesh observability | Auth, routing, throttling at the edge |
| Example | Custom Stripe-ambassador | Envoy in Istio | Kong, AWS API Gateway, Apigee |

If you have a service mesh, most "ambassador" needs for **internal** services are already handled by the mesh's sidecar. Custom ambassadors then specialize in **external** and **legacy** integrations.

See [Service Mesh →](../03-apis/service-mesh.md), [API Gateway →](../03-apis/api-gateway.md).

---

## 5. Adapter vs Sidecar vs Ambassador — Quick Disambiguation

If you remember nothing else:

- **Sidecar:** generic; "do something useful beside the app."
- **Ambassador:** sidecar for **outbound** calls. App ← localhost → Ambassador → external.
- **Adapter:** sidecar that **translates** between the app and an external management plane (monitoring, configuration, etc.).

Many real sidecars play multiple roles. Vault Agent is sidecar + adapter (translates app's filesystem reads into Vault API calls). Envoy in Istio is sidecar + ambassador + adapter (proxies outbound, exposes metrics in Prometheus format, intercepts inbound TLS).

The pattern names matter less than understanding *what direction the helper is operating in and what it's translating*.

---

## 6. Worked Example — Ambassador for a Sharded Cache

A multi-tenant SaaS uses Redis sharded by tenant. Without an ambassador, every app:

- Resolves the right Redis shard for a tenant ID.
- Keeps connection pools per shard.
- Handles shard failure (failover to replica).
- Updates routing when shards rebalance.

That's a lot of code per service. An ambassador sidecar:

```
App ─SET tenant_42:cart─► localhost:6379  ──► Redis Ambassador
                                              - hash(tenant_42) → shard 3
                                              - pool connection to shard 3 primary
                                              - on failure: failover to replica
                                              - emit shard, latency metrics
                                              │
                                              ▼
                                            Redis shard 3
```

App code looks like talking to a local Redis. The sharding, failover, metrics live in the ambassador. The team that owns Redis owns the ambassador.

This is how some companies abstract their sharded data layers — Discord (cache), Slack (lookaside cache), and others have shipped variants of this pattern.

---

## 7. Worked Example — Adapter for Legacy Health Checks

A legacy app writes a status string to `/var/run/legacy/status.txt` every 30 seconds. K8s expects an HTTP `/healthz`. An adapter sidecar:

```python
# health-adapter sidecar
from http.server import HTTPServer, BaseHTTPRequestHandler
import os, time

STATUS_PATH = "/var/run/legacy/status.txt"

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        try:
            with open(STATUS_PATH) as f:
                line = f.read().strip()
            mtime = os.path.getmtime(STATUS_PATH)
            if line == "OK" and (time.time() - mtime) < 60:
                self.send_response(200); self.end_headers(); self.wfile.write(b"OK")
            else:
                self.send_response(503); self.end_headers()
        except FileNotFoundError:
            self.send_response(503); self.end_headers()

HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

K8s liveness probe hits the adapter at `:8080/healthz`. Legacy app is untouched.

This pattern lets you adopt modern orchestration without modifying decade-old codebases — a frequent benefit during migration.

---

## 8. Operational Concerns (Same as Sidecar)

Ambassadors and adapters inherit all sidecar trade-offs:

- **Lifecycle.** Use K8s sidecar containers (`restartPolicy: Always`) so the ambassador starts before and shuts down after the main app.
- **Latency.** Localhost calls are fast but not free. Account for ~0.1–1 ms per hop.
- **Memory.** Each pod carries the ambassador's RAM footprint. Right-size.
- **Observability.** The ambassador is a process — monitor it like any other workload.
- **Versioning.** Roll ambassadors out independently when feasible.
- **Failure modes.** If the ambassador crashes, the app's outbound calls fail. Set health relationships and tune restart policies.

See [Sidecar Pattern →](./sidecar.md).

---

## 9. Anti-Patterns

- **Putting business logic in the ambassador.** Ambassadors are infrastructure — retries, routing, observability. If you're conditionally choosing what to charge based on customer data inside the ambassador, the boundary is wrong.
- **One ambassador per third-party service per app, no reuse.** The whole point is to share infra logic. Build one Stripe ambassador for the org.
- **Building ambassadors when service mesh already covers it.** Internal service-to-service calls? The mesh handles it. Reserve ambassadors for external/legacy.
- **Adapters that re-shape data formats inconsistently.** Each team's adapter renames fields differently; central aggregation degrades. Standardize via shared adapter modules.
- **No version contract between app and ambassador.** Upgrade the ambassador, app breaks because it relied on a specific localhost API.
- **Ambassador holds long-lived state that's lost on restart.** Move state out — to Redis, DB, or stream — or accept "warming" on restart.
- **Tightly coupling apps to specific ambassador implementations.** Hard to swap or upgrade.
- **Using an adapter to fix a poorly designed app instead of fixing the app.** Sometimes legitimate (legacy, third-party); sometimes it's avoidance.
- **Forgetting graceful shutdown.** App exits cleanly but ambassador kills outbound retries mid-flight. Coordinate `preStop` hooks.

---

## 10. When to Use Each

| Situation | Pattern |
| --- | --- |
| Calling a third-party API consistently from many services | Ambassador |
| Sharded backend (DB, cache) abstraction | Ambassador |
| Cross-cutting infra (logs, metrics, secrets) | Sidecar |
| Internal service-to-service mTLS / retries | Service Mesh sidecar |
| Bridging a legacy protocol/format | Ambassador or Adapter |
| Translating between app's interface and an external standard | Adapter |
| North-south traffic (clients → cluster) | API Gateway (not a sidecar) |
| Per-language SDK avoidance | Ambassador |

---

## 11. Common Mistakes / Anti-Patterns

(See §9 — main ones are: business logic in sidecar, duplication across teams, building ambassadors that the mesh already provides, no version contract, ambassador holding ephemeral state, no graceful shutdown coordination.)

Additional:

- **Custom ambassador without strong tests.** It's now a single point of failure for outbound traffic.
- **Building "an ambassador for every dependency."** Over-engineering; pick high-leverage cases (external APIs, sharded stores).
- **Treating Ambassador as a synonym for Sidecar.** They overlap but have distinct emphases. Be precise so design conversations stay clear.
- **No SLA on the ambassador itself.** It's now in the critical path; treat it as production infrastructure.
- **Letting ambassadors accumulate behind-the-scenes business policy.** "Add a 10% discount if X" — that's not infra, it's logic.

---

## 12. Cheat Card

```
SIDECAR     "anything beside the app"
AMBASSADOR  sidecar for OUTBOUND — proxies app→remote calls
ADAPTER     sidecar that TRANSLATES between app and external interface

AMBASSADOR DUTIES   discovery · LB · retry · timeout · circuit-break ·
                    TLS/mTLS · auth to remote · sharding · rate limit ·
                    observability · caching

USE AMBASSADOR FOR  external APIs (Stripe, Twilio) · sharded DB/cache ·
                    legacy protocol bridging · per-language SDK avoidance ·
                    no-service-mesh environments

ADAPTER DUTIES      expose /metrics from app's native format ·
                    bridge custom health protocols ·
                    normalize logs ·
                    add tracing context ·
                    expose JMX/SNMP-like interfaces

VS SERVICE MESH     internal s2s already handled by mesh.
                    Custom ambassadors = external + legacy + sharded backends.

VS API GATEWAY      gateway is north-south (external client → cluster).
                    Ambassador is east-west outbound (app → remote).

TRADE-OFFS (same as sidecar)
  latency · memory · lifecycle ordering · observability · versioning

ANTI-PATTERNS
  business logic in ambassador · per-app duplication · re-building mesh ·
  state lost on restart · no graceful shutdown · accumulated policy

RULE: ambassador is the app's outbound diplomat.  Adapter is its translator.
       Build one per category for the org, not one per service.
```

---

## 13. Resources

### Books
- *Kubernetes Patterns* — Bilgin Ibryam & Roland Huß. Chapters on Sidecar, Ambassador, Adapter.
- *Designing Distributed Systems* — Brendan Burns. The book that popularized the trio.
- *Cloud Native Patterns* — Cornelia Davis.
- *Istio in Action* — Posta, Maloku.

### Articles
- "Pattern: Service Sidecar / Ambassador / Adapter" — microservices.io.
- "Prana: A Sidecar for your Netflix PaaS based Applications" — Netflix tech blog.
- "Ambassador pattern" — Microsoft Azure Architecture Center.
- "Adapter pattern in microservices" — various engineering blogs.

### Videos
- "Designing Distributed Systems" — Brendan Burns, KubeCon.
- "Patterns for Cloud-native Apps" — Bilgin Ibryam.

### Tools
- **Ambassador (third-party API):** custom; sometimes Envoy with API-specific filters.
- **Database / cache proxies (ambassador-shaped):** PgBouncer, AWS RDS Proxy, ProxySQL, Twemproxy.
- **Adapters for metrics:** node-exporter (kind of), JMX exporter, custom Prometheus exporters.
- **Adapters for logs:** Fluent Bit, Vector — when configured to parse legacy formats.
- **Service mesh proxies (also act as ambassador for internal):** Envoy, Linkerd-proxy.

### Adjacent reading
- [Sidecar Pattern →](./sidecar.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [BFF — Backend for Frontend →](../03-apis/bff.md)
- [Microservices Architecture →](./microservices.md)
- [Circuit Breaker Pattern →](../11-reliability/circuit-breaker.md)
- [Retry, Timeout, Exponential Backoff →](../11-reliability/retry-timeout-backoff.md)

---

*Previous:* [← Sidecar Pattern](./sidecar.md)  |  *Next:* [Lambda vs Kappa Architecture →](./lambda-kappa.md)
