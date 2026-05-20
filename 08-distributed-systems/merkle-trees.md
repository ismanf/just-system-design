# Merkle Trees

> **TL;DR** — A **Merkle tree** (hash tree) is a binary tree where each leaf is a hash of a data block and each internal node is the hash of its children's hashes. A single **root hash** summarizes the entire dataset. To detect differences between two copies, **compare root hashes** — if equal, the data is identical; if different, recursively compare child hashes to find which blocks diverged. Used to **reconcile replicas** efficiently (Cassandra, Riak, Dynamo anti-entropy), **verify data integrity** (Git, IPFS, BitTorrent), and **prove inclusion** in blockchains (Bitcoin, Ethereum). The key win: with N blocks, finding which ones differ takes `O(log N)` hash comparisons, not O(N). Merkle trees are how distributed systems compare massive datasets without transferring them.

---

## 1. The Picture

```
Data blocks: D1, D2, D3, D4

leaves: hashes of data
   H(D1)   H(D2)   H(D3)   H(D4)
     \     /         \     /
   H(H1 || H2)     H(H3 || H4)
              \   /
        H( H(H1||H2) || H(H3||H4) )
                ↑
            root hash
```

Each leaf hashes a block. Each internal node hashes the concatenation of its children's hashes. The root depends on every block in the tree.

If any block changes, all hashes on the path to the root change. Comparing root hashes tells you whether **anything** differs.

If roots differ, descend: which child's hash differs? Then descend into that child, and so on. In `log N` steps you've found the exact differing leaves.

---

## 2. Why You Care

Imagine two replicas of a 10 TB dataset. After a partition heals, you want to find what's different. Options:

- **Send everything**: 10 TB over the wire. Death.
- **Compare keys one by one**: same scale.
- **Merkle tree comparison**: maybe a few MB of hashes; ship only differing blocks.

The win is **proportionate to similarity**. If both replicas are mostly the same (typical after partition), you ship the small diff, not the full set.

---

## 3. Construction

1. Split data into N blocks (rows, key-ranges, byte-chunks).
2. Hash each block → leaves.
3. Pair leaves; hash each pair → next level.
4. Recurse until one root.

If N isn't a power of 2, pad or carry odd leaves up (variants).

```python
def merkle_root(blocks):
    # leaves
    nodes = [hash(b) for b in blocks]
    while len(nodes) > 1:
        if len(nodes) % 2 == 1:
            nodes.append(nodes[-1])  # duplicate odd leaf (one convention)
        nodes = [hash(nodes[i] + nodes[i+1]) for i in range(0, len(nodes), 2)]
    return nodes[0]
```

Real implementations use **cryptographic hashes** (SHA-256 typically) so that:
- Birthday collisions are infeasible.
- An adversary can't forge a different dataset with the same root.

---

## 4. Comparing Two Trees

```
Replica A's root: 0xabc...
Replica B's root: 0xdef...
                 different → start descending

Replica A internal: left=0xa1, right=0xa2
Replica B internal: left=0xa1, right=0xb2
                    left matches → right differs

Replica A right child: leaves H(D3), H(D4)
Replica B right child: leaves H(D3), H(D4')   ← different
                                ↑ ship D4 from one to the other
```

Each comparison is a hash. For N blocks, worst-case log N levels. Network cost: O(K log N) for K differing blocks.

This is **fundamentally** how anti-entropy in distributed databases works.

---

## 5. Where Merkle Trees Are Used

### 5.1 Database anti-entropy
- **Cassandra**: build Merkle trees per token range; compare; ship differing blocks. Used by `nodetool repair`.
- **Riak**: AAE (active anti-entropy) uses Merkle trees.
- **Dynamo / DynamoDB**: internal anti-entropy.
- **BigTable / HBase**: per-region.

The pattern: scheduled background process builds a Merkle tree per shard/range; gossip with peers; reconcile.

### 5.2 Distributed file systems
- **IPFS**: every file is chunked; Merkle-DAG (DAG = directed acyclic graph, generalization). Content-addressable.
- **BitTorrent**: torrent files contain Merkle root + piece hashes. Verify pieces as they arrive.

### 5.3 Version control
- **Git**: every commit is a tree of file hashes. Commits, trees, blobs all SHA-1 (now SHA-256 in newer Git) — content-addressable. Comparing two trees: same Merkle algorithm.
- **Mercurial**, **Pijul**: similar.

### 5.4 Blockchains
- **Bitcoin**: each block's header includes a Merkle root of all transactions in the block. Light clients (SPV) request **Merkle proofs** to verify a transaction is in a block without downloading the block.
- **Ethereum**: state, transactions, receipts all Merkle-Patricia tries. The state root is part of every block.

