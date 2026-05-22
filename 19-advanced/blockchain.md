# Blockchain & Distributed Ledger Basics

> **TL;DR** — A **blockchain** is a replicated, append-only log of transactions ordered by a **consensus algorithm** that doesn't require participants to trust each other. Each block contains a hash of the previous block, making the chain **tamper-evident** — modifying past data requires re-mining or re-attesting every block since. The original innovation (Bitcoin, 2008) was **Byzantine-fault-tolerant consensus over an open network of mutually-distrustful nodes**, achieved via **Proof-of-Work**. Modern chains use **Proof-of-Stake** (Ethereum, 2022) or classical **BFT consensus** (Cosmos, Avalanche, Hyperledger). The honest take: **blockchains solve a narrow problem — coordination among mutually-distrustful parties — at a significant cost in throughput, latency, and complexity**. For *most* system-design problems where a centralized database with audit logs would work, **a centralized database with audit logs is the right answer**. The genuinely interesting use cases are **digital currency, cross-chain settlement, public coordination games, certain supply-chain provenance, and decentralized identity**. Engineers should understand the core primitives — hashes, Merkle trees, consensus, smart contracts — because they reappear in non-blockchain systems (Git, certificate transparency, IPFS, signed-feed protocols).

---

## 1. The big picture

```
   Block N-1            Block N             Block N+1
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │ Prev hash:  │     │ Prev hash:  │     │ Prev hash:  │
  │ ABCDEF...   │ ◄── │ Hash(N-1)   │ ◄── │ Hash(N)     │
  │ Merkle root │     │ Merkle root │     │ Merkle root │
  │ Timestamp   │     │ Timestamp   │     │ Timestamp   │
  │ Nonce / sig │     │ Nonce / sig │     │ Nonce / sig │
  │             │     │             │     │             │
  │ TX 1        │     │ TX 1        │     │ TX 1        │
  │ TX 2        │     │ TX 2        │     │ TX 2        │
  │ ...         │     │ ...         │     │ ...         │
  └─────────────┘     └─────────────┘     └─────────────┘
```

Every block:
- Contains a batch of transactions.
- References the previous block by cryptographic hash.
- Has a Merkle root summarizing the transactions (see [Merkle Trees →](../08-distributed-systems/merkle-trees.md)).
- Is sealed by a consensus mechanism (PoW nonce, PoS signature, BFT quorum).

A network of nodes maintains the chain. Each node holds a full (or partial) copy. New transactions broadcast through the network; periodically, one node assembles them into a block and proposes it; consensus determines whether that block becomes canonical.

The defining trait: **changing any historical block requires changing every block after it**, because every subsequent block references its predecessor's hash. Tampering becomes detectable instantly.

---

## 2. Why blockchains exist

The classical distributed systems problem: how do `N` nodes agree on a shared sequence of values when some nodes might be slow, crash, or actively malicious?

- **Crash-tolerant consensus** (Paxos, Raft, ZAB) handles slow / crashed nodes. See [Consensus Algorithms →](../08-distributed-systems/consensus.md).
- **Byzantine fault-tolerant** (BFT) consensus handles **malicious** nodes — ones that lie, send conflicting messages, or attack the protocol. See [Byzantine Fault Tolerance →](../08-distributed-systems/bft.md).

Both assume a *fixed set of known participants*. Blockchain extends BFT to an **open, permissionless network** where anyone can join — and any node might be an adversary.

That's the hard problem Bitcoin solved. Satoshi Nakamoto's 2008 paper combined:
- **Cryptographic hashing** to chain blocks.
- **Public-key cryptography** for transaction authorization.
- **Proof-of-Work** to make creating new blocks costly enough that attackers can't cheaply outrun honest participants.
- **Longest-chain rule** to converge on a canonical history.

The result: trustless agreement over a global network of strangers. That's the genuine innovation.

The cost: a system that processes ~7 transactions per second (Bitcoin) or ~30 (Ethereum L1), with multi-minute confirmation times, at enormous energy / capital expense. For internet-scale workloads, blockchains are slow.

---

## 3. The core primitives

### Cryptographic hash functions

A function `H(x) → fixed-length digest` such that:
- **Deterministic**: same input → same output.
- **One-way**: given `H(x)`, you cannot find `x`.
- **Collision-resistant**: practically impossible to find `x ≠ y` with `H(x) = H(y)`.
- **Avalanche effect**: tiny input change → completely different output.

Used everywhere: block linking, transaction IDs, Merkle trees, mining puzzles. Standard hashes: **SHA-256** (Bitcoin), **Keccak-256** (Ethereum), **BLAKE2/3**.

