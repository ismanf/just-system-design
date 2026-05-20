# Graceful Degradation

> **TL;DR** — **Graceful degradation** is the design discipline of keeping a system *useful* — not perfect, not complete, but useful — when components fail or are overloaded. Instead of crashing the whole page when the recommendation service is down, you show popular items. Instead of erroring when the personalization API times out, you return generic content. Instead of refusing to load when the avatar service is slow, you render initials. The goal is to *break the multiplicative reliability problem*: a service composed of N dependencies each 99.9% reliable is itself 99.9%^N reliable — until you make the service work even when a dependency fails. Done right, an outage in a non-critical subsystem is invisible to users. Done wrong, the home page goes down because the "you might also like" widget couldn't load. The discipline is design-first, not bolt-on: every feature needs an explicit "what happens when this dependency is unavailable?" answer, baked in from the start.

---

## 1. The Core Insight

```
Service ────depends on──► A (99.9%)
        ────depends on──► B (99.9%)
        ────depends on──► C (99.9%)
        ────depends on──► D (99.9%)

Without graceful degradation:
  service availability = 0.999 × 0.999 × 0.999 × 0.999 = 99.6%
  (a four-nine increase in dependencies → loss of nines for users)

With graceful degradation around B, C, D (non-critical):
  service availability ≈ availability of A (the critical dep)
  = 99.9%
  (you've broken the multiplication)
```

The math is unforgiving. A service that calls 10 dependencies and *requires* every one to be up is at the mercy of the worst link. The remedy isn't to make each dependency 99.99% reliable — that's expensive and doesn't scale. The remedy is to make most dependencies **optional from the user's perspective**.

The user doesn't care if the recommendations are personalized or generic; they care that the page loads.

---

## 2. The Four Honest Options When a Dependency Fails

```
1. FAIL          return an error to the user
2. DEGRADE       return a reduced or stale answer
3. SUBSTITUTE    return a generic / default answer
4. SKIP          omit the feature from the response entirely
```

Each is appropriate sometimes:

- **Fail** for genuinely required operations: process payment, log in, send the email.
- **Degrade** when stale or partial data is still useful: yesterday's recommendations, last-known inventory count.
- **Substitute** when a generic answer is reasonable: default avatar, popular items, base ranking.
- **Skip** when the feature is purely additive: dismiss the "people you may know" widget; render the page without it.

Graceful degradation is the discipline of explicitly picking option 2, 3, or 4 for **every dependency** that isn't truly required.

---

## 3. Required vs Optional Dependencies

The starting point: classify every dependency.

```
REQUIRED                    OPTIONAL
─────────                   ─────────
auth service                personalization
payment processor           recommendations
inventory check at checkout related products
write to primary DB         "users who bought also bought"
session validation          activity feed
DB read for the resource    avatars
                            profile completion percentage
                            badge counts
                            search "did you mean"
                            unread count in nav
```

Most user-facing services have **5–10× more optional dependencies than required ones**. Each optional dependency that's treated as required is a self-inflicted reliability tax.

The exercise: for every dependency in your service, write down what happens when it fails. If the answer is "users see an error," challenge that — is it really required, or have we just not designed the fallback?

---

## 4. Patterns

A small set of techniques covers most cases:

### Static defaults
Hard-coded fallback values. Cheap and always available.

```
def get_user_avatar(user_id):
    try:
        return avatar_service.fetch(user_id, timeout=200ms)
    except Exception:
        return GENERIC_AVATAR_URL
```

Best for: tiny pieces of UI that have an obvious default.

### Stale-while-revalidate / serve stale
Cache the last good response; serve it when the upstream fails.

```
def get_recommendations(user_id):
    cache_key = f"recs:{user_id}"
    try:
        result = recs_service.fetch(user_id, timeout=500ms)
        cache.set(cache_key, result, ttl=1h)
        return result
    except Exception:
        return cache.get(cache_key, default=POPULAR_ITEMS)
```

Best for: data that's still useful even if a few hours old.