### 5.5 Certificate transparency
Google's CT logs use Merkle trees to provide tamper-evident logs of issued TLS certificates. Anyone can verify a cert is in the log.

### 5.6 Backup verification
Rsync, ZFS send/receive use Merkle-like ideas for incremental sync. Borg backup uses content-addressable chunks.

---

## 6. Merkle Proofs (Inclusion Proofs)

You have a root hash; you want to prove a specific block is part of the tree without revealing the entire tree.

```
Verify block D3 is in the tree with root R.

Proof path = sibling hashes along D3's path to root:
   H(D4), H(D1||D2)  ← the "sister hashes"

Recompute: H( H(D3) || H(D4) ) → matches H3-4
           H( H1-2  || H3-4 ) → matches R?

If yes → D3 was in the tree. Proof is O(log N) in size.
```

Used by:
- Bitcoin SPV: prove a transaction was in a block.
- Certificate Transparency: prove a cert is logged.
- Blockchain rollups: prove state without sending it all.

A Merkle proof is **succinct** — `O(log N)` hashes regardless of dataset size.

---

## 7. Variants

### 7.1 Merkle-DAG
Internal nodes can have multiple parents. Used in IPFS — same content reused across many "files" is stored once.

### 7.2 Merkle-Patricia Trie (Ethereum)
A trie (prefix tree) with cryptographic hashes at each node. Allows key-value-style queries with Merkle proofs.

### 7.3 Sparse Merkle Tree
Tree includes every possible key, with default "empty" values for unused keys. Useful when you want to prove non-existence too.

### 7.4 Binary Merkle Tree
The classic. Simple, well-understood.

### 7.5 N-ary Merkle Tree
Each node has more than 2 children. Trade: shorter trees, larger proofs per level.

For most uses, **binary Merkle tree** is sufficient.

---

## 8. Implementation Considerations

### Chunk size
Larger chunks = smaller tree, larger diffs per leaf.
Smaller chunks = bigger tree, finer-grained diffs.

Rsync uses ~1 KB chunks; Cassandra repair uses entire token ranges; Bitcoin uses transactions.

### Hash function
- SHA-256 — gold standard.
- SHA-1 — used in Git originally; cryptographically broken (2017); Git migrating to SHA-256.
- BLAKE2 / BLAKE3 — faster, modern alternatives.
- Non-cryptographic (xxHash, MurmurHash) — only if adversaries aren't a concern (anti-entropy among trusted replicas).

For systems where data can be forged (blockchain, CT logs), cryptographic hashes are essential. For internal replica reconciliation, faster non-cryptographic hashes are sometimes acceptable.

