# HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC)

> **TL;DR** — Three generations of the same protocol, each fixing the previous one's bottleneck:
> - **HTTP/1.1** (1997) — plain-text, one request per connection at a time. Simple. Slow on modern pages.
> - **HTTP/2** (2015) — binary, multiplexed requests on a single TCP connection. Faster, but TCP head-of-line blocking still hurts.
> - **HTTP/3** (2022) — runs on **QUIC** (UDP-based) with TLS 1.3 built in. No HOL blocking, 0/1-RTT connections, connection migration.
>
> All three are the same HTTP semantics (methods, status codes, headers); only the **wire format** changed. As a designer you choose H/2 or H/3 on the server and let the client negotiate.

---

## 1. The Whole Story In One Picture

```
HTTP/1.1                         HTTP/2                         HTTP/3
─────────                         ──────                         ──────
text frames                       binary frames                  binary frames
1 request at a time per conn      multiplexed streams            multiplexed streams
keep-alive                        single conn, many streams      single conn, many streams
no header compression             HPACK header compression       QPACK header compression
HOL blocking at HTTP layer        HOL blocking at TCP layer      NO HOL (per stream)
TLS optional                      TLS effectively required       TLS 1.3 mandatory
                                  on TCP                          on UDP (QUIC)

Same: methods (GET/POST/...), status codes (200/404/500), headers, URLs.
```

---

## 2. HTTP/1.1 — The Workhorse for 20 Years

### How it works
- **Text-based protocol.** Requests and responses are ASCII you can literally type into `telnet`.
- One request per connection at a time (per TCP socket).
- **Keep-Alive** (persistent connections) reuses a TCP connection for multiple sequential requests.
- **Pipelining** lets a client send multiple requests without waiting — but responses must come back *in order*, so it was rarely used in practice (proxies broke it).

