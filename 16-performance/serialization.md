# Serialization Formats (JSON, Protobuf, Avro, MessagePack)

> **TL;DR** — Serialization is converting in-memory objects to bytes (and back). The format you pick determines **payload size**, **CPU cost**, **language portability**, **schema evolution rules**, and **debuggability**. **JSON** is the default for external APIs because every language and every human reads it. **Protobuf** is the default for internal RPC because it's compact, fast, and schema-typed. **Avro** dominates the data-lake / Kafka world because schemas live next to the data. **MessagePack** is JSON-shaped binary. **FlatBuffers / Cap'n Proto** are "zero-copy" formats for extreme-low-overhead reads. **CBOR** is a JSON-aligned binary standard you'll see in IoT and WebAuthn. The honest take: there is no "fastest" format in the abstract — only "fastest for your workload, with your tooling, your schema, and your evolution discipline." The single most expensive serialization mistake teams make isn't picking the wrong format; it's **changing schemas without a compatibility strategy**. Get that right and most formats are fine.

---

## 1. The big picture

Every distributed system is plumbed by serialization. Three axes shape the choice:

```
   compactness ──────────────►
   speed       ──────────────►   protobuf / capnp / flatbuf
                                 avro
                                 msgpack / cbor
                                 json / xml
   ◄──────────  human-readable
   ◄──────────  language ubiquity
   ◄──────────  schemaless flexibility
```

The right format depends on where you sit on those axes for the specific channel:

- **Public API for unknown clients** → JSON, period.
- **Internal RPC where both sides ship together** → Protobuf or similar.
- **Streaming data on Kafka with consumers in many languages and long lifetimes** → Avro with a schema registry.
- **Telemetry from a microcontroller** → CBOR or MessagePack.
- **A 10 GB game asset loaded at startup** → FlatBuffers (zero-copy read).
- **A configuration file a human edits** → YAML or TOML (not serialization in the wire sense, but in the same family).

Two formats often coexist: JSON at the edge for clients, Protobuf or Avro between services and into the warehouse.

---

## 2. The formats that matter

| Format | Binary? | Schema | Self-describing | Strengths | Weaknesses |
|---|---|---|---|---|---|
| **JSON** | No | Optional (JSON Schema) | Yes | Universal, debuggable, easy | Verbose, slow, no native types (no int64, no bytes, no date) |
| **YAML** | No | Optional | Yes | Human config | Spec is large, edge cases bite, slow |
| **XML** | No | XSD | Yes | Historical, namespaced | Verbose, parser complexity, mostly legacy |
| **MessagePack** | Yes | None | Yes | JSON-shaped, smaller, faster | Still schemaless; weaker tooling than Protobuf |
| **CBOR** | Yes | Optional (CDDL) | Yes | IETF standard, JSON-aligned | Niche but growing (WebAuthn, COSE) |
| **BSON** | Yes | None | Yes | MongoDB's format | Larger than MessagePack; not really general-purpose |
| **Protobuf** | Yes | **Required (.proto)** | No | Compact, fast, mature, polyglot | Schema must be shared; less debuggable |
| **gRPC** | Uses Protobuf | — | — | Streaming, HTTP/2 wire | Same caveats as Protobuf |
| **Avro** | Yes | **Required (in writer header or registry)** | Yes (when schema is bundled) | Schema evolution rules, registry pattern | More complex tooling |
| **Thrift** | Yes | Required | No | Like Protobuf, with RPC built-in | Lost market share to Protobuf/gRPC |
| **FlatBuffers** | Yes | Required | No | Zero-copy access; mmap-friendly | Awkward API; mutate-by-rebuild |
| **Cap'n Proto** | Yes | Required | No | Zero-copy + RPC | Smaller community |
| **SBE (Simple Binary Encoding)** | Yes | Required | No | Ultra-low latency for trading | Niche |

The four mentioned in the title are the four you'll encounter most often. The rest you should at least recognize.

---

## 3. JSON — why it won the external API

JSON is verbose and slow and you should still use it for most public APIs.

Why:
- **Every language has a parser.** Stable, fast (jsoniter, simdjson, RapidJSON, orjson, fast-json-stringify can hit ~1 GB/s).
- **Humans read it.** Debugging in browsers, terminals, log lines — JSON wins.
- **Tooling everywhere** — curl, jq, Postman, browser DevTools.
- **Schemaless tolerance** — you can add a field and old clients ignore it.

