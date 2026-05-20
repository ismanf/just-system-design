# Global Server Load Balancing (GSLB)

> **TL;DR** — **GSLB** is load balancing across **regions, data centers, or POPs** — the top of the LB hierarchy. It decides which region a user connects to, based on **geography**, **latency**, **availability**, **weight**, or **cost**. The two primary mechanisms are **DNS-based GSLB** (the authoritative DNS server returns different IPs to different users) and **anycast** (the same IP is advertised from many sites; BGP routes packets to the nearest healthy one). DNS-based is more flexible — health-checks, weights, geo-rules per user. Anycast is more responsive — failover is at the speed of BGP withdrawal, not DNS TTL. Real-world systems combine them: anycast for the public IP, DNS for fine-grained per-customer or per-region overrides. The hardest problems in GSLB are **failover latency** (DNS TTLs are slow, BGP convergence isn't always fast either), **measuring "best"** (geographic distance isn't latency; latency isn't health), and **avoiding traffic skews** when one region absorbs load it can't handle.

---

## 1. Why Global

Load balancing across a single data center solves throughput and pod-level resilience. It doesn't solve:

- **Latency for global users** — a user in Singapore talking to us-east-1 sees 200+ ms RTT.
- **Regional outages** — a single AWS region failure, an undersea cable cut, a DNS provider going down.
- **Compliance** — GDPR / data residency may require EU users hit EU infrastructure.
- **Cost** — egress is cheaper local; cross-region traffic costs more.
- **DDoS absorption** — distributing the attack across many entry points.

GSLB exists to route each user to the **right region** for that user at that moment.

```
                 user in Sydney
                       │
                       ▼
                ┌──────────────┐
                │ Global DNS / │   answer = nearest healthy region IP
                │ Anycast      │
                └──────┬───────┘
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
        ap-southeast us-east   eu-west
        (the right one)
```

---

## 2. The Two Primary Mechanisms

### 2.1 DNS-based GSLB
The authoritative DNS server has logic. When a query arrives, it picks the answer based on:
- The querier's geo location (inferred from resolver IP, or `EDNS Client Subnet`).
- Health of each region.
- Configured weights.
- Latency probes.

```
client → DNS resolver → authoritative GSLB DNS
                         │
                         picks best IP for this client
                         │
                       returns A record (TTL ~30s)
                         │
client ← (IP, TTL=30) ←  resolver ← authoritative
```

The DNS TTL governs cache lifetime. Short TTLs = faster failover; more DNS queries.

**Used by**: AWS Route 53 (Geolocation, Latency, Failover, Multivalue, Weighted policies), Azure Traffic Manager, Akamai DNS, Cloudflare Load Balancer DNS Steering, NS1, Dyn.

### 2.2 Anycast
The same IP is announced via BGP from many sites. The internet's routing protocol picks the "closest" path (in terms of AS hops, not always latency). Each user's packets land at whichever site is topologically closest.

```
client packets → ISP routes by BGP → nearest POP announcing the IP
                                     │
                                     handles the request
```

When a POP fails, it withdraws the BGP announcement; routing reconverges to the next-best POP, typically in seconds.

**Used by**: Cloudflare (all traffic), Fastly, Google's GFE, AWS Global Accelerator, public DNS resolvers (1.1.1.1, 8.8.8.8), CDNs broadly.

### 2.3 Hybrid
Most large deployments use both:
- **Anycast** for the public IP block — every region announces the same IPs; users hit whichever is closest.
- **DNS** for fine-grained per-customer overrides — paying customer X gets routed to a dedicated region; geo rules; latency optimization.

Example: Cloudflare uses anycast for routing; their DNS can also return different IPs for advanced routing scenarios.

---

## 3. DNS-Based GSLB In Detail

### 3.1 Geo-routing
Look up the resolver's IP → infer country/region → return a region-appropriate IP.

```
US user → DNS lookup → returns 198.51.100.1   (us-east IP)
EU user → DNS lookup → returns 198.51.101.1   (eu-west IP)
APAC user → DNS lookup → returns 198.51.102.1 (ap-southeast IP)
```

Used for compliance (GDPR), localization, latency. Simple to set up; coarse — country granularity at best.

### 3.2 Latency-based routing
The GSLB provider continuously measures latency from each major resolver/region to each backend region. The DNS answer is the lowest-latency region.

Used by: AWS Route 53 Latency-based Routing, Akamai Global Traffic Management.

**Pros**: best perceived performance.
**Cons**: requires global probe infrastructure; latency is per-resolver, not per-user (EDNS Client Subnet helps); fluctuates with internet conditions.

### 3.3 Weighted / canary
Return region IPs in proportion to weights. 95% of traffic to stable region, 5% to canary.

```
weights: us-east 50, eu-west 30, ap-southeast 20
DNS returns each IP in roughly that fraction
```

Used for gradual region rollouts, traffic-shifting tests, intentional cost-tuning.

### 3.4 Failover
A primary region; secondary if primary is unhealthy. Health checks drive the swap.

```
primary = us-east, secondary = us-west
us-east passes health check → return us-east IP
us-east fails → return us-west IP after detection delay
```

Used for active-passive multi-region. Detection time = `health_check_interval × failure_threshold` + DNS TTL.

### 3.5 EDNS Client Subnet (ECS)
A DNS extension where the resolver passes the client's subnet to the authoritative server. Lets geo-routing make decisions per-client rather than per-resolver.

Without ECS: a user in Australia using Google DNS (8.8.8.8, anycasted) shows up as some Google datacenter location.

With ECS: the resolver tells the authoritative "the actual client is in 203.0.x.x", so the answer is correct.

ECS is widely supported but has privacy trade-offs; Cloudflare's 1.1.1.1 famously doesn't send ECS. Plan for both cases.

---

## 4. Anycast In Detail

### How it works
Each anycast IP block is announced from multiple PoPs via BGP. Internet routers pick the "best" path (lowest AS-path length) — typically the topologically closest PoP. The same IP, different machines, different cities.

### 4.1 Strengths
- **Failover at BGP speed** — when a PoP fails, it withdraws the announcement; routing reconverges in seconds (sometimes < 1 second on modern internet routes, sometimes 10–30 seconds during global reconvergence).
- **No DNS TTL waiting** — the IP doesn't change, only the route to it does.
- **Single IP for everywhere** — easy for clients; same TLS cert.
- **DDoS absorption** — attack traffic distributes across many PoPs automatically.

### 4.2 Weaknesses
- **You need ASN + IP block + BGP peering** — heavy infrastructure.
- **Stateful flows are tricky** — if mid-flow the BGP route changes, the new PoP has no TCP state for that flow → reset. Typically rare in practice but real.
- **No fine-grained control** — you can't say "users in France go here, except enterprise customer X."
- **BGP convergence isn't always fast** — some networks take minutes.
- **Hijack risk** — bad actors can announce your prefix (BGP hijack). RPKI helps.

### 4.3 Why CDNs use anycast
A CDN POP cluster announces the same IPs from hundreds of cities. A user in Lagos hits the Lagos POP automatically; a user in Stockholm hits the Stockholm POP automatically. No DNS magic needed beyond resolving the hostname once.

---

## 5. Comparison Table

| Aspect | DNS-based GSLB | Anycast |
|---|---|---|
| Granularity | Per-resolver (or per-client with ECS) | Per-network-path |
| Failover speed | DNS TTL bound (typ 30s–5min) | BGP convergence (seconds) |
| Fine-grained control | Yes (geo, weight, health) | Limited |
| Infrastructure | Authoritative DNS + health checks | ASN + IP block + BGP |
| Cost to operate | Cheap | Significant |
| Per-customer overrides | Easy | Hard |
| Stateful flow stability | TCP terminates at one endpoint | BGP can flip mid-flow |
| Used by | AWS R53, Akamai, Azure TM, NS1 | Cloudflare, Fastly, Google, AWS Global Accelerator |

---

## 6. Failover Speed and DNS TTL

DNS TTL is the dominant variable for DNS-based GSLB failover. Three layers can cache:

```
authoritative TTL = 30s
ISP resolver respects ~30s
browser DNS cache: typically 30s–60s (varies by OS, browser)
                  (Chrome's "predictive caching" can extend)
```

Real failover time = `health_check_detect + authoritative_swap + max(resolver_cache, browser_cache)`.

With 30s TTL: practical failover is 1–3 minutes.

To get faster:
- **Shorter TTL** (10–30s). DNS query rate goes up; provider charges may apply.
- **Anycast instead** — sub-second to seconds.
- **Client SDKs that re-resolve on errors** — apps that explicitly refresh DNS on connection failures bypass cache.
- **Multi-CDN setups** — different DNS-driven failure modes.

If your SLO requires sub-minute regional failover, DNS-based GSLB alone won't get there. Use anycast.

---

## 7. How "Best Region" Is Measured

Geographic distance ≠ latency. A user in Mumbai might have a worse path to a Bangalore data center than to a Singapore one if their ISP routes via Singapore.

Real GSLB measures by:
- **Continuous synthetic probes** from many resolvers/networks to each backend region.
- **Real-user measurement (RUM)** — JS beacons or app SDKs report measured latency back to the GSLB.
- **BGP / AS-path heuristics** — sometimes used as approximations.

Akamai's GTM, Cloudflare's Magic Transit, AWS R53 latency, and similar all maintain global latency maps. The map updates continuously; the DNS answer reflects the current best.

Edge cases:
- Mobile networks vary minute-to-minute.
- Some ISPs add huge "egress hairpins" so the topologically-near region is slow.
- Cross-Pacific links have unique latency profiles.

The honest answer: best-effort + continuous correction.

---

## 8. Health Checks at Global Scale

GSLB needs to know each region is healthy. Two flavors:
- **Synthetic** — provider's probes hit each region's endpoint periodically.
- **Reported** — your apps push health to the GSLB control plane.

Considerations:
- **Probe from multiple geos** — a region appearing healthy from US probes might be unreachable from Asia.
- **Probe diversity** — different probe types (HTTP, TCP, application-level).
- **Anti-flapping** — multiple-strike thresholds.
- **Bias for staying** — small wobbles shouldn't drain a region. Big and persistent should.

See [Health Checks →](./health-checks.md).

---

## 9. Patterns: Active-Active vs Active-Passive

### Active-Active
All regions serve traffic. Users hit nearest. On region loss, GSLB shifts the affected users; remaining regions absorb load.

- **Pros**: lowest latency, regional independence, less wasted capacity.
- **Cons**: each region must handle peak load when a peer dies (capacity planning is 2× per region for 2 regions, etc.).

Used by: most modern internet services.

### Active-Passive
One primary region; secondary stands by, mostly cold. Failover swings traffic.

- **Pros**: simpler data flow; canonical primary for writes.
- **Cons**: standby is mostly idle (waste); cold-start risk during failover; failover drills required (often skipped in practice → "did we test this?").

Used by: legacy enterprise, regulated industries with strict failover semantics, smaller orgs.

### Pilot light / Warm Standby
The standby region runs a minimal version of the stack. Failover scales it up. Compromise between cost and recovery time.

### Multi-region with master writer
Reads served from nearest; writes go to a master region. Trade-off: write latency for some users; consistency simpler.

---

## 10. Data Concerns (the Hardest Part)

GSLB routes requests. Your **data** is the harder problem. Some patterns:

### Region-local data
Each region's data is independent (multi-tenant SaaS where tenants are pinned to a region). Easiest.

### Globally replicated read-only
Data written centrally, replicated to all regions read-only. Easy reads; writes funnel to one region.

### Multi-master with conflict resolution
Writes accepted anywhere; conflicts resolved via CRDTs (counters, sets), last-writer-wins (with vector clocks), or app-specific logic. Hard but powerful. See [Multi-Region →](../10-scalability/multi-region.md), [CRDTs →](../08-distributed-systems/crdts.md).

### Global transactions
Spanner, CockroachDB. Synchronous global commits via Paxos/Raft. Write latency = cross-region RTT. Used when correctness > latency.

GSLB doesn't fix data architecture. It only routes requests to whichever region can serve them, given the data architecture you've built.

---

## 11. Real-World GSLB Stacks

### Netflix
Multi-region active-active on AWS. Each region is a full copy. GSLB via Route 53 latency-based + Atlas (their global routing). Chaos Kong drills full-region failover.

### Stripe
Multi-AZ within region. Region failover via DNS. Strong primary-writer model.

### Cloudflare
Anycast everywhere. The 1.1.1.1 model: every PoP serves every user. Internal failover is BGP-driven.

### Google
Maglev + GFE + global load balancing. Single global VIP backed by anycast + global load balancer software. Single anycast IP for `google.com` hits the user's closest PoP automatically.

### AWS Global Accelerator
Provides anycast IPs in front of ELBs. Lets non-anycast AWS resources benefit from anycast routing. Useful for non-HTTP services.

### Discord
Cloudflare at the edge (anycast). Internal routing via DNS / service discovery.

---

## 12. Pitfalls and Failure Modes

### 12.1 DNS pinning
Browsers and OSes can cache DNS longer than the TTL. Chrome's "DNS prefetch" can extend it. Some mobile apps cache indefinitely. Users stay on a dead region after failover.

**Fix**: in-app retry-with-resolve on connection error.

### 12.2 ISP DNS ignores TTL
Some ISPs cache DNS responses for hours. Common in some countries.

**Fix**: nothing perfect. Use anycast for the case where this matters.

### 12.3 Asymmetric failover
DNS shifts US users away, but Asia users still hit the broken region because DNS hasn't updated for them yet. Or BGP convergence is faster in some regions than others.

**Fix**: layer the mechanisms; expect uneven shifts.

### 12.4 Capacity insufficient post-failover
You shifted 100% of one region's traffic to another. Other region has 1.5× normal capacity but received 2× normal load. It dies. **Cascade.**

**Fix**: capacity model for N-1 (single-region failure). Pre-scale before failover if possible. Shed load gracefully.

### 12.5 Stateful sessions across regions
User authenticated in us-east; failover sends them to eu-west; session not replicated → re-login. Worse if mid-checkout.

**Fix**: replicate session state globally (Redis Active-Active, JWT with global trust), or accept session loss as the cost of regional failover.

### 12.6 DNS provider outage
The DNS service is a SPOF. The 2016 Dyn outage took down half the internet for hours.

**Fix**: use **multiple DNS providers** for the apex domain (NS records to two providers). NS1 + Route 53. Cloudflare + Route 53. Hardly anyone does this; the few who do thank themselves later.

### 12.7 BGP hijack
A malicious or misconfigured ASN announces your prefix. Traffic detours.

**Fix**: **RPKI** signing of your routes. Most CDNs and cloud providers do this; verify your own announcements.

---

## 13. Worked Example: Globally-Distributed API

You serve an API to 1M users worldwide. SLO: p99 < 200 ms, 99.95% availability.

### Architecture
- Three regions: us-east, eu-west, ap-southeast. Active-active.
- Anycast IPs in front (via Cloudflare or AWS Global Accelerator).
- Anycast directs each user to nearest region's edge.
- DNS as a fallback / fine-grained override layer (Route 53 latency + health).

### Data
- Postgres with logical replication (read-only) to other regions. Writes go to us-east primary.
- Redis cluster per region for cache (locally populated).
- Kafka mirror across regions for events.

### Failover plan
- us-east fails: anycast withdraws us-east; traffic moves to next-best PoP. Cross-region writes shift to eu-west (failover writer promoted). Within 1–2 minutes.
- DNS-based fallback: Route 53 health checks shift any clients still using DNS-resolved endpoints.
- Capacity per region sized for N-1 failure.

### Observability
- Real-user latency from each region per user-region pair.
- Per-region request rates and health.
- BGP route monitoring.
- Synthetic probes from 20+ external locations.

### Drills
- Quarterly chaos: terminate a region. Watch traffic shift. Measure recovery time. Tune.

---

## 14. Common Mistakes

- **Long DNS TTLs in production.** 24-hour TTL = 24-hour failover.
- **One DNS provider.** Single point of failure for global resolution.
- **No anycast, requiring DNS for everything.** Slow failover.
- **No N-1 capacity plan.** Failover crushes the survivor.
- **No real-user latency telemetry.** Optimizing geo-routes blind.
- **Region-locked data without thinking about failover.** "We can't fail over because our DB is regional."
- **Anycast with stateful TLS or WebSocket without sticky-by-flow.** Mid-flow flips break.
- **DNS pinning ignored.** Apps cache forever and never failover.
- **Skipping BGP hijack protection.** RPKI is free and easy.

---

## 15. Cheat Card

```
WHAT      LB across regions/DCs/POPs

MECHANISMS
  DNS-based GSLB      flexible (geo/weight/latency/health),
                      TTL-bound failover
  Anycast             same IP from many sites, BGP routing,
                      seconds-scale failover

USE BOTH  anycast for the public address; DNS for overrides

POLICIES
  geo-routing         country-based answers
  latency-based       lowest-latency region
  weighted            staged rollouts
  failover            primary → secondary on health failure
  ECS                 client-subnet for accurate per-client geo

PATTERNS
  active-active       lowest latency, N-1 capacity planning
  active-passive      simpler, costly idle, drift risk
  warm standby        compromise

PITFALLS
  long TTL, DNS pinning, single DNS provider,
  no capacity for failover, asymmetric DNS shifts,
  BGP hijack, stateful flows mid-anycast-flip

RULE
  GSLB routes users to a region. Your data architecture
  decides whether each region can serve them.
```

---

## 16. Resources

### Books
- *Site Reliability Engineering* — Google SRE Book (load balancing chapters).
- *DNS and BIND* — Cricket Liu et al.

### Documentation
- **AWS Route 53 routing policies**: <https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html>
- **AWS Global Accelerator**: <https://docs.aws.amazon.com/global-accelerator/>
- **Cloudflare Load Balancing**: <https://developers.cloudflare.com/load-balancing/>
- **Azure Traffic Manager**: <https://learn.microsoft.com/en-us/azure/traffic-manager/>
- **NS1 / IBM**: <https://ns1.com/resources/>

### Papers and articles
- "Maglev: A Fast and Reliable Software Network Load Balancer" — Google, NSDI 2016.
- "Cloudflare anycast architecture" — Cloudflare engineering blog (multi-part).
- "Building 1.1.1.1" — Cloudflare blog.
- "Netflix global traffic management" — Netflix Tech Blog.
- "RPKI: Securing the Internet's routing" — APNIC and others.

### Videos
- ByteByteGo — "DNS Load Balancing", "Anycast Explained".
- Hussein Nasser — "How anycast works".

### Tools
- **AWS Route 53**, **Azure Traffic Manager**, **NS1**, **Cloudflare LB**.
- **Akamai GTM**, **Citrix ITM**.
- **bgp-stats**, **RIPE Stat** for BGP analysis.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [Health Checks →](./health-checks.md)
- [DNS →](../02-networking/dns.md)
- [Geographically Distributed Systems (Multi-Region) →](../10-scalability/multi-region.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)
- [CDN →](../05-caching/cdn.md)
- [Replication →](../04-databases/replication.md)

---

*Previous:* [← Sticky Sessions](./sticky-sessions.md)  |  *Next:* [Reverse Proxy vs Forward Proxy →](./proxies.md)
