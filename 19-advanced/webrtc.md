# WebRTC for Real-Time Media

> **TL;DR** — **WebRTC** (Web Real-Time Communication) is a browser-native stack for **peer-to-peer audio, video, and data channels** with sub-100 ms latency. It is the technology behind Google Meet, Discord voice/video, Zoom Web, Slack huddles, Twitch's low-latency mode, and most modern WebRTC-based live-streaming products. The stack combines **media codecs (Opus, VP8/VP9, H.264, AV1)**, **UDP-based transport with congestion control (SCTP, DTLS, ICE)**, **NAT traversal (STUN/TURN)**, and a **signaling channel** you provide. Once a connection is established, media flows directly between peers; nothing routes through a server. The honest take: **WebRTC is the right answer for any low-latency interactive media** — but "peer-to-peer" rarely scales past 4 participants. Real production stacks use **SFUs (Selective Forwarding Units)** or **MCUs (Multipoint Control Units)** as centralized media routers. Most of the engineering work is in **signaling, NAT traversal, codec choice, jitter buffers, and bandwidth adaptation**, not in WebRTC itself.

---

## 1. The big picture

```
┌────────────┐                                        ┌────────────┐
│  Alice     │                                        │   Bob      │
│  browser   │                                        │  browser   │
└─────┬──────┘                                        └─────┬──────┘
      │  1. Signaling (offer/answer SDP)                    │
      │  via WebSocket / HTTPS / SSE                        │
      ▼                                                     ▼
   ┌────────────────────────────────────────────────────────────┐
   │ Signaling server (you build this)                          │
   └────────────────────────────────────────────────────────────┘
      │                                                     │
      │  2. ICE candidate gathering                         │
      │  STUN → public IP/port                              │
      │  TURN → relay fallback                              │
      │                                                     │
      ▼                                                     ▼
   ┌────────────────────────────────────────────────────────────┐
   │ STUN / TURN servers                                        │
   └────────────────────────────────────────────────────────────┘
      │                                                     │
      │  3. Direct peer connection                          │
      ▼                                                     ▼
  ╔════════╗ ◄────  DTLS-SRTP encrypted media ────► ╔════════╗
  ║ Alice  ║ ◄────  SCTP encrypted data channel ──► ║  Bob   ║
  ╚════════╝                                        ╚════════╝
```

Three distinct phases:

1. **Signaling** — Alice and Bob exchange offers, answers, and ICE candidates. WebRTC doesn't specify how; you provide a side channel (WebSocket is the default).
2. **ICE / NAT traversal** — Each peer figures out reachable IP/port combinations. STUN discovers the public address; TURN relays when direct fails.
3. **Direct media flow** — Once a connection is established, audio/video/data flow peer-to-peer over encrypted UDP.

That's the whole protocol shape. Everything else is implementation detail.

---

## 2. Why WebRTC exists

Before WebRTC (~2011), real-time browser communication required **plugins** — Flash, Java, custom native apps. Each platform was different. There was no standard way to capture a camera, encode video efficiently, traverse NAT, and stream over the open internet in a browser.

WebRTC standardized:

- Browser APIs for camera, microphone, and screen capture (**`getUserMedia` / `getDisplayMedia`**).
- An **encoding/decoding pipeline** for industry-standard codecs.
- A **peer connection abstraction** (`RTCPeerConnection`) that handles SDP, ICE, DTLS, SRTP, congestion control.
- A **data channel** abstraction (`RTCDataChannel`) for arbitrary application data over the same connection.

The killer property: **it works in every modern browser, with no plugin, no install, no extension**. Click a link, the browser does the rest.

Today, every browser plus iOS Safari, Android WebView, Electron, React Native, mobile native SDKs (libwebrtc) support it. Production real-time media at internet scale is built on this stack.

---

## 3. The signaling layer (you build this)

WebRTC doesn't include signaling. You connect peers via your own channel — typically:

- **WebSocket** server (most common).
- **Server-Sent Events** + HTTP POST.
- **Firebase Realtime Database** / **Supabase Realtime** (for prototypes).
- **Matrix federation**, **XMPP** (for chat-rooted systems).

Signaling carries:

- **Session Description Protocol (SDP)** offers and answers — the codec list, supported features, network capabilities.
- **ICE candidates** — pairs of IP/port/protocol that each peer thinks it can be reached on.
- Optional metadata: room joining, user identity, permission grants.

```
Alice         Signaling server         Bob
  │                  │                  │
  │── offer SDP ────►│                  │
  │                  │── offer SDP ────►│
  │                  │                  │
  │                  │◄── answer SDP ───│
  │◄── answer SDP ───│                  │
  │                  │                  │
  │── ICE cand ─────►│── ICE cand ─────►│
  │◄── ICE cand ─────│◄── ICE cand ─────│
  │  ... many ICE candidates trickle ...│
```

After exchange, both peers attempt direct connections using each ICE candidate pair. The first pair that succeeds becomes the media path.

**Signaling is where most product-level engineering happens**: room management, user identity, presence, permissions, recording, multi-party orchestration. The actual WebRTC bits are off-the-shelf.

---

## 4. ICE, STUN, TURN — the NAT-traversal trio

The vast majority of internet users sit behind NAT. WebRTC's **Interactive Connectivity Establishment (ICE)** framework handles this.

### STUN — Session Traversal Utilities for NAT

A simple server that tells a client "this is what your public IP/port look like from out here." Free, lightweight, runs on UDP. Most ISPs allow STUN traffic.

