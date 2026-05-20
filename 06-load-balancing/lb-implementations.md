# Nginx, HAProxy, Envoy, AWS ELB/ALB/NLB

> **TL;DR** — The load-balancing market has consolidated around a handful of tools. **Nginx** is the most-deployed reverse proxy in the world — fast, well-documented, easy to learn, but its config language is showing its age. **HAProxy** is the load-balancing specialist — superb tracing, granular tuning, the go-to for high-stakes L4/L7 in self-hosted environments. **Envoy** is the modern, cloud-native, xDS-driven proxy — the substrate of Istio, Linkerd2-proxy precursor, Consul Connect, Cloudflare's "Pingora" predecessor; designed for service mesh and dynamic config. **AWS ELB family** — Classic (legacy), **ALB** (L7), **NLB** (L4, ultra-fast), and **GLB** (Gateway, niche) — gives you cloud-managed, scalable, integrated LBs without operating servers. There is no universal "best." Choose by **layer needed** (L4 vs L7), **environment** (cloud-managed vs self-hosted), **dynamism** (static config vs frequent updates), and **observability needs**.

---

## 1. The Landscape

```
                          static config        dynamic config (xDS, K8s API)
                       ┌─────────────────┬─────────────────────────┐
   self-hosted (FOSS)  │  NGINX, HAProxy │  Envoy, Traefik, Caddy  │
                       ├─────────────────┼─────────────────────────┤
   commercial          │  NGINX Plus,    │  Istio, Kong, Tyk       │
                       │  HAProxy Enter. │                         │
                       ├─────────────────┼─────────────────────────┤
   cloud-managed       │  AWS ELB (old)  │  AWS ALB/NLB, GCLB,     │
                       │                 │  Azure Front Door       │
                       └─────────────────┴─────────────────────────┘
```

The four tools in this page's title cover ~80% of the production load-balancing market. Knowing them in some depth covers most engineering interviews and real-world decisions.

---

## 2. NGINX

### What it is
A high-performance HTTP server, reverse proxy, and load balancer. Originally written by Igor Sysoev (2004) to solve the C10K problem with an async event loop. Available as FOSS NGINX, NGINX Plus (commercial), and as the basis for many products (OpenResty, Tengine, Kong).

### Strengths
- **Mature, ubiquitous** — runs more public websites than any other web server.
- **Easy to learn** — config files read like English (mostly).
- **High throughput** — easily handles 50k–150k RPS per instance.
- **Static content** — fastest in class for serving files.
- **Caching** — built-in HTTP cache with rich invalidation.
- **TLS** — robust, well-tested.
- **Ecosystem** — every cloud has Nginx images; every CDN can sit in front; every CMS knows about it.

### Weaknesses
- **Config syntax** — directives, inheritance, regex blocks. Subtle bugs from accidental `if` inside `location`.
- **Static config by default** — reload (`nginx -s reload`) for changes; can be made dynamic via NGINX Plus' API.
- **Less modern HTTP/3, gRPC support** vs Envoy.
- **Observability** — basic access logs and stub_status; full observability via NGINX Amplify or third-party (Prometheus exporter).
- **No native xDS** — for dynamic K8s service-discovery you need NGINX Ingress Controller or NGINX Plus.

### A minimal reverse-proxy config

```nginx
http {
    upstream app {
        least_conn;
        server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
        server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate     /etc/ssl/certs/api.crt;
        ssl_certificate_key /etc/ssl/private/api.key;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

### Where Nginx fits today
- **Public-facing reverse proxy** for moderate-traffic sites.
- **Static file server** + ingress for traditional apps.
- **K8s ingress** via `ingress-nginx`, the most-popular Ingress Controller.
- **API gateway lite** when you don't need full Envoy/Kong.

---

## 3. HAProxy

### What it is
A load-balancing-first reverse proxy, written by Willy Tarreau (2000). Famously fast and famously stable — the canonical example of a tool designed for one job and excellent at it.

### Strengths
- **L4 and L7** — both modes in one binary.
- **Very fast** — typical numbers: 1M+ TCP connections, 200k+ HTTP RPS per instance.
- **Sophisticated load-balancing options** — far more algorithms (`roundrobin`, `leastconn`, `source`, `uri`, `url_param`, `random`, `static-rr`, `first`).
- **Excellent observability** — stats page, detailed logs, runtime API for live introspection.
- **Health checks** — granular, scriptable, including passive observation.
- **Stick tables** — keep state across connections (for rate limits, session affinity, abuse detection).
- **Dynamic via Runtime API** — change config without reload.
- **ACL system** — powerful matching DSL.

### Weaknesses
- **Less of a static web server** — you wouldn't pick it to serve files.
- **Config DSL** is dense and idiosyncratic; learning curve.
- **HTTP/3 / QUIC** support is later than Envoy.
- **Smaller ecosystem of third-party modules** than Nginx.

### A minimal LB config

```haproxy
global
    maxconn 100000
    log /dev/log local0

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    option httplog
    option dontlognull
    option forwardfor
    option http-server-close