### Tree storage
- In memory for small trees.
- On disk for large (Cassandra's repair).
- Lazily computed (only hash blocks that are accessed).

---

## 9. Worked Example: Cassandra Repair

You run `nodetool repair` on a 3-node Cassandra cluster after a partition.

```
1. Each node builds a Merkle tree of its data, broken into token-range buckets.
   Each leaf = hash of all data in that range.

2. Coordinator compares trees: gather root hashes from all nodes.

3. Where roots differ, descend recursively:
   - same hash at internal nodes → that range matches; skip.
   - different hash → recurse.

4. At the leaf level, identified differing ranges.

5. Coordinator orchestrates: ship the actual data for those ranges
   from the most-up-to-date replica to the others.

Network: a few KB of hashes per node + bulk data for differing ranges.
Without Merkle: ship all data → terabytes.
```

This is why repairs in Cassandra are practical at all.

---

## 10. Worked Example: Bitcoin SPV

A mobile wallet wants to verify "transaction T was included in block #800000" without downloading the block.

```
1. Wallet has block headers (~80 bytes each, all of them).
2. Wallet asks a full node: "Merkle proof for T in block 800000."
3. Full node returns:
   - The Merkle root from header 800000 (already known to wallet).
   - The sister hashes along T's path to root.
4. Wallet recomputes the path: hashes T, combines with sibling hashes,
   walks up the tree. If final hash matches root → T was in the block.
```

Wallet downloads: ~80 bytes/block headers (already had) + ~640 bytes (10-deep proof × 32 bytes) per verification.

Compare to downloading every block (~1 MB each, ~1 TB total): orders of magnitude less.

---

## 11. Worked Example: Git's Tree

A `commit` in Git is just:
```
parent: <hash of previous commit>
tree:   <hash of root tree object>
author: ...
message: "Fix the thing"
```

The `tree` hash is a Merkle root over all files in the working directory at that commit. Each directory is a tree object referencing file blobs (hashed) and subtrees (hashed).

```
commit (hash: a1b2)
  └── tree   (hash: c3d4)
        ├── README.md  (blob hash: e5f6)
        ├── src/       (subtree hash: 7890)
        │     ├── main.py (blob hash: aabb)
        │     └── lib.py  (blob hash: ccdd)
        └── docs/      (subtree hash: 1122)
```

If `main.py` changes, its blob hash changes → `src/`'s tree hash changes → the root tree hash changes → the commit hash changes.

This is why Git can show diffs across millions of files instantly: compare tree hashes, descend only into differing subtrees.

---

## 12. Operational Concerns

### Building the tree
For 10 TB of data, building the tree once takes the time to hash 10 TB. Incremental updates are easier: when a block changes, recompute only the path from leaf to root (`O(log N)` hashes).

### Concurrent updates
Tree is mutable in some implementations; hashes need to recompute on the fly. Some systems (Cassandra) rebuild periodically rather than incrementally.

### Tree drift between snapshots
If two replicas built their trees at different times, ongoing writes will look like differences. Cassandra coordinates by repairing **at a specific time** (snapshot point).

### Memory
Trees are smaller than data but not free. A 10 TB dataset with 1 MB blocks → 10M leaves × 32 bytes = 320 MB just for leaves; ~32 MB for internal nodes; ~350 MB total.

### Hash collisions
With SHA-256, infeasible. With MD5, theoretically possible. Use proper hash.

---

## 13. Common Mistakes

- **Comparing leaf hashes one by one.** Defeats the purpose; use root-and-descend.
- **Non-cryptographic hashes in adversarial contexts.** Blockchain / CT logs need crypto hashes.
- **Different block sizes between replicas.** Trees won't compare. Standardize.
- **Forgetting to verify the proof.** SPV clients sometimes skip; defeats security.
- **Excessive depth.** Very fine-grained chunks → deep trees → slower proofs.
- **Static trees on dynamic data.** Must rebuild or incrementally update on changes.
- **Treating root hash as random ID.** It depends on data layout and chunk boundaries — small layout changes change everything.

---

## 14. Cheat Card

```
MERKLE TREE   binary tree of hashes
               leaves: hash of data blocks
               internal: hash of children's hashes
               root: summary of entire dataset

COMPARE       compare roots; if equal, data identical
               if different, descend → O(log N) to find diffs

PROOF (inclusion)  sister hashes along path to root
                   O(log N) size; verify in seconds

USE CASES
  database anti-entropy   Cassandra, Riak, Dynamo
  distributed FS          IPFS, BitTorrent
  version control         Git (commits, trees, blobs)
  blockchain              Bitcoin, Ethereum
  cert transparency       Google CT logs

HASH FUNCTION
  SHA-256 (default), BLAKE3 (faster)
  non-crypto only if no adversaries

CHUNK SIZE    larger → smaller tree, coarser diffs
              smaller → bigger tree, finer reconciliation

PITFALLS      mismatched chunk size, static tree on dynamic data,
              skipped proof verification, weak hash

RULE          When two parties share data, Merkle trees let them
               find what's different without sharing what's the same.
```

---

## 15. Resources

### Papers
- "A Digital Signature Based on a Conventional Encryption Function" — Ralph Merkle, 1979 (the original tree).
- "Bitcoin: A Peer-to-Peer Electronic Cash System" — Satoshi Nakamoto, 2008 (Merkle tree for transactions).
- "IPFS: Content Addressed, Versioned, P2P File System" — Juan Benet.

### Books
- *Mastering Bitcoin* — Andreas Antonopoulos (great Merkle-in-Bitcoin coverage).
- *Designing Data-Intensive Applications* — Kleppmann (passing).

### Articles
- "Merkle trees explained" — visual guides (many).
- "Cassandra anti-entropy repair internals" — DataStax.
- "Git internals: trees, commits, blobs" — Git documentation Pro Git.
- "How Bitcoin SPV works" — bitcoin.org.

### Videos
- ByteByteGo — "Merkle Trees Explained".
- 3Blue1Brown — Bitcoin explainer with Merkle trees.
- Strange Loop talks on Git internals.

### Tools / Libraries
- **Python**: `merkletools`, `pymerkle`.
- **Go**: `cbergoon/merkletree`, `txaty/go-merkletree`.
- **Rust**: `rs_merkle`.
- **Built-in**: every blockchain / Git / IPFS implementation.

### Adjacent reading
- [Bloom Filters →](./bloom-filters.md)
- [Probabilistic Data Structures →](./probabilistic-data-structures.md)
- [Replication →](../04-databases/replication.md)
- [Quorum-Based Replication →](./quorum.md)
- [Gossip Protocol →](./gossip-protocol.md)
- [Blockchain & Distributed Ledger Basics →](../19-advanced/blockchain.md)
- [Wide-Column Stores →](../04-databases/wide-column-stores.md)

---

*Previous:* [← Count-Min Sketch & HyperLogLog](./probabilistic-data-structures.md)  |  *Next:* [CRDTs →](./crdts.md)
