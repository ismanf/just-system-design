# Layer 4 vs Layer 7 Load Balancing

> **TL;DR** — **L4 load balancers** route at the transport layer (TCP/UDP). They see IPs, ports, and bytes. They're fast, cheap, protocol-agnostic, and pass connections through without parsing the payload. **L7 load balancers** route at the application layer (HTTP/gRPC/TLS/WebSocket). They parse the request, see hosts, paths, headers, methods, and cookies. They can do path-based routing, header rewriting, response retries, A/B testing, mTLS, WAF — at the cost of higher CPU per request and TLS termination overhead. The rule: use **L4 for raw TCP/UDP services** (databases, message brokers, gRPC streams when path routing isn't needed), use **L7 for HTTP APIs and anywhere you need request-aware routing**. Most production stacks chain both: L4 at the edge for raw throughput and DDoS absorption, L7 inside the cluster for application-aware routing.

---

## 1. The OSI Refresher

Two layers of the OSI model are where load balancing typically happens:

```
  L7  APPLICATION   HTTP, HTTP/2, HTTP/3, gRPC, WebSocket, TLS (SNI)
  L6  PRESENTATION  ─
  L5  SESSION       ─
  L4  TRANSPORT     TCP, UDP, QUIC
  L3  NETWORK       IP
  L2  LINK          Ethernet
  L1  PHYSICAL      wires/radio
```

L4 LBs operate at the transport layer: they see source/destination IP and port. They don't open the byte stream. L7 LBs operate at the application layer: they decode the protocol (HTTP request line, headers, body), make routing decisions on content.

This isn't formally accurate — modern L7 LBs do TLS termination (L6/L5), parse SNI (L5), inspect HTTP/2 streams — but "L4 vs L7" is the working vocabulary.

---

## 2. What L4 Balancers See

```
TCP SYN → LB
  src IP: 203.0.113.7
  dst IP: 198.51.100.1  (the LB's VIP)
  src port: 50321
  dst port: 443

LB decisions:
  - pick a backend based on (src IP, port) hash, or
  - round-robin among healthy backends,
  - then relay all bytes for this connection.
```

That's it. The LB sees the 4-tuple (source IP, source port, dest IP, dest port), maybe inspects TLS SNI before forwarding, then becomes a TCP relay. The payload is opaque.

**Pros of opacity:**
- Works for any TCP/UDP protocol. Postgres, Redis, MySQL, MQTT, RTMP, anything.
- Very low per-packet cost. Modern XDP/eBPF L4 LBs (Katran, Cilium) process tens of millions of packets per second per core.
- TLS handshake terminates at the backend, not the LB — useful for mTLS and end-to-end encryption.
- Cheap. CPU-light, memory-light.

**Cons of opacity:**
- Can't route on URL path / method / header.
- Can't retry an HTTP 5xx — the LB has no idea the response was a 5xx.
- Sticky sessions only by IP (terrible behind NAT) or via connection-state tables.
- Limited observability: connection counts, byte counts, durations. No per-request metrics.

L4 LBs include: AWS NLB, GCLB TCP/UDP, Azure Load Balancer, IPVS, HAProxy in `mode tcp`, Nginx `stream` module, Envoy TCP proxy, Maglev, Katran.

---

## 3. What L7 Balancers See

```
HTTP request → LB
  GET /api/v2/products/42?ref=email HTTP/1.1
  Host: shop.example.com
  Authorization: Bearer eyJ...
  User-Agent: ...
  Accept: application/json

LB decisions:
  - match host=shop.example.com → cluster "shop"
  - match path /api/v2/* → backend pool "api-v2"
  - apply rate limit by Authorization claims
  - retry on 502/503
  - rewrite header X-Forwarded-For
  - emit metrics by URI template
```

The LB parses HTTP, applies a routing table, modifies headers, retries, can transform request/response. This is the world of **application gateways**, **API gateways**, **service mesh** — all built on L7 LBs.

**Pros of awareness:**
- Path-based routing (`/api` → service A, `/static` → CDN).
- Method-based routing (POST → write tier, GET → read replicas).
- Retry on server errors. The LB knows what a 5xx looks like.
- Header rewriting (`X-Forwarded-For`, `X-Request-ID`).
- A/B testing, canary deployments, traffic splitting by header.
- mTLS termination, WAF rules, OAuth introspection.
- Per-route metrics, latency histograms, error rates.
- HTTP/2 connection coalescing — many requests over fewer connections.