frontend https_in
    bind *:443 ssl crt /etc/ssl/api.pem alpn h2,http/1.1
    default_backend app

backend app
    balance leastconn
    option httpchk GET /health
    http-check expect status 200
    server app1 10.0.0.1:8080 check inter 5s fall 3 rise 2 slowstart 30s
    server app2 10.0.0.2:8080 check inter 5s fall 3 rise 2 slowstart 30s
```

### Where HAProxy fits today
- **High-stakes L4/L7** — financial, ad-tech, ultra-high-throughput sites.
- **TCP load balancing** for databases, Redis, custom protocols.
- **Self-hosted environments** where you want fine control.
- **Rate limiting / abuse mitigation** via stick tables.

Reddit, Twitter (historically), Tumblr, GitHub (alongside others), Stack Overflow have all run HAProxy in production. Stack Overflow famously runs almost everything through HAProxy with very few instances.

---

## 4. Envoy

### What it is
A modern L7 (and L4) proxy designed at Lyft for cloud-native microservices. Open-sourced 2017. The data plane for Istio, AWS App Mesh, Consul Connect, and many service meshes. Configured via a runtime control plane (xDS) — dynamic by design.

### Strengths
- **Dynamic config (xDS)** — listeners, clusters, routes, endpoints, secrets all updatable at runtime from a control plane.
- **First-class HTTP/2, gRPC, HTTP/3** — designed for them.
- **Service mesh sidecar** — the most-deployed proxy inside Kubernetes pods.
- **Observability** — rich metrics (StatsD/Prometheus), tracing (Jaeger/Zipkin), access logs with structured output.
- **Extensibility** — Wasm filters, Lua filters, custom HTTP filters.
- **Advanced features** — circuit breakers, retries, outlier ejection, hedging, traffic mirroring, fault injection.
- **Modern codebase** — C++14, multi-threaded, NUMA-aware.

### Weaknesses
- **Complexity** — the config is YAML/JSON-heavy and verbose. Not for casual use.
- **Memory footprint** — bigger than HAProxy.
- **xDS requires a control plane** (Istio, App Mesh, custom).
- **Smaller community** than Nginx for vanilla web serving.

### A minimal Envoy config

```yaml
static_resources:
  listeners:
  - address:
      socket_address: { address: 0.0.0.0, port_value: 443 }
    filter_chains:
    - transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain: { filename: "/etc/ssl/api.crt" }
              private_key:       { filename: "/etc/ssl/api.key" }
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress
          route_config:
            virtual_hosts:
            - name: api
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: app
                  retry_policy: { retry_on: 5xx, num_retries: 2 }
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: app
    type: STRICT_DNS
    connect_timeout: 2s
    lb_policy: LEAST_REQUEST
    load_assignment:
      cluster_name: app
      endpoints:
      - lb_endpoints:
        - endpoint: { address: { socket_address: { address: app-pod-1, port_value: 8080 } } }
        - endpoint: { address: { socket_address: { address: app-pod-2, port_value: 8080 } } }
    health_checks:
    - timeout: 2s
      interval: 5s
      unhealthy_threshold: 3
      healthy_threshold: 2
      http_health_check: { path: "/health" }
    outlier_detection:
      consecutive_5xx: 5
      base_ejection_time: 30s
      max_ejection_percent: 50
