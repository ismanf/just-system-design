# TCP vs UDP

> **TL;DR** — Both are **Layer 4 transport protocols** that move data between processes (identified by ports). **TCP** is a *reliable, ordered, connection-oriented byte stream* — it guarantees delivery, in order, with congestion control, at the cost of latency and overhead. **UDP** is a *fire-and-forget datagram* protocol — it just throws packets and never looks back; the application deals with loss, ordering, retries. TCP is the default for "I need every byte to arrive correctly" (HTTP, SSH, databases). UDP is for "I'd rather have fresh data than late data" (DNS, video, games, real-time audio). **QUIC** is the modern third option — UDP underneath, TCP-like reliability built into the app layer.

---

## 1. The Big Picture

```
┌──────────────────────────────────────────────────────────────┐
│                       Application                             │
│  HTTP   gRPC   SSH   DB   DNS   Video   VoIP   Gaming  …      │
├──────────────────────────────────────────────────────────────┤
│                         TCP    │    UDP    │   QUIC           │
│   (reliable, ordered)          │  (fast,    │  (UDP base,     │
│                                │   lossy)   │   TCP-like API) │
├──────────────────────────────────────────────────────────────┤
│                            IP                                 │
└──────────────────────────────────────────────────────────────┘
```

| Feature | TCP | UDP |
| --- | --- | --- |
| Connection-oriented | ✅ handshake first | ❌ just send |
| Reliable delivery | ✅ retransmits lost packets | ❌ app's problem |
| Ordering | ✅ in-order | ❌ may arrive out of order |
| Flow control | ✅ receiver window | ❌ |
| Congestion control | ✅ slow start, AIMD, BBR, CUBIC | ❌ |
| Header size | 20+ bytes | 8 bytes |
| Speed (raw) | Slower (state, ACKs) | Faster |
| Use case | Anything where bytes must arrive | Latency-critical, loss-tolerant |
| Packet unit | "Segment" (part of a stream) | "Datagram" (one whole message) |
| Common ports | 80, 443, 22, 5432, 6379 | 53, 123, 161, 1812, 5060 |

---

## 2. TCP — Reliable Byte Stream

### Properties
- **Connection-oriented.** Two endpoints negotiate a session (the 3-way handshake) before data flows.
- **Reliable.** Every byte sent gets an acknowledgment; lost segments are retransmitted.
- **In-order delivery.** Receiver buffers out-of-order arrivals and presents them in sequence.
- **Stream-based.** The app writes/reads bytes; TCP doesn't preserve message boundaries.
- **Flow controlled.** Receiver advertises a window so the sender doesn't drown it.
- **Congestion controlled.** Sender backs off when the network shows signs of overload.

### The 3-Way Handshake
```
Client                          Server
  │ ── SYN ────────────────────► │     "Open connection, my seq = X"
  │ ◄── SYN-ACK ──────────────── │     "OK, my seq = Y, ack X+1"
  │ ── ACK ────────────────────► │     "Got it, ack Y+1"
  │                                          │
  │ ◄════ data flows ═══════════►  │
```

This costs **1 RTT** before any data is sent. Across continents that's 100–200 ms of pure wait time. (Mitigations: TCP Fast Open, TLS 1.3 1-RTT, HTTP/3 0-RTT.)

### Connection Termination — 4-Way
```
Client                          Server
  │ ── FIN ────────────────────► │
  │ ◄── ACK ──────────────────── │
  │ ◄── FIN ──────────────────── │
  │ ── ACK ────────────────────► │
```
The TIME_WAIT state then sits on a socket for 2× the maximum segment lifetime (~30–120 s) to avoid old packets confusing a new connection on the same ports. This is the source of countless "address already in use" errors.

### The TCP Header (the bits that matter)
```
0       8       16      24      32
+-------+-------+-------+-------+
| Source port   | Dest port     |
+---------------+---------------+
|       Sequence number         |
+-------------------------------+
|     Acknowledgment number     |
+---+---+-----+-+-+-+-+-+-+-+---+
|HL |   |Flags| F| S| R| P| A| U| Window |
|   |   |     | I| Y| S| S| C| R|        |
|   |   |     | N| N| T| H| K| G|        |
+---+---+-----+-+-+-+-+-+-+-+---+
| Checksum      | Urgent ptr    |
+---------------+---------------+
| Options (variable, 0–40 B)    |
+-------------------------------+
| Data...                       |
```
Notable flags:
- **SYN** — start of connection.
- **ACK** — acknowledges data up to ack#.
- **FIN** — graceful close.
- **RST** — abrupt close (often: "go away").
- **PSH** — "flush this immediately".