Public STUN servers exist (Google's `stun.l.google.com:19302`), but production should run their own to avoid privacy / availability concerns.

### TURN — Traversal Using Relays around NAT

When STUN doesn't work (symmetric NAT, restrictive firewalls), TURN servers **relay traffic** for both peers. The TURN server sees all the media; you pay for that bandwidth.

About **5–20% of WebRTC sessions** fall back to TURN, depending on network conditions and user demographics (corporate, mobile carrier-grade NAT, etc.).

TURN is expensive — you're paying for media-rate bandwidth. Production services use:
- **`coturn`** (open source, self-hosted).
- **Twilio**, **Xirsys**, **Cloudflare Calls** — managed TURN-as-a-service.
- **Multi-region TURN** to minimize relay latency.

### ICE candidate gathering

Each peer gathers candidates: host, server-reflexive (STUN-discovered), peer-reflexive (discovered during ICE itself), relayed (TURN). All exchanged via signaling. The pair with the lowest priority that connects wins.

ICE is part of why WebRTC takes 1–5 seconds to establish a connection. Trickle ICE (sending candidates as they're discovered rather than batching) shaves time.

---

## 5. SDP — the negotiation language

**Session Description Protocol** is a 30-year-old text format inherited from VoIP. A WebRTC offer is a multi-page SDP blob describing:

- Audio codecs and parameters (`opus/48000/2`).
- Video codecs (`VP8`, `VP9`, `H264`, `AV1`).
- RTCP feedback (`nack`, `pli`, `goog-remb`, `transport-cc`).
- DTLS fingerprints.
- ICE ufrag and password.
- Bandwidth caps, simulcast layers.

You almost never write SDP by hand. The browser generates it; you exchange it; you let the browser parse it. Where you do touch SDP is **munging** — manually editing the offer/answer to force codec preferences, enable simulcast, change bandwidth limits, etc. This is operationally fragile but sometimes necessary.

---

## 6. Media — codecs, jitter, congestion

### Codecs

| Codec | Type | Pros | Cons |
|---|---|---|---|
| **Opus** | Audio | Royalty-free, low latency, wide bitrate range (6–512 kbps) | None worth mentioning — it's a clear default |
| **VP8** | Video | Royalty-free, broadest compatibility | Slightly lower compression than newer |
| **VP9** | Video | ~30% better than VP8 | Higher CPU |
| **H.264** | Video | Hardware decode on every device | Patent licensing varies |
| **AV1** | Video | Highest compression | High encode CPU; hardware support still growing in 2026 |

In practice: **Opus + VP8** is the lowest-common-denominator pair. Production stacks negotiate the best mutually supported pair.

### Jitter buffer

UDP packets arrive out of order, delayed, sometimes lost. The receiver buffers a small window (50–200 ms typically) to smooth playback. Larger buffer = fewer glitches but higher latency. WebRTC implementations adapt dynamically.

### Congestion control

WebRTC uses **GCC (Google Congestion Control)** or newer **transport-cc** to detect available bandwidth and adapt video bitrate / resolution. Without this, a single sender saturates the link and quality collapses.

Two control loops:
- **Encoder side** — reduce bitrate, drop resolution, drop frame rate based on feedback.
- **Receiver side** — RTCP feedback to senders.

### Simulcast

Sender encodes **multiple resolutions** (e.g., 1080p, 360p, 180p) simultaneously and ships them all. The SFU (see §7) decides which to forward to each receiver based on their bandwidth and viewport.

Simulcast is **the** secret to good multi-party video. Without it, one slow receiver drags everyone's quality down.

### SVC (Scalable Video Coding)

A single encoded stream with multiple "layers" — base + enhancements. The SFU drops layers per receiver. More CPU-efficient than simulcast but less mature; AV1 has strong SVC support.

---

## 7. SFU vs MCU vs Mesh — the multi-party question

Two peers — easy. Direct connection. Five peers — much harder. Pure peer-to-peer mesh requires each peer to send 4 outbound streams (4 × upload bandwidth).

Three architectures:

### Mesh (full peer-to-peer)

```
A ─── B
│ \ / │
│  X  │     each peer connects to every other
│ / \ │
C ─── D
```

For 4 peers: each sends 3× upload bandwidth. Above ~4 participants, bandwidth and CPU on each peer collapse. **Use mesh only for ≤4 participants.**

### SFU — Selective Forwarding Unit

```
A ───►│                   ┌──► A
B ───►│ ◄── SFU ──►       ├──► B
C ───►│                   ├──► C
D ───►│                   └──► D
```

Each peer sends **one** stream to the SFU; the SFU forwards copies to other peers (often filtered by simulcast layer). Peers' upload bandwidth stays manageable. The SFU **doesn't decode media** — it just routes. CPU cost on the server is low; bandwidth cost is high.

This is the dominant production pattern. Used by every major video conferencing product (Zoom, Meet, Teams, Slack huddles, Discord). Open-source SFUs:

- **mediasoup** (Node.js, very popular).
- **Janus** (mature C, many plugins).
- **LiveKit** (Go, opinionated, modern).
- **Pion** (Go library, build your own SFU).
- **Jitsi Videobridge** (Java, ships with Jitsi Meet).
- **Ant Media Server**.
- **OvenMediaEngine**.

Managed: **Twilio Video**, **Vonage**, **LiveKit Cloud**, **Daily**, **Agora**, **Dyte**, **Whereby Embedded**, **Cloudflare Calls** (SFU-as-a-service).

### MCU — Multipoint Control Unit

```
A ───►│                   ┌──► A (one mixed stream)
B ───►│ ── MCU mixes ──►  ├──► B (one mixed stream)
C ───►│                   ├──► C (one mixed stream)
D ───►│                   └──► D (one mixed stream)
```

The MCU **decodes, mixes, re-encodes** a single composite stream per receiver. Receivers see and download only one stream. Bandwidth-efficient on receivers; **CPU-heavy on the server** (transcoding for every participant). Used historically for limited-bandwidth telephony / conference room equipment. Largely displaced by SFU + simulcast for modern web/mobile.

The picture in 2026: **SFU is the default**. MCU lives in interop scenarios (calling into the PSTN, broadcasting to RTMP/HLS).

---

## 8. Data channels

Beyond audio/video, WebRTC supports `RTCDataChannel` — arbitrary message-passing over the same encrypted connection. Built on **SCTP over DTLS over UDP**, with two configuration knobs:

- **Reliability**: ordered + reliable (TCP-like), unordered, unreliable.
- **Backpressure**: per-channel buffering with `bufferedAmount` API.

Use cases:
- Cursor / pointer sync in collaborative tools (Figma early days used this).
- Game state in browser-based games.
- File transfer (WebTorrent, ShareDrop).
- Low-latency control signals alongside media (mute, gesture).

Bandwidth: gigabits per second peer-to-peer on a good link. Often faster than HTTP for browser-to-browser transfers because there's no server hop.

---

## 9. Security

WebRTC is **encrypted by default**, end-to-end between peers (or peer-to-SFU). The encryption layers:

- **DTLS** — handles the handshake and key exchange.
- **SRTP** — encrypts media (audio/video) using keys derived from DTLS.
- **SCTP over DTLS** — encrypts data channels.

Identity is **not** automatic. The signaling channel must authenticate users; if not, an attacker could swap SDP and MITM. Production stacks always pair WebRTC with their auth system.

For multi-party SFU sessions, the SFU sees all media in transit (E2E encryption is then **client-to-SFU**, not client-to-client). True **E2E encryption** in group calls requires additional schemes:

- **DTLS-SRTP per pair** (works only for mesh).
- **Frame-level E2E** with derived keys (Insertable Streams, MLS — Messaging Layer Security).
- **Zoom End-to-End** and **Apple FaceTime** use variations.

In 2026, end-to-end encrypted group video is becoming standard for premium tiers (Meet, Discord Stage Channels, etc.) but operational complexity is real.

---

## 10. Recording, broadcasting, integrating

Real-world products often need more than peer media:

### Recording

The server-side SFU can fork media to a recorder that mixes and saves an MP4. Tools: **mediasoup-recorder**, **GStreamer pipelines**, **FFmpeg**, **LiveKit's egress**.

### Live streaming / broadcast

For one-to-many at scale (10K+ viewers), WebRTC becomes inefficient. Convert to **HLS** or **DASH** chunked HTTP streams. Latency goes from ~200 ms (WebRTC) to 2–10 s (LL-HLS) or 10–30 s (regular HLS). Trade-off for scale.

**WebRTC-to-HLS bridges** are routine in modern stacks (Mux, Daily.co, Restream).

### RTMP ingest

Streamers (OBS, Twitch Studio) push to your service via RTMP, you transcode and republish as WebRTC / HLS. Standard in livestreaming.

### PSTN — phone integration

WebRTC ↔ telephone numbers via SIP gateways (FreeSWITCH, Asterisk) or managed services (Twilio, Vonage).

---

## 11. Latency budgets

Where the time goes in a typical call:

```
20 ms   audio capture buffer
20 ms   encoder
~30 ms  network one-way (varies wildly)
50 ms   jitter buffer
20 ms   decoder
20 ms   audio playback buffer
────────
~160 ms typical mouth-to-ear (good network)
```

For comparison:
- **Phone call** (PSTN): ~200 ms.
- **Zoom** (SFU): 150–300 ms typical.
- **WebRTC P2P** on good network: 80–150 ms.
- **HLS / DASH**: 2,000–10,000 ms.

Sub-200 ms feels live. Above ~300 ms, conversations start stepping on each other. WebRTC's whole point is staying below that threshold.

---

## 12. Operational concerns

### Bandwidth shape

WebRTC is **upload-heavy** for senders. A 720p video upload is ~1.5 Mbps; 1080p ~3 Mbps. Mesh calls with many peers blow this up fast.

### CPU

Video encode/decode is the dominant CPU cost. Hardware acceleration (VPU, GPU) helps a lot on modern devices. Older hardware struggles with VP9/AV1.

### Monitoring

`getStats()` exposes detailed metrics per connection: jitter, loss, RTT, bitrate, codec, FEC. Production stacks ingest this into observability pipelines (Datadog, Honeycomb) to track call quality at fleet scale.

### Mobile / battery

WebRTC drains battery quickly. Native SDKs use platform-specific optimizations. Backgrounding a call disables video. Background audio is allowed via VoIP entitlements.

### Geographic distribution

Latency between Tokyo and São Paulo is bad. Multi-region SFUs route media via the closest SFU per peer, with SFU-to-SFU mesh. Cloudflare Calls, Twilio's Global Low Latency, LiveKit's multi-region are examples.

### Cost

For a 4-person 1080p meeting:
- Bandwidth: ~12 Mbps per user upstream, ~36 Mbps downstream → tens of GB/hr cluster-wide.
- TURN fallback adds bandwidth-relay cost.
- Recording adds storage and transcoding.

A managed service like Daily or LiveKit Cloud charges roughly $0.001–0.004 per minute per participant for media. At scale, building your own SFU on commodity cloud is 30–60% cheaper but is a real engineering investment.

---

## 13. Common Mistakes / Anti-Patterns

- **Building peer mesh for >4 participants.** Bandwidth and CPU collapse. Use an SFU.
- **No TURN fallback.** 5–20% of users can't connect peer-to-peer. Forget TURN, lose those users.
- **Free public STUN/TURN in production.** Rate-limited, unreliable, privacy concerns. Run your own or use a managed provider.
- **Trying to write your own SFU as the first step.** Use mediasoup, LiveKit, Janus, or Pion — all mature.
- **Ignoring simulcast.** One slow receiver drags everyone to 180p. Always enable simulcast for >2 participants.
- **Hard-coded resolutions/bitrates.** WebRTC's congestion control needs adaptive layers, not fixed targets.
- **Signaling over an unauthenticated channel.** MITM swap of SDP. Always pair with your auth.
- **Forgetting to renegotiate.** Changing tracks (add screenshare) requires a new SDP exchange.
- **Treating recording as free.** Mixing 4×720p streams is real CPU work.
- **No metric dashboards.** "It's broken for this user" with no `getStats()` data is unfixable.
- **Single-region SFU.** A São Paulo user joining a us-east-1 SFU has 130+ ms baseline. Multi-region is essential.
- **Heavy CPU work on the main JS thread on call.** Audio underruns; choppy video. Use Web Workers / Insertable Streams off-thread.
- **Insufficient jitter buffer for mobile networks.** LTE / 5G jitter is much worse than wired; auto-adapt or set higher.
- **Ignoring battery on mobile.** Background, pause video, drop framerate, prefer audio-only modes.
- **Mixing E2E encryption with SFU media manipulation.** Recording / transcoding becomes impossible (which may be the goal — but be deliberate).
- **No call-quality SLO.** Without an MOS-like target, you don't know if quality is regressing.

---

## 14. Cheat Card

```
PURPOSE   Low-latency audio, video, and data between browsers
          (and native apps) — encrypted, NAT-traversing, codec-
          aware, with congestion control built in.

THE STACK
  Capture        getUserMedia / getDisplayMedia
  Negotiation    SDP offer / answer + ICE candidates
  Transport      UDP + DTLS + SRTP + SCTP
  Media          Opus (audio), VP8/VP9/H.264/AV1 (video)
  Data           RTCDataChannel (reliable / unreliable, ordered / unordered)

CONNECTION FLOW
  1. Signaling   (your WebSocket; not part of WebRTC)
  2. ICE         STUN to find public IP, TURN to relay if needed
  3. DTLS        key exchange + encryption setup
  4. Media       SRTP between peers (or peer ↔ SFU)

ARCHITECTURE
  Mesh   ≤4 peers; each peer sends to every other
  SFU    centralized router; one upload, fan-out forwarding
  MCU    server mixes; CPU-heavy; legacy/interop

PRODUCTION SFUs
  mediasoup · Janus · LiveKit · Pion · Jitsi Videobridge
  Managed: LiveKit Cloud · Twilio Video · Daily · Cloudflare Calls

NON-NEGOTIABLES
  TURN servers (5–20% of users need them)
  Simulcast (multi-resolution streams)
  Multi-region SFU for global users
  Signaling auth (MITM-prevention)
  getStats() observability

LATENCY BUDGET
  Capture+encode ~40 ms
  Network        ~20–100 ms
  Jitter buffer  ~50 ms
  Decode+play    ~40 ms
  Total          ~160–300 ms

WHEN WEBRTC WINS
  Interactive audio/video (calls, meetings, telehealth)
  Real-time games and collaboration
  Browser-to-browser file transfer
  Anything where ≤300 ms latency is required

WHEN IT LOSES
  One-to-many broadcast at huge scale → HLS / DASH
  PSTN integration → SIP gateways
  Recording-first workflows → server-side mix is heavier

PITFALLS
  Mesh for >4 participants
  No TURN fallback
  Free public STUN in production
  Hand-rolled SFU as first move
  No simulcast (slow receiver drags everyone)
  Single-region SFU for global users
  Signaling without auth → SDP swap MITM
  No call-quality monitoring

RULE   Direct between peers when you can; SFU when you must.
       TURN as fallback always. Simulcast always. Mediate signaling
       in your own auth. Use a battle-tested SFU; don't roll your
       own first.
```

---

## 15. Resources

### Specifications and standards
- **WebRTC** — W3C: <https://www.w3.org/TR/webrtc/>
- **WebRTC API on MDN** — <https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API>
- **RFC 8825** — overview of WebRTC.
- **RFC 8829** — JSEP (JavaScript Session Establishment Protocol).
- **ICE / STUN / TURN** — RFC 8445 / RFC 8489 / RFC 8656.

### Documentation
- **mediasoup** — <https://mediasoup.org>
- **Janus** — <https://janus.conf.meetecho.com>
- **LiveKit** — <https://livekit.io>
- **Pion** — <https://github.com/pion/webrtc>
- **Jitsi** — <https://jitsi.org>
- **coturn** — <https://github.com/coturn/coturn>
- **Cloudflare Calls** — <https://developers.cloudflare.com/calls/>

### Books
- *Real-Time Communication with WebRTC* — Salvatore Loreto & Simon Pietro Romano.
- *WebRTC Cookbook* — Andrii Sergiienko.
- *High Performance Browser Networking* — Ilya Grigorik (excellent chapter on WebRTC).

### Articles
- "WebRTC for the Curious" — free book: <https://webrtcforthecurious.com>
- "How Discord built their voice/video infrastructure" — Discord engineering.
- "Inside Google Meet" — various Google blog posts on Meet's SFU.
- "How Zoom's architecture works" — Zoom engineering writeups.
- "How Slack built huddles" — Slack engineering blog.

### Videos
- *Kranky Geek* annual conference — WebRTC industry deep dives.
- *WebRTC: The Future of Web Communication* — Sam Dutton talks.
- ByteByteGo — "WebRTC Explained."

### Tools
- **Browser DevTools `chrome://webrtc-internals/`** — invaluable for debugging.
- **fippo's webrtc-stats** — stats parsing for observability.
- **trickle-ice tester** — <https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/>
- **callstats / ClusterCockpit / LiveKit Analytics** — call-quality dashboards.

### Adjacent reading
- [Peer-to-Peer Systems & DHTs →](./p2p-dht.md)
- [Edge Computing →](./edge-computing.md)
- [QUIC & HTTP/3 Internals →](./quic.md)
- [TCP vs UDP →](../02-networking/tcp-vs-udp.md)
- [WebSockets →](../02-networking/websockets.md)
- [Long Polling vs Short Polling →](../02-networking/polling.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)
- [Design Zoom / Video Conferencing →](../18-case-studies/zoom.md)

---

*Previous:* [← Peer-to-Peer Systems & DHTs](./p2p-dht.md)  |  *Next:* [QUIC & HTTP/3 Internals →](./quic.md)