```

### Where Envoy fits today
- **Service mesh sidecar** — Istio, Consul, Linkerd2-proxy uses Linkerd's own proxy (Rust); Envoy is the dominant sidecar.
- **API gateway** — via Emissary-ingress, Gloo, Tetrate, AWS App Mesh.
- **Edge LB at scale** — Lyft, Stripe, Netflix's Zuul-like, and a growing list of edge stacks.
- **Anywhere you want xDS + observability** — Envoy is the cleanest choice.

---

## 5. AWS ELB Family

AWS's load balancer offerings, from oldest to newest:

### 5.1 Classic Load Balancer (CLB)
- The original. EC2-Classic era.
- Supports HTTP, HTTPS, TCP, SSL.
- Limited features compared to ALB/NLB.
- **Use only if you're on EC2-Classic** (rare). Otherwise — migrate.

### 5.2 Application Load Balancer (ALB) — L7
- HTTP/HTTPS/gRPC/WebSocket.
- Host-based and path-based routing.
- Native authentication (Cognito / OIDC).
- WAF integration.
- HTTP/2 in front; HTTP/1.1 or HTTP/2 to targets.
- Target groups can be EC2, ECS, Lambda, IP addresses.
- TLS termination with ACM certificates (free for AWS-issued).
- **Pricing**: hourly + LCU (Load Balancer Capacity Units, a composite of new connections, active connections, processed bytes, rule evaluations).

**Strengths**
- Tight AWS integration; auto-scaling target groups.
- WAF, Shield, CloudWatch integration.
- Sticky sessions built in.
- gRPC support since 2020.

**Weaknesses**
- L7 only. Doesn't do UDP or non-HTTP TCP.
- Per-request latency adds ~few ms.
- Limited routing rule complexity vs Envoy.
- "Idle timeout" minimum 60s, max 4000s.

### 5.3 Network Load Balancer (NLB) — L4
- TCP, UDP, TLS (passthrough).
- Ultra-high throughput, very low latency (< 100 µs added).
- Preserves source IP (no `X-Forwarded-For` — source IP is real).
- Static IP per AZ; supports Elastic IPs.
- Can handle millions of connections.
- **Pricing**: hourly + NLCU.

**Strengths**
- Lowest latency in the ELB family.
- Static IPs — useful for firewall allowlisting.
- Handles non-HTTP protocols (Postgres, Redis, MQTT, etc.).
- Passes source IP directly to backend.

**Weaknesses**
- L4 only — no path routing, no HTTP-aware retries.
- Can't terminate TLS at L7 features; only TLS passthrough or termination without HTTP logic.
- Sticky only by source IP.

### 5.4 Gateway Load Balancer (GWLB)
- Used for inserting third-party network appliances (firewalls, deep packet inspection) into traffic flow.
- L3 with GENEVE encapsulation.
- Niche; mostly for security-stack vendors.

### 5.5 ALB vs NLB Decision

| Use case | Pick |
|---|---|
| HTTP/HTTPS API, web app | **ALB** |
| gRPC | ALB or NLB (NLB if low latency critical) |
| WebSocket | ALB |
| TCP service (Postgres, Redis, custom) | **NLB** |
| UDP (DNS, gaming, IoT) | **NLB** |
| Need source IP preservation in headers | NLB (real source) or ALB (X-Forwarded-For) |
| Static IPs for allowlisting | **NLB** |
| Latency-critical (sub-ms) | NLB |
| Path-based routing, host-based routing | **ALB** |
| Lambda backends | **ALB** |

### 5.6 ALB + NLB Together
Common pattern: **NLB in front of ALB** for static IPs + L7 features. Or **NLB in front of self-hosted Envoy/Nginx** for full control with NLB's throughput.

---

## 6. Beyond the Four: Honorable Mentions

### Traefik
A Go-based reverse proxy with native Docker / K8s / Consul integration. Auto-discovers services and updates routing without manual config. Great DX for small-to-medium clusters. Used widely in startups and Docker Compose setups.

### Caddy
Go-based, automatic HTTPS via Let's Encrypt. Trivially simple config. Great for hobby projects and simple internal services. Has a load-balancer mode.

### Linkerd2-proxy
Linkerd's own Rust-based sidecar (formerly used Envoy in Linkerd1). Smaller footprint, simpler than Envoy, opinionated. Excellent for service mesh.

### Cloudflare Pingora
Cloudflare's replacement for Nginx (Rust). Powers most of Cloudflare's edge. Open-sourced 2024. Designed for the most demanding edge scenarios. Not really a "self-deploy" tool but worth knowing.

### Kong / Apigee / Tyk
API gateways built on or alongside Nginx/Envoy. Add API-specific features: rate limiting, key auth, plugins. See [API Gateway →](../03-apis/api-gateway.md).

### GCLB / Azure Front Door
GCP and Azure's managed LB products. Architecturally similar to ALB/NLB but with cloud-specific integrations.

---

## 7. Big Comparison Table

| Aspect | NGINX | HAProxy | Envoy | AWS ALB | AWS NLB |
|---|---|---|---|---|---|
| Type | L7 (+L4 via stream) | L4/L7 | L4/L7 | L7 only | L4 only |
| Throughput per instance | ~100k RPS | ~200k RPS | ~50–100k RPS | autoscales | line-rate millions |
| Latency added | 0.5–2 ms | 0.3–1 ms | 0.5–2 ms | 1–10 ms | < 100 µs |
| Config | static, declarative | static, declarative | dynamic xDS | console / IaC | console / IaC |
| Hot reload | reload | reload + runtime API | hot via xDS | managed | managed |
| HTTP/3 | recent | recent | full | yes | n/a |
| gRPC | basic | yes | first-class | yes (2020+) | passthrough only |
| Sticky sessions | ip_hash, hash | many | cookie, hash | cookie | source-IP |
| Observability | basic (stub) + exporters | stats page, runtime API | rich Prometheus + tracing | CloudWatch | CloudWatch |
| WAF | ModSecurity module | basic ACLs | filters / external | AWS WAF | none |
| Service mesh role | Ingress only | not designed for it | sidecar (gold) | n/a | n/a |
| Cost | FOSS | FOSS | FOSS | $-$$ | $-$$ |
| Best for | web reverse proxy, simple ingress | self-hosted LB, TCP, observability | service mesh, edge, dynamic | AWS L7 HTTP | AWS L4 TCP/UDP |

---

## 8. Choosing in Practice

A pragmatic decision tree:

```
On AWS, simple public HTTPS app?
  → ALB. Maybe with WAF + CloudFront in front.