### Public-key cryptography

Each user has a key pair: a private key (secret) and a public key (shared). A transaction signed with the private key can be verified by anyone with the public key. The **address** is typically a hash of the public key.

Used algorithms: **ECDSA** over secp256k1 (Bitcoin, Ethereum), **EdDSA / Ed25519** (Solana, others).

### Merkle trees

A tree where each leaf is the hash of a transaction, and each internal node is the hash of its two children. The root summarizes the entire set with a single hash. Used to:
- Prove a transaction is in a block in O(log N) proof size.
- Enable lightweight clients (SPV — Simplified Payment Verification) that don't store the full chain.

See [Merkle Trees →](../08-distributed-systems/merkle-trees.md).

### Digital signatures

Sign(private_key, message) → signature. Verify(public_key, message, signature) → true/false. This is how transactions prove the spender authorized the spend.

These four primitives compose into the entire blockchain stack. Nothing more exotic is needed.

---

## 4. Consensus mechanisms

The defining choice of any blockchain.

### Proof of Work (PoW)

Original Bitcoin / Ethereum-pre-Merge / Litecoin / Dogecoin. Miners compete to find a nonce such that `H(block_header + nonce)` starts with N zero bits — a numerically improbable target. The first miner to find one publishes the block; the network accepts it. Mining is computationally expensive (electricity + ASIC hardware); difficulty auto-adjusts to maintain ~10 min block time (Bitcoin) or ~13 s (Ethereum-pre-Merge).

Pros: open participation; well-understood security model (51% attack requires majority hashrate); proven robust.
Cons: enormous energy use; slow; high latency; expensive transactions; centralization risk if hashrate consolidates.

### Proof of Stake (PoS)

Modern Ethereum (post-Merge 2022), Cosmos, Cardano, Tezos, Avalanche, Polkadot, Solana variants. Validators put up "stake" (locked tokens) for the right to propose / vote on blocks. Misbehavior (signing two conflicting blocks, going offline too long) results in **slashing** — losing some or all of the stake.

Pros: orders of magnitude less energy; faster finality possible; lower transaction fees.
Cons: "rich get richer" stake concentration; complex slashing rules; subtle attack vectors (long-range attacks, nothing-at-stake).

### Classical BFT (PBFT, Tendermint, HotStuff)

Used by permissioned chains (Hyperledger Fabric, Quorum, Corda) and some public chains (Cosmos via Tendermint, Diem/Aptos via HotStuff). A known set of validators run a multi-round voting protocol; a quorum (typically 2f+1 of 3f+1 to tolerate f Byzantine nodes) commits a block. Strong, fast finality — once committed, the block cannot be reverted.

Pros: instant finality; high throughput; well-understood theory.
Cons: requires known validator set; doesn't scale to thousands of validators easily; permissioned model not always desirable.

### Other consensus families

- **Proof of Authority (PoA)** — a fixed set of trusted validators sign blocks. Suitable for permissioned / consortium chains. No real "consensus" beyond signing.
- **Proof of Space / Storage** (Chia, Filecoin) — miners commit disk space rather than CPU.
- **Avalanche consensus** (Avalanche, others) — repeated random sampling of peers; probabilistic agreement.
- **DAG-based** (IOTA, Hedera Hashgraph) — transactions form a graph instead of a single chain.

The choice of consensus drives throughput, finality, decentralization, and energy. There's no universal best.

---

## 5. UTXO vs account model

Two ways to track who owns what.

### UTXO model (Bitcoin)

Every transaction consumes one or more **Unspent Transaction Outputs (UTXOs)** and creates new ones. Your "balance" is the sum of UTXOs you can spend with your private key. There's no persistent "account" — just a graph of outputs.

```
TX_A: outputs   →   [UTXO1: Alice, 5 BTC]
                    [UTXO2: Bob,   2 BTC]

TX_B: consumes UTXO1  →  [UTXO3: Carol, 3 BTC]
                          [UTXO4: Alice, 2 BTC]  (change)
```

Pros: trivially parallelizable, strong privacy (no global account state to query).
Cons: cumbersome for stateful computation (smart contracts).

### Account model (Ethereum)

Each account has a balance and (for contracts) persistent state. A transaction debits one account, credits another, or invokes a contract.

```
state:  Alice = 5 ETH
        Bob   = 2 ETH

TX:     debit Alice 3 ETH, credit Bob 3 ETH

state:  Alice = 2 ETH
        Bob   = 5 ETH
```

Pros: clean model for smart contracts and stateful applications.
Cons: harder to parallelize; transactions must serialize per account.