**Cons of awareness:**
- More CPU per request — parsing, allocating, copying.
- TLS termination eats CPU (mitigated by hardware accel, TLS 1.3, session resumption).
- More state to track per connection.
- Misconfiguration risk: wrong header rewrite leaks data.
- Slower than L4 for raw throughput at the same hardware.

L7 LBs include: AWS ALB, GCLB HTTPS, Azure Application Gateway, Azure Front Door, Nginx, HAProxy in `mode http`, Envoy, Traefik, Istio (Envoy), Linkerd, Caddy.

---

## 4. A Side-by-Side

| Concern | L4 | L7 |
|---|---|---|
| Inspects payload | No | Yes |
| Routes by | IP/port, SNI | Host, path, method, header, cookie, body |
| Protocols | Any TCP/UDP | HTTP/1.1, HTTP/2, HTTP/3, gRPC, WebSocket, MQTT (L7-aware) |
| TLS | Passthrough usually | Terminates (most common) |
| Retries on 5xx | No | Yes |
| Cookie / session affinity | IP-hash only | Cookie-based |
| Per-request observability | No | Yes (URI, status, latency) |
| Connection multiplexing | No | HTTP/2 / gRPC streams |
| CPU cost | Low | Medium–high |
| Throughput per core | ~1M+ conn/sec | ~50–200k req/sec |
| Latency overhead | µs | ~0.1–1 ms |
| Examples | AWS NLB, IPVS, Maglev, Katran | AWS ALB, Nginx, Envoy, HAProxy http |
| Best for | TCP services, DDoS scrubbing, raw throughput | HTTP APIs, microservices, edge |

---

## 5. When to Use Which

### Use L4 when:
- The protocol isn't HTTP. Postgres replication, Redis cluster, MQTT broker, Game UDP traffic, SMTP, IRC.
- You want **end-to-end encryption** or **end-to-end mTLS** — the backend, not the LB, terminates TLS.
- Throughput / latency is more critical than routing intelligence (think 10+ Gbps).
- You're already handling app-aware routing elsewhere (service mesh inside the cluster).

### Use L7 when:
- The protocol is HTTP/gRPC and you need any of: path routing, header-based routing, retries, header rewriting, A/B testing, mTLS termination + identity propagation.
- You want **per-route metrics and observability**.
- You want **request-level retry semantics** (retry idempotent GETs on 502/503).
- You're building an API gateway, service mesh, or ingress.

### Use both (the common case):
```
client → L4 edge LB (TLS passthrough, DDoS absorb) → L7 ingress (path routing) → service
```

AWS commonly: **NLB** (L4) in front of **ALB** (L7) in front of pods. Or **CloudFront** (L7 CDN) in front of **ALB**. Or **NLB** in front of **Envoy** in front of pods.

---

## 6. The Subtle Cases

### SNI-based L4 routing
A pure L4 LB can peek at the **TLS Server Name Indication** during the handshake without terminating TLS. That gives it just enough info to route by hostname while preserving end-to-end encryption.

```
client TLS → LB sees SNI=api.example.com → routes to api cluster
LB doesn't decrypt; backend terminates TLS.
```

Used by: AWS NLB (TLS listener with `SslPolicy`), HAProxy `tcp` mode with `req.ssl_sni`, Nginx `stream` with `ssl_preread`. Useful when you have several services on the same IP but want passthrough TLS.

### HTTP/2 and gRPC
HTTP/2 multiplexes many concurrent streams over one TCP connection. An L4 LB **cannot route per-stream** — it picks a backend at connection-setup time and stays there. If you have multiple gRPC clients connecting through an L4 LB, each client gets pinned to one backend and traffic skews badly.

