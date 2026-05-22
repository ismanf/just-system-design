# Compression (Gzip, Brotli, Zstd)

> **TL;DR** — Compression trades **CPU cycles** for **bytes on the wire (or on disk)**. For web traffic, the right default is **gzip** (universal), **Brotli** (better ratio, well-supported, slower to compress at high levels but great as static pre-compressed content), or **Zstandard (zstd)** (a near-Pareto-optimal newcomer with excellent ratio *and* speed). The decision is rarely about which compressor is "best" — it's about **where the bottleneck is**: bandwidth, latency, CPU, or storage. **Compress everything text** on the wire (HTML, JS, CSS, JSON, SVG, fonts). **Don't compress already-compressed bytes** (images, video, archives). **Tune compression level by content** — level 1 for hot dynamic responses, level 9–11 for assets cached forever. The deep lesson: a smaller payload almost always wins on the open Internet, even at meaningful CPU cost, because **bandwidth is shared, slow, and lossy at scale** — and faster bytes hit the user faster.

---

## 1. The big picture

```
Original:    [ HTML 280 KB ]
gzip -6:     [ HTML  82 KB ]   ratio 3.4×, fast both ways
br  -6:      [ HTML  68 KB ]   better ratio, slower compress
br  -11:     [ HTML  56 KB ]   best ratio, very slow compress (precompute it)
zstd -3:     [ HTML  76 KB ]   excellent ratio + speed both ways
zstd -19:    [ HTML  58 KB ]   slower compress, fast decompress
```

The wins come from one observation: text has redundancy. HTML repeats tags, JSON repeats keys, JS repeats identifiers, log files repeat IPs and timestamps. A general-purpose compressor finds those repetitions and encodes them shorter. The decompressor reverses the trick.

Where you compress:

- **HTTP responses** — by far the biggest win for most apps.
- **HTTP requests** — less common but supported (`Content-Encoding: gzip` on the request body).
- **Storage** — Parquet/ORC, columnar DBs, object storage, backups.
- **Inter-service RPC** — gRPC supports per-message compression; many internal APIs gzip JSON.
- **Logs and metrics** — almost always compressed at the agent and again at the store.
- **Caches** — Memcached/Redis blobs are sometimes compressed; cost varies.

---

## 2. The compressors that actually matter

| Algorithm | Year | Ratio | Compress speed | Decompress speed | Use case |
|---|---|---|---|---|---|
| **gzip** (DEFLATE) | 1992 | Decent | Medium | Fast | Universal compatibility. Default for everything. |
| **Deflate** (raw) | 1993 | Same as gzip | Same | Same | Inside zlib, PNG, WebP, zip files. |
| **Brotli** | 2015 (Google) | Best ratio for text | Slow at high levels | Fast | HTTP responses, esp. precompressed static. |
| **Zstandard (zstd)** | 2016 (Facebook) | Near-Brotli | Very fast | Very fast | Storage, logs, backups, gRPC, modern HTTP. |
| **LZ4** | 2011 | Lower ratio | Extremely fast | Extremely fast | RAM-bound caches, where CPU > ratio. |
| **Snappy** | 2011 (Google) | Lower ratio | Very fast | Very fast | Kafka, Cassandra, BigTable. Speed first. |
| **LZMA / xz** | 2001/2008 | Excellent ratio | Slow | Slow | Archive distribution (source tarballs). |
| **zstd dictionary** | — | Best for small payloads | Fast | Fast | Logs, messages, small JSON — trained dictionary. |

The headline practical comparison for HTTP compression:

```
Asset: Jquery 3.6 min.js  (90 KB)
  none      90.0 KB
  gzip -6   31.0 KB
  brotli 6  28.5 KB
  brotli 11 27.5 KB   (asymmetric: slow to compress, normal to decompress)
  zstd -3   29.2 KB
  zstd -19  27.8 KB
```

For larger HTML, the Brotli advantage over gzip is typically 15–25% smaller. For binary payloads the difference shrinks. For already-compressed bytes (PNG, JPEG, MP4), all general-purpose compressors break even or *expand* the file.

---

## 3. Where the bytes go on the modern web

The default HTTP setup most teams should have:

- **`Accept-Encoding`** in the request: `gzip, deflate, br, zstd` (the client lists what it can decompress).
- **Server** picks the best one the client supports, based on file type and configuration.
- **`Content-Encoding`** in the response: the chosen algorithm.
- **`Vary: Accept-Encoding`** so caches don't serve the wrong variant to the wrong client.

```http
> GET /app.js HTTP/1.1
> Accept-Encoding: gzip, deflate, br, zstd

< HTTP/1.1 200 OK
< Content-Encoding: br
< Content-Type: application/javascript
< Vary: Accept-Encoding
< Content-Length: 27583
```