Why it costs:
- **No native types.** Integers larger than 2^53 lose precision in JavaScript. No bytes type (you base64). No date type. No decimal. No null vs missing.
- **No schema by default.** Validation lives in your code, or in JSON Schema, or in a runtime contract.
- **Big.** 3–10× larger than Protobuf for equivalent data. Compresses well but you still parse the bytes.
- **Slow.** Standard `json.dumps`/`JSON.stringify` are 5–20× slower than Protobuf for the same data. Specialty parsers narrow but don't eliminate the gap.

Production tips for JSON-heavy services:
- **Pick a fast parser** — orjson (Python), simdjson (C++/bindings), fast-json-stringify (Node), jsoniter (Java/Go), serde_json (Rust).
- **Validate at the boundary.** JSON Schema, Zod, Pydantic, Ajv, GoValidator.
- **Decide on `null` vs absent** for every field. Be consistent.
- **Avoid 64-bit integers as numbers** when JS clients are involved. Send as string.
- **Don't ship JSON over the wire uncompressed.** gzip/br/zstd is mandatory. See [Compression →](./compression.md).

---

## 4. Protobuf — the internal RPC default

Protocol Buffers (Google, 2008) define a `.proto` schema and generate code for serialization in 30+ languages.

```proto
syntax = "proto3";

message Order {
  uint64 id = 1;
  string customer_id = 2;
  uint32 amount_cents = 3;
  repeated string items = 4;
  Status status = 5;
}

enum Status {
  PENDING = 0;
  PAID = 1;
  REFUNDED = 2;
}
```

Properties:
- **Compact** — varint-encoded integers, field tags, no field names on the wire. Typically 3–10× smaller than JSON.
- **Fast** — generated code, no reflection in the hot path. Encoders push ~1–5 GB/s on commodity hardware.
- **Strongly typed** — types known at compile time on both sides.
- **Polyglot** — first-class support in Go, Java, C++, Python, Rust, Ruby, JS/TS, C#, Swift, Kotlin, Dart.
- **Evolution rules are simple and unforgiving** (see §7).

The killer combo: **Protobuf + gRPC**. gRPC is an RPC framework that runs over HTTP/2 with Protobuf messages, multiplexed streams, deadlines, metadata, and built-in load balancing primitives. The default for inter-service comms at most modern infra-heavy companies. See [gRPC, Protocol Buffers, Thrift →](../02-networking/grpc-protobuf.md).