On AWS, TCP / UDP service?
  → NLB. Maybe with self-hosted Envoy/Nginx behind for L7 features.

On Kubernetes, public ingress?
  → ingress-nginx or Envoy (via Emissary/Gloo/Istio Gateway)

On Kubernetes, service mesh?
  → Istio (Envoy data plane), Linkerd (own proxy), or Consul Connect

Self-hosted, you control hardware?
  → HAProxy or Nginx. HAProxy if you need fine-grained LB and stick tables.

You want a tool that just works for a small site?
  → Caddy or Traefik.

You're at edge scale (Cloudflare-style)?
  → Custom proxy or Pingora. Envoy or NGINX as starting point.
```

For most teams in 2026:
- **Edge**: ALB / GCLB / Cloudflare.
- **Cluster ingress**: ingress-nginx or Envoy.
- **Service mesh**: Istio (Envoy) or Linkerd.
- **TCP/UDP**: NLB (cloud) or HAProxy (self-hosted).

---

## 9. Operating Notes

### Reload vs hot config
- Nginx / HAProxy: `nginx -s reload` / `haproxy -sf <pid>` — graceful, new process started, old drains.
- Envoy: xDS push, no restart needed.
- Cloud LBs: provider handles.

### Connection draining
- All tools support drain on shutdown.
- AWS target groups have `deregistration_delay_timeout_seconds` — default 300; tune for your traffic.

### Logging
- Nginx: access_log + error_log to disk or syslog.
- HAProxy: highly configurable; syslog by default.
- Envoy: structured JSON logs, multiple sinks, gRPC ALS protocol for streaming.
- AWS: CloudWatch + S3 access logs.

### Metrics
- Nginx: `stub_status` + Prometheus exporter or NGINX Amplify.
- HAProxy: stats page + Prometheus exporter.
- Envoy: built-in Prometheus endpoint, rich histograms.
- AWS: CloudWatch metrics.

### TLS cert rotation
- Nginx / HAProxy: reload required (graceful).
- Envoy: SDS — push new cert via xDS; no restart.
- AWS: ACM rotates automatically (most cases).

---

## 10. Worked Example: Architecting an LB Stack

A new B2B SaaS API on AWS:
- 95% HTTPS REST traffic; 5% gRPC.
- Multi-region, multi-AZ.
- 50k RPS peak.
- Customers want static egress IPs from us (for their allowlists).

### Edge
- **CloudFront** in front of public assets and cacheable APIs.
- **ALB** as origin behind CloudFront for L7 routing. WAF on ALB.
- ACM certs for ALB + custom domain.

### gRPC path
- **NLB** for gRPC (sub-ms latency matters). Or ALB with HTTP/2 — fine for B2B scale.
- Behind, **Envoy sidecars** in pods do the per-pod retry/circuit-breaker.

### Static egress
- **NAT Gateway** (or VPC NLB) with Elastic IP. All outbound traffic routes through it. Customers allowlist that EIP.

### Internal mesh
- **Istio** (Envoy) for service-to-service. mTLS, retries, traffic split for canary.

### Cluster ingress (for non-public services)
- **ingress-nginx** for internal admin tools.

### Result
- One CloudFront → ALB → service for public HTTP.
- One NLB → gRPC service for low-latency.
- Istio mesh inside.
- NAT GW with EIP for static egress.

A mix of cloud-managed (CloudFront, ALB, NLB, NAT GW) and self-hosted (Envoy, ingress-nginx) — each chosen for the job.

---

## 11. Common Mistakes

- **Using ALB for high-throughput TCP** — wrong tool. Use NLB.
- **Using NLB and then complaining about no path routing** — wrong tool. Use ALB or self-host.
- **Nginx `if`-block bugs** — `if` inside `location` is "evil" per the docs.
- **HAProxy without `option http-server-close` or proper keep-alive** — connection reuse falls apart.
- **Envoy without a control plane** — manually editing static config defeats half its value.
- **Not setting `proxy_set_header Host $host`** in Nginx → backend sees wrong Host.
- **Reloading on every config change in HAProxy / Nginx** — use runtime API / xDS instead.
- **Not using slow_start** on a new backend in NGINX / HAProxy.
- **Letting ALB idle timeout differ from backend keep-alive** — silent connection resets.
- **Treating ALB / NLB as one-size-fits-all** — pick per service.
- **Forgetting cross-zone load balancing** — disabled by default on NLB; can cause hot AZ.

---

## 12. Cheat Card

```
NGINX        easy, ubiquitous, web-server-first reverse proxy
             use for: ingress, web serving, simple LB