### Cached / precomputed alternative
Maintain a second, lower-quality data store as a fallback (popular items list, default rankings, generic content).

### Per-feature kill switches
A feature flag that disables a specific feature server-wide. When the recommendation service is broken, flip a flag to skip rendering that section entirely.

```
if not feature_flag("recommendations_enabled"):
    return render_page_without_section("recommendations")
```

Best for: known-broken features during an incident.

### Best-effort, parallel fan-out
Issue all optional dependency calls in parallel, with timeouts. Whatever returns by the deadline is included; the rest is skipped.

```
def render_homepage(user):
    deadline = 200ms
    futures = {
        "feed":     async_fetch(feed_service.get,    user, timeout=deadline),
        "recs":     async_fetch(recs_service.get,    user, timeout=deadline),
        "ads":      async_fetch(ad_service.get,      user, timeout=deadline),
        "badges":   async_fetch(badge_service.get,   user, timeout=deadline),
    }
    return assemble_page(
        feed   = futures["feed"].result_or_default([]),
        recs   = futures["recs"].result_or_default(POPULAR),
        ads    = futures["ads"].result_or_default(None),
        badges = futures["badges"].result_or_default(0),
    )
```

Best for: feed-style pages with many independent sections.

### Read from a replica when primary is down
Accept staleness in exchange for availability. The reader bypasses the failed primary and serves from the most recent replica.

### Reduced functionality mode
Mark the system "degraded" globally; some features become read-only or unavailable; the rest works.

```
Examples:
  - Comments disabled, posts still visible.
  - Cart works, checkout requires retry.
  - Search returns titles only, no thumbnails.
```

Best for: known incidents where partial functionality keeps users productive.

### Sample / approximate
When the exact computation is unavailable, return an approximation.

```
exact_count = expensive_query()           # primary path
fallback    = "1.2k"                       # rounded estimate from cache
```

Best for: counters, leaderboards, search ranking — anywhere "good enough" is acceptable.

### Offline / queue for later
Accept the input but defer processing. The user gets confirmation; the work is processed when downstream recovers.

```
def submit_order(order):
    try:
        return payment_service.charge(order)
    except CircuitBreakerOpenError:
        order_queue.enqueue(order)
        return "queued"   # user sees order received; processed later
```

Best for: write-heavy operations where async completion is OK.

---

## 5. Designing for Degradation From the Start

Graceful degradation isn't a bolt-on. It's a design discipline. The questions to ask at design time:

```
For every feature on this screen:
  1. Is this required for the screen to be useful at all?
  2. If we couldn't get this data, what's the best alternative?
  3. What's the timeout budget for fetching it?
  4. Will the user notice if it's missing? If yes, how do we tell them?
  5. Is the fallback always available (truly, or also dependent on the
     same downstream)?
  6. Is the fallback tested?
```

The last two are where most graceful-degradation efforts break:
- "Our cache is the fallback" — but the cache is in the same region that just went down.
- "We have a generic default" — but no one's tested whether that path actually works.

Test the fallback paths. They will be the only thing working during an incident.

---

## 6. Static vs Dynamic Degradation

Two flavors of "degrade":

### Static
Hard-coded fallbacks. Always available. Same answer for everyone. Cheap to implement and test.

```
fallback recommendations = TOP_100_POPULAR_HARDCODED
fallback price           = LAST_PUBLISHED_PRICE
fallback ad              = HOUSE_AD_HARDCODED
```

### Dynamic
Computed from a less-reliable but more-up-to-date source.

```
fallback recommendations = redis.get("popular:24h")
fallback inventory       = "approximately N in stock" from BI snapshot
fallback notifications   = poll on next page load (skip realtime)
```

Static is the most reliable; dynamic is more useful. Production systems use both — static is the floor when dynamic also fails.

---

## 7. Operational Degradation — The Toolkit

When an incident is in progress, degradation needs operator levers:

### Feature flags
Toggle individual features on/off without a deploy. Disable recommendations, ads, comments, personalization, search ranking, anything optional.