When *not* to use Protobuf:
- Public APIs (clients can't always generate stubs).
- Quick prototyping (overhead of `.proto` + codegen).
- Heavy text fields where wire size barely differs from gzipped JSON.

---

## 5. Avro — Kafka's lingua franca

Apache Avro was built for the data-pipeline world. Its defining trait: **the writer's schema is embedded in the file** (Avro Object Container Files), or pinned by ID in a **Schema Registry** for streaming.

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.acme.orders",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "customer_id", "type": "string"},
    {"name": "amount_cents", "type": "int"},
    {"name": "items", "type": {"type": "array", "items": "string"}},
    {"name": "status", "type": {"type": "enum", "name": "Status", "symbols": ["PENDING", "PAID", "REFUNDED"]}}
  ]
}
```

Why Avro dominates Kafka and data lakes:
- **Compact** — comparable to Protobuf in size.
- **Schema evolution baked in** — readers and writers can have different schemas; the framework reconciles fields by name + type rules.
- **Schema Registry** (Confluent / Apicurio) tracks versions and enforces compatibility (`BACKWARD`, `FORWARD`, `FULL`).
- **Block-based** — efficient for batch reads from object storage.

When you need many independent producers and consumers, evolving over years, with strict compatibility tracking — Avro plus a registry is the canonical pattern. Protobuf can do this too (with Buf Schema Registry or similar), but Avro got there first and dominates the Hadoop / Kafka ecosystem.

Avro's downside: more moving parts (registry, schema files, codegen optional but common), and tooling outside the JVM is thinner than Protobuf's.

---

## 6. MessagePack, CBOR, BSON — JSON-shaped binary

**MessagePack** ("msgpack") is JSON in fewer bytes. Same data model (maps, arrays, strings, numbers, bools, null) — just binary-encoded for size and speed. Great default when you want JSON's flexibility but want it smaller and faster.

```python
import msgpack
data = {"id": 12345, "name": "Alice", "active": True}
packed = msgpack.packb(data)   # smaller than json.dumps(data)
```

Typical wins over JSON: **30–50% smaller**, 2–5× faster encode/decode. Used by Redis (`MSGPACK` integration in some clients), some game engines, Pinterest's earlier infra, microservices that want minimal change from a JSON design.

**CBOR** (RFC 8949) is essentially an IETF-standardized MessagePack with extras (typed arrays, tags, deterministic encoding). Less common as a general format; the killer use case is **WebAuthn / FIDO2** (passkeys use CBOR), **COSE** (signed CBOR objects), and IoT systems that need a JSON-like model in a binary protocol.

**BSON** is MongoDB's storage format. You'll see it in MongoDB drivers and dumps. It's less compact than MessagePack (BSON includes type tags and field lengths inline). Not generally useful outside MongoDB's ecosystem.

The rule of thumb: if you'd reach for JSON but want it smaller/faster and both sides are under your control, MessagePack is a clean upgrade. If you want a typed schema and codegen, jump to Protobuf or Avro instead.

---

## 7. Schema evolution — the part that matters most

Every wire format has rules for how schemas can change without breaking deployed services. Get this wrong and you cause outages.

### Protobuf evolution rules

- **Field tags are forever.** Tag `1` is `string customer_id`; you cannot reuse tag `1` later for `uint64 amount`. Reserve removed tags: `reserved 1, 2; reserved "customer_id";`.
- **Adding new optional fields** — safe. Old clients ignore them; new clients see defaults if missing.
- **Removing fields** — safe if you reserve the tag and name.
- **Changing field types** — mostly unsafe. `int32` ↔ `int64` is safe within bounds; `string` ↔ `bytes` is safe; otherwise dangerous.
- **Changing field number** — never safe.
- **Adding required fields (proto2)** — you broke clients. In proto3, `required` doesn't exist; that's by design.
- **Enums**: adding values is safe. Old clients see "unknown" but don't crash.

### Avro evolution rules

Avro names compatibility modes explicitly:

- **Backward** — new schema can read old data. Safe to add fields with defaults, remove fields with defaults.
- **Forward** — old schema can read new data. Safe to add fields with defaults, remove non-default fields.
- **Full** — both directions.
- **None** — anything goes (don't).

Avro reconciles by **field name** + type rules. Tag numbers don't exist. Aliases bridge renames.

### JSON evolution rules

JSON has no enforced schema, so evolution is whatever your team agrees on. **Common discipline**:

- Adding fields is safe (clients ignore unknown).
- Removing fields breaks clients that read them.
- Changing types breaks clients.
- Renaming fields is removing + adding — breaks both.
- Optional fields should be **omitted** when absent, not sent as `null`, unless you've defined `null` to mean something specific. Pick one rule per API.

### The universal lesson

**Pick a compatibility policy and enforce it in CI.** Tools:

- **Buf** for Protobuf — `buf breaking` checks against a baseline. Fails the PR if you broke wire compatibility.
- **Confluent Schema Registry** for Avro — refuses to register an incompatible schema.
- **OpenAPI / JSON Schema diff** — `oasdiff`, `openapi-diff` for REST APIs.

Without CI enforcement, evolution rules are guidelines that nobody follows under deadline pressure.

---

## 8. Real performance numbers — order of magnitude

For a representative object (~10 fields, mix of ints, strings, arrays):

| Format | Size | Encode | Decode |
|---|---|---|---|
| JSON (stdlib) | 1.0× | 1.0× | 1.0× |
| JSON (fast parser, e.g. orjson) | 1.0× | 3–5× faster | 3–5× faster |
| MessagePack | 0.5–0.7× | 2–4× faster | 2–4× faster |
| CBOR | 0.5–0.7× | similar to msgpack | similar |
| Protobuf | 0.3–0.5× | 5–10× faster | 5–10× faster |
| Avro | 0.3–0.5× | similar to Protobuf | similar |
| FlatBuffers | 0.3–0.5× | 1–3× faster encode | **~free decode** (mmap, no copy) |
| Cap'n Proto | 0.3–0.5× | similar | **~free decode** |

These vary hugely by message shape, library, and language. Benchmark your own data with your own tools before committing.

The "free decode" of FlatBuffers and Cap'n Proto deserves a sentence: the wire format *is* the in-memory format. You mmap the file and dereference into it. Zero parse time. Useful for game assets, embedded systems, very-high-throughput pipelines. Awkward API (everything is offset accessors); mutating means rebuilding.

---

## 9. Cross-cutting concerns

### Versioning at the API level

Format compatibility is one layer; **API versioning** is another. You can have a backward-compatible Protobuf change that is still a breaking change to your API contract (e.g., a field whose semantics changed). Compatibility is a wire-format concept; product behavior is your responsibility. See [API Versioning Strategies →](../03-apis/versioning.md).

### Time, money, big numbers

- **Time**: ISO-8601 strings in JSON, epoch-millis in Protobuf (well, `google.protobuf.Timestamp`), explicit logical types in Avro.
- **Money**: never doubles. String decimals, or smallest unit as int (cents, mills). Protobuf supports `google.type.Decimal` (community) or just `int64` cents.
- **Big integers**: JSON loses precision past 2^53 in JavaScript. Send as strings, or use `BigInt` on the JS side.

### Polymorphism / tagged unions

Modeling "Order is either ShippedOrder or RefundedOrder":

- **JSON**: discriminator field convention (`"type": "shipped"`).
- **Protobuf**: `oneof` field.
- **Avro**: union types `["null", "ShippedOrder", "RefundedOrder"]`.

Discriminate explicitly; never assume.

### Bytes / binary blobs

- **JSON**: base64 (33% size overhead).
- **Protobuf / Avro / msgpack / CBOR**: native `bytes` type.

For big blobs, don't embed at all — store in object storage, send a reference.

### Compression interplay

Most binary formats compress *less well* than JSON because they're already entropy-dense. JSON+gzip vs Protobuf raw is often surprisingly close in wire size, especially for verbose data. Protobuf+zstd usually wins on size *and* CPU for large messages. See [Compression →](./compression.md).

---

## 10. Picking the format — a practical decision tree

```
Who consumes it?
├── Browsers, unknown 3rd-party clients, public docs
│       → JSON  (with OpenAPI / JSON Schema)
│
├── Your own services, schemas shipped together
│       → Protobuf + gRPC
│
├── Kafka topic with many producers/consumers, long retention
│       → Avro + Schema Registry
│
├── Tiny embedded device
│       → CBOR or MessagePack
│
└── Big assets read at startup with no need to mutate
        → FlatBuffers or Cap'n Proto