Browser support today (2026):
- **gzip**: 100% of HTTP clients ever. Always safe.
- **Brotli (br)**: ~96% of browsers in use. Effectively universal on HTTPS.
- **Zstandard (zstd)**: ~85%+ of modern browsers (Chrome 123+, Firefox 126+, Safari 17.4+). Becoming routine on HTTPS.

`Vary: Accept-Encoding` is critical when intermediaries cache responses — without it, a gzip variant can be served to a client that asked for Brotli, breaking it.

---

## 4. Compression levels — the real lever

Every compressor has a level knob. Higher levels search harder, find more repetitions, and produce smaller output, but cost more CPU and time. **Decompression speed is much less sensitive to level than compression speed**.

A representative shape:

```
Level    1    3    6     9    11(br)/19(zstd)
ratio   1.0  1.3  1.5   1.6  1.65
compress time × 1   ×2   ×8   ×30   ×300
decompress time     ×1.0 ×1.0 ×1.0  ×1.05
```

The shape says everything:

- **Dynamic responses** — generated per request, compressed once, served once. Use low levels (gzip 1–4, brotli 4–6, zstd 3). High levels eat CPU for marginal byte savings.
- **Static / cacheable assets** — compressed once at build time, served forever. Use high levels (brotli 11, zstd 19+). Spend the CPU once; ship the smallest bytes for every future user.
- **Real-time / streaming** — every millisecond matters; low levels. Or skip compression for tiny payloads where the cost of the compressor's framing exceeds the savings.

The split is so common it's now built into nginx, Cloudflare, Fastly, Caddy, and Apache: separate `gzip` (dynamic) and `gzip_static` (precompressed file) directives, plus `brotli` / `brotli_static`.

---

## 5. The cost / benefit math

Compression is worth it when:

```
(bytes_saved / bandwidth) > (compress_time + decompress_time)
```

The breakeven shifts dramatically by environment:

- **Mobile networks** — bandwidth is precious, RTTs are high, last-mile is slow. Even expensive compression pays back.
- **In-datacenter (10 Gbps, 0.5 ms RTT)** — bandwidth is cheap and fast. Compression often *loses* on small messages; it's near-neutral on medium ones; it wins on big ones. Many gRPC services run uncompressed internally.
- **Cross-region (60–200ms RTT)** — bandwidth costs money and adds latency. Compression usually wins.
- **Storage at rest** — compression always wins eventually; you pay CPU once to write, decompress on read. Most modern columnar formats (Parquet, ORC) compress per column block by default.

**Rule of thumb for HTTP**: compress responses ≥ 1 KB; below that the framing overhead doesn't pay back.

---

## 6. The "don't compress already-compressed bytes" rule

Compressors lose, or do nothing useful, on:

- **JPEG, PNG, GIF, WebP, AVIF** — already lossy/entropy-coded.
- **MP3, AAC, OGG, FLAC** — audio codecs do the compression.
- **MP4, WebM, AV1, HEVC** — video codecs ditto.
- **ZIP, 7z, RAR, gz, br, zst** — already compressed.
- **Encrypted bytes (TLS payloads at rest, age-encrypted files)** — encryption removes redundancy by design.

Symptoms of trying anyway:
- Higher CPU for ~0% savings.
- Sometimes a small *increase* in size (compressor framing + non-redundant data).
- Wasted CPU on every read in storage tiers that auto-compress.

This is why HTTP servers exclude image types from their `gzip_types` / `brotli_types` lists. And why object stores like S3 don't auto-compress most data — by default they assume you're storing whatever the format dictates.

There's one important exception: **fonts**. WOFF and WOFF2 are themselves compressed containers (WOFF uses zlib; WOFF2 uses Brotli). Don't double-compress.

---

## 7. Compression and security — the CRIME/BREACH gotcha

Compression interacts badly with secrets when both attacker-controlled and secret data are compressed together. The CRIME (2012) and BREACH (2013) attacks exploit this in TLS/HTTP compression: the attacker observes compressed sizes after injecting guesses and infers parts of the secret.

Mitigations:

- **TLS compression is disabled** in TLS 1.3 and effectively all of TLS 1.2 — CRIME is dead.
- **HTTP-level compression** can still leak if the response mixes user-controlled input with secrets (CSRF tokens, session IDs). Standard defenses:
  - Don't reflect attacker-controlled data into responses that contain secrets.
  - Use per-request CSRF tokens that randomize per response.
  - Length-mask responses if necessary.
  - Modern frameworks generally do this correctly; check before assuming.