### Load shedding
Drop low-priority work when the system is overloaded. Health checks > user-facing requests > background jobs. See [Backpressure →](../10-scalability/backpressure.md).

### Read-only mode
Disable writes; serve cached or replica reads. Often the difference between "users can browse" and "site is down" during a primary DB failure.

### Region failover with degraded data
Failover to a secondary region that has slightly stale data. Users see yesterday's state; service stays up.

### Cached responses for everything
At the CDN / edge level, extend cache TTLs aggressively during incidents. Even an hour-old homepage is better than a 503.

### "Lite" version
Some services maintain a stripped-down version (Facebook Lite, GMail HTML view) that uses minimal dependencies and ships during heavy load or known incidents.

---

## 8. The "Static Stability" Idea — AWS Style

AWS coined **static stability**: the system continues functioning at its current state without any control-plane dependencies. If the control plane (Kubernetes scheduler, autoscaler, DNS) goes down, the data plane continues serving with its last-known configuration.

The implication for degradation: **the most reliable degraded state is one that doesn't require any new operations to enter**. If the system is already serving cached recommendations, it doesn't need to switch modes when the recommender breaks — it's already in the degraded mode by design.

This argues for **always-serve-from-cache** patterns, where the "freshness" is just a property of the cached value rather than a runtime decision.

---

## 9. Communication During Degradation

When users experience a degraded service, tell them — but appropriately.

### Visible degradation
- Banner: "Some features are temporarily unavailable."
- Inline message: "Recommendations are currently being refreshed."
- Replacement: "Showing popular items" where personalized would normally appear.

