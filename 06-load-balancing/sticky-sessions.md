# Sticky Sessions

> **TL;DR** — A **sticky session** (session affinity) pins a client's requests to the same backend across multiple calls. The LB does this by hashing some stable identifier (IP, cookie, header, key) and routing the same value to the same backend. Sticky sessions are usually a workaround for **stateful backends** — sessions in memory, WebSockets, sharded caches that benefit from locality. **The honest take: sticky sessions are a smell most of the time.** They defeat horizontal scaling, complicate deploys (draining a sticky backend kicks out active users), create hot pods, and don't survive pod restarts. The right answer is usually to make your service stateless and push session state to a shared store (Redis, DB, encrypted cookie). When you do need affinity, use **cookie-based stickiness** at L7 over IP-based stickiness — IP hashing breaks behind NAT — and bound the stickiness duration tightly.

---

## 1. What Sticky Sessions Are

Without affinity:
```
request 1 from user A → backend 2
request 2 from user A → backend 5
request 3 from user A → backend 1
```
Each request is independent. If the backends are stateless, all three see the same state via shared storage. Done.

With sticky session:
```
request 1 from user A → backend 2  (LB decides "user A → backend 2")
request 2 from user A → backend 2  (LB remembers)
request 3 from user A → backend 2
```

User A's traffic concentrates on backend 2 until something invalidates the affinity (cookie expires, backend dies, pool changes).

---

## 2. Why You Might Want Stickiness

Real reasons (in declining order of legitimacy):

### 2.1 Session state in memory
Old-school servlets stored sessions in process memory. Sticky session needed because only that backend knew the user. **Solution today: externalize sessions (Redis, JWT, signed cookie).**

### 2.2 WebSocket / SSE / long-lived connection
The connection is bound to one backend for its lifetime. Once established, every message goes to that backend. This is **implicit stickiness** for the lifetime of the connection — not session affinity in the LB-config sense.

### 2.3 In-process caches with locality benefits
A pod holds a hot cache of user A's data. Routing user A back to that pod gives cache hits.

This can be legit at large scale. The standard pattern: **consistent hashing on user id** so user A predictably hits pod 2, populating that pod's cache. See [Consistent Hashing →](../04-databases/consistent-hashing.md).

### 2.4 Stateful workloads
Game servers (one game session per pod), real-time analytics windowing, video transcoders. The pod *owns* the state for that user/session — affinity is required for correctness.

### 2.5 Migration / legacy
You inherited a monolith with session state in memory. Modernizing the session store is on the backlog. Stickiness keeps it working.

### 2.6 Test / canary by user
"Route the same user to the canary so they have a consistent experience during the test." Done via consistent hashing on user id.

---

## 3. Where Sticky Sessions Hurt

The cost is real.

### 3.1 Defeats load balancing
The whole point of an LB is to spread traffic. Stickiness pins each user. If users have very different traffic patterns (a heavy user vs many light users), the heavy user's pod gets overloaded.

### 3.2 Hot pods
Celebrity user, high-fanout user, or just bad luck → one pod handles 10× the load of others.

### 3.3 Deploys are painful
A draining pod has active sticky sessions. Either:
- Drop them (users see errors).
- Wait for them to drain (sometimes minutes if sessions are long).
- Migrate sessions to another pod (complex; requires shared session store anyway — at which point why are you sticky?).

### 3.4 Pod restarts break sessions
The pod dies; the user's session is gone. They're force-logged-out or routed to a different pod that has no context.

### 3.5 Autoscaling is awkward
Adding pods doesn't help current sticky users; they stay on their original pods. Scaling out doesn't relieve hot pods.

### 3.6 Multi-LB inconsistency
If you have multiple LB instances and they don't share state, two requests from the same user can land on different LBs and (potentially) be routed to different backends. Cookie-based stickiness solves this; LB-table-based stickiness fails.

---

## 4. The Mechanisms

### 4.1 IP hash
LB hashes the client IP, routes to `hash(ip) % N`. Stateless across LB instances. No cookies needed.

```nginx
upstream app {
    ip_hash;
    server 10.0.0.1;
    server 10.0.0.2;
    server 10.0.0.3;
}
```

**Pros**
- Works without changes to the application.
- Stateless across LB instances.

**Cons**
- **NAT / CGNAT skew** — corporate users, mobile carriers, VPNs hash all to one IP. One backend serves thousands of users.
- IPv6 reduces but doesn't eliminate this.
- Adding/removing backend reshuffles `1/N` of users.

**Verdict**: cheap, but rough in the modern internet. Avoid for public traffic.

### 4.2 Application-controlled cookie
The application sets a cookie identifying the backend (or any stable token); the LB routes by it.

```
backend sets: Cookie: BACKEND=pod-2
LB reads cookie, routes to pod-2
```

Sometimes the application sets the cookie value (e.g., a session token), and the LB hashes it.

**Pros**
- Survives client IP changes (mobile network handoffs).
- Per-user precision.

**Cons**
- Requires app cooperation.