HAPROXY      LB specialist, L4/L7, deep observability
             use for: self-hosted LB, TCP, stick tables, abuse mit

ENVOY        modern, dynamic xDS, service-mesh data plane
             use for: mesh sidecar, edge at scale, gRPC, HTTP/3

AWS ALB      L7 HTTP/gRPC, host/path routing, WAF, Cognito
             use for: public HTTP APIs on AWS

AWS NLB      L4 TCP/UDP, ultra-fast, static IPs, source IP
             use for: TCP services, static IPs, low latency

PATTERN      CloudFront → ALB → (Envoy mesh) → pods
             NLB → custom TCP service
             NAT GW with EIP for static egress

PITFALLS     wrong layer (ALB for TCP, NLB for HTTP),
             reload-instead-of-runtime-API,
             forgot cross-zone LB, idle timeout mismatch

RULE         Right tool for the layer. The fancy proxy isn't
             always the right answer; the boring one usually is.
```

---

## 13. Resources

### Documentation
- **NGINX**: <https://nginx.org/en/docs/>
- **HAProxy**: <https://www.haproxy.com/documentation/>
- **Envoy**: <https://www.envoyproxy.io/docs>
- **AWS ALB**: <https://docs.aws.amazon.com/elasticloadbalancing/latest/application/>
- **AWS NLB**: <https://docs.aws.amazon.com/elasticloadbalancing/latest/network/>
- **GCLB**: <https://cloud.google.com/load-balancing/docs>

### Books
- *NGINX Cookbook* — Derek DeJonghe.
- *HAProxy in Action* — work in progress; the docs are the bible.
- *Istio: Up and Running* — Lee Calcote (Envoy via Istio).

### Articles
- "NGINX vs HAProxy vs Envoy" — many engineering blog posts.
- "How Stack Overflow uses HAProxy" — Nick Craver / Stack Overflow Engineering.
- "Lyft's Envoy: Embracing a service mesh" — Matt Klein.
- "Why Cloudflare built Pingora" — Cloudflare blog.
- "ALB vs NLB: choosing the right load balancer" — AWS Architecture Blog.

### Videos
- ByteByteGo — "Top 6 Load Balancing Tools".
- Hussein Nasser — "NGINX internals", "HAProxy deep dive", "Envoy basics".
- AWS re:Invent talks on ELB internals.

### Tools
- **Self-hosted**: NGINX, HAProxy, Envoy, Traefik, Caddy.
- **Cloud-managed**: AWS ALB/NLB, GCLB, Azure Front Door.
- **Edge/CDN with LB**: Cloudflare, Fastly, Akamai.
- **API gateway flavors**: Kong, Apigee, Tyk, Emissary, Gloo.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [L4 vs L7 →](./l4-vs-l7.md)
- [Algorithms →](./algorithms.md)
- [Health Checks →](./health-checks.md)
- [Sticky Sessions →](./sticky-sessions.md)
- [GSLB →](./gslb.md)
- [Proxies →](./proxies.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [CDN →](../05-caching/cdn.md)

---

*Previous:* [← Reverse Proxy vs Forward Proxy](./proxies.md)  |  *Up:* [README ↑](../README.md)
