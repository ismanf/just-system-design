# CDN — Content Delivery Networks

> **TL;DR** — A **CDN** is a globally distributed fleet of reverse-proxy caches (Points of Presence, or **POPs**) that sits between users and your origin. It caches your responses near every user, terminates TLS close to them, absorbs DDoS traffic, and increasingly runs code at the edge. The dominant providers are **Cloudflare**, **Fastly**, **Akamai**, **AWS CloudFront**, **Google Cloud CDN**, and **Azure Front Door**. The mechanics that matter: how requests are **routed** to the nearest POP (anycast + DNS GSLB), how **caching keys** and **TTLs** are derived from HTTP headers, how to handle **cache misses** efficiently (tiered caching, shield POPs), how to **purge** cached content (URL, tag, soft purge), and how **edge compute** is changing what "the CDN" even means. For any internet-facing system that serves more than a few users, putting a CDN in front of your origin is the single highest-ROI infrastructure decision.

---

## 1. What a CDN Actually Does

The simple version:

```
   user (Tokyo)  ──►  POP-tokyo  ──►  origin (us-east-1)
                       │
                       └── caches response
```

The real version, in priority order:

1. **Cache** static and dynamic responses near the user, sometimes at hundreds of POPs.
2. **Terminate TLS** close to the user (TLS handshakes are RTT-bound; saving 100 ms × 5 round trips matters).
3. **Absorb DDoS traffic** at the edge before it reaches origin (anycast distributes load globally; capacity is enormous).
4. **Route intelligently** — pick the optimal path from POP to origin, sometimes via the CDN's private backbone.
5. **Compress and optimize** — gzip/brotli/zstd on the fly; serve WebP/AVIF based on `Accept`; image resize at edge.
6. **Apply WAF rules** — drop malicious requests at the edge.
7. **Run code** — edge functions (Cloudflare Workers, Fastly Compute@Edge, Lambda@Edge) for routing, A/B, personalization.
8. **Provide a stable address** with anycast IPs that survive origin region failures.

If you're not using a CDN, you're paying for every one of these in your own infrastructure — or going without.

---

## 2. How Requests Get to the Nearest POP

Two mechanisms work together: **anycast** and **DNS-based routing**.

### Anycast
Multiple POPs around the world advertise **the same IP address** via BGP. The internet's routing protocol naturally sends each user's traffic to the closest POP. From a user's perspective, `cloudflare.com` resolves to a single IP, but the packets land in dozens of different cities depending on where the user is.

Used by: Cloudflare (all traffic), Fastly (all traffic), Google, AWS Global Accelerator. Anycast is **fast** — DNS only resolves once, then routing is automatic.

### DNS-based GSLB (Global Server Load Balancing)
DNS returns different IPs to different users based on their location, latency probes, or origin health. The CDN's authoritative DNS server picks the best POP for each query.

Used by: AWS CloudFront (via Route 53), Akamai (legacy and modern), Azure Front Door. Slightly slower failover (DNS TTL bounds), but more flexible.

Most modern CDNs use both: anycast for the IP block, DNS for fine-grained per-customer routing and per-region overrides.

See [GSLB →](../06-load-balancing/gslb.md).

---

## 3. The Cache Key

Every CDN caches by a **cache key** derived from the request. The defaults vary by provider but typically include:

- The **URL** (scheme + host + path + query string).
- Optionally selected request headers (`Accept-Encoding`, `Accept-Language`).
- Optionally the request method (`GET`/`HEAD` cached; `POST` rarely).
- Optionally the device class (mobile vs desktop) or cookie value.

Two requests with the same cache key share a cached response. Two requests that should share a response but have different keys cause cache fragmentation. **Cache key design is the most underestimated lever** in CDN optimization.

### Things to normalize out
- Tracking parameters: `utm_source`, `utm_campaign`, `gclid`, `fbclid`, `_ga`. They never affect the response; they shatter cache. Modern CDNs let you ignore listed query params.
- Sort and lower-case query strings if they're equivalent.
- Strip cookies that don't affect the response. **Cookies are cache killers** — many CDNs by default refuse to cache responses with a `Set-Cookie` header.

