# Peer-to-Peer Systems & DHTs

> **TL;DR** — A **peer-to-peer (P2P)** system has no central server: peers connect to each other directly and share resources (data, compute, bandwidth). The defining challenge is **how does a peer find which other peer has the data it wants?** The dominant answer is a **Distributed Hash Table (DHT)** — a logical key-value store spread across all peers, where each key maps deterministically to a peer (or small set of peers) responsible for it. Lookup takes **O(log N) hops** thanks to clever routing (**Kademlia**, **Chord**, **Pastry**, **CAN**). DHTs underpin **BitTorrent's trackerless mode**, **IPFS**, **Ethereum's node discovery**, **Cassandra's gossip-based partition lookup (loosely)**, and countless overlay networks. The honest take: **most "P2P" success stories combine DHT lookup with some centralized coordination** (bootstrap nodes, trackers, signaling servers), because pure P2P struggles with NAT traversal, sybil attacks, and bootstrap. The interesting engineering ideas — **consistent hashing, hop-bounded routing, gossip, churn tolerance** — show up in distributed systems even where peer-to-peer isn't the architecture.

---

## 1. The big picture

A client-server system has one (or a few) servers known to everyone:

```
clients
  │ │ │
  ▼ ▼ ▼
[ Server ]
```

A peer-to-peer system has many peers, each both client and server:

```
   peer ─── peer
    │  X  X  │
    │ X    X │
   peer ─── peer
    │  X  X  │
   peer ─── peer
```

No single server. No single point of failure. No single point of cost. But also: **no obvious place to ask "where's the data?"**. Pure flooding (ask everyone) is O(N) per query and doesn't scale. The DHT solves this with structure.

---

## 2. Why P2P matters

Three reasons, historically:

- **Cost** — distribute bandwidth and storage across users. BitTorrent moves petabytes daily, none of it from "BitTorrent's servers."
- **Censorship resistance** — no central server to take down. Useful for human rights work, content distribution in restrictive regimes.
- **Resilience** — no single point of failure.

The reality is more mixed. Modern internet infrastructure (NAT, mobile networks, asymmetric bandwidth, ISP throttling) makes pure P2P harder than it was in 2003. Most production "P2P" systems are **hybrid**: peers do the work, but central coordinators help discovery, bootstrap, and signaling. Examples:

- **WebRTC** for video calls — P2P media, but signaling server arranges the handshake. See [WebRTC →](./webrtc.md).
- **BitTorrent** — DHT works, but most clients also use trackers.
- **IPFS** — DHT is core, but gateways and pinning services are common.
- **Tailscale / WireGuard mesh VPN** — direct peer connections with central coordination.
- **Bitcoin / Ethereum** — every node is a peer, but bootstrap nodes are well-known.

Pure P2P is rare in production. The interesting engineering is the **discovery and routing layer**, where DHTs shine.

---

## 3. The DHT contract

A **Distributed Hash Table** offers two operations:

```
put(key, value)        # store this value somewhere in the network
get(key) → value       # retrieve it
```

Like a normal hash table. The trick: the data lives on **whichever peer the key hashes to**, with replication to nearby peers for fault tolerance.

Three properties matter:

- **Deterministic routing** — given a key, you can compute which peer should hold it.
- **Logarithmic lookup** — `O(log N)` hops on a network of N peers.
- **Churn tolerance** — peers join and leave constantly; the system self-heals.

Famous DHTs:

- **Chord** (Stoica et al., 2001) — circular keyspace with `finger tables` for routing.
- **Pastry** (Rowstron & Druschel, 2001) — prefix-based routing with locality awareness.
- **Kademlia** (Maymounkov & Mazières, 2002) — XOR-based distance metric; symmetric; the dominant design today (BitTorrent, IPFS, Ethereum).
- **CAN — Content Addressable Network** (Ratnasamy et al., 2001) — d-dimensional coordinate space.
- **Tapestry** (Zhao et al., 2001) — similar to Pastry, used by some research systems.

Production DHTs in 2026 are almost all Kademlia derivatives.

---

## 4. Kademlia — the dominant DHT

Kademlia introduced two key ideas:

### XOR distance

