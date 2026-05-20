# Reverse Proxy vs Forward Proxy

> **TL;DR** — Both are proxies — software that sits in the middle of a network conversation and intermediates between client and server. The difference is **whose side they're on**. A **forward proxy** sits on the **client side** and represents the client: it intermediates outgoing traffic for one or many internal clients (corporate egress proxy, ISP proxy, Tor, content filter). A **reverse proxy** sits on the **server side** and represents one or many servers: it intermediates incoming traffic to backend services (Nginx, HAProxy, Envoy, ALB). Load balancers are reverse proxies. CDNs are giant globally-distributed reverse proxies. Most "proxy" discussion in system design is about reverse proxies. The two share the same primitive — TCP/HTTP forwarding — but their use cases, deployment positions, and concerns are different.

---

## 1. The Direction Test

The easiest mental model: **which side does the proxy hide?**

```
FORWARD PROXY (hides clients from the internet)

   ┌─────────┐
   │ client  │──┐
   ├─────────┤  │
   │ client  │──┼─►  forward proxy  ─►  internet  ─► server
   ├─────────┤  │
   │ client  │──┘
   └─────────┘
   (the proxy is "between you and the world")


REVERSE PROXY (hides servers from the internet)

                                            ┌─────────┐
                                       ┌────┤ backend │
                                       │    ├─────────┤
   client  ─►  internet  ─►  reverse  ─┼────┤ backend │
                            proxy      │    ├─────────┤
                                       └────┤ backend │
                                            └─────────┘
   (the proxy is "between the world and the backends")
```

A forward proxy talks to *any server* on behalf of *its clients*. A reverse proxy talks to *one set of backends* on behalf of *any clients*.

---

## 2. Forward Proxies in Depth

A forward proxy intermediates outgoing traffic. The client is **explicitly configured** to use it (HTTP `Proxy` setting, `HTTPS_PROXY` env var, PAC file, system policy).

### What they do
- **Egress control** — corporate or school policy decides which sites are allowed.
- **Content filtering** — block known malware domains, ads, NSFW.
- **Caching** — share fetched resources across many clients. (Less common now that most traffic is HTTPS, which is opaque.)
- **Logging / auditing** — every external request goes through one chokepoint.
- **Anonymization** — the destination sees the proxy IP, not the client.
- **Bandwidth shaping** — quotas, QoS.

### Common examples
- **Squid** — the classic open-source forward proxy. Still widely used in corporate networks.
- **Apache Traffic Server** — also bidirectional, but commonly deployed as a forward proxy.
- **Privoxy** — privacy-focused.
- **Tor** — anonymizing forward proxy with onion routing.
- **iCAP servers** for content scanning.
- **mitmproxy** — debugging.
- **Corporate "web filtering" appliances** — Bluecoat, Zscaler.

### HTTP CONNECT
For HTTPS traffic, the forward proxy can't see inside the encrypted tunnel. Instead, the client sends:

```
CONNECT example.com:443 HTTP/1.1
Host: example.com
```

The proxy opens a TCP tunnel to `example.com:443` and blindly relays bytes. It can see *which host* but not the content.

For full HTTPS interception (corporate "TLS inspection"), the proxy must terminate TLS using a private CA installed on every client device. Privacy- and security-sensitive; common in regulated industries.

### Transparent proxies
Sometimes the network forces traffic through a proxy without client configuration — a transparent proxy intercepts via routing or firewall rules. Common in ISPs and some corporate networks.

---

## 3. Reverse Proxies in Depth

A reverse proxy intermediates incoming traffic. Clients don't know it exists; they think they're talking directly to the service.