### Flow Control vs Congestion Control
These two are often confused.
- **Flow control** — protects the *receiver*. The receiver advertises a "window" (how many bytes it can buffer). Sender doesn't exceed it.
- **Congestion control** — protects the *network*. Sender starts slow (slow start), grows window until loss, then backs off (AIMD), recovers, repeats. Algorithms: Reno, CUBIC (Linux default), BBR (Google, latency-aware).

### Retransmissions
- Sender sets a Retransmission Timeout (RTO). If no ACK arrives, retransmit.
- **Fast retransmit** — receiver sees out-of-order, sends duplicate ACKs; after 3 dup-ACKs, sender retransmits immediately without waiting for timeout.
- **SACK** (Selective ACK) — receiver tells sender exactly which bytes are missing so only those get resent.

### Head-of-Line Blocking
Because TCP is **in-order**, one lost packet stalls every subsequent packet — even if they arrived. This is why HTTP/2 (multiplexed over a single TCP connection) suffered head-of-line blocking and why HTTP/3 moved to QUIC (UDP).

---

## 3. UDP — Send and Forget

### Properties
- **Connectionless.** No handshake. First packet is *the* packet.
- **Unreliable.** No retransmits, no acknowledgments, no guarantees.
- **No ordering.** Packets may arrive in any order or not at all.
- **Datagram-based.** Each `sendto()` is one message; receiver sees that boundary.
- **No flow / congestion control.** App must handle this.
- **Tiny header.** 8 bytes.

### The UDP Header
```
0       8       16      24      32
+---------------+---------------+
| Source port   | Dest port     |
+---------------+---------------+
| Length        | Checksum      |
+---------------+---------------+
| Data...                       |
```
That's it. Compare to TCP's 20+ bytes — UDP is *the* minimum viable transport.

### When UDP makes sense
- **Latency matters more than completeness.** Real-time voice/video — a missed audio packet is better than a 500 ms wait.
- **Many short requests.** DNS queries fit in one packet; opening a TCP connection per lookup would be wasteful.
- **Broadcast / multicast.** TCP can't do these; UDP can.
- **Custom reliability.** Games and QUIC build their own reliability scheme on top of UDP — they want control.

### When UDP burns you
- The middle box (firewall, NAT, ISP) drops UDP traffic. UDP is treated as second-class on much of the internet.
- The app has no idea a packet was lost. *You* must build retry, ordering, congestion control if you need them.
- Packet size limits — IP-level MTU (~1500 bytes) without fragmentation; ~64 KB hard max.

---

## 4. QUIC — The Modern Third Option

QUIC = "Quick UDP Internet Connections." Designed by Google, standardized as IETF QUIC (RFC 9000). It's the transport under HTTP/3.

```
HTTP/3
  └─ over QUIC (reliable streams, TLS 1.3 built in)
        └─ over UDP
              └─ over IP
```

### Why UDP underneath?
TCP is implemented in the OS kernel and can't be upgraded easily (firewalls, middleboxes, OS kernels everywhere). UDP is the lowest common denominator that gets through. QUIC's reliability lives in **user space** so it can evolve.