```

For most teams, the answer in practice is: **JSON at the edge, Protobuf between services, Avro into the warehouse**. That's a fine default and most successful infra is shaped that way.

---

## 11. Common Mistakes / Anti-Patterns

- **No schema for "JSON" APIs.** Every consumer reverse-engineers the contract from examples. Use JSON Schema, OpenAPI, or Zod/Pydantic — somewhere.
- **Reusing Protobuf field tags after deletion.** Silent corruption when an old consumer reads new data.
- **Changing the meaning of a field, not the field itself.** "It still parses!" Yes — and your data is now wrong.
- **Sending 64-bit integers to JavaScript clients as numbers.** Loss of precision past 2^53.
- **Doubles for money.** Floating-point errors in financial systems. Use string decimals or int cents.
- **Embedding large binary blobs in JSON via base64.** 33% overhead, plus parsing pain. Use object storage + reference.
- **Sharing Protobuf files via copy-paste between repos.** They drift. Use a schema repo or a registry.
- **No CI check for breaking changes.** "It compiles" isn't compatibility.
- **Choosing the smallest format for everything.** Saving 200 bytes on a 1MB response is irrelevant. CPU and tooling cost is not.
- **Pretty-printing JSON in production logs.** 30% more bytes, no readability win (machine-parsed anyway).
- **Choosing Avro for a low-volume internal API.** All the operational complexity, none of the benefit.
- **Choosing FlatBuffers because "zero-copy is fast."** Awkward API; usually unnecessary.
- **Forgetting the `Vary` and `Content-Type`** when one endpoint serves both JSON and Protobuf.
- **JSON parsing in the hot path with stdlib parsers.** A fast parser is often a 5× CPU win.
- **No way to tell which version of a schema produced a message.** Bury a schema ID in the wire format or in a header.
- **Mismatched timezones in serialized timestamps.** Always UTC; document it.
- **Letting consumers mutate a serialized message in place expecting it to be sent back unchanged.** Many formats don't round-trip unknown fields by default.

---

## 12. Cheat Card

```
PURPOSE   Convert in-memory data ↔ bytes. Choice shapes size,
          speed, evolution, tooling, debuggability.

DEFAULTS (BORING & RIGHT)
  Public API           JSON  (+ OpenAPI / JSON Schema)
  Internal RPC         Protobuf + gRPC
  Kafka / data lake    Avro + Schema Registry
  IoT / embedded       CBOR or MessagePack
  Big read-only asset  FlatBuffers / Cap'n Proto