### Example
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0
Accept: */*

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

### Why it became slow
Modern pages load 50–500 resources (HTML, CSS, JS, fonts, images, API calls). HTTP/1.1's "one at a time" model meant browsers opened **6+ parallel TCP connections per origin** as a workaround. That's wasteful (handshake costs, congestion windows starting from 1), and you still hit the 6-connection ceiling.

### Other H/1.1 baggage
- **Verbose headers** repeated on every request (no compression).
- **No prioritization** of what loads first.
- **No server push** (had to wait for client to request).
- **Plain text** — relatively expensive to parse.

### When you'll still see it
- Internal tools, legacy systems.
- Server-to-server when both endpoints control the stack and simplicity wins.
- Many `curl` scripts and webhooks.
- Some health-check endpoints.

---

## 3. HTTP/2 — Multiplexing the Same Connection

Released as RFC 7540 in 2015 (now RFC 9113). Designed by Google based on their SPDY experiment.

### What's new
- **Binary framing.** Requests/responses split into binary frames (HEADERS, DATA, etc.). Faster to parse, less ambiguous.
- **Multiplexing.** Many requests/responses share one TCP connection as independent **streams**, interleaved at the frame level. No more "6 connection" hack.
- **Header compression (HPACK).** Eliminates the repeated bytes on every request.
- **Server push** (officially deprecated in browsers, 2022 — was rarely useful in practice).
- **Stream prioritization.** Hints to the server about which resources matter most.
- **Flow control.** Per-stream, in addition to TCP's connection-level flow control.

### Visually
```
HTTP/1.1 (no pipelining):
  conn1: REQ-A ─wait─ RES-A   REQ-B ─wait─ RES-B   ...
  conn2: REQ-C ─wait─ RES-C   ...
  (6 connections per origin)

HTTP/2:
  conn1: ┌── REQ-A ──┐ ┌── REQ-B ──┐ ┌── REQ-C ──┐
         │  RES-A    │ │  RES-B    │ │  RES-C    │  ── all interleaved
         └───────────┘ └───────────┘ └───────────┘
         (1 connection, many streams)
```

### What it still suffers from: TCP Head-of-Line Blocking
HTTP/2 multiplexes at the *application* layer, but it still runs on TCP. **TCP guarantees ordered delivery of bytes**, so if one packet is lost, *every* stream behind it stalls until that one packet is retransmitted. The streams are logically independent, but TCP sees them as one byte stream.

On a lossy link (mobile, WiFi), HTTP/2 can perform *worse* than HTTP/1.1 with 6 separate connections — because one drop blocks all streams instead of just one.

### Other H/2 gotchas
- **TLS effectively required.** Spec allows cleartext (h2c), but no major browser will speak H/2 without TLS.
- **One big connection** = one big failure domain. Connection drop affects every in-flight request.
- **Server push** was added with great fanfare and then deprecated — it was hard to use right and often pushed things the client already had.
- **HPACK** is stateful — keep this in mind for HTTP-aware proxies.

### Where you'll see it
- The default for almost every modern HTTPS site.
- gRPC runs on H/2.
- Most CDNs and reverse proxies (Nginx, Envoy, ALB).

---

## 4. HTTP/3 — Throwing Away TCP

Standardized as RFC 9114 (2022). The interesting part is the transport beneath it: **QUIC**, defined in RFC 9000.

### Why QUIC?
The TCP-HOL problem can't be fixed inside TCP — kernels and middleboxes won't change. So:
- **Replace TCP with QUIC** (a new transport built on UDP).
- **Move reliability and ordering into the application layer**, per stream.
- **A lost packet only stalls its own stream**, not the others.

```
HTTP/3
  └─ QUIC (streams, reliability, TLS 1.3 built in)
        └─ UDP
              └─ IP
```

### What QUIC buys you
- **No HOL blocking** between streams. A drop in stream X doesn't stall stream Y.
- **Built-in TLS 1.3.** No separate TLS handshake.
- **0-RTT** connection resumption (for repeat visitors).
- **1-RTT** for first-time connection (vs TCP+TLS's 2 RTT).
- **Connection migration.** A connection is keyed by a *connection ID*, not the 5-tuple. Switch from WiFi to LTE? Same connection survives.
- **Better congestion control** isolation per stream.
- **All encrypted at the transport level.** Even most of the QUIC headers are encrypted, frustrating middlebox interference.

### The handshake savings
```
TCP + TLS 1.3      QUIC (first conn)     QUIC (resumed)
──────────────    ──────────────────    ───────────────
TCP SYN/SYN-ACK   QUIC + TLS combined   "0-RTT" data
TLS handshake     in 1 RTT              with the first packet
First data
══════ 2 RTT      ════ 1 RTT            ════ 0 RTT
```

### What HTTP/3 keeps from H/2
- Same HTTP semantics (methods, status codes).
- Binary framing.
- Header compression (renamed **QPACK** — adapted for unordered delivery).
- Multiplexed streams.

### Caveats
- **UDP gets dropped by some networks.** Corporate firewalls and old middleboxes don't always allow UDP. Clients have to **fall back to H/2 over TCP**.
- **CPU cost** of QUIC was historically higher than TCP (user-space crypto, more packets). Modern implementations are catching up.
- **Tooling** (load balancers, proxies, ops) is still maturing.
- **Connection ID** privacy implications (designed to migrate, so trackers could in theory follow you).

### Adoption
- **All major browsers** support HTTP/3 (Chrome, Firefox, Safari, Edge).
- **CDNs**: Cloudflare, Fastly, Google Cloud CDN, Akamai, AWS CloudFront — all serve HTTP/3.
- **Origin servers**: Nginx (via experimental modules), Caddy, LiteSpeed, NGINX Plus, HAProxy 2.6+, Envoy.
- Google reports ~50% of its traffic is over QUIC.

---

## 5. Feature Matrix

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| Year | 1997 | 2015 | 2022 |
| Transport | TCP | TCP | QUIC (UDP) |
| Wire format | Text | Binary | Binary |
| Connections per origin (browser default) | 6 | 1 | 1 |
| Multiplexing | No | Yes (app-level) | Yes (transport-level) |
| HOL blocking | Yes (app + TCP) | Yes (TCP only) | No |
| Header compression | None | HPACK | QPACK |
| TLS | Optional | Effectively required | Mandatory (TLS 1.3 built in) |
| Handshake RTTs | 1 TCP + 2 TLS | 1 TCP + 1 TLS (1.3) | 1 (or 0 resumed) |
| Server push | No | Yes (deprecated) | No (intentionally dropped) |
| Connection migration | No | No | Yes |
| Easy to debug | Yes (telnet) | Harder (binary) | Hardest (encrypted + UDP) |

---

## 6. Negotiation: How Browsers Pick a Version

- **HTTP/2** is selected via **ALPN** during the TLS handshake (`h2` advertised by server).
- **HTTP/3** is signaled by the server via the **`Alt-Svc`** header (or HTTP/3 SVCB DNS records). The client makes one H/2 request, learns "I also speak H/3 on UDP port 443", and uses H/3 next time.

So your first request is always H/2; subsequent requests upgrade to H/3 if available.

---

## 7. Real-World Performance

The right answer depends on your workload:

- **Fast, low-loss networks** (datacenters, fiber): H/2 and H/3 are similar; H/2 is often slightly faster due to mature TCP.
- **Lossy / mobile networks**: H/3 is meaningfully faster. Connection migration alone saves user-visible reconnects.
- **Tiny one-shot requests**: HTTP/1.1 with keep-alive is often *fine*. The advantages of H/2/3 only show up when there are many concurrent requests.
- **Server-to-server APIs**: gRPC over H/2 is the norm; H/3 not yet widely adopted server-side.

---

## 8. Which Version Should I Run?

### Public-facing web app
- Run **HTTP/2 + HTTP/3** at the edge (your CDN handles it).
- Origin (behind the CDN) can be H/2 or H/1.1 — doesn't matter much.

### Server-to-server APIs
- **gRPC** = HTTP/2 (mandatory).
- **REST** over H/1.1 with keep-alive is fine for most; switch to H/2 if you have *many parallel calls per client* (rare server-to-server).

### Long-lived connections (WebSockets, streaming)
- WebSocket runs over **HTTP/1.1 Upgrade**. WebSocket over H/2 (RFC 8441) exists but is rarely used.
- For event streams: **SSE** (HTTP/1.1 or H/2), **gRPC streaming** (H/2), or build on **WebRTC / QUIC** directly.

### Edge of-the-world devices, mobile-first
- Prefer **H/3** if your platform supports it.

---

## 9. Same HTTP Semantics, Different Wire

Important: applications usually don't see a difference. Your code calls:
```python
requests.get("https://example.com/users/42")
```
Whether that's H/1.1, H/2, or H/3 underneath, the request looks like:
```
GET /users/42
Host: example.com
```
and the response is `200 OK` with headers and a body. **HTTP semantics didn't change; only the encoding did.**

This is why you can flip versions at the proxy layer without rewriting your app.

---

## 10. Common Mistakes

- **Treating H/2 as a magic bullet.** A poorly-designed app with a chatty API doesn't get *that* much from H/2.
- **Disabling keep-alive** on H/1.1 thinking it's "safer." It's not. It's just slower.
- **Opening many H/2 connections** instead of one. Defeats the whole point.
- **Assuming server push helps.** It's deprecated for a reason.
- **Not enabling H/3** when your CDN supports it free.
- **Putting H/2 behind a load balancer that downgrades to H/1.1.** Look up your LB settings (AWS ALB, GCP LB, Nginx) — many keep H/2 only client-side.
- **Forgetting that gRPC requires H/2.** Some networks block or downgrade it.

---

## 11. Debugging

### Inspect from the command line
```bash
curl -v --http1.1 https://example.com
curl -v --http2 https://example.com
curl --http3 https://example.com   # requires curl built with HTTP/3 (recent versions)
```

### See ALPN in the TLS handshake
```bash
openssl s_client -connect example.com:443 -alpn h2,http/1.1 -tls1_3 -servername example.com
```

### Browser tools
- Chrome DevTools → Network → "Protocol" column shows `h1`, `h2`, `h3`.
- Chrome flag: `chrome://flags/#enable-quic`.
- Firefox `about:networking#http3`.

### Wireshark
- See real frames. For HTTP/2 / HTTP/3 you need TLS keylog files to decrypt (or capture before TLS).

---

## 12. Cheat Card

```
HTTP/1.1   text, 1 req/conn at a time, keep-alive, 6 conns/origin
            still ubiquitous; fine for simple server-to-server

HTTP/2     binary, multiplexed streams over 1 TCP conn, HPACK, TLS expected
            TCP HOL blocking is its Achilles heel
            REQUIRED for gRPC

HTTP/3     binary, multiplexed over QUIC (UDP), TLS 1.3 built in
            no TCP HOL, 0/1-RTT handshake, connection migration
            falls back to H/2 when UDP blocked

SAME HTTP semantics across all three: methods, status codes, headers.

CHOOSING:
  Public web         enable H/2 + H/3 at the CDN edge.
  gRPC               H/2 (no choice).
  Internal API       H/1.1 keep-alive is OK; H/2 if many parallel calls.
  WebSocket          H/1.1 Upgrade.
  Streaming          SSE / gRPC streaming / WebRTC.

NEGOTIATION:
  H/2 via ALPN ("h2")        H/3 via Alt-Svc header / SVCB DNS.

LOOK OUT:
  Server push is dead. Don't use it.
  H/2 over plaintext (h2c) is fine internally but not for browsers.
  Some networks block UDP — your client must fall back to H/2.
```

---

## 13. Resources

### Specs
- **RFC 9112** — HTTP/1.1 (revised 2022).
- **RFC 9113** — HTTP/2 (revised 2022).
- **RFC 9114** — HTTP/3.
- **RFC 9000** — QUIC.
- IETF datatracker — <https://datatracker.ietf.org/>

### Books
- **Ilya Grigorik, *High Performance Browser Networking*** — Chapters on HTTP/2 are gold. Free online: <https://hpbn.co/>
- *Learning HTTP/2* — Stephen Ludin & Javier Garza (O'Reilly).

### Articles
- Cloudflare on HTTP/3: <https://blog.cloudflare.com/http3-the-past-present-and-future/>
- Daniel Stenberg's HTTP/3 explainers (the curl maintainer): <https://daniel.haxx.se/blog/>
- "HTTP/2 is slower than HTTP/1.1 in the real world" (lossy networks) — various blogs benchmarking the HOL issue.
- Google QUIC project history: <https://www.chromium.org/quic/>

### Videos
- ByteByteGo — "HTTP/1, HTTP/2, HTTP/3": <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser — comprehensive HTTP/2 and QUIC deep dives: <https://www.youtube.com/@hnasr>
- "Move Fast & Fix Things" — talks on HTTP/3 deployment at Google/Cloudflare.

### Tools
- `curl` (built with `--http3` support in recent versions).
- `nghttp2` — pure HTTP/2 client/server for testing.
- `wireshark` with HTTP/3 dissector.
- Chrome DevTools / Firefox DevTools.
- <https://http3check.net/>, <https://h3check.net/>.

---

*Previous:* [← DNS — How It Works](./dns.md)  |  *Next:* [HTTPS, TLS/SSL Handshake →](./https-tls.md)