### What they do
- **Load balancing** — distribute requests across backends.
- **TLS termination** — handle handshakes, certificates.
- **Caching** — serve cached responses without bothering the backend (the CDN job).
- **Compression** — gzip/brotli responses.
- **Security** — WAF, rate limit, IP blocklist, DDoS absorption.
- **Routing** — path-based, host-based, header-based.
- **Authentication offload** — verify JWTs, OAuth tokens, mTLS.
- **Header rewriting** — `X-Forwarded-For`, `X-Real-IP`, custom routing headers.
- **Protocol translation** — HTTP/2 in front, HTTP/1.1 to backends; gRPC-Web to gRPC.
- **Observability** — access logs, metrics, traces.
- **Service discovery integration** — backend list from Consul / etcd / K8s API.

### Common examples
- **Nginx**, **HAProxy**, **Envoy**, **Traefik**, **Caddy** — software.
- **AWS ALB / ELB / NLB**, **GCLB**, **Azure App Gateway / Front Door** — cloud-managed.
- **F5 BIG-IP**, **Citrix NetScaler** — hardware (legacy).
- **Cloudflare**, **Fastly**, **Akamai**, **CloudFront** — globally distributed reverse proxies (CDNs).
- **Istio / Linkerd** (service mesh) — each pod has its own reverse proxy sidecar.

### A reverse proxy IS a load balancer (mostly)
Most "load balancers" in software contexts are L7 reverse proxies. Nginx is both. HAProxy is both. The terms have collapsed in modern usage.

The distinction that remains: **load balancer** is the role; **reverse proxy** is the implementation. A reverse proxy with no upstream choice (one backend) isn't load balancing; a load balancer that doesn't touch payload (NLB, IPVS) is more LB than proxy.

---

## 4. Side-by-Side

| Aspect | Forward proxy | Reverse proxy |
|---|---|---|
| Whose side | Client | Server |
| Configured by | Client | Server operator |
| Visible to | Client knows | Client doesn't know |
| Purpose | Egress control, anonymization, filtering, caching outbound | Ingress LB, TLS, WAF, caching inbound, routing |
| Sees source IP | Client IP | Hides backend IPs from client |
| Sees destination | Any (the open internet) | Configured backends only |
| TLS | Tunnels (`CONNECT`) or intercepts | Terminates and optionally re-encrypts |
| Examples | Squid, Tor, Zscaler | Nginx, ALB, Cloudflare, Envoy |
| Common in | Corporate egress, schools, censorship circumvention | Every public web service |

---

## 5. The Proxy as Pattern

Both share the same skeleton:

```
client ──► [proxy] ──► origin
   ◄─────  [proxy] ◄─────
```

Inside the proxy:
1. Accept connection / request.
2. Apply policy (block, allow, transform).
3. Route / select upstream.
4. Forward.
5. Receive response.
6. Apply policy on response.
7. Return to client.

Everything fancy — caching, retries, header rewriting, observability, auth — fits in steps 2 and 6.

---

## 6. Special Reverse Proxy Use Cases

### 6.1 TLS termination
The reverse proxy holds the TLS certificate. Clients connect via HTTPS; the proxy speaks HTTP (or HTTPS again — "bridging") to backends. Most public web services do this.

### 6.2 API gateway
A reverse proxy with API-aware features: authentication, rate limiting, schema validation, request routing by path/method, response transformation. Kong, Apigee, AWS API Gateway, Tyk. See [API Gateway →](../03-apis/api-gateway.md).

### 6.3 Web application firewall (WAF)
Inspects HTTP requests for attacks (SQL injection, XSS, command injection patterns). ModSecurity, Cloudflare WAF, AWS WAF, F5 ASM. Often deployed as a reverse proxy or as a module within one.

### 6.4 Edge caching (CDN)
Globally distributed reverse proxies that cache responses. Cloudflare, Fastly, CloudFront. See [CDN →](../05-caching/cdn.md).

### 6.5 Service mesh sidecar
Each pod has its own Envoy (or Linkerd-proxy) attached. All in-cluster traffic passes through these sidecars, which act as reverse proxies for the pod's services. See [Service Mesh →](../03-apis/service-mesh.md).