Bitcoin and most "money" chains use UTXO. Ethereum and most "smart contract" chains use the account model. Some chains (Cardano) use **extended UTXO** to get both.

---

## 6. Smart contracts

A **smart contract** is a program stored on-chain whose code is executed by every node when invoked. Inputs (transaction arguments) → deterministic outputs (state changes, events). Once deployed, the code is immutable unless the contract itself supports an upgrade pattern.

Ethereum's **EVM** (Ethereum Virtual Machine) is the dominant smart-contract platform. Contracts are written in **Solidity** or **Vyper**, compiled to EVM bytecode, deployed by a transaction.

Other VMs: **CosmWasm** (WebAssembly on Cosmos), **MoveVM** (Aptos, Sui), **Solana's Sealevel** (parallel BPF), **Stylus** (Arbitrum's WASM).

A real ERC-20 token contract (the standard for fungible tokens) is roughly 200 lines of Solidity. Once deployed, anyone in the world can interact with it.

Smart contracts power:
- **Tokens** (ERC-20 fungible, ERC-721 NFTs).
- **DeFi** — automated market makers (Uniswap), lending (Aave, Compound), derivatives.
- **DAOs** — on-chain governance.
- **Bridges** — cross-chain asset transfer.

The economic value moves through them. So do the exploits — billions of dollars have been lost to smart contract bugs.

---

## 7. Layers — L1, L2, sidechains, rollups

L1 blockchains (Bitcoin, Ethereum, Solana) provide base-layer security but limited throughput. **Layer 2** solutions extend throughput while inheriting L1 security.

### Rollups

The dominant L2 pattern (2026). Batch many transactions off-chain; submit a compressed proof or batch on-chain.

- **Optimistic rollups** (Arbitrum, Optimism, Base) — assume submitted batches are valid; allow challengers to dispute within a window (typically 7 days). Cheap, but withdrawals are slow.
- **ZK rollups** (zkSync, Starknet, Polygon zkEVM, Scroll) — submit a zero-knowledge proof that the batch is valid. Cheaper long-term; expensive to compute proofs.

L2s scale Ethereum from ~30 TPS to thousands. Most retail DeFi activity has migrated to L2s.

### Sidechains

A separate blockchain pegged to a main chain via a bridge. Polygon PoS, BSC, Avalanche subnets. Faster but with weaker security assumptions than rollups.

### State channels

Off-chain transactions between two parties; the chain settles only the final state. Bitcoin's **Lightning Network** is the canonical example. High throughput for known counterparties.

### Cross-chain bridges

Move assets between chains. Lock on chain A, mint on chain B, or burn on chain B, unlock on chain A. The most attacked component in crypto — bridge hacks account for billions in losses.

---

## 8. Hashing economics — security and throughput

The defining trade-off:

- **More decentralized** = more nodes verifying = slower throughput, harder coordination.
- **Stronger consensus** = more confidence in finality = higher latency.
- **Higher TPS** = either tighter validator set or weaker security model.

