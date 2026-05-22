# Design Rate Limiter

> **TL;DR** — A rate limiter answers "is request N from key K in the last T seconds allowed?" in **sub-millisecond time** at the edge of every API call. Five algorithms cover 99% of cases: **fixed window** (simplest, burst at window boundaries), **sliding window log** (accurate, memory-heavy), **sliding window counter** (the sweet spot), **token bucket** (allows bursts up to capacity), **leaky bucket** (smooths to constant rate). Distributed rate limiting adds the coordination problem — most production systems pick Redis with Lua scripts or sharded counters. The hardest decision isn't the algorithm; it's **what the key is** (per-user, per-IP, per-API-key, per-endpoint, combinations).

---

## 1. Requirements

### Functional
- Reject requests that exceed configured rate.
- Multiple policies (e.g., 100 req/min per user, 10K req/min per API key).
- Return appropriate HTTP 429 + `Retry-After` headers.
- Configurable per endpoint.

### Non-Functional
- Decision latency p99 < 1 ms.
- Throughput: matches your API throughput (millions/sec at scale).
- Accuracy: within ~1% of configured limit.
- Survives single-node failure (no false denies during failover).

---

## 2. Where It Sits

```mermaid
flowchart LR
    Client --> LB --> Gateway[API Gateway] --> RL[Rate Limiter] --> Service
```

Typically in the API gateway / reverse proxy (Nginx, Envoy) or a sidecar. The check happens before any real work.

---

## 3. The Five Algorithms

### 3.1 Fixed Window

Bucket per (key, time-window). Reset at boundary.

```
window = floor(now / 60)
count[key:window] += 1
allow if count <= limit
```

**Pros**: trivially simple, O(1) memory per key.
**Cons**: 2× burst at window boundaries (100 in last second + 100 in first second = 200 in 2 seconds, limit was 100/minute).

### 3.2 Sliding Window Log

Store timestamp of every request in last window. Drop entries older than window. Allow if count < limit.

**Pros**: exact.
**Cons**: O(limit) memory per key.

### 3.3 Sliding Window Counter

Approximation. Weighted average of current and previous fixed windows.

```
weight = (window_size - time_in_current_window) / window_size
estimated = previous_count × weight + current_count
```

**Pros**: O(1) memory, much smoother than fixed window.
**Cons**: approximate; can over-count by a few percent.

### 3.4 Token Bucket

Bucket of tokens, refilled at rate R per second up to capacity C. Each request consumes one token. Empty bucket → reject.

```
tokens[key] = min(C, tokens[key] + elapsed × R)
if tokens[key] >= 1: allow, tokens[key] -= 1
```

**Pros**: allows bursts up to C, then smooths to R. Used by AWS, Stripe, GCP.
**Cons**: parameters less intuitive ("rate" + "burst").

### 3.5 Leaky Bucket

Queue of requests; drains at constant rate. Overflow → reject.

**Pros**: enforces constant output rate.
**Cons**: queue size = max latency; requires actual queue.

See [Rate Limiting →](../03-apis/rate-limiting.md) for deeper analysis.

---

## 4. Choosing the Algorithm

| Need | Algorithm |
|---|---|
| Simplest, can tolerate 2× burst | Fixed window |
| Exact, low traffic | Sliding window log |
| Smooth, scalable | Sliding window counter |
| Want bursts allowed | Token bucket |
| Constant output rate | Leaky bucket |

Token bucket and sliding window counter are the workhorses.

---

## 5. Storage

In-memory: only viable for non-distributed services.

**Redis** is the canonical store:
- Atomic counters via `INCR` + `EXPIRE`.
- Lua scripts for read-check-write atomicity.
- TTL handles cleanup automatically.

For token bucket:
```lua
-- Lua script (atomic in Redis)
local tokens = tonumber(redis.call('HGET', key, 'tokens')) or capacity
local last_refill = tonumber(redis.call('HGET', key, 'ts')) or now
tokens = math.min(capacity, tokens + (now - last_refill) * rate)
if tokens >= 1 then
  tokens = tokens - 1
  redis.call('HSET', key, 'tokens', tokens, 'ts', now)
  return 1
else
  return 0
end
```