Every node and every key has a 160-bit ID (SHA-1 hash). The "distance" between two IDs is their **XOR**:

```
distance(a, b) = a XOR b
```

Why XOR? It's:
- **Symmetric** — `d(a,b) = d(b,a)`. Routing tables built by one node are useful to others.
- **Triangle-inequality-respecting**.
- **Easy** — just XOR the IDs.

For a key K, the responsible node is the one whose ID is **closest to K under XOR distance**. Routing is "send the query to the peer in your contact list closest to K."

### k-buckets and finger tables

Each node maintains a contact list organized by distance. Specifically, **k-buckets** — buckets of (up to) k peers each, where bucket `i` holds peers whose distance from us is in the range `[2^i, 2^(i+1))`. Routing involves checking k-buckets at the right distance level, sending the query to the closest contacts, and recursing.

With this structure:
- Lookups complete in `O(log N)` hops.
- Joining the network is cheap: bootstrap from a known node, populate buckets via lookups.
- Failures self-heal — drop dead contacts, replace with live ones.

Kademlia's elegance is that routing, replication, and discovery all share the same XOR-distance machinery.

---

## 5. Lookup walkthrough

Suppose the network has 1M nodes and we want `get(key_K)`.

1. Hash the key: `K = SHA-1(key_K)`.
2. Find the α (typically 3) closest peers we know to `K` from our k-buckets.
3. Send `FIND_VALUE(K)` to each.
4. Each peer responds with either:
   - The value (if they hold it), or
   - Their own α closest peers to `K`.
5. Repeat with newly-discovered peers, always pursuing nodes closer to `K` than what we've already seen.
6. Stop when no closer peers are returned.

The walk is parallel (3+ simultaneous requests), iterative (we drive it, not blind forwarding), and bounded by the network's logarithmic depth.

