# Edge Computing

> **TL;DR** — **Edge computing** moves computation from a few large centralized data centers to many small points-of-presence (PoPs) that sit close to users — sometimes within 20 ms of every internet user on the planet. The motivation is **latency** (users feel faster systems), **bandwidth** (don't ship raw data to a central cloud), **regulatory** (data residency), and **availability** (regional failures only affect that region). The dominant platforms — **Cloudflare Workers**, **Fastly Compute@Edge**, **AWS Lambda@Edge / CloudFront Functions**, **Vercel Edge Functions**, **Deno Deploy**, **Fly.io** — each give you a sandboxed execution environment at every PoP. The honest take: **the edge is exceptional for transformation, routing, caching, auth, and personalization at the request layer**, but a poor substitute for traditional compute when you need persistent state, large datasets, long-running processes, or strong consistency. The right mental model is **"the new CDN" — programmable, but with the same constraints**.

---

## 1. The big picture

Three eras of where compute lives:

```
1990s        2010s             2020s+
─────        ─────             ──────

Origin       Origin            Origin (your cloud)
            +CDN cache          +CDN cache
                                +CDN compute (edge)

  o            o      ──┐         o      ──┐
  │            │        │         │        │
  │           cache    │        cache+code │
  ▲           ▲ ▲      │       ▲ ▲ ▲ ▲ ▲   │
  │          /   \     │      / | | | | \  │
  user      user user  │   user user ... user
```

Edge computing is **CDN + arbitrary code**. The CDN-cached HTML is now generated, transformed, or rewritten at the PoP. The auth check happens before the request leaves your continent. The personalization happens 5 ms from the user.

Three distinct edge tiers (with different cost / capability trade-offs):

| Tier | Examples | Cold start | Use case |
|---|---|---|---|
| **Edge workers** (ms cold start) | Cloudflare Workers, Vercel Edge, Fastly | <5 ms (V8 isolates) | Per-request transforms, auth, routing, personalization |
| **Regional / lambda@edge** | Lambda@Edge, CloudFront Functions, Cloudfront Functions L1 | ~100 ms cold | Heavier per-request logic, multi-step |
| **Regional VMs / containers** | Fly.io, Akamai EdgeWorkers, Equinix Metal | seconds | Stateful apps closer to users |

Most "edge computing" conversations are about the first tier — V8 / WASM isolates running JavaScript / TypeScript / Rust at hundreds of PoPs worldwide.

---

## 2. Why latency matters this much

Users in São Paulo pinging us-east-1 (Virginia) see ~120 ms RTT. London ↔ Tokyo is ~230 ms RTT. Add the TLS handshake, HTTP/2 ramp-up, and database round trips, and a "fast" homepage can take 800 ms to first contentful paint.

Putting compute and cache 5 ms from the user collapses this. A page that takes 800 ms from us-east-1 can take 80 ms from a Cloudflare PoP in São Paulo — same code, same product, dramatic UX shift.

Bandwidth has a similar story. The edge transformation can:
- Strip PII before it leaves the region.
- Aggregate IoT telemetry before shipping.
- Filter / route per request without hitting origin.
- Cache personalized responses (varied by signed cookie).

The pattern: **do the obvious work where it's free**, send only what must go to origin.

---

## 3. The execution model

Different platforms use different isolation tech:

### V8 isolates (Cloudflare Workers, Vercel Edge, Deno Deploy)

A single V8 process runs thousands of isolated JavaScript contexts. **Cold start is ~5 ms** because there's no process startup — just allocating an isolate inside an already-warm process.

Trade-off: limited to JS / WASM. No filesystem. No long-running processes. Memory and CPU caps. APIs are V8 + Web Standard (`fetch`, `WebStreams`, `WebCrypto`) plus a small platform-specific layer.

### WASM (Fastly Compute@Edge, others)

Run WebAssembly modules in lightweight sandboxes. Lower memory footprint than full V8 isolates; multi-language (Rust, Go, AssemblyScript, JS via QuickJS). Slightly higher cold-start than pure V8 isolates, but still in the milliseconds.

### Lambda@Edge / CloudFront Functions

AWS's edge tier sits between traditional Lambda and lightweight isolates. CloudFront Functions are JS-only, microsecond-scale, and limited. Lambda@Edge runs full Node.js / Python in fewer locations, with cold starts in the hundreds of milliseconds.

### Regional VMs (Fly.io, container-based)

Run normal Docker containers at multiple PoPs. Closer to "small-cloud-per-region" than to ultra-lightweight isolates. Seconds-scale cold start; stateful apps possible; full ecosystem.

---

## 4. What the edge is good at

### Routing, redirects, A/B testing

Decisions you don't want to round-trip to origin. Personalize URLs, redirect based on geo, split traffic between variants — all in <10 ms per request.

### Authentication and authorization

Validate JWTs, check API keys, terminate OAuth flows at the edge. The protected resource only sees verified requests; origin auth cost goes way down. Stripe, Cloudflare Access, Auth0 all do this.

### Image optimization

Resize / format-convert / compress images per request, per device. Cloudflare Images, Vercel Image Optimization, Fastly's Image Optimizer. Saves bandwidth, saves origin CPU.

### Response transformation

Inject A/B variant, rewrite HTML server-side for personalization, strip headers, modify cookies. HTMLRewriter (Cloudflare) and similar APIs make this a streaming, low-memory pass.

### Caching with personalization

Traditional CDN caches whole responses keyed on URL. The edge can vary the cache key by signed cookie, geographic region, or user tier — caching personalized content that used to require origin per request.

### Edge SSR / ISR

Server-side render at the edge instead of origin. Next.js, SvelteKit, Remix, Astro all support edge SSR. Cold start matters less; first paint drops dramatically.

### Geo-routing for compliance

Route EU users to EU origins. Decide regulatory rules before the request leaves the region. Data residency without rebuilding the app.

### IoT and stream pre-aggregation

Devices send raw telemetry to the nearest PoP; the edge aggregates / filters before forwarding. Saves origin bandwidth and storage.

### Bot detection and rate limiting

Block / challenge requests at the PoP before they consume origin resources. Cloudflare, Akamai, Fastly all have edge bot-management.

---

## 5. What the edge is bad at

### Persistent state

Hundreds of PoPs each see a slice of traffic. There's no "shared memory." You can:
- Use **edge KV stores** (Cloudflare KV, Vercel KV, Fly Redis) — eventually consistent reads, slow writes.
- Use **edge databases** (Cloudflare D1 / Durable Objects, Turso, Neon, PlanetScale) — small data, regional reads, primary in one region.
- Use **edge sync** patterns — broadcast changes via Pub/Sub.

But **strongly consistent global state on every PoP** is fundamentally a different beast. CAP theorem applies. See [CAP Theorem →](../08-distributed-systems/cap-theorem.md).

### Long-running processes

Edge runtimes have wall-clock timeouts measured in seconds (Cloudflare Workers: 30 s CPU, plus I/O wait limits). Background tasks, long polls, file processing — usually wrong place.

### Large datasets

Edge nodes don't have hundreds of GB of RAM. A query that scans a multi-GB table doesn't belong at the edge.

### Heavy CPU work

ML inference, video transcoding, image generation — sometimes feasible (Cloudflare Workers AI does inference at the edge), but expensive and limited. Most CPU-heavy work stays in regional compute.

### Strong consistency

Globally consistent counters, leader election across PoPs, distributed transactions — beyond what most edge platforms easily support. Specialized services (Durable Objects, CockroachDB, Spanner) get partway, with performance compromises.

### Long-lived connections at scale

WebSockets and SSE work at the edge but are billed differently. Plan for it.

---

## 6. Edge storage — the new primitives

### Edge KV (Cloudflare KV, Vercel KV, Workers KV)

Eventually consistent key-value across PoPs. Reads are fast and local; writes propagate in seconds. Great for config, feature flags, session data that tolerates eventual consistency.

### Durable Objects (Cloudflare)

A novel primitive: a **single-instance stateful object** pinned to one PoP, addressable from anywhere. You can build "single-writer per chat room / per game match / per user session" without a separate database. Strong consistency for that object.

### D1 (Cloudflare)

SQLite at the edge with read replication. Primary in one region; read replicas elsewhere. Small-scale but easy.

### Turso / LiteFS

SQLite-based, embedded edge databases with replication. Good for read-heavy small data.

### Upstash Redis at the edge

Multi-region Redis with per-region replicas. Lightweight enough for edge functions to use.

### Neon / Supabase / PlanetScale

Centralized Postgres / MySQL with global edge connectivity. You still have one primary region; edge functions connect via secure tunnels.

### Object storage

S3 / R2 / B2 / Google Cloud Storage. Often combined with CDN caching for read-heavy workloads.

The pattern: **read-heavy state replicates to the edge; write-heavy state stays centralized**. Combine deliberately.

---

## 7. The PoP map

Modern edge networks have hundreds of PoPs:

- **Cloudflare** — 330+ cities in 120+ countries (2025).
- **Akamai** — historically the densest network; very mature.
- **Fastly** — fewer but bigger PoPs; "fewer-but-fatter" philosophy.
- **AWS CloudFront** — 600+ PoPs (a mix of edge and regional edge caches).
- **Google Cloud CDN** — heavy on the global backbone.
- **Vercel** — built on AWS + Cloudflare.

Different shapes for different workloads. Cloudflare's massive PoP count optimizes for global reach + ultra-low cold start. Fastly's fewer PoPs are more capable per-PoP (more compute, more storage).

For most apps, the difference is operationally invisible — you write the same code; the network handles distribution.

---

## 8. Worked example — an edge personalization stack

A modern SaaS landing page:

```
User in Berlin
   │
   ▼
┌──────────────────────────────────────────────┐
│ Cloudflare PoP in Frankfurt                  │
│  1. Terminate TLS                            │
│  2. Geo-IP check; route EU users to EU origin│
│  3. Auth: validate session JWT (KV cache)    │
│  4. Personalize: read user tier from KV      │
│  5. SSR rendered page with variant A/B       │
│  6. Image optimization for the user's device │
│  7. Cache the response with key (URL, tier)  │
└──────────────────────────────────────────────┘
   │ origin miss
   ▼
┌──────────────────────────────────────────────┐
│ Origin in eu-central-1                       │
│  - business logic                            │
│  - Postgres (primary)                        │
└──────────────────────────────────────────────┘
```

What used to be 6 origin round-trips becomes 1 (or 0 with cache hit). Latency drops, origin costs drop, and the user feels the difference.

---

## 9. The new architectural patterns

### Edge as application server

Frameworks (Next.js App Router, SvelteKit, Remix, Astro, Nuxt) ship edge runtimes that render pages at the PoP. Origin is now a data layer.

### Edge as gateway

Route, authenticate, rate-limit, transform — then forward to origin. Often replaces a traditional API gateway sitting in a regional cloud.

### Edge as cache + invalidation engine

Beyond static cache, the edge can subscribe to events (Cloudflare's Queues, Pub/Sub) and invalidate / refresh content reactively.

### Edge for compliance partitions

Multi-tenant SaaS where some tenants must keep data in specific regions. Route by tenant attribute; each region's data stays put.

### Edge for fan-out

A single client request triggers parallel calls to multiple downstream services from the edge; the edge fans out and merges responses. BFF-on-steroids.

### Edge for AI inference

Cloudflare Workers AI, Vercel AI SDK, Fastly's bot detection — small models run at the PoP. Cuts inference cost and latency for things that don't need GPU-class power. Will grow as models shrink.

---

## 10. Cost shape

Edge platforms generally bill per-request (a few microcents) and per-CPU-millisecond (sub-cent). Often free tiers cover small workloads.

Pricing examples (2026 approximations):
- **Cloudflare Workers**: ~$0.30 per million requests + ~$12.50 per million CPU-seconds.
- **AWS Lambda@Edge**: ~$0.60 per million requests + duration billing.
- **Vercel Edge Functions**: ~$0.65 per million invocations.
- **Fastly Compute@Edge**: ~$0.50 per million requests.

For high-traffic apps, edge can be **cheaper than origin compute** because:
- No idle cost — pay only when invoked.
- CDN caching reduces origin requests.
- Bandwidth from origin is often more expensive than edge egress.

But also possibly **more expensive** for compute-heavy work — edge CPU is metered, and per-millisecond costs add up if your function does real work.

The rule: edge is cheap for cheap work, fair for medium work, and not the right place for heavy compute.

---

## 11. The shift in operational thinking

Traditional cloud: one or a few regions, one or two replicas, traffic routed via DNS / Anycast / L7.

Edge: hundreds of locations, none with full state, decision logic per-PoP, deploy is "global push to all PoPs in seconds."

Implications:

- **Observability** must aggregate across hundreds of PoPs. Each PoP emits its own metrics; centralized dashboards merge them.
- **Debugging** without local file system or shell. Tools like Cloudflare's Wrangler tail, Vercel's logs, Fastly's stream logs are how you see what's happening.
- **Deploys** are eventually consistent across PoPs (seconds). Plan for a brief "mixed-version" window.
- **Feature flags** at the edge are powerful — flip a flag, all PoPs reflect it within seconds.
- **Rollbacks** are similar — global revert in seconds.
- **State migrations** need careful planning when data spans regions.

The edge is operationally simpler than a multi-region cloud build, *if* you stay within its constraints. The complexity returns the moment you push past.

---

## 12. Common Mistakes / Anti-Patterns

- **Treating the edge as a full app server.** Long-running processes, big datasets, heavy CPU work — wrong place.
- **Synchronous calls from edge to origin for every request.** Defeats the point. Cache or pre-compute.
- **Globally consistent counters at the edge.** Possible (Durable Objects) but expensive. Often the wrong design.
- **Different code paths for edge vs origin.** Drift; bugs in one and not the other. Use a shared codebase or a clear platform abstraction.
- **Edge KV used as a relational database.** It's eventually consistent K/V — not a Postgres replacement.
- **Cold-start budgets violated.** Vercel claimed 5 ms cold start; your TypeScript bundle is 8 MB → cold start is 200 ms. Profile.
- **No regional fallback.** Edge PoP outage / DDoS → users in that region down. Plan for failover to another PoP or origin.
- **Storing PII in edge KV without encryption.** Logs, replication, sub-processor scrutiny all apply.
- **Ignoring compliance / data-residency.** EU users routed to US PoPs (or vice versa) when contract requires otherwise.
- **Long-lived WebSockets billed as compute time.** Some platforms count WebSocket open time; budget accordingly.
- **Logging too verbosely.** Logs scale per request × per PoP. Costs add up.
- **No load test of the edge configuration.** Synthetic local tests don't reveal PoP-specific cold-start patterns.
- **Treating edge deploys as instant globally.** They're seconds-eventually-consistent.
- **Heavy crypto / regex / parsing in the hot path.** Edge CPU is fast but small. Burn through your budget per request.

---

## 13. Cheat Card

```
PURPOSE   Run code at points-of-presence close to users for low
          latency, reduced bandwidth, and regulatory locality.

EXECUTION TIERS
  V8 isolates           Cloudflare Workers / Vercel / Deno Deploy (<5 ms cold)
  WASM                  Fastly Compute@Edge (multi-language)
  Lambda@Edge / CFF     AWS (heavier, fewer PoPs)
  Regional VMs / hosts  Fly.io, Akamai EdgeWorkers (full apps)

EDGE EXCELS AT
  Per-request transforms (HTML, headers)
  Routing, redirects, A/B testing
  Auth (JWT verification, OAuth)
  Image / asset optimization
  Geo-routing for compliance
  Caching with personalization
  Edge SSR / ISR
  Bot detection, rate limiting
  IoT / stream pre-aggregation

EDGE STORAGE
  KV stores       eventual consistency, fast reads
  Durable Objects single-writer stateful primitive
  D1 / SQLite     small read-replicated DB
  Centralized DB  via tunnel; primary stays one region

EDGE STRUGGLES WITH
  Long-running compute (seconds-bounded)
  Heavy CPU / GPU work
  Globally strongly-consistent state
  Multi-GB datasets
  Long-lived connections at huge scale

PATTERNS
  Edge as gateway (route + auth + rate limit)
  Edge as renderer (SSR per request near user)
  Edge as cache+invalidation engine
  Edge as fan-out coordinator
  Origin for write-heavy persistence

PITFALLS
  Big TypeScript bundle → killed cold start
  Edge KV used as relational DB
  No regional fallback
  Heavy crypto / regex per request
  Stateful logic that needs strong consistency
  Code drift between edge and origin builds

RULE   Edge is the new CDN — programmable, but constrained.
       Use it for transformation, routing, and caching at the
       request layer. Keep heavy state and compute centralized.
```

---

## 14. Resources

### Documentation
- **Cloudflare Workers** — <https://developers.cloudflare.com/workers/>
- **Cloudflare Durable Objects / KV / D1 / R2** — <https://developers.cloudflare.com/>
- **Vercel Edge Functions** — <https://vercel.com/docs/functions/edge-functions>
- **Fastly Compute@Edge** — <https://docs.fastly.com/products/compute-at-edge>
- **AWS Lambda@Edge / CloudFront Functions** — <https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html>
- **Deno Deploy** — <https://deno.com/deploy>
- **Fly.io** — <https://fly.io/docs/>

### Articles
- "How Cloudflare Workers actually work" — Cloudflare engineering.
- "The future of edge computing" — Fastly, Cloudflare, Vercel blogs.
- "Edge databases compared" — multiple engineering blogs.
- "Why the edge is the next big thing" — Lee Robinson (Vercel) talks.

### Videos
- *Cloudflare Connect* / *Vercel Ship* / *Fastly Yield* — annual events.
- *State of the Edge* — Linux Foundation reports.
- ByteByteGo — "Edge Computing Explained."

### Tools
- **Cloudflare Workers / KV / Durable Objects / D1 / R2 / Workers AI**.
- **Vercel Edge Functions / Edge Config / KV**.
- **Fastly Compute@Edge**.
- **AWS Lambda@Edge / CloudFront Functions / S3 + CloudFront**.
- **Deno Deploy** / **Bun on Cloudflare**.
- **Fly.io** for regional apps.
- **Turso / Upstash Redis / Neon** for edge-friendly databases.
- **Honeycomb / Datadog / Logflare** for cross-PoP observability.

### Adjacent reading
- [WebRTC for Real-Time Media →](./webrtc.md)
- [QUIC & HTTP/3 Internals →](./quic.md)
- [CDN →](../05-caching/cdn.md)
- [DNS — How It Works →](../02-networking/dns.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 →](../02-networking/http-versions.md)
- [Multi-Region →](../10-scalability/multi-region.md)
- [CAP Theorem →](../08-distributed-systems/cap-theorem.md)
- [Caching Overview →](../05-caching/caching-overview.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [Rate Limiting →](../03-apis/rate-limiting.md)
- [Feature Flags & Dark Launches →](../15-deployment/feature-flags.md)

---

*Previous:* [← Real-Time Analytics](./real-time-analytics.md)  |  *Next:* [Blockchain & Distributed Ledger Basics →](./blockchain.md)