### Things to include in the key
- `Accept-Encoding` — different bodies for gzip vs br vs uncompressed.
- `Accept-Language` (if you localize).
- A custom header for A/B variants.
- The `Vary` header tells the CDN to do this for you:
  ```
  Vary: Accept-Encoding, Accept-Language
  ```

### Vary anti-patterns
- `Vary: User-Agent` — shatters the cache into thousands of fragments (every UA string is unique).
- `Vary: Cookie` — same problem, plus cache rarely matches anyway.
- `Vary: Authorization` — only works if you really want per-user cached responses.

The right pattern: parse the things you care about (mobile vs desktop, locale) into a small set of normalized values, and key on those. Cloudflare Workers and Fastly VCL make this easy.

---

## 4. What the CDN Caches

Three tiers, by ease of caching:

### 4.1 Static assets — easy
JS, CSS, images, fonts, video segments. Always cacheable, long TTLs, content fingerprinting (`app.f3a9b2.js`) makes invalidation free. **If your CDN isn't caching these at 99%+ hit rate, you have a misconfiguration.**

```
Cache-Control: public, max-age=31536000, immutable
```

That's it. Set this on every fingerprinted asset.

### 4.2 HTML and dynamic pages — medium
Pages that don't depend on the logged-in user can be cached. For anonymous traffic — which is enormous for media sites, ecommerce browse pages, marketing pages — caching the HTML is transformative.

```
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=3600
```

`s-maxage` lets the CDN cache for 5 minutes even while browsers revalidate every minute. `stale-while-revalidate=3600` lets the CDN serve stale for up to an hour while refreshing in background.

For personalized pages, the modern pattern is to cache an **anonymous shell** at the edge and use **ESI / edge fragments / hydration** to fill in personalized parts. Done well (Akamai, Fastly Compute@Edge, Cloudflare Workers), you can serve a personalized page from the edge with one origin round trip for the personalization data.

### 4.3 API responses — harder
GET endpoints with bounded user-set, public data, can absolutely be cached. Search results, product detail JSON, public profiles, leaderboards.

```
GET /api/products/42
Cache-Control: public, max-age=30, s-maxage=120
```

A 30-second cache on a hot API endpoint at the CDN cuts origin load by 30–1000×.

For authenticated APIs, options:
- **Don't cache.** Send `Cache-Control: private, no-store`. Sometimes the right answer.
- **Cache the unauthenticated parts** of the response and let the client mix in user-specific bits.
- **Use a custom cache key** that includes a user token; expensive (one cached copy per user) but sometimes worth it.

---

## 5. Tiered Caching and Shield POPs

A naive CDN has every POP independently pull from origin. With 200 POPs, a single cold object causes up to 200 origin requests — one per POP. That's a **fan-out miss** problem.

**Tiered caching** routes regional POP misses to a smaller set of "tier-2" or "shield" caches before going to origin.

```
user (Tokyo) → POP-tokyo (L1)
              miss → shield-asia (L2)
                     miss → origin

user (Berlin) → POP-berlin (L1)
               miss → shield-eu (L2)
                      hit if anyone in EU already fetched
```

Effect: origin sees at most one request per shield-region per object, not one per POP. Reduces origin load by ~10× and improves hit rate.

- Cloudflare calls this **Argo Tiered Cache**.
- Fastly's **Origin Shield**.
- AWS CloudFront's **Origin Shield**.
- Akamai's tiered architecture is baked in.

Always enable tiered caching for any production setup. The cost is minimal; the benefit is large.

---

## 6. Cache-Control Headers Refresher

The headers that drive a CDN:

