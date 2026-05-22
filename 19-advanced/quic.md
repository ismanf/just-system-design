# QUIC & HTTP/3 Internals

> **TL;DR** — **QUIC** is a transport protocol that runs over **UDP** and provides TCP-like reliability, encryption, and multiplexing with significantly better performance for modern network conditions. **HTTP/3** is HTTP semantics layered over QUIC. Together they fix three long-standing problems with TCP-based HTTP: **head-of-line blocking** across streams, **slow connection setup** (TCP handshake + TLS handshake = 2–3 RTTs), and **broken connections on network change** (Wi-Fi to LTE drops your TCP socket). QUIC was prototyped at Google around 2012, standardized as RFC 9000 in 2021, and is now used for **~30% of internet traffic** (Cloudflare, Google, Meta, Akamai). HTTP/3 is supported by every major browser, all major CDNs, and most modern HTTP clients. The honest take: **enable HTTP/3 at your edge for free latency wins, especially for mobile and high-loss networks**. The hard parts aren't your problem — they're the server/library implementor's. The interesting engineering lives in **0-RTT, connection migration, congestion control, and the deliberate redesign of how transport, encryption, and multiplexing fit together**.

---

## 1. The big picture

```
HTTP/2 over TCP (since 2015):

  HTTP/2 frames
       │
       ▼
  TLS 1.2/1.3
       │
       ▼
  TCP   ◄── ordered byte stream
       │
       ▼
  IP

HTTP/3 over QUIC (since 2022):

  HTTP/3 frames
       │
       ▼
  QUIC streams (multiplexed, independent)
       │
       ▼
  QUIC packets (encrypted; built-in TLS 1.3)
       │
       ▼
  UDP
       │
       ▼
  IP
```

The defining changes:

- **TLS is built in.** QUIC requires TLS 1.3; the handshake combines transport and crypto into one round trip.
- **Multiplexed streams without head-of-line blocking.** TCP's "one byte stream" means one lost packet stalls everything; QUIC has independent streams that share a single connection.
- **Connection IDs, not 4-tuples.** A connection survives when the client changes IP (Wi-Fi → cellular) — TCP can't do this.
- **0-RTT data on repeat connections.** Send data with the first packet to a known server.
- **Runs over UDP**, so it works on the existing internet without kernel upgrades or middlebox cooperation.

These add up to *fewer round trips for the same work*. On a typical mobile network with 60 ms RTT and 1% packet loss, page load time can drop 20–40% by switching from HTTPS over TCP to HTTP/3.

---

## 2. Why TCP wasn't enough

By the late 2010s, TCP's limits were obvious:

- **Connection setup latency.** TCP SYN + SYN-ACK + ACK + TLS hello + server hello + ... is 2–3 RTTs minimum. For mobile users on 60 ms RTT links, that's a 120–180 ms tax before the first byte of payload.
- **Head-of-line blocking inside one TCP connection.** HTTP/2 multiplexed many streams onto one TCP socket; one lost packet stalls *every* stream until retransmission. The protocol that was supposed to fix HTTP/1.1's HOL blocking introduced its own.
- **Ossification.** Middleboxes (firewalls, NATs, "smart" enterprise gear) inspect TCP headers and refuse anything that doesn't look like Web 1.0 traffic. Changing TCP itself is essentially impossible.
- **Connection death on network change.** A 4-tuple (`src IP, src port, dst IP, dst port`) defines a TCP connection. Change any element — by switching networks, by NAT rebinding — and the connection is dead. Mobile apps re-establish constantly.
- **Slow congestion control evolution.** Improving TCP's congestion control requires kernel updates rolled out across the whole internet.

Google's solution was to move transport into **user space** (where it can iterate fast), encrypt **everything** (so middleboxes can't see and can't ossify), and run over **UDP** (which is universally supported but unstructured).

QUIC went from "Google research project" (2012) to "RFC 9000" (2021) to "30% of the web" (2024–2026). One of the faster protocol transitions in internet history.

---

## 3. The QUIC handshake

```
HTTPS over TCP+TLS 1.3 (cold start):

   client                          server
     │       TCP SYN ──────────────►│
     │◄── TCP SYN-ACK ───────────────│
     │── TCP ACK ───────────────────►│
     │── TLS ClientHello ───────────►│   (RTT 1 done)
     │◄── TLS ServerHello + cert ────│
     │── TLS Finished + data ───────►│   (RTT 2 done)
     │◄── data ──────────────────────│

QUIC handshake (cold start):

   client                          server
     │── Initial+ClientHello ───────►│   (combined!)
     │◄── Initial+ServerHello+data ─│   (RTT 1 done)
     │── Handshake+Finished+data ──►│
     │◄── data ──────────────────────│
```