### 4.3 LB-issued cookie ("duration-based" or "controlled" stickiness)
The LB injects a cookie on the first response and uses it to route subsequent requests.

```
LB → response → Set-Cookie: AWSALB=abc123; Path=/; Max-Age=86400
LB ← next request with cookie AWSALB=abc123 → routes to pod that 'abc123' maps to
```

AWS ALB calls these "duration-based sticky sessions". The cookie embeds either:
- The target's encrypted identity (so any LB instance can decode).
- A token mapped via the LB cluster's shared state.

**Pros**
- No app changes.
- Encrypted so client can't tamper.
- Per-LB-cluster state managed transparently.

**Cons**
- Stickiness bounded to cookie lifetime.
- Cookie can be lost on browser clear / new device.
- Doesn't work for non-cookie clients (raw API calls, mobile native).

**This is the default sticky-session mechanism for most public web apps.**

### 4.4 Header-based hash / route
Hash on a custom header. Useful for APIs.

```
header: X-Tenant-ID: 42
LB consistent-hash → tenant 42 → pod 5
```

**Pros**
- Per-tenant routing for multi-tenant apps.
- Survives without browser cookies.

**Cons**
- Client must set the header.
- Same hot-key risk if one tenant dominates.

### 4.5 Path / URL hash
Hash on the URL path or a query param. Used for cache routing.

```
GET /image/42 → consistent-hash on "/image/42" → pod that has this image cached
```

CDNs and cache hierarchies use this heavily.

### 4.6 Consistent hashing
Same idea as the above mechanisms, but the hashing algorithm minimizes reshuffling when the pool changes. The standard for sharded caches and stateful services.

```
Adding a pod: only ~1/N of users move
Removing a pod: only that pod's users move
```

See [Consistent Hashing →](../04-databases/consistent-hashing.md).

---

## 5. Sticky Session Lifecycles

Every sticky-session setup needs to answer:

- **When does affinity start?** Usually on first request (LB sets cookie) or first authenticated request.
- **How long does it last?** A duration (15 min, 1 hour, until browser close) or a count (next 100 requests).
- **What if the target dies?** LB must fall back to another backend gracefully. The user may lose their session if state was in memory.
- **What if the pool changes?** Cookie-based affinity may now point to a backend that left; LB picks another. Consistent-hash affinity minimizes this.
- **What if a deploy drains the target?** The user might need to re-authenticate or lose mid-flow state. Plan for it.

A 24-hour stickiness with cookie may seem reasonable. In production it means a user pinned across multiple deploys to a single pod's local state. When that pod is killed, every action it owned breaks. Tighter bounds (5–30 min) are usually better.

---

## 6. Right Patterns for Common Cases

### Web application with login sessions
- **Externalize sessions to Redis** (or use signed JWT cookies).
- No sticky session. All pods serve any user equally.
- Modern default.

### WebSocket chat / collaboration
- Implicit stickiness for the WebSocket connection.
- For multi-pod fanout: use a pub/sub layer (Redis Pub/Sub, NATS, Kafka) so any pod can publish events that any connected pod delivers.

### Game servers
- True ownership: one server owns one match. Sticky is correct.
- The LB routes the match-join request; further messages go directly to the assigned server.

### Sharded in-memory cache (e.g., Twemproxy in front of Memcached)
- Consistent hashing on key.
- No "user affinity"; key affinity.

### Multi-tenant SaaS with tenant-local cache
- Consistent hash on tenant ID at the L7 LB.
- Each pod owns a subset of tenants' cached data; cache hit rate is high.
- Adding pods only reshuffles `1/N` of tenants.

### Long-running upload / download
- The connection itself binds traffic to one pod.
- For chunked / resumable uploads (S3 multipart), each PUT is independent; the LB doesn't need stickiness for it.

---

## 7. Worked Example: A Multi-Tenant SaaS

You run a SaaS. Each tenant has a heavy per-tenant cache (lots of computed views). Tenants vary in size — biggest tenant is 100× the smallest.

**Bad design**: round-robin. Every pod tries to cache every tenant's views; cache misses dominate; cost is high.

**Better**: consistent hash on `tenant_id`. Each pod owns a slice of tenants. Cache hit rate goes from 50% to 95%.

**Even better**: weighted consistent hashing. Biggest tenants get virtual nodes proportional to their size, so they spread across multiple pods. The single-tenant hot-pod risk is bounded.

**Edge case**: a single tenant grows to 30% of total load. Now they need to be sharded across multiple pods themselves. Add a sub-shard: hash on `tenant_id + user_id`.

---

## 8. Sticky Sessions in Specific LBs

### NGINX
- `ip_hash` — IP-based.
- `hash <key> consistent` — consistent hashing on a variable (cookie, header, URI).
- `sticky cookie` (commercial NGINX Plus).

### HAProxy
- `cookie SERVERID insert indirect nocache` — LB-injected cookie.
- `balance source` — IP hash.
- `balance uri / hdr(...) / url_param consistent` — consistent hash.

