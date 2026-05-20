# Load Balancer Fundamentals

> **TL;DR** — A **load balancer** is a device or software that sits in front of a pool of servers and decides where each incoming connection or request goes. It exists for three reasons: **scale** (one box can't do it all), **availability** (one box can fail and traffic keeps flowing), and **operability** (you can deploy, drain, and replace servers without dropping requests). Load balancers live at OSI Layer 4 (TCP/UDP) or Layer 7 (HTTP/gRPC/TLS), come in hardware, software, and cloud-managed flavors, and choose targets via algorithms like round-robin, least-connections, or consistent hashing. The decisions that matter: **algorithm**, **health-check strategy**, **stickiness**, **TLS placement**, **failover topology**, and **how the LB itself stays available** (anycast, ECMP, HA pairs).

---

## 1. Why Load Balance

The simplest production system has three problems the moment it gets popular:

1. **A single server can't handle the traffic.** You scale horizontally — N servers behind one address.
2. **Any single server will eventually fail.** A request landing there must not fail with it.
3. **You need to deploy without downtime.** Drain a server, replace it, return it.

A load balancer solves all three by abstracting "a fleet of servers" behind one stable endpoint.

```
                ┌─────────────┐
                │     LB      │   one address (DNS, anycast IP)
                └──────┬──────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌────────┐    ┌────────┐    ┌────────┐
   │ pod 1  │    │ pod 2  │    │ pod 3  │
   └────────┘    └────────┘    └────────┘
   any can fail; LB skips it. any can be added; LB picks it up.
```

Almost every layer of modern infrastructure uses load balancing somewhere: DNS-based for global; anycast for edge networks; L4 for TCP services; L7 for HTTP APIs; service-mesh sidecars for pod-to-pod.

---

## 2. The Job Description

What a load balancer actually does, in order on each new connection:

1. **Accept the connection** on its public address (TCP socket, TLS handshake, sometimes HTTP/2 stream).
2. **Match a routing rule** — listener / host / path / SNI / port.
3. **Pick a backend** from a healthy target pool using an algorithm.
4. **Forward** the traffic (TCP relay, HTTP proxying, or DSR).
5. **Track liveness** of backends via active and passive health checks.
6. **Decide stickiness** if needed (session affinity).
7. **Apply policy** — rate limits, retries, timeouts, header rewrites, mTLS.
8. **Emit telemetry** — access logs, metrics, traces.

The complexity scales with the layer: an L4 LB just relays bytes. An L7 LB parses HTTP, routes by URL, retries on 5xx, modifies headers.

---

## 3. Where Load Balancers Live in the Stack

```
                       ┌────────────────┐
                       │   user (web)   │
                       └────────┬───────┘
                                │
                     ┌──────────▼──────────┐
                     │   Global DNS / GSLB │  ← geo / latency / failover
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │  CDN / Edge anycast │  ← caching, DDoS, TLS
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │ Public LB (L4/L7)   │  ← AWS ALB/NLB, ELB, GCLB, Azure FD
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │ Ingress (Nginx,     │  ← in-cluster path routing
                     │  HAProxy, Envoy)    │
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │ Service mesh sidecar│  ← pod-to-pod L7, mTLS, traffic split
                     └──────────┬──────────┘
                                │
                              backend
```

Real systems chain three or four LBs without thinking. Each layer optimizes for a different concern.

---

## 4. L4 vs L7 (One-Paragraph Preview)

- **L4** balances **connections** based on IP and port. Fast, transparent, doesn't inspect payload. Great for TCP/UDP services where the LB just relays bytes (databases, gRPC streams, raw sockets).
- **L7** balances **requests** based on application data (HTTP path, host, headers, cookies). More expensive but allows path-based routing, header rewriting, A/B testing, advanced retries.

Full discussion in [L4 vs L7 →](./l4-vs-l7.md).

---

## 5. Algorithms (One-Paragraph Preview)

Pick a backend by:
- **Round-robin** — simplest.
- **Weighted round-robin** — unequal pool sizes.
- **Least-connections** — best for varying request durations.
- **Least-response-time / EWMA** — adapts to backend latency.
- **IP-hash / consistent hashing** — same client → same backend.
- **Power-of-two-choices** — pick two random, choose the less loaded. Excellent fairness at almost no cost.
- **Random** — surprisingly competitive at high pool sizes.

Full discussion in [Algorithms →](./algorithms.md).

---

## 6. Health Checks (One-Paragraph Preview)

The LB pulls a target out of rotation when it's unhealthy. Two flavors:
- **Active** — periodic probes (TCP connect, HTTP `/health`, gRPC health protocol).
- **Passive** — observe live traffic; eject on errors.

Full discussion in [Health Checks →](./health-checks.md).

---

## 7. Stickiness (One-Paragraph Preview)

Sometimes you want a specific client to always hit the same backend — for session affinity, in-memory caches, WebSocket continuity. Sticky sessions are usually a workaround for a stateful service that should be stateless. Full discussion in [Sticky Sessions →](./sticky-sessions.md).

---

## 8. TLS Placement

The LB usually terminates TLS for incoming HTTPS connections. There are three patterns:

```
1. SSL TERMINATION                client ──TLS──► LB ──HTTP──► backend
   simplest; LB is the only TLS endpoint; backend traffic is plaintext (private network)

2. SSL PASSTHROUGH (L4 only)      client ──TLS──► LB ──TLS──► backend
   LB sees encrypted bytes; can't route on HTTP; mTLS works end-to-end

3. SSL BRIDGING / RE-ENCRYPTION   client ──TLS──► LB ──TLS──► backend
   LB terminates AND re-encrypts to backend; can route on HTTP and still encrypt internally
```

For most public-facing HTTPS APIs, **termination** is the default. For compliance-sensitive environments and gRPC mTLS, **bridging** is common. For pure TCP services (Postgres, Redis Cluster), **passthrough** is the only option.

TLS termination is expensive (handshake CPU). LBs benefit from session resumption, TLS 1.3 0-RTT, and ECDSA certs to keep CPU manageable.

---

## 9. Connection Lifecycle: What the LB Tracks

The LB is stateful even if your backends are not. It tracks:

- **Open connections** per backend (for least-conn algorithm).
- **Recent latencies / errors** per backend (for EWMA, outlier ejection).
- **Health check state** per backend.
- **Persistent connections to upstreams** (HTTP keep-alive, HTTP/2 streams, gRPC channels).
- **Sticky-session tables** if affinity is enabled.

This state is what makes a multi-instance LB tier interesting. Either:
- LBs **share state** (rare, expensive — ZooKeeper, etcd, gossip).
- LBs **work independently** and accept some inconsistency (the typical case — round-robin per LB is fine; sticky-session keying must be deterministic across LBs).

Cloud LBs (ALB, NLB, GCLB) abstract this; you just configure the policy. Self-hosted (HAProxy, Nginx, Envoy) you operate it.

---

## 10. The LB Itself Must Be Available

A load balancer is by definition a critical path. Its own failure modes:

- **Single instance** — never. One process, one bug, no traffic.
- **HA pair** — active/passive with VRRP/keepalived. Sub-second failover, but only doubles capacity.
- **Active/active with ECMP** — multiple LB instances behind ECMP hashing in the router. Linear scale + redundancy.
- **Anycast** — same IP advertised from many locations; BGP routes to the nearest healthy one. Used by CDNs and cloud LBs.
- **DNS round-robin to multiple LBs** — fallback when nothing else is available. Slow failover (DNS TTL).

Cloud-managed LBs (AWS ALB/NLB, GCLB, Azure Front Door) handle all of this for you, scaling and failing over transparently.

---

## 11. Direct Server Return (DSR) and Modern Variants

A traditional LB sees both directions of traffic — requests in, responses out. Throughput bound by LB bandwidth.

**Direct Server Return (DSR)** sends requests through the LB but lets backends reply directly to clients, bypassing the LB on the return path.

```
   client ──► LB ──► backend
        ◄─────────── backend  (response skips LB)
```

DSR was huge for high-bandwidth services (video, downloads) because the response is typically much larger than the request.

Modern variants:
- **L3DSR** with tunneling (IPIP, GRE) — backends see the original client IP via tunneling.
- **Maglev** (Google) — consistent-hash LB with DSR; published paper 2016. Cloudflare's Unimog is similar.
- **Katran** (Facebook) — eBPF/XDP based L4 LB. Open source.
- **AWS NLB** — does DSR-like behavior at scale.

For ultra-high-throughput L4 LB, XDP/eBPF-based designs are state-of-the-art.

---

## 12. The Three LB Form Factors

### 12.1 Hardware
F5 BIG-IP, Citrix NetScaler, A10. Purpose-built appliances. Excellent throughput per unit; expensive; on-prem. Used in financial services, telco, and big enterprises. Declining outside those.

### 12.2 Software
Nginx, HAProxy, Envoy, Traefik. Run on commodity hardware. Free and open. The default for most companies. Operational burden is yours.

### 12.3 Cloud-managed
AWS ELB classic, ALB (L7), NLB (L4), GCLB (L4/L7), Azure Load Balancer / Application Gateway, Azure Front Door. You pay per hour and per request. They scale automatically and handle redundancy.

For a green-field deployment in 2026, the choice is usually:
- **Cloud-managed** for the edge LB.
- **Envoy/Nginx/HAProxy** for in-cluster ingress and service mesh.
- **Hardware** only if you have specific compliance or perf requirements.

Full per-tool comparison in [LB Implementations →](./lb-implementations.md).

---

## 13. Real-World Examples

- **Netflix** uses AWS ELB for the public edge, Zuul as the application gateway, and Eureka for service discovery — Ribbon (client-side LB) inside Java services.
- **Google** uses Maglev (custom L4 + consistent hash + ECMP) and GFE (the global HTTP frontend) — billions of requests/sec.
- **Cloudflare** uses Unimog (their own eBPF L4 LB) and proxies all traffic through Cloudflare-built workers + reverse proxies.
- **Stripe** uses NLB + ALB at the edge, Envoy as the internal mesh.
- **Discord** uses Cloudflare at the edge, HAProxy + Envoy internally.
- **Slack** uses HAProxy + Envoy.

There's no single answer. The pattern is: cloud at the edge, software in the middle.

---

## 14. What's Easy to Get Wrong

The classic LB mistakes:

- **No health checks** — dead pods get traffic forever.
- **Health checks that lie** — `/health` returns 200 even when the DB is unreachable.
- **Wrong algorithm** — round-robin on heterogeneous backends, where some are 10× slower.
- **No idle-connection drain** — deploys interrupt in-flight requests.
- **Sticky sessions by default** — defeats horizontal scale; one backend gets hot.
- **TLS termination without HSTS** — downgrade attacks.
- **No timeouts** — slow backends block the LB.
- **Insufficient backend connection pool** on the LB → exhaustion under load.
- **One LB layer** — flat single LB tier with no edge protection in front.
- **Stickiness via client IP** — NAT'd clients all hash to one backend.

We cover these in detail in the specific topic pages.

---

## 15. A Mental Picture of the Decision Space

You build a load-balancing setup by answering, in order:

1. **What protocol?** TCP/UDP raw → L4. HTTP/gRPC → L7. Both — different LBs in front.
2. **Where in the stack?** Global edge / public LB / cluster ingress / mesh sidecar.
3. **Cloud-managed or self-hosted?** Time-to-value vs control.
4. **Algorithm?** Round-robin is fine until it isn't. Then least-connections. Then power-of-two.
5. **Sessions stateful?** If yes, fix the stateful service first, sticky sessions second.
6. **TLS?** Terminate at the edge LB usually. Bridge if compliance demands.
7. **Health check strategy?** Active + passive; outlier ejection; bounded eviction.
8. **Failover?** Multi-AZ minimum; multi-region if SLOs demand.
9. **Observability?** Access logs + RED metrics from day one.

If you answer all nine before turning the LB on, you're already ahead of most production deployments.

---

## 16. Common Mistakes

- **Treating the LB as a simple TCP relay** when you have an HTTP API — you lose visibility into request-level retries, timeouts, and observability.
- **Mixing AZ traffic without zone-affinity** — cross-AZ data transfer can dominate cloud bills.
- **Single LB instance behind DNS** — outage = global outage.
- **No connection draining** — every deploy drops in-flight requests.
- **Long health-check intervals** — dead backends serve traffic for minutes.
- **Short health-check intervals + sensitive thresholds** — flapping ejects healthy backends.
- **Sticky sessions everywhere as default** — load imbalance and migration pain.
- **HTTP keepalive timeout mismatched between LB and backend** — random "connection reset" errors.
- **Forgot the LB needs telemetry too** — when the LB drops requests, you find out from customers, not dashboards.

---

## 17. Cheat Card

```
WHY              scale, availability, ops (deploys, drains)

JOB              accept → match → pick → forward → check → policy → log

LAYERS           L4 (TCP/UDP, fast, opaque)
                 L7 (HTTP/gRPC, smart, expensive)

ALGORITHMS       round-robin · weighted · least-conn ·
                 least-response · ip-hash / consistent-hash ·
                 power-of-two-choices · random

TLS              terminate · passthrough · bridge

HEALTH CHECKS    active probes + passive ejection

LB SCALE         ECMP · anycast · HA pair · cloud-managed

FORM FACTOR      hardware (declining) · software · cloud-managed

PITFALLS         no health checks, no drain, sticky by default,
                 wrong algorithm for heterogeneous pool,
                 single LB instance

RULE             A load balancer that you can't see, monitor,
                 deploy without, and fail without is not done.
```

---

## 18. Resources

### Books
- *Site Reliability Engineering* — Google SRE Book, especially "Load Balancing at the Frontend".
- *Web Performance in Action* — Jeremy Wagner.

### Papers
- "Maglev: A Fast and Reliable Software Network Load Balancer" — Google, NSDI 2016.
- "Beamer: Stateless, Resilient Load Balancing" — UCL.
- "Unimog — Cloudflare's edge load balancer" — Cloudflare blog (multi-part).

### Documentation
- **AWS ELB**: <https://docs.aws.amazon.com/elasticloadbalancing/>
- **Envoy**: <https://www.envoyproxy.io/docs>
- **HAProxy**: <https://www.haproxy.com/documentation/>
- **NGINX**: <https://nginx.org/en/docs/>

### Articles
- "Open-sourcing Katran, a scalable network load balancer" — Facebook Engineering.
- "Load Balancing is impossible" — Tyler Treat (clear-eyed on trade-offs).
- "Choosing a load balancing algorithm" — HAProxy and NGINX blogs.

### Videos
- ByteByteGo — "Load Balancing Algorithms".
- Hussein Nasser — "Reverse Proxy, Load Balancer, API Gateway".

### Tools
- **HAProxy**, **NGINX**, **Envoy**, **Traefik**, **Caddy**.
- **AWS ELB/ALB/NLB**, **GCLB**, **Azure Front Door**.
- **Katran**, **Maglev** (academic / FOSS implementations).

### Adjacent reading
- [L4 vs L7 →](./l4-vs-l7.md)
- [Algorithms →](./algorithms.md)
- [Health Checks →](./health-checks.md)
- [Sticky Sessions →](./sticky-sessions.md)
- [GSLB →](./gslb.md)
- [Proxies →](./proxies.md)
- [LB Implementations →](./lb-implementations.md)
- [DNS →](../02-networking/dns.md)
- [TCP vs UDP →](../02-networking/tcp-vs-udp.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Layer 4 vs Layer 7 Load Balancing →](./l4-vs-l7.md)