A cold QUIC connection is **1 RTT** for full handshake — and the second leg already carries useful data. That's about 1 RTT cheaper than TCP+TLS 1.3, or 2+ RTTs cheaper than TCP+TLS 1.2.

### 0-RTT — sending data in the first packet

If the client has talked to this server before and saved a session ticket, it can encrypt and send actual payload in the **first packet** of the next connection. The server can respond immediately. Effective latency: **0 RTT** for repeat connections.

Caveats:
- **Replay attacks**. 0-RTT data isn't bound to a fresh exchange; an attacker could resend it. Standard mitigation: only use 0-RTT for **idempotent operations** (GET, idempotent POST with idempotency keys). Servers can reject 0-RTT and force fallback.
- **Limited data**. The server signals an early-data limit.

For mobile and high-RTT environments, 0-RTT is the practical superpower. Returning users skip the latency tax entirely.

---

## 4. Streams and multiplexing

QUIC has **first-class independent streams** inside a single connection.

```
QUIC connection
  ┌──────────────────────────────────────────────┐
  │ Stream 1 ────► packet packet packet         │
  │ Stream 2 ────► packet         packet        │
  │ Stream 3 ────► packet packet                │
  │ Stream 4 ────►        packet packet packet  │
  └──────────────────────────────────────────────┘
```

If a packet belonging to Stream 1 is lost, **only Stream 1 stalls** waiting for retransmission. Streams 2, 3, 4 keep flowing. This is the single biggest performance improvement over HTTP/2-on-TCP for lossy networks.

HTTP/3 maps each HTTP request to one bidirectional QUIC stream. Browser opening 50 sub-resources? 50 independent streams sharing one connection, one congestion window, but no HOL blocking across streams.

Stream types:
- **Bidirectional client-initiated** — most HTTP requests.
- **Unidirectional server-initiated** — push (rare; HTTP/3 server push was effectively deprecated).
- **Unidirectional client-initiated** — control / settings frames.

---

## 5. Connection migration

A QUIC connection is identified by a **connection ID**, not the 4-tuple. When the client's IP/port changes:

1. Client sends a packet from the new path, including the existing connection ID.
2. Server validates the new path (anti-spoofing).
3. Connection continues, no re-handshake.

Why this matters:
- Phone moves from Wi-Fi to LTE — the call / video / page-load doesn't drop.
- Carrier-grade NAT rebinds — the connection survives.
- Multi-path opportunities (future QUIC versions add explicit multi-path).

TCP simply can't do this. Mobile apps have spent a decade papering over the problem with reconnection logic; QUIC obsoletes that work for new connections.

---

## 6. Encryption and packet structure

QUIC's encryption is built in. Every QUIC packet header is partially obfuscated; the payload is encrypted with TLS 1.3 keys.

```
QUIC packet:
  ┌──────────────────────────────────────────────┐
  │ Header                                        │
  │  - flags                                      │
  │  - version                                    │
  │  - connection ID                              │
  │  - packet number (encrypted "header prot.")  │
  ├──────────────────────────────────────────────┤
  │ Payload (encrypted with AEAD, TLS 1.3 keys)  │
  │  - one or more frames                         │
  │    (STREAM, ACK, CRYPTO, NEW_CONNECTION_ID,  │
  │     PATH_CHALLENGE, etc.)                    │
  └──────────────────────────────────────────────┘
```

Implications:
- **Middleboxes can't inspect or modify QUIC**. Some firewalls block all QUIC because they can't deep-inspect. Some networks have "QUIC fallback" mechanisms.
- **DDoS amplification protections** are baked in — the server demands proof of address before sending big responses.
- **Always-encrypted means no plaintext acks for diagnostics.** Easier to operate; harder to debug at the network layer.

---

## 7. Congestion control

QUIC moves congestion control to **user space**. Implementations choose algorithms; current production ones include:

- **CUBIC** — TCP's default, ported to QUIC.
- **BBR / BBRv2 / BBRv3** — Google's bandwidth-delay-product-based control. Used in YouTube, search, Meet. Often dramatically better on long-fat-pipe networks (high RTT, high bandwidth).
- **NewReno** — for legacy compatibility.