### 6.6 SSL inspection (form of reverse proxy + forward proxy combo)
Some enterprise setups have the same appliance act as a reverse proxy for inbound and a forward proxy for outbound, applying policy in both directions.

---

## 7. Proxy-Specific Concerns

### 7.1 Preserving client info
The backend wants to know who the real client was. The reverse proxy adds:

```
X-Forwarded-For: 203.0.113.7, 198.51.100.5
X-Forwarded-Proto: https
X-Forwarded-Host: api.example.com
X-Real-IP: 203.0.113.7
Forwarded: for=203.0.113.7;proto=https;host=api.example.com  (RFC 7239)
```

Backends parse these. **But trust no one** — only trust headers from your own proxy. Strip any incoming `X-Forwarded-For` at the edge before adding your own.

### 7.2 Path normalization
Different proxies normalize paths differently — `/foo//bar`, `/foo/./bar`, URL-decoded characters. Mismatched normalization between proxies and backends is a source of routing bugs and security holes (auth bypass via path tricks).

### 7.3 Connection coalescing
HTTP/2 (and HTTP/3) reuse one connection for many requests. A reverse proxy that maintains an upstream HTTP/2 connection sends many client requests over one backend connection — great efficiency, but breaks per-connection rate limits and certain load-balancing patterns.

### 7.4 Buffering vs streaming
Default proxies buffer the request body before sending to backend. For large uploads this hurts; configure pass-through streaming. Same for responses — Server-Sent Events and chunked responses must not be buffered.

### 7.5 Timeouts at every layer
Client timeout, proxy idle timeout, proxy upstream timeout, backend timeout. Misalignment causes silent failures. **Backend timeouts should be < proxy timeout < client timeout.** Otherwise the client gives up before the proxy gets an answer.

### 7.6 Connection reuse to backends
Maintaining keep-alive to backends saves handshake cost. But if a backend dies, the cached connections fail until refreshed. Tune keep-alive carefully.

### 7.7 mTLS
The proxy can present a client cert to authenticate to the backend (mTLS upstream). Common in zero-trust internal networks.

---

## 8. Forward Proxy Use in Engineering

When do engineers (not enterprise IT) set up forward proxies?

- **Egress control from microservices** — pin all outbound traffic through a known proxy. Useful for compliance, allowlisting, observability.
- **Egress IP for whitelisting** — a payment provider requires you to call from a known IP; route all your egress through a proxy with that IP.
- **Outbound caching for repeat fetches** — rare but useful for crawlers.
- **MITM debugging** — local mitmproxy to see what an app sends.
- **Server-side rendering with an outbound proxy** — Vercel and other platforms use this internally.
- **Migration / sidecar** — slow re-route of legacy egress through a proxy as you modernize.

Inside a service mesh, the "egress gateway" is essentially a forward proxy for pod-to-external traffic.

---

## 9. Forward + Reverse: Side by Side in One Stack

A realistic enterprise picture:

```
   internal client  ──► forward proxy (egress)  ──► internet
                                                     │
                                                     ▼
                                              CDN reverse proxy
                                                     │
                                                     ▼
                                              edge reverse proxy / WAF
                                                     │
                                                     ▼
                                              cluster ingress (reverse proxy)
                                                     │
                                                     ▼
                                              service mesh sidecar
                                              (reverse proxy in,
                                               forward proxy out)
                                                     │
                                                     ▼
                                                  backend
```

A single request can go through 4–6 proxies. Each does a specific job. The skill is keeping the policy clear, the latency bounded, and the observability complete.

---

## 10. Real-World Examples

### Cloudflare (reverse proxy at planetary scale)
Cloudflare reverse-proxies for millions of customer websites. Each PoP runs a stack: Nginx (or now custom) + Workers (edge compute) + WAF + caching. Their entire business is "globally distributed reverse proxy."

### Squid in universities
Many university networks deploy Squid as a transparent forward proxy. Filters content, caches popular downloads, logs activity. Less effective now that ~95% of traffic is HTTPS.