`put(key, value)` works similarly: find the k closest peers to `K`, store the value on all of them. Replication factor `k` (~20 in BitTorrent's DHT) protects against churn.

---

## 6. Real DHT deployments

### BitTorrent

The classic. Originally relied on central **trackers** that knew which peers had which torrent. Faced legal pressure; designed the **Mainline DHT** (Kademlia-based) so peers could find each other without a tracker. Today the world's largest active DHT: tens of millions of nodes, billions of lookups.

A `.torrent` file or magnet link contains an info hash. The DHT maps info hash → peer list. Peers exchange pieces of the file directly.

### IPFS

The InterPlanetary File System uses **Kademlia** for both peer discovery and content routing. Every file is hashed (CID — Content Identifier); the DHT maps CID → list of nodes that have it.

In practice, IPFS uses a layered system: bitswap for actual transfer, libp2p for connection management, DHT for discovery. Pinning services (Pinata, Filebase, Filecoin pins) provide "always-on" availability that consumer nodes can't guarantee.

### Ethereum node discovery

Ethereum's **discv5** is a Kademlia-derived DHT for peer discovery in the Ethereum network. Nodes join, find peers near them in keyspace, build their peer set. Block data flows over libp2p; discovery happens over the DHT.

### Bitcoin

Bitcoin doesn't use a DHT. It uses **gossip + DNS seeds**. Bootstrap from DNS, then exchange `addr` messages with peers to learn about more peers. Simpler than a DHT, viable because Bitcoin doesn't need to look up arbitrary content — only to find any peers.

### libp2p

Protocol Labs' libp2p is the open-source toolkit for building P2P apps. Provides DHT, pub/sub, transport multiplexing, NAT traversal. Used by IPFS, Ethereum, Filecoin, Polkadot, many others.

### Cassandra (sort of)

Cassandra's data partitioning uses consistent hashing — conceptually a DHT structure. But Cassandra doesn't do P2P discovery; the gossip protocol distributes cluster membership across known nodes. Same primitives, different operational model.

---

## 7. Consistent hashing — the foundation

Before talking DHT topology, the underlying idea: **consistent hashing**. Hash both nodes and keys onto the same circular keyspace; each key belongs to the next node clockwise.

```
       0
       │
   ┌───┼───┐
   │   │   │   nodes at positions 30, 90, 150, 210, 280
   │       │   key K hashes to 100
   │       │   K belongs to the next node clockwise: 150
   └───┼───┘
       │
      180
```

Adding or removing a node only redistributes keys in a single arc — typically `1/N` of the total — rather than rebuilding the entire mapping. This is what makes elastic scaling possible.

Consistent hashing is used in:
- DHTs (Chord, Kademlia, Pastry — different ways to navigate the ring).
- Distributed caches (Memcached client-side hashing).
- Database partitioning (Cassandra, DynamoDB, Riak).
- Load balancers (request → backend).

See [Consistent Hashing →](../04-databases/consistent-hashing.md). The DHT is consistent hashing + a routing layer for "I don't know all the nodes."

---

## 8. The hard problems P2P keeps fighting

### NAT traversal

Most consumer machines sit behind NAT. They can make outgoing connections but can't easily receive incoming ones. Solutions:

- **STUN** — discover your public IP / port. See [WebRTC →](./webrtc.md).
- **TURN** — relay through a known server when STUN fails.
- **UDP hole punching** — coordinate simultaneous outgoing UDP packets so NATs let them through.
- **uPnP / NAT-PMP** — ask the router to forward a port.

None of these work 100%; production systems combine all of them and fall back to relays for the 5–20% of connections that fail direct.

### Sybil attacks

An attacker creates many fake nodes to disrupt the network. In a DHT, sybils near a target key can swallow lookups (eclipse attacks) or corrupt data. Defenses:

- **Proof of work** for joining (rare).
- **Crypto puzzles / staking** (Web3 networks).
- **Reputation** (peer scoring, slow-build).
- **Limited node IDs per IP** (heuristic, easy to evade with botnets).

No DHT is fully sybil-resistant in the open internet. Production systems accept the risk or build hybrid models with permissioned overlays.

### Churn

Peers join and leave constantly. The DHT must self-heal. Kademlia handles this well through lazy refresh and replication; some other DHTs (Chord) need explicit stabilization protocols.

### Bootstrap

A new node needs at least one existing node to talk to. **Bootstrap nodes** are well-known, long-lived peers maintained by the project (or its community). If they all go down, new joiners can't find the network.

### Censorship / blocking

ISPs can block specific ports / protocols / IPs. P2P traffic is famously throttled or deprioritized. Encrypted transports and protocol obfuscation help, but determined adversaries (state-level) can still cause headaches.

---

## 9. Gossip protocols — the P2P sibling

DHTs are about **structured** routing. **Gossip protocols** are about **unstructured** information dissemination. Each peer periodically picks a few random peers and exchanges state with them. Information spreads epidemically — eventually everyone knows what they should.

Used for:
- Membership / failure detection (SWIM, Serf, Cassandra).
- State distribution (HashiCorp Consul, etcd watch propagation).
- Anti-entropy in eventually-consistent databases (Cassandra, Riak, DynamoDB).
- Configuration broadcast.

Gossip is simpler than DHTs and tolerates churn beautifully. Trade-off: it's eventually consistent, not addressable — you can't say "tell me which peer has X." See [Gossip Protocol →](../08-distributed-systems/gossip-protocol.md).

Most P2P systems use both: DHTs for content addressing, gossip for membership and metadata.

---

## 10. Tracker-based vs trackerless

In BitTorrent's history:

- **Tracker-based** (original): a central server lists which peers have which file. Simple, fast, single point of failure / takedown.
- **Trackerless** (post-DHT): peers use the Mainline DHT for discovery. Resilient, slower for new content, no central liability.
- **PEX** (Peer Exchange): once you have one peer in a swarm, exchange peer lists with them.

Modern BitTorrent clients use all three simultaneously. Same pattern in IPFS, Ethereum, Polkadot — **redundant discovery layers**.

---

## 11. P2P beyond file sharing

A short tour of where P2P thinking actually wins in 2026:

### Real-time media (WebRTC)

Video calls between two parties: direct peer connection, low latency, no media-relay cost for the platform. Multi-party calls usually mix P2P with SFU/MCU centralization for efficiency. See [WebRTC →](./webrtc.md).

### CRDT-based collaborative editing

Tools like **Yjs**, **Automerge**, **Figma's libcrdt** allow many users to edit shared data with peer-to-peer sync. Often combined with a central server for persistence, but the data model is P2P-friendly. See [CRDTs →](../08-distributed-systems/crdts.md).

### Mesh VPN

**Tailscale**, **ZeroTier**, **Nebula**, **WireGuard mesh** — peers establish direct encrypted tunnels with central coordination for keys and policy. Massive bandwidth savings over hub-and-spoke VPN.

### Decentralized storage

**Filecoin**, **Storj**, **Arweave**, **Sia** — pay peers to store your data; cryptographic proofs verify they actually have it. Niche but real.

### Decentralized social

**Bluesky's AT Protocol**, **Mastodon (federated, not pure P2P)**, **Nostr** — different points on the centralization spectrum. Bluesky in particular uses DIDs and PDS-as-a-service rather than pure P2P, but the architectural inspiration is clear.

### Blockchain

Every blockchain is a P2P network at its base layer. See [Blockchain →](./blockchain.md). The DHT-style discovery primitives (devp2p, discv5) are direct descendants of this lineage.

### CDNs with P2P offload

**Peer5 (now Microsoft)**, **WebTorrent** — video / file delivery where viewers' browsers cache and serve to other viewers. Lowers CDN bills for popular content.

---

## 12. Common Mistakes / Anti-Patterns

- **Treating P2P as "free CDN."** Bandwidth, NAT, and churn all introduce real costs. Many P2P deployments end up paying for relays at the long tail.
- **Pure P2P with no bootstrap plan.** When the well-known seed nodes go down, new joiners can't find the network.
- **Ignoring NAT traversal.** 30–50% of connections won't be direct on the public internet without STUN/TURN.
- **No sybil defense at all.** Open DHTs can be eclipsed; data can be poisoned.
- **DHT for low-latency lookup.** It's `O(log N)` hops, each across the public internet. Don't expect millisecond responses.
- **Encrypting everything but not authenticating peers.** MITM is easy when keys aren't verified.
- **Assuming "decentralized" means "trustless."** Bootstrap nodes, validators, and devs hold real power even in P2P networks.
- **Building a custom DHT.** Use libp2p or an existing library. Custom routing tables have many subtle bugs.
- **Storing PII on a P2P network.** Once data is replicated to peers, you can't recall it. GDPR-incompatible.
- **Trusting peer-reported availability.** A peer claiming to have the data may lie. Verify hashes.
- **No churn tolerance.** Peers leave constantly; your replication factor and timeout settings need to absorb that.
- **Treating P2P as a replacement for HTTPS/CDN.** For most app delivery, a CDN is still the right answer.
- **Ignoring legal exposure.** Some P2P networks have hosted illegal content; your software shouldn't make it easy to participate involuntarily.

---

## 13. Cheat Card

```
PURPOSE   Coordinate among many peers with no central server,
          distributing data, compute, and bandwidth across the
          participants.

KEY PRIMITIVES
  Consistent hashing  same keyspace for nodes + keys
  DHT                 put(k, v) / get(k) → distributed routing
  Gossip              epidemic state propagation
  Bootstrap nodes     entry points for joiners

DHT FAMILIES
  Kademlia (Maymounkov & Mazières, 2002)  — dominant; BitTorrent, IPFS, Ethereum
  Chord (Stoica, 2001)                    — finger tables on a ring
  Pastry (Rowstron & Druschel, 2001)      — prefix routing + locality
  CAN, Tapestry                           — research / niche

KADEMLIA SPECIFICS
  XOR distance metric on 160-bit IDs
  k-buckets (k≈20) maintain peers by distance
  Lookup = iterative, parallel (α≈3), O(log N) hops
  put / get share routing machinery
  Replication factor k for churn tolerance

WHERE DHTs LIVE
  BitTorrent Mainline DHT
  IPFS (libp2p / kad-dht)
  Ethereum (discv5)
  libp2p (Polkadot, Filecoin, many others)

HARD PROBLEMS
  NAT traversal (STUN / TURN / hole punching)
  Sybil / eclipse attacks
  Churn (peers come and go)
  Bootstrap dependency
  Censorship / port blocking

P2P SUCCESS PATTERNS (often hybrid)
  Direct media (WebRTC) + signaling server
  Trackerless BT + tracker + PEX (redundant discovery)
  Mesh VPN + central key/policy plane (Tailscale)
  Blockchain L1 + RPC providers + DHT discovery

WHEN P2P WINS
  Bandwidth offload for popular content
  Censorship resistance
  No central authority desired
  Many-to-many media (calls, collaboration)
  Trustless coordination (blockchain L1)

WHEN P2P LOSES
  Low-latency interactive apps without central coordination
  Strong privacy of stored data (replicated forever)
  Easy onboarding (NAT, bootstrap, key mgmt are hard)
  Simple deployment / debugging

PITFALLS
  Pure P2P with no relay fallback
  Custom DHT implementation
  Ignoring sybil + eclipse risk
  PII on P2P storage
  Treating "decentralized" as "trustworthy"
  Bootstrap nodes neglected

RULE   DHTs solve "find which peer has X" in O(log N) hops.
       Real P2P deployments are almost always hybrid — peers do
       the work, central services help discovery, bootstrap,
       and NAT traversal.
```

---

## 14. Resources

### Papers
- "Kademlia: A Peer-to-Peer Information System Based on the XOR Metric" — Maymounkov & Mazières, 2002.
- "Chord: A Scalable Peer-to-Peer Lookup Service" — Stoica et al., 2001.
- "Pastry: Scalable, Decentralized Object Location and Routing" — Rowstron & Druschel, 2001.
- "A Scalable Content-Addressable Network (CAN)" — Ratnasamy et al., 2001.
- "BitTorrent: Mainline DHT" — community specifications.

### Books
- *Distributed Algorithms* — Nancy Lynch.
- *Peer-to-Peer: Harnessing the Power of Disruptive Technologies* — Andy Oram (historical context).
- *Designing Data-Intensive Applications* — Kleppmann. Indirectly relevant (consistent hashing, gossip).

### Documentation
- **libp2p** — <https://docs.libp2p.io>
- **IPFS** — <https://docs.ipfs.tech>
- **BitTorrent BEPs** — <https://www.bittorrent.org/beps/bep_0000.html>
- **Ethereum devp2p / discv5** — <https://github.com/ethereum/devp2p>
- **WebRTC** — <https://webrtc.org>

### Articles
- "Inside the Mainline DHT" — many engineering writeups.
- "How Kademlia works" — Brave engineering, Protocol Labs blog.
- "Tailscale: How NAT traversal works" — Tailscale engineering blog.
- "IPFS principles" — Protocol Labs documentation.

### Videos
- *Kademlia explained* — multiple academic + community talks.
- *Inside IPFS* — Juan Benet (creator) talks.
- *Tailscale technical deep dives* — Avery Pennarun talks.
- ByteByteGo — "P2P Networks Explained."

### Tools
- **libp2p** — modular P2P toolkit (Go, JS, Rust, others).
- **bittorrent-go**, **transmission**, **qBittorrent** — BT clients.
- **IPFS Kubo**, **Helia** — IPFS implementations.
- **Tailscale**, **WireGuard**, **ZeroTier**, **Nebula** — mesh VPN.
- **WebTorrent** — browser-based BT.
- **Yjs**, **Automerge** — CRDT libraries (P2P-friendly sync).

### Adjacent reading
- [Consistent Hashing →](../04-databases/consistent-hashing.md)
- [Gossip Protocol →](../08-distributed-systems/gossip-protocol.md)
- [Merkle Trees →](../08-distributed-systems/merkle-trees.md)
- [CRDTs — Conflict-free Replicated Data Types →](../08-distributed-systems/crdts.md)
- [Byzantine Fault Tolerance →](../08-distributed-systems/bft.md)
- [Blockchain & Distributed Ledger Basics →](./blockchain.md)
- [WebRTC for Real-Time Media →](./webrtc.md)
- [Edge Computing →](./edge-computing.md)
- [DNS — How It Works →](../02-networking/dns.md)

---

*Previous:* [← Blockchain & Distributed Ledger Basics](./blockchain.md)  |  *Next:* [WebRTC for Real-Time Media →](./webrtc.md)