Why this matters: improving congestion control no longer requires kernel upgrades and operator cooperation. Cloudflare, Google, Meta deploy new algorithms in months, A/B-test them at scale, and adopt what wins.

The result: QUIC connections often outperform TCP **even on the same network**, because the congestion algorithms are newer and better-tuned.

---

## 8. HTTP/3 specifics

HTTP/3 = HTTP semantics over QUIC. Specifically:

- **One stream per request/response** — clean separation, no HOL.
- **QPACK header compression** — like HPACK (HTTP/2's compression) but designed for QUIC's out-of-order delivery. Slightly less compact than HPACK but works with QUIC's stream model.
- **No HTTP/2-style server push** (effectively deprecated; alternative is `103 Early Hints`).
- **Same HTTP semantics** — methods, status codes, headers — as HTTP/1.1 and HTTP/2.

For an application developer, HTTP/3 looks identical to HTTP/2. Same APIs, same observables. The wins are at the network layer.

### Alt-Svc and discovery

How does a client know a server supports HTTP/3? The server advertises with an `Alt-Svc` header on its TCP+TLS response:

```
Alt-Svc: h3=":443"; ma=86400
```

The client caches that for the `max-age` window and uses HTTP/3 for subsequent connections. On first visit, you still pay TCP+TLS handshake cost.

**HTTPS DNS RR** (RFC 9460, 2023) lets browsers discover HTTP/3 support in DNS, eliminating that first-connection cost. Cloudflare, Google, and Apple all support it.

---

## 9. The performance picture

Real-world benchmarks (CloudFlare, Google, others):

| Scenario | TCP/TLS | QUIC | Improvement |
|---|---|---|---|
| Cold connection, low RTT | similar | similar | ~0–10% |
| Cold connection, high RTT (mobile) | baseline | 1 RTT faster | 15–30% page load |
| Warm connection, 0-RTT | baseline | 0 RTT | 60–100 ms saved |
| 1% packet loss | bad (HOL) | mostly fine | 20–40% |
| Network switch (Wi-Fi → LTE) | reconnect | seamless | huge UX win |
| Long-distance high-bandwidth | OK | better with BBR | 10–30% |
| Local LAN low loss | similar | similar | minimal |

Where QUIC really shines: **mobile networks, cross-continent connections, packet-loss-prone Wi-Fi, repeat visits with 0-RTT**.

Where it barely matters: **datacenter networks where loss is zero and RTT is microseconds**. Inside a datacenter, TCP is usually fine. Inter-DC and to the user is where QUIC wins.

---

## 10. Operational reality — deploying HTTP/3

For consumers:
- **Cloudflare, Fastly, Akamai, AWS CloudFront, Google Cloud CDN** all support HTTP/3 with a flag flip. Free wins for most sites.
- **Modern browsers** (Chrome, Edge, Firefox, Safari, mobile equivalents) support it.
- **Servers** — nginx (since 1.25), Caddy (default since 2.6), Envoy, HAProxy, HTTPx, LiteSpeed.
- **Client libraries** — Chromium's QUIC, msquic (Microsoft), quiche (Cloudflare), aioquic (Python), ngtcp2, picoquic, quic-go, s2n-quic (AWS).

For server operators:
- **Open UDP/443**. Many firewalls / load balancers historically blocked it. Check before turning on.
- **Tune UDP buffer sizes**. Default Linux UDP buffers are tiny; high-throughput QUIC servers need `sysctl -w net.core.rmem_max=...` tuning.
- **Watch out for amplification attacks.** QUIC has built-in protections (3× client-data rule for unverified addresses) but UDP is naturally attractive to reflection attacks.
- **Load balancing.** Stateless L4 LB needs to route QUIC by connection ID — not by 4-tuple — or migration breaks. Modern LBs (Envoy, F5, Cloudflare) handle this; some don't.
- **Observability.** TCP's clear-text headers are gone. Use server-side metrics and qlog (a JSON-based structured logging format for QUIC) for debugging.
- **Fallback.** Some networks block UDP/443. Your CDN should fall back to HTTPS over TCP transparently.

---

## 11. QUIC beyond HTTP

QUIC is a general-purpose transport. Other protocols on top:

- **WebTransport** — a browser API exposing QUIC streams (and datagrams) to JavaScript. Use case: games, real-time apps, transferring large data with less HTTP overhead. Successor to WebSockets where applicable.
- **MASQUE** — proxying via QUIC (Apple's iCloud Private Relay uses MASQUE internally).
- **MOQ — Media over QUIC** — IETF work-in-progress for video streaming over QUIC, aiming to displace HLS for low-latency live.
- **DoQ — DNS over QUIC** — DNS encryption + speed via QUIC (already production at Cloudflare, AdGuard, NextDNS).
- **SMB over QUIC** — Microsoft moved file sharing onto QUIC.
- **WireGuard-over-QUIC** experiments.

The pattern: QUIC is becoming the general-purpose transport that TCP was for 40 years. We're early in that transition.

---

## 12. WebTransport

A particularly interesting use case for application developers. **WebTransport** exposes QUIC's streams and datagrams to JavaScript:

```javascript
const transport = new WebTransport("https://example.com/wt");
await transport.ready;

// Reliable, ordered stream
const stream = await transport.createBidirectionalStream();
const writer = stream.writable.getWriter();
writer.write(encoded);

// Unreliable, unordered datagrams
transport.datagrams.writable.getWriter().write(encoded);
```

Compared to WebSockets:
- **Multiple streams** without HOL blocking.
- **Datagrams** for unreliable best-effort delivery (cursor positions, game state).
- **Better congestion control** out of the box.
- **No upgrade handshake** — runs natively over HTTP/3.

Use cases: web games, real-time collaboration, video calls (alternative to WebRTC for some scenarios), data sync. Browser support is reaching critical mass in 2026.

---

## 13. Common Mistakes / Anti-Patterns

- **Treating QUIC as a drop-in replacement that needs no config.** Buffer sizes, firewall rules, load balancer settings all matter.
- **Blocking UDP/443 at the firewall.** All QUIC traffic dies; clients fall back to TCP silently.
- **Not advertising Alt-Svc.** Clients never discover that you support HTTP/3.
- **Not enabling HTTPS DNS RR.** Forces first connections to do TCP+TLS, then upgrade. Minor but real.
- **Using a load balancer that routes UDP by 4-tuple.** Connection migration breaks because new packets land on a different backend.
- **Logging only TCP-level metrics.** You see Layer 4 dropouts but not QUIC stream stalls. Use qlog.
- **Allowing 0-RTT for non-idempotent operations.** Replay attacks possible. Restrict 0-RTT.
- **Assuming kernel UDP buffer defaults are fine.** They aren't on Linux. Tune `net.core.rmem_max`, `net.core.wmem_max`.
- **Assuming HTTP/3 means HTTP/2 / 1.1 can be retired.** They can't — corporate networks, old clients, some middleboxes. Keep all three.
- **Performance-testing HTTP/3 on a LAN.** It barely helps. Test on lossy, high-RTT, real-world conditions.
- **Server stack mismatch with client.** Old client lib, new server, surprising bugs.
- **Forgetting that HTTP/3 still uses TLS 1.3.** Certificate management, OCSP stapling, key rotation are unchanged — you just don't run them at a separate layer.
- **Confusing QUIC with HTTP/3.** QUIC is the transport; HTTP/3 is one application over it. WebTransport, DNS-over-QUIC, MOQ are others.
- **Believing HTTP/2 server push will be saved by HTTP/3.** It won't. Use `103 Early Hints` instead.

---

## 14. Cheat Card

```
PURPOSE   A modern transport that fixes TCP+TLS's biggest
          performance and operational pains by running over UDP
          with built-in TLS 1.3, independent streams, and
          connection migration.

THE STACK
  HTTP/3 frames
    over QUIC streams (multiplexed, independent)
      over QUIC packets (TLS 1.3 encrypted)
        over UDP

KEY WINS
  1 RTT handshake (down from 2–3)
  0-RTT for repeat connections
  No HOL blocking across streams
  Connection survives IP change (Wi-Fi ↔ LTE)
  User-space congestion control (BBR, evolving)
  Encrypted everything → can't be ossified by middleboxes

HTTP/3 SPECIFICS
  One QUIC stream per request/response
  QPACK header compression
  No server push (use 103 Early Hints)
  Discovered via Alt-Svc header or HTTPS DNS RR

WHERE QUIC WINS
  Mobile networks, high RTT, packet loss
  Cross-continent or last-mile connections
  Repeat visits (0-RTT)
  Networks where users switch (Wi-Fi/LTE)

WHERE IT BARELY MATTERS
  Datacenter networks (0% loss, µs RTT)
  Pure LAN
  Very simple single-request flows

OPERATIONAL CHECKLIST
  UDP/443 open through your stack
  UDP buffer sysctls tuned
  Load balancer routes by connection ID (not 4-tuple)
  qlog / metrics on the QUIC layer
  Alt-Svc + HTTPS DNS RR advertised
  Restrict 0-RTT to idempotent operations
  Fallback to TCP+TLS preserved for blocked networks

QUIC BEYOND HTTP
  WebTransport          browser API for streams + datagrams
  DNS over QUIC (DoQ)   encrypted, fast DNS
  MOQ                   media streaming (work in progress)
  MASQUE                proxy-via-QUIC (iCloud Private Relay)

PITFALLS
  Blocking UDP/443 at firewall
  Connection-ID-unaware load balancer
  Buffer defaults too small under load
  0-RTT enabled for non-idempotent endpoints
  No qlog / observability for the QUIC layer
  Testing only on local low-loss networks

RULE   Turn on HTTP/3 at the edge — it's almost free latency.
       The hard parts are someone else's problem; the wins are
       largely yours.
```

---

## 15. Resources

### Specifications
- **RFC 9000** — QUIC: A UDP-Based Multiplexed and Secure Transport.
- **RFC 9001** — Using TLS to Secure QUIC.
- **RFC 9002** — QUIC Loss Detection and Congestion Control.
- **RFC 9114** — HTTP/3.
- **RFC 9204** — QPACK.
- **RFC 9460** — HTTPS / SVCB DNS Resource Records.

### Documentation
- **HTTP/3 Explained** — Daniel Stenberg (curl): <https://http3-explained.haxx.se/>
- **Cloudflare QUIC** — <https://blog.cloudflare.com/tag/quic/>
- **Chromium QUIC** — <https://www.chromium.org/quic/>
- **quiche (Cloudflare)** — <https://github.com/cloudflare/quiche>
- **msquic (Microsoft)** — <https://github.com/microsoft/msquic>
- **ngtcp2** — <https://github.com/ngtcp2/ngtcp2>
- **aioquic (Python)** — <https://github.com/aiortc/aioquic>
- **quic-go (Go)** — <https://github.com/quic-go/quic-go>
- **s2n-quic (Rust, AWS)** — <https://github.com/aws/s2n-quic>

### Articles
- "Inside QUIC" — Cloudflare engineering blog posts.
- "HTTP/3 from A to Z" — Robin Marx series (excellent).
- "The HTTPS-only Web" — Tim Berners-Lee and W3C posts.
- "Connection migration in QUIC" — RFC and engineering writeups.
- "qlog: a tool for QUIC debugging" — Robin Marx et al.

### Books
- *HTTP/2 in Action* — Barry Pollard (covers HTTP/2 thoroughly; HTTP/3 inherits most ideas).
- *High Performance Browser Networking* — Ilya Grigorik. Chapters on transports are foundational.

### Videos
- *HTTP/3 talks* — Daniel Stenberg, Robin Marx, Lucas Pardue.
- *QUIC: a new transport* — Jana Iyengar talks.
- ByteByteGo — "QUIC and HTTP/3 Explained."
- IETF / QUIC working group recordings.

### Tools
- **curl** — supports HTTP/3 with `--http3`.
- **`nghttp3`** — HTTP/3 testing client.
- **`quiche-client`** — Cloudflare's QUIC test tool.
- **qlog viewers** — qvis.io, qlog2csv.
- **Wireshark** — captures and dissects QUIC packets when given the keys.
- **`server-timing`** + **RUM** for client-side latency measurement.

### Adjacent reading
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC) →](../02-networking/http-versions.md)
- [TCP vs UDP →](../02-networking/tcp-vs-udp.md)
- [HTTPS, TLS/SSL Handshake →](../02-networking/https-tls.md)
- [WebSockets →](../02-networking/websockets.md)
- [Edge Computing →](./edge-computing.md)
- [WebRTC for Real-Time Media →](./webrtc.md)
- [DNS — How It Works →](../02-networking/dns.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)
- [Connection Pooling & Keep-Alive →](../16-performance/connection-pooling.md)

---

*Previous:* [← WebRTC for Real-Time Media](./webrtc.md)  |  *Next:* [Multi-Tenant SaaS Architecture →](./multi-tenant-saas.md)