### Silent degradation
- Generic avatar instead of the user's actual photo (no notice needed).
- Slightly stale data (no notice needed if the user can't tell).
- Skipping decorative widgets entirely.

The rule: **notify when the user might notice something is missing**, stay silent when the substitute is indistinguishable. Notifying about silent degradations creates more confusion than the original problem.

---

## 10. Worked Example — A Product Page Under Stress

A product detail page during a major Black Friday outage of the recommendation service:

```
Section            Critical?  Fallback
─────────────────  ─────────  ────────────────────────────
Product title       YES       fail / cached page
Product price       YES       cached (5 min) acceptable
Product images      YES       fail if origin down; CDN cached
Inventory count     no        omit count; show "in stock"
Add to cart         YES       fail / queue for later
                              (acceptable for short outage)
Reviews             no        omit section
Recommendations     no        fallback to "popular in category"
Recently viewed     no        omit
Ads / promos        no        omit
Question/answer     no        omit
Personalized ship   no        show generic "free over $50"
  date
```

Under failure: the page renders with title, price, image, "in stock", and "add to cart." Everything else either falls back or is omitted. The user can still buy the product.

Without degradation: any failure in any dependency takes the whole page down. With degradation: only the title or price failure does.

The difference is one of architectural intent. The product page renders as a *composition* of independent sections, not a monolithic blob.

---

## 11. Operational Reality

### Test the degraded path
The degraded path is the one path that almost never runs in production. Test it explicitly:
- Chaos drills that disable the recommender → does the page still load?
- Feature flags toggled in staging to force degraded mode.
- Synthetic monitoring that exercises the fallback periodically.

If the degraded path hasn't been exercised in 30 days, assume it's broken.

### Monitor the degraded mode
- Dashboard: percentage of requests served from fallback.
- Alert: persistent fallback rate above threshold = upstream is broken.
- Metric: how long the system has been in degraded mode.

### Beware: degradation that masks real problems
A robust fallback can hide a broken dependency from monitoring. The recommender service might be down for hours; users see popular items; nobody notices. Counter-measure: explicit metrics on fallback rate, alerts when fallback is firing.

### Feature flag hygiene
Operational kill switches are a critical tool. Discipline:
- Document each switch and what it disables.
- Test them quarterly.
- Default to "on" but verify "off" works.
- Audit what's off; "temporary" flips that became permanent are a smell.

### Cached fallbacks can be stale at the worst moment
The cache that backs your fallback should be:
- Refreshed by a different system than the one being faulted (so a single dependency failure doesn't kill both).
- Multi-AZ or multi-region.
- TTL'd long enough to outlast typical incidents (24 h+ for popular-items lists).

### Degradation under retry storms
A degraded path is often more expensive than the normal path (more cache hits, more fallback queries). A surge of retries during an incident can take down the fallback path too. Apply circuit breakers and rate limits to both paths.

### "Just show an error" is sometimes correct
For destructive or critical operations (delete, pay, transfer), erroring is correct. The user understands "we couldn't do that" better than a silent fail. Don't degrade away the truth on critical paths.

---

## 12. Real-World Examples

### Facebook Lite
A heavily-degraded version of the Facebook app for low-bandwidth users. Has saved Facebook during regional infrastructure failures: when full FB is unavailable, Lite still loads because it has fewer dependencies and aggressive caching.

### Amazon during prime day
Amazon explicitly trades personalization for availability during peak: less personalized but always-available product pages.

### Google search "did you mean"
When the spelling-suggestion service is slow, results are returned without the suggestion. Users see only their query results; nobody notices the missing suggestion.

### Slack during outages
Slack's typical incident pattern: degrade real-time features (typing indicators, presence), keep message send/receive working. Users notice but can still communicate.

### GitHub status banners
GitHub's banner pattern: when Actions are degraded, you can still push, pull, browse code, file issues. Only Actions UI shows a "degraded" indicator.

### Netflix's Pareto principle
Netflix doesn't make every microservice highly available. They invest in 99.99% for the playback path and accept lower SLOs for browsing, search, recommendations. When recommendations are down, you see a generic carousel and keep watching.

### Discord during DNS failures
Famous incidents where Discord's voice channels kept working via cached connection state even when DNS / control plane was unreachable. Static-stability in action.

---

## 13. Common Mistakes / Anti-Patterns

- **Bolt-on degradation.** Adding fallbacks after an outage; you'll always be retrofitting. Design with degradation in mind from the start.
- **No fallback at all.** Every dependency failure becomes a user error.
- **Fallback that depends on the same broken thing.** Cache populated by the failed service; can't recover.
- **Untested fallback.** It will be the only path during an incident — and broken.
- **Silent fallback that masks the outage.** Engineers don't know the dependency is broken.
- **Fallback that's expensive.** Substitute is more expensive than the original; cascading collapse.
- **Generic message for every degraded state.** "Something went wrong" — for missing avatar, recommendations, and the entire page. Indistinguishable.
- **Required dependencies that aren't really required.** "Show stock count" was never critical; treating it as such ties product availability to inventory uptime.
- **"Just retry" as the entire degradation strategy.** Retries help with transients; for sustained outages, retries amplify, not heal.
- **Treating every section of a page as a hard requirement.** Monolithic page; one fault kills it.
- **No operator levers.** During an incident, engineers can't disable the broken feature without a code change + deploy.
- **Feature flags that don't work.** Tested at deploy time but not exercised; the "off" path is broken when you need it.
- **No metric on fallback frequency.** Can't tell when degradation is firing.
- **Degraded mode communications that confuse users.** "We are experiencing issues" everywhere, even for invisible substitutions.

---

## 14. Decision Rule

```
For each piece of data / functionality:
  1. Is it required for the user's primary task?
     YES → fail explicitly on dependency failure
     NO  → degrade

For each "degrade" choice:
  - Static default, stale cache, generic alternative, or skip?
  - Is the fallback truly independent of the failing dependency?
  - Is the fallback tested?

For each feature:
  - Operator kill switch?
  - Metric on fallback rate?
  - User communication for the degraded state?

For the system as a whole:
  - Can the home page render with N dependencies down?
  - Can users still complete the most important journey?
  - Is "degraded mode" a tested mode, not a hypothetical?

Bias: more degradation is almost always better than less.
The single most user-affecting reliability investment is
turning required dependencies into optional ones.
```

---

## 15. Cheat Card

```
PURPOSE     Keep the system useful when dependencies fail. Break
            the multiplicative reliability problem by making most
            dependencies optional from the user's perspective.

FOUR OPTIONS ON DEPENDENCY FAILURE
  Fail        return an error (for truly required operations)
  Degrade     reduced / stale answer
  Substitute  generic default
  Skip        omit feature entirely

PATTERNS
  Static defaults              cheap, always available
  Stale-while-revalidate       serve last good cache
  Cached / precomputed alt     "popular items" fallback
  Feature flags / kill switch  operator levers per feature
  Parallel best-effort fanout  whatever responds by deadline
  Read from replica            accept staleness for availability
  Reduced functionality mode   read-only, simplified
  Approximate / sampled        good-enough numbers
  Offline / queue for later    accept now, process later

DESIGN-FIRST  Every feature: explicit "what if this dep fails?"
              answer, baked in from the start.

REQUIRED VS OPTIONAL
  Required: auth, payment, primary write, the user's primary task
  Optional: personalization, recs, badges, counts, decorations
  Most features are optional. Treat them as such.

TEST THE FALLBACK
  Chaos drills disabling dependencies. If unexercised for 30 days,
  assume it's broken.

PITFALLS    No fallback · fallback depends on the broken thing ·
            untested fallback · silent fallback masks outage ·
            generic-error-everywhere · expensive fallback ·
            "just retry" as strategy · monolithic page · no operator
            levers · feature flags that don't work · no fallback
            metrics

RULE        100% reliability is impossible. Useful service during
            partial failure is achievable — by design, not luck.
            More features should be optional than is comfortable.
```

---

## 16. Resources

### Books
- *Release It!* — Michael Nygard. Stability patterns include fail-fast and graceful degradation.
- *Site Reliability Engineering* — Google. Designing for graceful degradation.
- *Reactive Design Patterns* — Roland Kuhn. Isolation and degradation in reactive systems.
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Articles
- "Static Stability Using Availability Zones" — AWS Builders' Library: <https://aws.amazon.com/builders-library/static-stability-using-availability-zones/>
- "Avoiding Insidious Failure Modes" — AWS Architecture Blog.
- "How Netflix Handles Cascading Failures" — Netflix tech blog.
- "Graceful Degradation at Slack" — Slack engineering blog.
- "Failing Gracefully" — Google SRE essays.
- "Why You Should Build a Lite Version" — Facebook engineering on Facebook Lite.
- "Feature Flags Are Critical Infrastructure" — Charity Majors / Honeycomb.

### Videos
- "Failure is Not an Option, It's a Feature" — various conference talks.
- "How Complex Systems Fail" — Richard Cook.
- ByteByteGo — "Graceful Degradation."
- SREcon — many talks on degrading gracefully.

### Tools
- **LaunchDarkly / Split / Statsig / Unleash** — feature flag platforms.
- **resilience4j / Polly** — fallback policies.
- **Envoy / Istio** — service-mesh-level fallbacks via outlier detection.
- **Hystrix Dashboard (archived)** — visualization of degraded states.
- **Honeycomb / Datadog** — observability for fallback rates.

### Adjacent reading
- [Fault Tolerance Patterns](./fault-tolerance.md)
- [Circuit Breaker Pattern](./circuit-breaker.md)
- [Bulkhead Pattern](./bulkhead.md)
- [Feature Flags & Dark Launches](../15-deployment/feature-flags.md)
- [Backpressure](../10-scalability/backpressure.md)
- [Cache Strategies](../05-caching/cache-strategies.md)
- [SLA, SLO, SLI, Error Budgets](./sla-slo-sli.md)
- [Chaos Engineering →](./chaos-engineering.md)

---

*Previous:* [← Bulkhead Pattern](./bulkhead.md)  |  *Next:* [Failover & Disaster Recovery →](./failover-dr.md)