| Header | Effect |
|---|---|
| `Cache-Control: public` | All caches may cache. |
| `Cache-Control: private` | Only browser cache. CDN won't cache. |
| `Cache-Control: max-age=N` | Browser+CDN TTL (seconds). |
| `Cache-Control: s-maxage=N` | CDN-only TTL (overrides `max-age` for shared caches). |
| `Cache-Control: no-store` | Never cache. |
| `Cache-Control: no-cache` | Cache but revalidate every time. |
| `Cache-Control: must-revalidate` | If stale, revalidate (don't serve stale on error). |
| `Cache-Control: stale-while-revalidate=N` | Serve stale for N seconds while refreshing in background. |
| `Cache-Control: stale-if-error=N` | Serve stale for N seconds if origin returns 5xx. |
| `Cache-Control: immutable` | Browser won't revalidate. The URL is content-addressed. |
| `Expires: <date>` | Older equivalent of `max-age`. Use `max-age` instead. |
| `ETag: "abc"` | Opaque version tag. Sent back in `If-None-Match`. |
| `Last-Modified: <date>` | Timestamp version. Sent back in `If-Modified-Since`. |
| `Vary: ...` | Add these request headers to the cache key. |
| `Age: N` | How long the response has been cached. |
| `Surrogate-Control: max-age=N` | Origin-set, CDN-only directive (RFC 3040). |
| `Surrogate-Key: ...` | Tags for tag-based purging (Fastly, Cloudflare). |

`stale-if-error` is underused: it lets the CDN serve a stale copy when origin is down. Free uptime if you can tolerate slightly stale data during incidents.

---

## 7. Cache Invalidation (CDN Edition)

You can't avoid it. CDN content goes stale, and you need to clear it.

Three approaches, in order of operational maturity.

### TTL-only
Set short TTLs for things that change often, long TTLs for fingerprinted assets. Let the cache expire naturally. Simplest; never wrong; mostly enough for static content sites.

### URL-based purge
Explicit API call: "purge `https://x.com/p/42`".

```bash
# Cloudflare
curl -X POST "https://api.cloudflare.com/.../purge_cache" \
  -H "Authorization: Bearer ..." \
  -d '{"files":["https://x.com/p/42"]}'

# Fastly
curl -X PURGE "https://x.com/p/42" -H "Fastly-Key: ..."
```

Latency: 100 ms to a few seconds globally.

### Tag-based purge (a.k.a. surrogate keys)
Tag each cached response with one or more labels. Purge by tag clears every cached object with that tag.

```
Surrogate-Key: product-42 category-shoes brand-nike
```

```bash
# Purge everything tagged product-42 (product page, category lists, search results...)
curl -X POST "https://api.fastly.com/service/{id}/purge/product-42"
```

This is the right approach at scale. You stop needing to enumerate every URL that needs purging — your origin emits tags and the CDN tracks them.

Fastly invented surrogate keys; Cloudflare added "cache tags" later (Enterprise plan). AWS CloudFront has "wildcard invalidations" (path-prefix based) but no tags as of writing.

### Soft purge
Mark cached content as stale but don't delete it. The CDN serves stale-with-revalidate. Default on Fastly. Survives origin outages while you publish updates.

```bash
curl -X PURGE -H "Fastly-Soft-Purge: 1" https://x.com/p/42
```

### Versioned URLs (the cheat code)
Embed a version in the URL path itself. New version = new URL = automatic cache invalidation. Old URLs naturally expire. This is what static asset fingerprinting does.

```
/static/v123/app.css   ← old
/static/v124/app.css   ← new
```

For dynamic content, you can include the version in a query string parameter and have the CDN ignore it from the cache key (so the URL is unique-per-version but the cache is shared). Less common.

---

## 8. Edge Compute

Modern CDNs run code at the edge — close to the user, before the origin is consulted.

### What runs at the edge
- **Cloudflare Workers** — V8 isolates, JS/Rust/WASM, sub-ms cold start, KV/Durable Objects.
- **Fastly Compute@Edge** — Wasm-based, JS/Rust/Go.
- **AWS Lambda@Edge / CloudFront Functions** — lighter functions or full Lambda.
- **Akamai EdgeWorkers** — JS V8.
- **Vercel Edge Functions**, **Netlify Edge Functions** — built on Cloudflare or Deno Deploy.

### What you do at the edge
- **A/B testing** — route to variant without origin trip.
- **Authentication** — verify JWT, deny invalid before origin.
- **Personalization** — fetch user-specific data, inject into cached HTML shell.
- **Geo-routing** — block by country, redirect to localized.
- **Rate limiting** — closer to abusers; cheaper than origin.
- **Image transformations** — resize, format conversion, watermark.
- **Bot mitigation** — challenge or block at the edge.
- **Static-on-the-fly** — render and cache.

The mental model: the **edge is the new web server**. Many sites now do most of their request handling in edge workers and only call origin for canonical data.

### Constraints
- Memory/CPU is bounded (Workers: 128 MB, 50 ms wall time on free tier).
- No persistent local disk.
- Limited library compatibility (must run in the runtime).
- Database calls from edge add latency back; use edge KV for hot data.

---

## 9. Multi-CDN

For mission-critical systems, **one CDN is a single point of failure**. Multi-CDN setups route traffic between multiple providers based on health, performance, or cost.

```
   DNS resolution
        │
        ▼ (per request)
   ┌─────────┬─────────┐
   │ CDN A   │ CDN B   │
   └─────────┴─────────┘
```

Common patterns:
- **Active-active** — round-robin or weighted between CDNs.
- **Active-standby** — primary CDN serves all traffic; on failure, DNS shifts to backup.
- **Latency-based** — choose the faster CDN for each user.
- **Geo-based** — different CDNs for different regions.

Implementations:
- **NS1**, **Cedexis (Citrix ITM)**, **Catchpoint** — multi-CDN load balancers.
- **AWS Route 53** weighted/health-checked records.
- **Custom DNS GSLB**.

Trade-offs: doubles your CDN cost; complicates purge (must purge in both); makes log analysis harder. Worth it for major properties (Netflix uses multiple, video CDNs in particular).

---

## 10. Origin Configuration

Things to do on origin to make the CDN happy:

- **Set strong `Cache-Control` headers.** The CDN follows them. If you set `no-store` accidentally, the CDN won't cache.
- **Set `Vary` precisely.** Wrong Vary → cache fragmentation or wrong response.
- **Compress responses.** Most CDNs gzip/brotli automatically; you can also pre-compress and serve `Content-Encoding: br`.
- **Use HTTP/2 or HTTP/3 to origin.** Cuts handshake cost; multiplexes.
- **Health checks.** CDN should detect origin failure and serve stale-if-error.
- **Origin shield IPs** — restrict origin to accept traffic only from your CDN's IP range (CDN providers publish them).
- **Avoid `Set-Cookie` on cacheable responses.** Default CDN behavior is to skip caching. Either strip cookies or move them to a separate request.
- **Limit response body sizes.** Most CDNs cap object size (Cloudflare: 512 MB on Enterprise, lower on Free).

For video streaming and large assets, use **chunked / segmented delivery** (HLS, DASH) so the CDN caches 2–10 second video segments efficiently.

---

## 11. Pricing Model

CDNs charge mostly for **bandwidth** (egress GB) and sometimes for **requests** (per 10k) and **purges**.

Rough ballpark (mid-2025 list pricing for non-discounted Enterprise volume):
- **CloudFront**: ~$0.085/GB egress in US/EU, ~$0.114/GB in Asia. Per-request ~$0.0075 / 10k.
- **Cloudflare**: Free tier covers most websites. Pro plan flat-rate. Enterprise negotiated.
- **Fastly**: ~$0.12/GB in US/EU; per-request fees; pay-as-you-go.
- **Bunny.net**: ~$0.01–$0.04/GB; best price-per-byte; younger.

For high-bandwidth sites (streaming, large downloads), CDN cost can dominate infrastructure spend. Tactics:
- Maximize hit rate (cached bytes are cheaper than origin-pull bytes).
- Compress aggressively.
- Use cheaper "second-tier" CDNs for bulk static traffic, premium ones for dynamic.
- Negotiate. CDN sales reps are happy to discuss at volume.

---

## 12. Observability

What you watch:

- **Hit ratio** — global, per-POP, per-URL-prefix. Sudden drop = misconfiguration or invalidation storm.
- **Origin bandwidth** — should be a small fraction of CDN egress.
- **Status codes by edge** — 5xx from edge means CDN issues; 5xx from origin means origin pain even with cache.
- **TLS handshake time** — should be < 50 ms p99.
- **Time-to-first-byte from edge** — global view via synthetic monitoring.
- **Cache fill rate** — POPs requesting from origin. Bursts mean tiered caching is off or there's a hot path.

Most CDNs ship logs (real-time on Fastly and Cloudflare Enterprise) you can pipe to your log pipeline. Set up dashboards from day one.

---

## 13. CDN Comparison

| Provider | Strengths | Weaknesses | Best for |
|---|---|---|---|
| **Cloudflare** | Generous free tier; security suite; Workers; huge POP fleet | Locked into their stack; Enterprise pricing opaque | Most internet-facing apps |
| **Fastly** | Real-time purge (~150 ms); VCL flexibility; instant config | Smaller POP count; needs more skill to operate | Media sites, news, high-trust ops |
| **Akamai** | Largest POP footprint; bulletproof | Expensive; complex; legacy | Largest enterprises, video, broadcast |
| **AWS CloudFront** | AWS integration; Lambda@Edge; simple billing | Slower purges; smaller POP count | AWS-heavy stacks |
| **Google Cloud CDN** | Tight GCP integration; HTTPS by default | Smaller global presence; tied to GCP LB | GCP-heavy stacks |
| **Azure Front Door** | Tight Azure integration; WAF | Azure-only | Azure-heavy stacks |
| **Bunny.net** | Cheap; simple | Smaller ecosystem | Bulk static delivery, indie |
| **KeyCDN / BunnyCDN** | Pay-as-you-go simple | Smaller features | Low-budget static |

For most workloads, the choice is Cloudflare (default), Fastly (when programmability and purge speed matter), or CloudFront (when you're deep in AWS). The rest are niche or legacy.

---

## 14. Real-World Patterns

### News sites (NYT, BBC, Guardian)
Cache HTML aggressively (~30s s-maxage). Purge by surrogate-key on article publish or update. Serve personalized header / paywall logic via edge fragments or client-side hydration.

### Ecommerce (Amazon, Shopify, Etsy)
Cache product detail HTML for anonymous users for ~1 min. Use edge personalization for "recently viewed" / cart. Aggressively cache catalog images with fingerprinted URLs. Sub-second invalidation for price changes via tag purge.

### SaaS APIs (Stripe, GitHub, Slack)
Cache public read-only endpoints (rate limits, public profiles, status pages) at the CDN. Use private/no-store for authenticated. Edge workers for authentication, geo-routing, abuse rules.

### Video streaming (Netflix, YouTube, Twitch)
Custom CDN (Netflix Open Connect) or massive third-party (Akamai). Cache video segments (HLS/DASH chunks) for hours. Personalization data hits origin or a separate API.

### Marketing sites
Cache everything for hours; purge on deploy. Use tag-based purge per page. Static site generators (Next.js, Astro, Hugo) work beautifully here.

---

## 15. Common Mistakes

- **No CDN at all.** "It works fine in dev" → "it's down in production" because origin can't handle a fraction of real traffic.
- **`Cache-Control: no-cache` everywhere.** Means revalidate every request — slow. Use `no-store` if you actually mean "don't cache."
- **Cookies on cacheable responses.** Most CDNs default to no-cache when `Set-Cookie` is present.
- **`Vary: User-Agent`.** Cache shattered into thousands of pieces, ~0% hit rate.
- **No tiered caching.** Cold objects cause hundreds of origin pulls.
- **No purge automation.** Updates rely on TTL expiry; users see stale content for hours.
- **Trusting cache hit rate as a single metric.** Hit rate of 98% is great if those misses don't kill origin. Always measure origin-pull bandwidth too.
- **Caching error responses.** A brief origin 500 cached for an hour becomes an hour-long outage.
- **Not setting `stale-if-error` / stale-while-revalidate.** Free uptime left on the table.
- **One CDN, no fallback, for a mission-critical site.** CDN outages happen. Plan for them.
- **Origin reachable from the open internet.** Lock origin to CDN IPs; otherwise attackers bypass the WAF.
- **Caching personalized HTML at the CDN.** Worst-case data leak between users. Edge fragments or `private` cache.

---

## 16. Cheat Card

```
WHAT A CDN DOES
  cache near user · terminate TLS · absorb DDoS ·
  optimize · run edge code · multi-region presence

ROUTING        anycast (Cloudflare/Fastly) + DNS GSLB (legacy/CloudFront)

CACHE KEY      URL + selected headers (Vary) + method
               normalize tracking params; strip cookies

TTL HEADERS    Cache-Control: public, max-age, s-maxage,
               stale-while-revalidate, stale-if-error,
               immutable (for fingerprinted assets)

TIERED CACHING enable always (Argo, Origin Shield)

INVALIDATION   TTL → URL purge → tag purge → versioned URLs
               soft purge for graceful staleness

EDGE COMPUTE   Workers, Compute@Edge, Lambda@Edge:
               auth, A/B, geo-routing, image transform

PRICING        bandwidth + per-request + purges
               hit rate is the cost lever

PITFALLS       no CDN; cookie-leaking responses; Vary: User-Agent;
               cached errors; CDN-bypassable origin

RULE           If users see it, the CDN should cache it.
               If it must be fresh, use short TTL + SWR.
```

---

## 17. Resources

### Books
- *High Performance Browser Networking* — Ilya Grigorik (CDN concepts in Chapters 11–13).

### Documentation
- **Cloudflare docs**: <https://developers.cloudflare.com/cache/>
- **Fastly docs**: <https://docs.fastly.com/en/guides/>
- **AWS CloudFront**: <https://docs.aws.amazon.com/AmazonCloudFront/>
- **Akamai documentation**: <https://techdocs.akamai.com/>
- **RFC 9111**: HTTP Caching.
- **RFC 5861**: stale-while-revalidate / stale-if-error.

### Articles
- "How CDN works" — Cloudflare Learning Center.
- "Surrogate keys at scale" — Fastly Engineering.
- "The economics of CDN" — various.
- "Multi-CDN strategies" — Catchpoint, NS1 writeups.
- "Cloudflare Workers internals" — Kenton Varda.
- Netflix Open Connect engineering blog.

### Videos
- ByteByteGo — "What is a CDN?".
- Hussein Nasser — "CDN deep dive".

### Tools
- **Cloudflare**, **Fastly**, **AWS CloudFront**, **Akamai**, **Bunny.net**.
- **NS1**, **Cedexis** — multi-CDN orchestration.
- **WebPageTest**, **Catchpoint**, **Pingdom** — measure edge latency globally.
- **curl -I**, **httpie**, **cloudflare-cli** — inspect cache headers.

### Adjacent reading
- [Cache Layers →](./cache-layers.md)
- [Cache Invalidation →](./cache-invalidation.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [DNS →](../02-networking/dns.md)
- [HTTP Versions →](../02-networking/http-versions.md)
- [HTTPS, TLS/SSL Handshake →](../02-networking/https-tls.md)
- [GSLB →](../06-load-balancing/gslb.md)
- [Edge Computing →](../19-advanced/edge-computing.md)

---

*Previous:* [← Cache Pitfalls](./cache-pitfalls.md)  |  *Up:* [README ↑](../README.md)