---

## 6. Distributed Rate Limiting

When you have N rate-limiter nodes:

### 6.1 Centralized (Redis)
All nodes hit same Redis. Accurate; adds ~1 ms network.

### 6.2 Distributed counters
Each node keeps a local count, sums periodically. Lower latency, less accurate.

### 6.3 Probabilistic
Each node enforces `limit / N` locally. Simple, can under-limit if traffic skews.

For most APIs, Redis is fine. For massive throughput (CDN edge), use probabilistic with periodic syncs.

---

## 7. Keys — The Hard Question

What identifies the "subject" of the limit?

- **By user_id**: requires auth. Misses anonymous traffic.
- **By IP**: catches anonymous but trivially bypassed (proxies, mobile IPs change).
- **By API key**: clean for B2B APIs.
- **Composite**: `user_id + endpoint`, `api_key + ip`.

Layer multiple policies: per-user limit AND per-IP limit AND global limit. Reject if any fails.

---

## 8. Response and Backoff

When limited:
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700000060
```

Clients are expected to back off. Good clients implement exponential backoff with jitter; bad clients retry immediately. Protect yourself with per-IP limits in addition to per-key.

---

## 9. Failure Modes

What if the Redis is down?
- **Fail-open**: allow all traffic (risk: DoS).
- **Fail-closed**: reject all traffic (kills your API).
- **Fail-degraded**: use stale local counts; alert.

Production answer: degraded mode + alert. Never silently fail-open without monitoring.

---

## 10. Hot Keys

One API key sending 1M req/sec hits one Redis shard hard. Mitigations:
- Pre-shard the key: hash to N sub-buckets and split traffic.
- Local in-memory counters with periodic sync.

---

## 11. Multi-Tier Limits

API gateways often layer:
1. **Edge** (CDN): block known abuse IPs.
2. **Gateway**: enforce per-account quotas.
3. **Service**: per-endpoint protections.

Each tier protects deeper layers from the next-larger problem.

---

## 12. Common Mistakes

- **Fixed window only** — 2× burst exploited intentionally by clients.
- **No `Retry-After` header** — well-behaved clients can't back off.
- **Single Redis** — single point of failure for the entire API.
- **No global cap** — per-user limits don't stop coordinated attack.
- **Counting AFTER the work** — rate limit must reject before doing work.
- **Per-host limits without per-account** — multi-tenant abuse goes through.

---

## 13. Cheat Card

```
PURPOSE    Allow/deny each request against a rate policy in <1 ms.

CORE       Token bucket and sliding window counter cover most needs
           Redis with Lua scripts = atomic + scalable storage
           Layer per-user + per-IP + per-endpoint + global

POLICIES   Per-user: 100/min, Per-IP: 1000/min,
           Per-API-key: 10K/min, Global cap as safety net

RESPONSES  HTTP 429 + Retry-After + X-RateLimit-* headers

PITFALLS   fixed-window bursts, single Redis,
           silent fail-open, no global cap.

RULE       Decide the key first.
           The algorithm is a tuning parameter.
```

---

## Resources

### Articles
- "Scaling your API with rate limiters" — Stripe blog
- "How we built rate limiting capable of scaling" — Figma engineering
- "Rate Limiting Algorithms" — Cloudflare

### Documentation
- **Envoy rate limiting** — <https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/rate_limit_filter>
- **Kong rate limiting** — <https://docs.konghq.com/hub/kong-inc/rate-limiting/>

### Tools
- Redis (with INCR, Lua), Bucket4j (Java), Gubernator (distributed rate limiter)

### Videos
- ByteByteGo: "Design a Rate Limiter"

### Adjacent reading
- [Rate Limiting →](../03-apis/rate-limiting.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [DDoS Protection & WAF →](../12-security/ddos-waf.md)

---

*Previous:* [← Notification System](./notification-system.md)  |  *Next:* [Distributed Cache →](./distributed-cache.md)