### Zscaler / Netskope (enterprise forward proxy)
Cloud-hosted forward proxy. All corporate traffic routes through Zscaler's global proxy fleet. Adds DLP, malware scanning, SSO-aware filtering. Replaces on-prem proxy appliances.

### Service mesh (per-pod reverse proxy)
Each Kubernetes pod gets an Envoy sidecar. The sidecar terminates incoming connections (reverse proxy), routes outgoing connections (forward proxy semantics), enforces mTLS, observability, traffic split.

### Stripe's Veneur / Envoy
Uses Envoy as a sidecar and edge proxy. Each service connects through Envoy for consistent retry/timeout/observability behavior.

---

## 11. Common Mistakes

- **Trusting incoming `X-Forwarded-For`.** Always strip and re-add at the edge.
- **TLS termination at the proxy but never re-encrypting internally.** Plaintext on the internal network — fine for some threat models, not others.
- **Mismatched timeouts** at proxy and backend → silent 504s.
- **Not draining proxy connections on deploy** → in-flight requests dropped.
- **Forward proxy with HTTPS interception** but no good private-CA rollout — every device gets cert errors.
- **CONNECT tunnels left open forever** — DoS via half-open connection floods.
- **No path normalization policy** — security holes from `/foo/../bar`-style smuggling.
- **Proxy without observability.** A black box you can't debug.
- **Two layers of proxies both compressing** — wastes CPU and bandwidth.
- **Forward proxy used as an egress chokepoint without HA.** Proxy down → all egress down.

---

## 12. Cheat Card

```
FORWARD PROXY     client-side; client knows; egress control,
                  anonymization, content filtering, audit
                  examples: Squid, Tor, Zscaler

REVERSE PROXY     server-side; client doesn't know; ingress LB,
                  TLS termination, caching, WAF, routing
                  examples: Nginx, HAProxy, Envoy, ALB,
                  Cloudflare, CDN

DIRECTION         forward = on behalf of client → to internet
                  reverse = on behalf of servers ← from internet

LB = REVERSE PROXY in modern parlance for L7 LBs

HEADERS           X-Forwarded-For / Forwarded (RFC 7239)
                  strip and re-add at trusted edge

PITFALLS
  trusting incoming X-F-F, mismatched timeouts,
  no observability, two layers compressing,
  HTTPS interception without private CA

RULE              Pick the proxy that matches the direction
                  you're trying to control.
```

---

## 13. Resources

### Books
- *HTTP: The Definitive Guide* — Gourley & Totty (excellent on proxies).
- *NGINX Cookbook* — Derek DeJonghe.

### Documentation
- **MDN HTTP — Proxy servers**: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Proxy_servers_and_tunneling>
- **RFC 7239 — Forwarded HTTP header**: <https://www.rfc-editor.org/rfc/rfc7239>
- **Squid wiki**: <https://wiki.squid-cache.org/>
- **Envoy listeners**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners>

### Articles
- "Reverse Proxy vs Load Balancer" — Cloudflare Learning Center.
- "Anatomy of a forward proxy" — Squid project.
- "How service meshes use proxies" — Istio / Linkerd blogs.

### Videos
- ByteByteGo — "Forward Proxy vs Reverse Proxy".
- Hussein Nasser — "Proxies explained".

### Tools
- **Forward**: Squid, Tor, mitmproxy, Privoxy, Zscaler.
- **Reverse**: Nginx, HAProxy, Envoy, Traefik, Caddy, ALB, GCLB.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [L4 vs L7 →](./l4-vs-l7.md)
- [LB Implementations →](./lb-implementations.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [CDN →](../05-caching/cdn.md)
- [DDoS Protection & WAF →](../12-security/ddos-waf.md)

---

*Previous:* [← Global Server Load Balancing (GSLB)](./gslb.md)  |  *Next:* [Nginx, HAProxy, Envoy, AWS ELB/ALB/NLB →](./lb-implementations.md)