This is a real-but-narrow concern. For most apps, enable HTTP compression and follow framework defaults for CSRF and session security.

---

## 8. Dictionary compression — the underused superpower

A trained **dictionary** primes the compressor with a sample of typical data. For small, repetitive payloads (log lines, JSON messages, structured events), a dictionary can deliver 2–5× better compression than raw zstd at the same level.

```bash
# zstd dictionary training
zstd --train logs/*.json -o app.dict
zstd -D app.dict input.json -o input.zst
```

Where this shines:

- **Per-message Kafka payloads** — many tiny JSON messages, all similar shape. Dictionary makes each tiny.
- **Distributed logs** — Loki, ClickHouse, columnar log stores.
- **Caches of small entries** — Redis blobs of similar shape.
- **Shared HTTP dictionaries** — recent Chrome/Cloudflare feature: serve a `Use-As-Dictionary` resource so subsequent compressed responses reference it. Massive savings for SPAs that ship many similar JS chunks.

Brotli was designed with a built-in 120 KB dictionary of common HTML/CSS/JS strings. That's part of why it beats gzip on web text: it's pre-trained on the web.

---

## 9. Streaming vs frame-based

Some compressors are stream-oriented (each compressor instance encodes a contiguous stream). Others are frame-based (each chunk is independently decodable).

- **gzip / DEFLATE** — stream by default.
- **Brotli** — stream by default; supports `BROTLI_OPERATION_FINISH` boundaries.
- **Zstandard** — both. Frame-based by default; supports streaming mode.
- **LZ4** — frame-based variant common in Kafka/Cassandra.

Why this matters:

- **Streaming** is ideal when you want to start sending bytes immediately (HTTP responses, real-time pipes). Latency to first byte is preserved.
- **Frame-based** is ideal for storage and parallel decoding (split a big log into chunks, decompress in parallel).
- **Random access** in storage requires frame boundaries — Parquet's column chunks are independently decompressible for this reason.

For HTTP responses, all three (gzip, br, zstd) stream fine. For storage, prefer frame-based with sensible block sizes (Parquet defaults to ~1 MB per column chunk).

---

## 10. Where the wall-clock wins really come from

A common surprise: compressing a 280 KB HTML response down to 70 KB on a desktop with Gigabit ethernet seems to save almost no time. The brand-new 4G phone user across town just experienced a 200ms speedup. The same response over a flaky mobile network in São Paulo experienced a 1 second improvement.

Three reasons compression matters more than raw bandwidth math suggests:

- **TCP slow start.** A response that fits in fewer round trips ships faster because TCP windows haven't ramped yet. The first 14 KB ships in the same RTT; everything beyond that depends on RTT and packet loss.
- **Packet loss compounds.** Less data = fewer packets = less chance of a retransmission stalling the stream.
- **Bandwidth is shared.** Even when one client has good bandwidth, the path competes with everyone else's traffic.

For mobile and emerging markets, compression isn't an optimization — it's a usability requirement. The same goes for users behind constrained backhaul (airplanes, ships, rural lines).

---

## 11. Compression in storage and analytics

Modern columnar formats are compression-first:

- **Parquet** — per-column dictionary + Snappy/Zstd/Brotli. Typical 4–10× compression ratios over CSV.
- **ORC** — similar approach, slightly different layout.
- **ClickHouse** — LZ4 by default, zstd available. Per-column codecs (delta, gorilla for floats, etc.).
- **TimescaleDB, InfluxDB** — Gorilla and similar specialized codecs for time-series.
- **HBase / Cassandra** — block-level Snappy or zstd by default.

The lesson: at storage scale, compression buys you both disk space and **read throughput** (smaller blocks → fewer IOPS for the same data). The CPU cost is amortized across many reads.

For object storage like S3, you pay per GB stored *and* per GB transferred. Compression cuts both bills.

---

## 12. Common Mistakes / Anti-Patterns

- **Compressing already-compressed bytes.** Wasted CPU, no savings, sometimes negative.
- **Compressing tiny responses.** Framing overhead exceeds the savings. Set a minimum size (1 KB) in your server config.
- **High compression level on dynamic responses.** gzip -9 on every HTTP response burns CPU for ~3% byte savings vs -4.
- **No `Vary: Accept-Encoding`.** Caches serve gzip to a Brotli-asking client; mysterious breakage.
- **Precompressed static assets generated at request time.** Build-time `brotli -11` once, serve forever. Don't re-compress per request.
- **Forgetting fonts and SVG.** SVG is text — gzip/Brotli love it. WOFF2 is already compressed — don't double-compress.
- **In-DC gRPC with gzip on every call, no measurement.** Often a perf *loss*. Measure or skip.
- **CRIME/BREACH-relevant secrets in compressed responses with reflected user input.** Standard frameworks usually fix this; verify for hand-rolled code.
- **No dictionary for tiny repetitive payloads.** A 200-byte JSON event compresses ~50% standalone, ~85% with a 10 KB trained dictionary.
- **Compression configured but never measured.** No metric showing compression ratio or bandwidth savings; nobody knows it's broken.
- **Mixing encodings via misconfigured proxies.** A CDN strips `br` because the origin sends `Content-Encoding: gzip` and the CDN re-encodes incorrectly.
- **Decompression bomb risk on user uploads.** A 100 KB gzip file expands to 4 GB. Cap decompressed size when accepting uncontrolled compressed inputs.