The **blockchain trilemma** (Vitalik Buterin's framing): decentralization, security, scalability — pick two. L2s exist to soften this by inheriting L1 security while scaling.

Throughput at 2026 levels:
- Bitcoin L1: ~7 TPS, ~10 min finality (probabilistic).
- Ethereum L1: ~15-30 TPS, ~12 s blocks, ~13 min finality.
- Ethereum L2 (Arbitrum, Optimism): ~1000+ TPS aggregate.
- Solana: claimed 1000s of TPS; periodic outages.
- Aptos / Sui: 1000+ TPS, MoveVM.
- Cosmos chains via Tendermint: 1000s of TPS per app-chain.

Compare to Visa (~24,000 TPS sustained). Public chains aren't there yet — and the gap is mostly about decentralization. Permissioned BFT chains (Hyperledger) can do 10K+ TPS with known validators.

---

## 9. What blockchain is actually good at

A short list, honest:

- **Digital cash / store of value across borders** — Bitcoin's original pitch.
- **Permissionless asset issuance** — anyone can issue a token without a centralized intermediary.
- **Cross-chain settlement** — DEXs, atomic swaps, decentralized stablecoins.
- **Public coordination games** — DAOs, prediction markets, public funding rounds.
- **Verifiable supply-chain provenance** — when many distrustful parties need a shared record (food safety, pharmaceuticals, conflict minerals).
- **Notarization / time-stamping** — proving a document existed at a specific date.
- **Decentralized identity (DID)** — self-sovereign identity standards (W3C DID, Verifiable Credentials).
- **Censorship-resistant publishing** — for narrow but real use cases.
- **Carbon credits / commitments** — public ledger of climate commitments (with caveats).

The common thread: **mutual distrust + need for shared truth + no acceptable central authority**. If you have a central authority (your company, a bank, a regulator), a centralized DB with audit logs is almost always cheaper, faster, and more flexible.

---

## 10. What blockchain is bad at

Equally honest:

- **High-throughput online services.** A few thousand TPS doesn't compete with Postgres.
- **Latency-sensitive workloads.** Multi-second-to-minute confirmation times kill responsiveness.
- **Mutable / private data.** Once on-chain, it's public and forever. GDPR's "right to be forgotten" is incompatible.
- **Storage of large blobs.** Per-byte cost is enormous; chains link to IPFS / Arweave / S3 for files.
- **Random access / queries.** Reading state is fast; running analytical queries is expensive.
- **Privacy.** Standard chains are fully transparent. Zero-knowledge tech adds privacy at high complexity.
- **Easy schema evolution.** Smart contracts are immutable by default; upgrades require proxy patterns and care.
- **Forgiving error handling.** Send to the wrong address, you lose the funds. No undo.

If your problem fits these constraints, blockchain is the wrong tool. Use a database.

---

## 11. The non-financial use cases

Distinct from cryptocurrency speculation, several blockchain ideas have real engineering value:

- **Certificate Transparency** — Google's public log of all TLS certificates. Append-only, Merkle-tree-backed, audited by browsers. Not a "blockchain" per se but uses the same primitives.
- **Git** — content-addressed, tamper-evident, distributed history. The mental model overlaps; git lacks consensus, but adds branching and merging.
- **Sigstore / Rekor** — software supply-chain attestations stored in an append-only transparency log.
- **Hyperledger Fabric** — permissioned blockchain for enterprise / consortium use.
- **Database audit logs** — append-only, hash-chained tables in PostgreSQL or similar. Inspired by blockchain ideas without the overhead.

The patterns — append-only logs, Merkle proofs, hash chains, signed transactions — are useful engineering primitives. You don't always need consensus.

---

## 12. Common Mistakes / Anti-Patterns

- **Using blockchain because it sounds modern.** Most "blockchain projects" should be databases. If you control all writers, you don't need consensus.
- **Storing private data on a public chain.** Forever on display; GDPR-incompatible.
- **Storing large files on-chain.** Use IPFS / Arweave / S3, store the hash on-chain.
- **Custom smart contract without audit.** Bugs in smart contracts have caused $5B+ in losses across DeFi.
- **Bridges as a trust assumption.** Cross-chain bridges have lost more value to hacks than any other crypto category. Treat as high-risk.
- **Hot wallets with significant assets.** Use hardware wallets / multi-sig / MPC.
- **Treating "decentralized" as binary.** Most "decentralized" projects have very centralized governance or validator sets.
- **Assuming smart contract upgrades are easy.** Proxy patterns work but introduce attack surface; mistakes are common.
- **No transaction monitoring.** Front-running, sandwich attacks, MEV — on public chains your mempool is open.
- **Confusing tokens with stocks.** Regulatory implications vary; treat with extreme care.
- **Over-engineering blockchain for problems with a clear central authority.** Almost certainly the wrong tool.
- **Ignoring gas economics.** Storage / compute per transaction costs real money; design with that in mind.
- **Trusting an unaudited oracle.** Many DeFi exploits begin with manipulating a price oracle.
- **Ignoring quantum risk for long-lived chains.** ECDSA is breakable by sufficiently large quantum computers (probably a decade+ away, but real for systems that need 30-year guarantees).

---

## 13. Cheat Card

```
PURPOSE   Replicated, append-only log of transactions ordered
          by a consensus algorithm that doesn't require trust
          between participants.

CORE PRIMITIVES (REUSABLE OUTSIDE BLOCKCHAIN)
  Hash chain        each block references previous by hash
  Merkle tree       summarize transactions in one root hash
  Public-key sig    transactions authorized by private keys
  Consensus         everyone agrees on the same history

CONSENSUS FAMILIES
  PoW       Bitcoin, pre-Merge ETH — energy-heavy, robust
  PoS       Ethereum post-Merge, Cosmos, Cardano — stake + slashing
  BFT       Tendermint, HotStuff, PBFT — fast, permissioned-ish
  PoA       trusted validators sign — consortia
  DAG       IOTA, Hashgraph — non-chain alternatives

LAYERS
  L1                 base chains (Bitcoin, Ethereum, Solana)
  Optimistic rollup  Arbitrum, Optimism, Base — challenge windows
  ZK rollup          zkSync, Starknet, Polygon zkEVM
  Sidechain          Polygon PoS, Avalanche subnets
  State channels     Lightning (Bitcoin) — off-chain settlement
  Cross-chain bridge most attacked component

STATE MODELS
  UTXO      Bitcoin — graph of unspent outputs; parallelizable
  Account   Ethereum — balances + contract state; smart contract friendly

SMART CONTRACTS
  EVM       Solidity / Vyper / Yul (Ethereum + many L2s)
  WASM      CosmWasm, Stylus
  Move      Aptos, Sui
  SVM       Solana's BPF

GOOD FOR
  Digital cash, asset issuance
  Cross-chain / DEX settlement
  Public coordination games (DAOs, prediction markets)
  Provenance with mutually-distrustful parties
  Notarization / time-stamping
  Decentralized identity

BAD FOR
  High-throughput online services
  Private data
  Large blobs (use IPFS / Arweave)
  Strong consistency at low latency
  Anything where a central authority is fine

PITFALLS
  Using a blockchain when a database would do
  PII on a public chain
  Unaudited smart contracts
  Bridges as a trust assumption
  Hot wallets at scale (use multisig / MPC)
  Confusing "decentralized" with "trustworthy"

RULE   Blockchain solves "no trusted party" — at a real cost.
       Without that constraint, use a database with audit logs.
       The primitives (hashes, Merkle, append-only, signatures)
       are valuable beyond blockchain itself.
```

---

## 14. Resources

### Papers and original sources
- "Bitcoin: A Peer-to-Peer Electronic Cash System" — Satoshi Nakamoto, 2008. The original.
- "Ethereum: A Next-Generation Smart Contract and Decentralized Application Platform" — Vitalik Buterin, 2013.
- "PBFT: Practical Byzantine Fault Tolerance" — Castro & Liskov, 1999.
- "The Tendermint paper" — Buchman, 2016.
- "HotStuff: BFT Consensus in the Lens of Blockchain" — Yin, Malkhi, et al., 2018.

### Books
- *Mastering Bitcoin* — Andreas Antonopoulos. The reference.
- *Mastering Ethereum* — Antonopoulos & Wood. Co-authored with Ethereum's co-creator.
- *The Bitcoin Standard* — Saifedean Ammous (advocacy lens; useful context).
- *Cryptoassets* — Burniske & Tatar.
- *Programming Bitcoin* — Jimmy Song.

### Documentation
- **Bitcoin Core** — <https://bitcoin.org/en/developer-documentation>
- **Ethereum** — <https://ethereum.org/en/developers/docs/>
- **Solidity** — <https://docs.soliditylang.org>
- **Cosmos SDK / Tendermint** — <https://docs.cosmos.network>
- **Hyperledger Fabric** — <https://hyperledger-fabric.readthedocs.io>

### Articles
- "Why Bitcoin matters" — Marc Andreessen, 2014 (historical context).
- "Why decentralization matters" — Chris Dixon.
- "Ethereum: Move to Proof of Stake" — Ethereum Foundation explainers.
- "How does Bitcoin work?" — 3blue1brown video.
- Vitalik Buterin's blog — <https://vitalik.ca>

### Videos
- *3blue1brown — Bitcoin Visually Explained* — best intro on YouTube.
- *Ethereum Devcon* talks.
- ByteByteGo — "How Blockchain Works."

### Tools
- **Bitcoin Core**, **bitcoind**.
- **Geth, Erigon, Reth** — Ethereum clients.
- **Hardhat, Foundry, Truffle** — Solidity development.
- **OpenZeppelin** — audited contract libraries.
- **Slither, Mythril, Echidna** — smart-contract security.
- **The Graph** — indexing for chain data.
- **IPFS / Arweave / Filecoin** — decentralized storage.

### Adjacent reading
- [Merkle Trees →](../08-distributed-systems/merkle-trees.md)
- [Byzantine Fault Tolerance →](../08-distributed-systems/bft.md)
- [Consensus Algorithms (Paxos, Raft, ZAB) →](../08-distributed-systems/consensus.md)
- [CAP Theorem →](../08-distributed-systems/cap-theorem.md)
- [Peer-to-Peer Systems & DHTs →](./p2p-dht.md)
- [Public-Key Cryptography Basics →](../12-security/pki.md)
- [Encryption at Rest & In Transit →](../12-security/encryption.md)

---

*Previous:* [← Edge Computing](./edge-computing.md)  |  *Next:* [Peer-to-Peer Systems & DHTs →](./p2p-dht.md)