SIZE / SPEED (vs JSON baseline)
  msgpack / CBOR       0.5–0.7×, 2–4× faster
  Protobuf / Avro      0.3–0.5×, 5–10× faster
  FlatBuffers/Cap'n    0.3–0.5×, near-zero decode (mmap)

SCHEMA EVOLUTION
  Protobuf  → field tags are forever; reserve removed tags
              add optional fields freely
              never reuse a tag for a different type
  Avro      → name + type matching; default values bridge
              registry enforces BACKWARD / FORWARD / FULL
  JSON      → no enforcement; pick rules; check OpenAPI diff

TYPE GOTCHAS
  64-bit ints in JS → send as string
  Money              → string decimal or int cents
  Time               → UTC, ISO-8601 or Timestamp type
  Bytes              → native type or base64 (33% bloat)
  Null vs absent     → pick one and be consistent

OPERATIONAL
  Validate at the boundary (Zod, Pydantic, Ajv)
  Use a fast JSON parser (orjson, simdjson, jsoniter)
  CI: buf breaking / oasdiff / schema registry compat
  Compress on the wire (br / zstd / gzip)
  Version in the URL / Accept header / wire envelope

PITFALLS
  Reusing protobuf tags
  Doubles for money
  No CI compat checks
  Embedding huge blobs in JSON via base64
  Pretty-printing JSON in prod logs
  Stdlib JSON parser in the hot path
  Avro for a low-volume internal API
  FlatBuffers because "zero-copy is cool"

RULE   JSON for humans and the public, Protobuf between your
       services, Avro for the data lake. Lock down schema
       evolution with CI. Pick fast parsers when CPU bites.
```

---

## 13. Resources

### Documentation
- **JSON (RFC 8259)** — <https://www.rfc-editor.org/rfc/rfc8259>
- **Protocol Buffers** — <https://protobuf.dev/>
- **Apache Avro** — <https://avro.apache.org/docs/>
- **MessagePack** — <https://msgpack.org>
- **CBOR (RFC 8949)** — <https://www.rfc-editor.org/rfc/rfc8949>
- **gRPC** — <https://grpc.io/docs/>
- **Confluent Schema Registry** — <https://docs.confluent.io/platform/current/schema-registry/>
- **Buf** (Protobuf schema tooling) — <https://buf.build/>
- **FlatBuffers** — <https://flatbuffers.dev/>
- **Cap'n Proto** — <https://capnproto.org/>

### Books and chapters
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 4 ("Encoding and Evolution") is the canonical reference.
- *gRPC: Up and Running* — Indrasiri & Kuruppu.
- *Protocol Buffers Reference Guide* — Google's online manual.

### Articles
- "Schema Evolution in Avro, Protocol Buffers and Thrift" — Martin Kleppmann (blog): <https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html>
- "Beating up on JSON" — Stripe / Cloudflare engineering pieces on faster parsers.
- "simdjson: parsing gigabytes of JSON per second" — Lemire et al.
- "How Discord stores billions of messages" — schema-related discussion.

### Videos
- *Protocol Buffers Deep Dive* — Kenton Varda (original author of Cap'n Proto / earlier Protobuf maintainer).
- *Kafka + Avro best practices* — Confluent talks.
- ByteByteGo — "Serialization Formats Compared."

### Tools
- **`protoc`**, **`buf`** — Protobuf tooling.
- **Apache Avro tools**, **avro-cli**.
- **orjson**, **simdjson**, **RapidJSON**, **jsoniter**, **fast-json-stringify**.
- **`jq`**, **`yq`**, **`xq`** — CLI for JSON/YAML/XML.
- **Confluent Schema Registry**, **Apicurio Registry**.
- **OpenAPI Generator**, **Pact** for contract testing.

### Adjacent reading
- [Compression →](./compression.md)
- [Profiling & Benchmarking →](./profiling.md)
- [gRPC, Protocol Buffers, Thrift →](../02-networking/grpc-protobuf.md)
- [REST vs GraphQL vs gRPC →](../02-networking/api-styles.md)
- [API Versioning Strategies →](../03-apis/versioning.md)
- [Kafka Deep Dive →](../07-messaging/kafka.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)

---

*Previous:* [← Compression](./compression.md)  |  *Next:* [N+1 Query Problem →](./n-plus-one.md)