The fix: use **L7 LB with HTTP/2 awareness** so it can balance individual streams. Or use **client-side load balancing** (gRPC's built-in round-robin resolver).

This is one of the most common production gotchas with gRPC.

### WebSocket / long-lived connections
WebSockets are HTTP/1.1 with `Upgrade: websocket`. L7 LBs handle the upgrade and then become an L4-style relay for the duration. L4 LBs treat it as plain TCP.

Trade-off: an L7 LB can route the initial HTTP request smartly, then hand off. An L4 LB is dumber but cheaper. For high-fan-out WebSocket services (real-time gaming, chat) most teams put an L7 LB at the front for routing and an L4 LB or direct backend connections behind.

### UDP / QUIC
UDP is L4-only. QUIC (the basis of HTTP/3) is UDP underneath but L7-aware at the protocol layer. Modern LBs (Envoy, GCLB, Cloudflare) handle QUIC natively. AWS NLB supports UDP.

---

## 7. Performance Numbers, Approximate

These are ballpark figures from production reports and benchmarks. Your numbers will differ.

| LB | Type | Throughput (per instance) | Latency added |
|---|---|---|---|
| **Maglev / Katran** (eBPF L4) | L4 | 10–40 Gbps per core | < 50 µs |
| **AWS NLB** | L4 | millions of conn/sec, line rate | < 100 µs |
| **HAProxy (tcp mode)** | L4 | 1–2M conn/sec on commodity CPU | < 200 µs |
| **IPVS** | L4 (kernel) | very high; CPU-bound | < 100 µs |
| **AWS ALB** | L7 | autoscaling; ~25k RPS per LCU | 1–10 ms |
| **Nginx** | L7 | 50–150k RPS per instance | 0.5–2 ms |
| **HAProxy (http)** | L7 | 100k+ RPS per instance | 0.3–1 ms |
| **Envoy** | L7 | 50–100k RPS per instance | 0.5–2 ms |

The pattern: L4 LBs are 10–100× cheaper per packet than L7. For the same hardware, you get an order of magnitude more throughput at L4.

---

## 8. Cost: TLS Termination at L7

TLS terminates at L7 LBs. The handshake is the expensive part — typically 1–5 ms of CPU on commodity hardware per new connection.

Mitigations:
- **TLS 1.3** cuts handshake to 1-RTT (or 0-RTT for resumption).
- **Session resumption** — reuse TLS sessions across connections.
- **ECDSA certs** are 5–10× faster handshake than RSA. Use ECDSA wherever possible.
- **TLS offload hardware** — modern CPUs have AES-NI, ARM has crypto extensions.
- **Keep-alive** — fewer handshakes per request.
- **HTTP/2** — one TLS connection serves many requests.

A modern L7 LB on commodity hardware can do ~10–30k TLS handshakes/sec per core. If you're on a high-volume site, this is the dominant CPU cost on your LB tier.

---

## 9. Architectural Pattern: The L4+L7 Sandwich

Almost every large stack ends up with:

```
            ─────  user  ─────
                    │
                    ▼
  ┌──────────────────────────────────┐
  │ CDN  (L7 cache + edge TLS)       │   global, anycast
  └──────────────────────────────────┘
                    │
                    ▼
  ┌──────────────────────────────────┐
  │ Edge L4 LB (NLB / Maglev / Katran)│   DDoS, raw throughput, anycast
  └──────────────────────────────────┘
                    │
                    ▼
  ┌──────────────────────────────────┐
  │ Cluster L7 ingress (Envoy/Nginx) │   path routing, retries, mTLS terminate
  └──────────────────────────────────┘
                    │
                    ▼
  ┌──────────────────────────────────┐
  │ Service mesh sidecar (Envoy)     │   pod-to-pod L7, mTLS to each peer
  └──────────────────────────────────┘
                    │
                    ▼
                  backend
```

Each layer does what it's best at. Edge L4 absorbs raw packet floods. CDN absorbs cacheable HTTP. Cluster L7 does the smart routing. Mesh sidecar handles pod-to-pod. The backend never sees a raw TLS handshake or a raw packet flood.

---

## 10. Migration Stories

### "We outgrew our L7-only setup"
Single NGINX-tier in front of services. As traffic grew past 50k RPS, the TLS termination CPU became the bottleneck. Solution: NLB in front of the Nginx tier for fast TCP load balancing across many Nginx instances. Each Nginx now handles a smaller slice.

### "We outgrew our L4-only setup"
A gRPC-heavy service balanced behind an L4 LB had hot pods because long-lived HTTP/2 connections pinned to single backends. Solution: introduced Envoy as an L7-aware proxy that re-balances per-stream.

### "We outgrew our ALB"
ALB latency budget got tight. Switched to NLB + Envoy for sub-ms internal routing while keeping the public IP behind NLB for stability.

### "We don't need an LB tier"
Tiny service with one pod and a service mesh that does client-side LB. Skip the dedicated LB. Works at small scale; doesn't scale to public traffic, DDoS, or 100+ pods.

---

## 11. Worked Example: Choosing for a New Service

Service: a Postgres + Node API + gRPC backend, deployed in Kubernetes on AWS.

**Public HTTP API**
- AWS ALB at the edge. L7 — does path routing for `/api/v2/*` to a new service, TLS termination, WAF rules. Logs every request.

**Public gRPC API**
- AWS NLB (L4) for low-latency raw TCP. Plus per-pod Envoy sidecars doing L7 gRPC balancing inside the cluster (or use ALB with HTTP/2 support, which is fine for moderate gRPC scale).

**Internal Postgres replica reads**
- HAProxy in `tcp` mode with health checks via Patroni endpoints. Pure L4.

**Internal Redis Cluster**
- No LB. Clients use Redis Cluster's smart routing directly. (The cluster IS the routing.)

**Real-time WebSocket service**
- ALB for the initial upgrade and routing by path; ALB transitions to long-lived after upgrade. Sticky sessions on if pods are stateful.

The mix is normal. The skill is picking the right one per case.

---

## 12. Common Mistakes

- **Using L4 for HTTP and missing out on retries / observability.** "We need per-route metrics" → you needed L7 a year ago.
- **Using L7 for raw TCP services**. Postgres doesn't speak HTTP. The LB just adds latency.
- **L4 LB for gRPC with many clients to few backends.** Backends get pinned; load skews. Fix with L7 LB or client-side balancing.
- **TLS termination at L7 but no HSTS / cert rotation.** Pile up of weak ciphers and forgotten certs.
- **Sticky sessions by IP** — NAT'd clients collapse to one backend. Use cookies or stateless services.
- **Mixing L4 and L7 SNI without thinking about cert placement.** L4 with passthrough means the backend owns the cert. L7 with termination means the LB owns it.
- **Health check at the wrong layer.** Doing TCP-only checks (L4) on an HTTP service can keep dead apps in rotation if the TCP socket still opens.
- **Forgetting connection-draining when rolling deploys at L7.** L7 LBs hold persistent upstream connections; you must drain before terminating pods.

---

## 13. Cheat Card

```
L4              transport layer: IP + port + (maybe) SNI
                fast, opaque, protocol-agnostic
                used for: TCP/UDP services, DDoS edge, raw throughput
                e.g., NLB, Maglev, Katran, IPVS, HAProxy tcp

L7              application layer: HTTP, gRPC, headers, paths
                smart, expensive, observable
                used for: HTTP APIs, ingress, service mesh
                e.g., ALB, Nginx, HAProxy http, Envoy, Traefik

CHOOSE L4 IF    payload doesn't matter, end-to-end TLS,
                throughput > smarts, raw TCP/UDP
CHOOSE L7 IF    HTTP/gRPC, path/header routing,
                retries, per-route metrics, mTLS

SNI ROUTING     L4 can peek at TLS SNI without decrypting

GRPC GOTCHA     L4 pins connections → skewed load.
                Use L7 LB or client-side LB.

PATTERN         CDN → edge L4 → cluster L7 → mesh sidecar → pod

RULE            Use the dumbest layer that still does the job.
```

---

## 14. Resources

### Books
- *Site Reliability Engineering* — Google SRE Book, "Load Balancing at the Frontend" and "Load Balancing in the Datacenter".
- *HTTP/2 in Action* — Barry Pollard.

### Papers
- "Maglev: A Fast and Reliable Software Network Load Balancer" — Eisenbud et al., NSDI 2016.
- "Beyond Jain's Fairness Index" — various.

### Documentation
- **AWS NLB vs ALB**: <https://aws.amazon.com/elasticloadbalancing/features/>
- **Envoy L4 vs L7 filters**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/network_filters>
- **HAProxy modes**: <https://www.haproxy.com/documentation/haproxy-configuration-tutorials/load-balancing/>

### Articles
- "L4 vs L7 load balancing" — Cloudflare Learning.
- "gRPC load balancing" — gRPC docs.
- "Beyond round-robin: load balancing inside Google" — Google Cloud blog.

### Videos
- ByteByteGo — "L4 vs L7 Load Balancers".
- Hussein Nasser — Multiple deep dives on L4/L7 mechanics.

### Tools
- L4: **AWS NLB**, **Katran**, **Maglev**, **IPVS**, **HAProxy tcp**.
- L7: **Nginx**, **HAProxy http**, **Envoy**, **AWS ALB**, **Traefik**, **Caddy**.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [Algorithms →](./algorithms.md)
- [Health Checks →](./health-checks.md)
- [Sticky Sessions →](./sticky-sessions.md)
- [LB Implementations →](./lb-implementations.md)
- [TCP vs UDP →](../02-networking/tcp-vs-udp.md)
- [HTTP Versions →](../02-networking/http-versions.md)
- [gRPC, Protobuf, Thrift →](../02-networking/grpc-protobuf.md)

---

*Previous:* [← Load Balancer Fundamentals](./load-balancer-basics.md)  |  *Next:* [Load Balancing Algorithms →](./algorithms.md)