---

## 13. Cheat Card

```
PURPOSE   Trade CPU for bytes. Faster bytes hit the user faster,
          especially on mobile / cross-region / shared networks.

PICK ONE
  gzip       universal, fast, the default
  Brotli     best ratio for text; great as static -11
  zstd       near-Brotli ratio, fastest decompress
  LZ4/Snappy speed first, for caches and brokers
  xz/LZMA    archive distribution only

RULES OF THUMB
  Text  →  always compress (HTML, JS, CSS, JSON, SVG, fonts*)
  Already-compressed (JPEG, MP4, PNG, ZIP, WOFF2) → don't
  Tiny  (<1 KB)  →  skip (framing overhead)
  Static  →  precompute at high level (brotli 11, zstd 19)
  Dynamic →  low level (gzip 1–4, br 4–6, zstd 3)
  Internal RPC small msgs → measure before enabling

HTTP CONFIG
  Accept-Encoding: gzip, br, zstd
  Content-Encoding: <chosen>
  Vary: Accept-Encoding         (always)
  Minimum length 1 KB
  Exclude image/video/font MIME types

STORAGE
  Parquet/ORC + zstd            warehouse default
  ClickHouse LZ4 / zstd
  Kafka producer compression (snappy/zstd/lz4)

DICTIONARIES
  Train on representative samples (zstd --train, brotli --dict)
  Best for many tiny similar payloads
  Shared HTTP dictionaries: Chrome/Cloudflare 2024+

WHERE WINS COME FROM
  Fewer packets → fewer RTTs in TCP slow start
  Mobile / emerging-market bandwidth ceilings
  Cross-region egress $ savings
  Smaller storage blocks → fewer IOPS

PITFALLS
  Compressing already-compressed bytes
  Compressing tiny responses
  Max level on hot dynamic responses
  No Vary header → cache poisoning
  Per-request brotli -11 on static assets
  CRIME/BREACH leaks (reflected secrets)
  Decompression bombs on user uploads

RULE   Compress text aggressively, leave binary alone, choose
       level by content lifetime, measure ratio and CPU together.
```

---

## 14. Resources

### Documentation
- **gzip / DEFLATE (RFC 1951, 1952)** — <https://www.ietf.org/rfc/rfc1951.txt>
- **Brotli (RFC 7932)** — <https://datatracker.ietf.org/doc/html/rfc7932>
- **Zstandard (RFC 8478, 8878)** — <https://datatracker.ietf.org/doc/html/rfc8478>
- **MDN — Content negotiation** — <https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation>
- **MDN — `Content-Encoding`** — <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding>

### Articles
- "Smaller and faster: brotli compression" — Google Developers.
- "Zstandard: real-time compression" — Yann Collet (creator).
- "Shared dictionary compression for HTTP" — Patrick Meenan / Chrome.
- "The fastest, deadliest gzip" — Cloudflare engineering.
- "On compressibility" — academic intros to Lempel–Ziv variants.

### Videos
- *Zstandard at Facebook* — engineering talks from Yann Collet.
- *Brotli for the web* — Jyrki Alakuijala (Google).
- ByteByteGo — "HTTP Compression Explained."

### Tools
- **`gzip` / `pigz`** (parallel gzip), **`brotli`**, **`zstd`** — CLI tools.
- **`hyperfine`** — benchmarking compression speed.
- **`mod_brotli`**, **`nginx_brotli`**, **`brotli` module in Cloudflare Workers / Vercel / Netlify**.
- **`zopfli`** — even better gzip-compatible compression (slow, for static).
- **`squash`** — compression benchmark suite.

### Adjacent reading
- [Serialization Formats →](./serialization.md)
- [Profiling & Benchmarking →](./profiling.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3 →](../02-networking/http-versions.md)
- [CDN →](../05-caching/cdn.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)

---

*Previous:* [← Connection Pooling & Keep-Alive](./connection-pooling.md)  |  *Next:* [Serialization Formats →](./serialization.md)