### AWS ALB
- Duration-based stickiness with LB-managed cookie (`AWSALB`).
- Application-based stickiness with app cookie name.
- Configured per target group.

### AWS NLB
- Source-IP affinity (the only option at L4).

### Envoy / Istio
- Cookie-based stickiness via `consistent_hash` policy with `cookie` source.
- Hash-policy by header / source IP / URI / query param.
- Ring hash or Maglev hash.

### GCLB
- Generated cookie affinity.
- Client IP affinity.
- HTTP cookie / header affinity.

---

## 9. When to Avoid Sticky Sessions Altogether

If you can answer "yes" to any of these, **don't use stickiness**:

- Session state can be moved to Redis / DB / encrypted cookie.
- The cache hit rate gain is < 20%.
- Tenants / users are roughly uniform in load.
- Pods restart frequently (more than once a day).
- You want to autoscale aggressively.
- You're behind a CDN and most traffic is cached anyway.

If you can answer "yes" to most of these, **consider stickiness**:

- The state is genuinely pod-local and can't be externalized (game server, in-memory analytics window).
- Cache hit rate moves from 50% to 90%+ with affinity.
- Tenants/users have very different sizes and you can shard accordingly.
- Pods are long-lived (hours to days).
- You're OK with a small population losing session on deploy.

---

## 10. Sticky Sessions and Stateless Authentication

A common confusion: "we need sticky sessions because we use cookies for auth."

**No.** Auth via session cookie (signed token or session ID) lets *any* backend serve the user. The cookie is the identity; the cookie holder doesn't care which backend it talks to. Auth ≠ affinity.

The only time auth implies stickiness is when *session state* (cart, partial form data, real-time state) lives on the backend in memory. Push that to Redis or DB and you're free.

---

## 11. Common Mistakes

- **IP-hash for public web traffic.** NAT/CGNAT crush you.
- **Sticky-by-default in cloud LB.** Check the AWS ALB target group config; new ones don't, but inherited ones might.
- **Stickiness as a workaround for stateful code.** Externalize state; remove stickiness.
- **Stickiness with no fallback when target dies.** User sees blank page or error.
- **Cookie stickiness with very long Max-Age.** Pinned to one pod for days; survives no deploys cleanly.
- **Stickiness on the LB but not at the cluster ingress.** Two layers, two affinity schemes; chaos.
- **Forgetting client-side LB.** gRPC clients can implement their own sticky logic; LB on top may conflict.
- **No `max_ejection_percent` on sticky pools.** A sticky pod dies → its users have no fallback.
- **Sticky on read paths.** Often unnecessary. Reads can hit any backend with a shared cache.

---

## 12. Cheat Card

```
WHAT       LB pins a client/key to a specific backend

MECHANISMS
  IP hash         cheap, NAT-blind, avoid for public
  LB cookie       AWS ALB default; encrypted; per-LB-cluster
  app cookie      app sets value; LB routes
  header hash     per-tenant routing, multi-tenant APIs
  consistent hash on key — sharded caches, stateful services

WHEN OK
  game servers / stateful workloads
  sharded cache locality (consistent hash on key)
  long-lived connections (implicit)
  legacy session-in-memory (transitional)

WHEN NOT
  stateless apps with externalized sessions
  uniform tenants
  frequent restarts / autoscaling
  hot tenants you must fan out

PITFALLS
  IP-hash + NAT  → one pod buried
  long-lived cookie  → pinned across deploys
  no fallback on death  → user sees error
  sticky as a substitute for fixing state

RULE
  Make it stateless. If you can't, hash on the right key.
  Bound the stickiness window. Plan for the pod dying.
```

---

## 13. Resources

### Documentation
- **NGINX upstream sticky**: <https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky>
- **HAProxy cookie-based stickiness**: <https://www.haproxy.com/documentation/haproxy-configuration-tutorials/load-balancing/>
- **AWS ALB stickiness**: <https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html>
- **Envoy consistent_hash**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#ring-hash>

### Articles
- "Why sticky sessions are usually wrong" — Various engineering blogs.
- "Cache affinity at scale" — multi-tenant SaaS engineering writeups.
- "Sticky sessions and the curse of NAT" — Cloudflare blog.

### Videos
- ByteByteGo — "Session Affinity / Sticky Sessions".
- Hussein Nasser — "Sticky sessions explained".

### Tools
- **HAProxy**, **NGINX**, **Envoy**, **ALB**, **GCLB** — all support sticky in various forms.
- **Redis** — your session store of choice.
- **JWT libraries** — for stateless session tokens.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [Algorithms →](./algorithms.md)
- [Stateful vs Stateless Services →](../01-foundations/stateful-vs-stateless.md)
- [Consistent Hashing →](../04-databases/consistent-hashing.md)
- [Session-Based Authentication →](../12-security/sessions.md)
- [JWT →](../12-security/jwt.md)

---

*Previous:* [← Health Checks & Failover](./health-checks.md)  |  *Next:* [Global Server Load Balancing (GSLB) →](./gslb.md)