### What QUIC gives you
- **0-RTT** or **1-RTT** connection establishment (vs TCP+TLS 1.3's 2 RTT).
- **Multiplexed streams** with no head-of-line blocking — a lost packet in one stream doesn't stall others.
- **Built-in TLS 1.3** — no separate handshake.
- **Connection migration** — survive a client switching from WiFi to cellular without dropping.
- **Modern congestion control** (BBR-friendly).

### Where it's deployed
- YouTube, Google Search, most Google services.
- Cloudflare's edge.
- Most modern browsers (HTTP/3).
- Many CDNs.

QUIC is gradually replacing TCP for HTTP, but TCP isn't going anywhere — most non-HTTP traffic still runs on it.

---

## 5. Ports

Both TCP and UDP use 16-bit ports (0–65535) to identify the *process* on a host.

- **0–1023** — Well-known ports (HTTP=80, HTTPS=443, SSH=22, DNS=53, etc.). Privileged on Unix.
- **1024–49151** — Registered ports (Postgres=5432, Redis=6379, MySQL=3306, etc.).
- **49152–65535** — Ephemeral / dynamic. Client connections get one of these.

A TCP connection is uniquely identified by the **5-tuple**: `(protocol, src IP, src port, dst IP, dst port)`. That's why a server on port 443 can handle millions of concurrent clients — each has a different src IP/port combination.

```
Server: 1.2.3.4:443
Client A: 10.0.0.5:53210  ◄────── 1 conn
Client B: 10.0.0.6:60112  ◄────── another conn (same server, same port)
Client A: 10.0.0.5:53211  ◄────── ANOTHER conn from same client (different src port)
```

### The 65k port myth
"A server can only have 65k connections" — wrong. The server side reuses port 443; the *uniqueness* is in the 5-tuple. Modern servers (epoll/kqueue) routinely handle 100k+ concurrent connections.

What *is* limited: a single client opening many connections to the *same* server (it runs out of ephemeral ports).

---

## 6. Connection Pooling & Keep-Alive

Because TCP handshake costs an RTT, modern clients **reuse** connections.

- **HTTP/1.1 Keep-Alive** — reuse the same connection for multiple requests sequentially.
- **HTTP/2** — multiplex many requests over one connection in parallel.
- **HTTP/3 / QUIC** — same, with no HOL blocking.
- **Database pools** (PgBouncer, HikariCP) — keep DB connections warm and lease them to threads.

If you don't pool, you'll pay an RTT (or two, if TLS) on every request. At 100 ms RTT across regions, that's an enormous cost.

---

## 7. Protocol Choice Decision Tree

```
Do you need every byte to arrive in order?
├── Yes  ─ Use TCP (or QUIC if you control both ends).
│         Examples: HTTP, file transfer, DB, SSH, email.
│
└── No   ─ Is fresh data more valuable than complete data?
           ├── Yes ─ Use UDP. Examples: video/voice, NTP, DNS, games.
           │
           └── No ─ Maybe TCP after all. Re-examine.

Do you need very low latency + reliability + your own protocol?
   → QUIC (or build on UDP).
```

---

## 8. Real-World Examples

| Protocol | L4 | Why that choice |
| --- | --- | --- |
| HTTP/1.1, HTTP/2 | TCP | Need exact bytes in order |
| HTTP/3 | QUIC (UDP) | Avoid TCP HOL blocking, faster handshake |
| SSH | TCP | Reliability, ordered shell |
| TLS | TCP (or DTLS on UDP) | The transport guarantees TLS expects |
| DNS query | UDP (TCP fallback for big responses) | Tiny, latency-critical |
| DNS zone transfer | TCP | Large, reliable |
| NTP | UDP | One-shot, fresh > complete |
| VoIP / video calls | UDP | Latency > completeness |
| SNMP | UDP | Monitoring, fire-and-forget |
| Postgres / MySQL | TCP | Long-lived reliable session |
| Redis | TCP | Same |
| Memcached | UDP or TCP (mostly TCP today) | Used to use UDP at scale; mostly TCP now |
| Kafka | TCP | Reliable log delivery |
| WebRTC media | UDP (via SRTP) | Real-time audio/video |
| QUIC / HTTP/3 | UDP | Modern internet transport |
| Online games | UDP (+ custom reliability) | Tick rate > completeness |

---

## 9. Visible Symptoms by Layer

If you're debugging, the kind of error you see tells you which layer is failing:

| Symptom | Likely cause |
| --- | --- |
| Connection refused | Server isn't listening on that port (TCP) |
| Connection timed out | SYN got no SYN-ACK — port blocked, host down, or wrong IP |
| Connection reset by peer | RST received — server closed abruptly |
| Broken pipe | You wrote to a socket the other side closed |
| Address already in use | Port still in TIME_WAIT (TCP); use `SO_REUSEADDR` |
| Too many open files | Out of file descriptors — tune `ulimit -n` |
| Cannot assign requested address | Out of ephemeral ports (rare, but happens) |
| UDP "lost" packets (silently) | UDP has no notion of "lost" — your app must handle this |

---

## 10. Performance Tips

### TCP
- Reuse connections. Connection pooling is non-negotiable.
- Tune the kernel for high-throughput: `net.core.somaxconn`, `tcp_max_syn_backlog`, `tcp_rmem`, `tcp_wmem`.
- Enable TCP BBR for lower latency under loss.
- Watch for TIME_WAIT exhaustion on clients (use `SO_LINGER` / `SO_REUSEADDR`).
- Set sensible timeouts. Forever-waiting connections kill you.

### UDP
- Build a backpressure / retry / dedupe strategy in the app.
- Pre-size buffers (`SO_RCVBUF`, `SO_SNDBUF`).
- Keep payloads under MTU to avoid IP fragmentation (~1400 bytes is safe).
- Don't trust the source IP — UDP is trivially spoofable.

### QUIC
- Use a modern library (`quiche`, `msquic`, `go-quic`).
- Make sure UDP isn't blocked between client and server (corporate networks sometimes block it).
- Expect to live alongside TCP for years.

---

## 11. Common Mistakes

- **Picking UDP "for speed" without handling loss.** You will lose data.
- **Assuming TCP guarantees apply to your message boundaries.** It doesn't — TCP is a *byte stream*. You need a length prefix or delimiter.
- **Not closing connections cleanly.** Leaked sockets pile up to file-descriptor limits.
- **Ignoring TIME_WAIT.** Common on load-test boxes hammering one server from one IP.
- **Using HTTP/1.1 with no keep-alive in latency-sensitive code.** Every request opens a fresh TCP+TLS.
- **Trusting UDP source IPs.** They can be spoofed.
- **Designing an in-house "reliable UDP" protocol when QUIC exists.** Don't.

---

## 12. Cheat Card

```
┌────────────────────────────────────────────────────────────────┐
│ TCP                          UDP                               │
│  connection-oriented         connectionless                    │
│  reliable, ordered           unreliable, unordered             │
│  flow + congestion control   none                              │
│  20-byte header              8-byte header                     │
│  3-way handshake (1 RTT)     no handshake                      │
│  HOL blocking                no HOL (each datagram independent)│
│  USE FOR: HTTP, SSH, DB,      USE FOR: DNS, NTP, video, voice, │
│           SMTP, files, gRPC          games, telemetry          │
│                                                                │
│ QUIC = UDP + TCP-like reliability + TLS 1.3 + 0/1-RTT.         │
│ Modern internet transport (HTTP/3).                            │
│                                                                │
│ 5-TUPLE: protocol, src IP, src port, dst IP, dst port          │
│ One server port handles millions of connections this way.      │
│                                                                │
│ ALWAYS connection-pool over the WAN.                           │
└────────────────────────────────────────────────────────────────┘
```

---

## 13. Resources

### Foundational
- **W. Richard Stevens, *TCP/IP Illustrated Vol. 1*** — the canonical text.
- **Kurose & Ross, *Computer Networking: A Top-Down Approach*** — accessible textbook.
- **RFC 9293** — TCP (modernized 2022): <https://datatracker.ietf.org/doc/html/rfc9293>
- **RFC 768** — UDP (one page!): <https://datatracker.ietf.org/doc/html/rfc768>
- **RFC 9000** — QUIC: <https://datatracker.ietf.org/doc/html/rfc9000>

### Articles
- "High Performance Browser Networking" — Ilya Grigorik, free online: <https://hpbn.co/>
- Cloudflare Learning — TCP vs UDP: <https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/>
- Cloudflare — HTTP/3 explained: <https://blog.cloudflare.com/http3-the-past-present-and-future/>
- Daniel Stenberg's QUIC explainers (curl maintainer): <https://daniel.haxx.se/blog/>

### Videos
- **Hussein Nasser** — entire playlists on TCP, UDP, QUIC: <https://www.youtube.com/@hnasr>
- **ByteByteGo** — TCP vs UDP animation: <https://www.youtube.com/@ByteByteGo>
- "How DNS Works" — animated tutorials.

### Tools
- `wireshark` / `tshark` — packet inspection, the canonical learning tool.
- `tcpdump` — CLI packet capture.
- `ss`, `netstat` — see active sockets.
- `iperf3` — throughput testing.
- `mtr` — traceroute + ping combined.

---

*Previous:* [← IP Addressing, Subnets, CIDR](./ip-subnets-cidr.md)  |  *Next:* [DNS — How It Works →](./dns.md)
