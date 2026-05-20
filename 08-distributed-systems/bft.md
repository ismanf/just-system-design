# Byzantine Fault Tolerance

> **TL;DR** — **Byzantine fault tolerance (BFT)** is the ability of a distributed system to reach consensus when some nodes are not just crashing but **actively behaving maliciously** — lying, sending different messages to different peers, colluding, or being compromised. The name comes from the **Byzantine Generals Problem** (Lamport, 1982). Standard consensus (Raft, Paxos) assumes **crash faults only** — nodes either work correctly or stop. BFT consensus survives a fraction of nodes acting arbitrarily badly. The math: BFT requires at least **3f+1 nodes** to tolerate `f` Byzantine failures (vs `2f+1` for crash-fault tolerance). Classical BFT algorithms: **PBFT** (Castro & Liskov, 1999) — the practical foundation. Modern blockchain-era variants: **HotStuff**, **Tendermint**, **Algorand**, used in **Diem/Libra**, **Cosmos**, **Hyperledger**. For most production distributed systems (internal networks, trusted nodes), **crash-fault tolerance is enough**. BFT is essential in **trustless** environments — blockchains, public networks, multi-organizational systems where you can't trust the peer code or operator.

---

## 1. The Byzantine Generals Problem

Lamport, Shostak, Pease, 1982 — a parable:

> Several Byzantine generals surround an enemy city. They communicate by messenger. They must agree: attack together or retreat together. Splitting the army loses. Some generals may be traitors — they can send conflicting messages to different peers.
> 
> Can they reach consensus despite traitors?

The result: **with synchronous communication, you need 3f+1 generals to tolerate f traitors.** Fewer, and traitors can prevent agreement or cause inconsistent decisions.

This formalizes the threat model where some participants don't just fail — they actively deceive.

---

## 2. Crash Faults vs Byzantine Faults

| Fault model | Behavior | Tolerance |
|---|---|---|
| **Crash-fault (CFT)** | Node either works or stops | 2f+1 to tolerate f |
| **Crash-recovery** | Node may crash and restart, loses state | Similar to CFT, more state mgmt |
| **Omission** | Node fails to send/receive specific messages | Between CFT and BFT |
| **Byzantine** | Node behaves arbitrarily — lies, sends conflicting messages, colludes | 3f+1 to tolerate f |

Raft, Paxos, ZAB assume **crash-fault** — the simplest and most common in practice.

BFT algorithms make no such assumption. They expect nodes might:
- Send different messages to different peers.
- Lie about state.
- Collude.
- Be slow.
- Be compromised.

---

## 3. Why 3f+1?

A node receives messages from peers. If `f` peers are Byzantine, they can lie. To distinguish truth:
- You need `f+1` honest votes to outvote `f` liars.
- But the honest nodes also need quorum among themselves.
- Worst case: `f` Byzantine + `f` slow → need `f+1` more for a real quorum.

Total: `3f + 1` minimum. With this, an honest majority always overcomes Byzantine influence + slow nodes.

```
f=1: need 4 nodes (tolerate 1 Byzantine)
f=2: need 7 nodes (tolerate 2 Byzantine)
f=3: need 10 nodes
```

Compare to crash-fault: `2f+1` (f=1 → 3 nodes, f=2 → 5 nodes). BFT is more expensive.

---

## 4. PBFT — Practical Byzantine Fault Tolerance

Castro & Liskov, 1999. The breakthrough that made BFT actually deployable.

### Setup
- N = 3f + 1 replicas.
- One **primary** (chosen round-robin).
- Clients send requests to primary.

### Three phases per request
1. **Pre-prepare** — primary sends `<PRE-PREPARE, view, seq, request>` to all backups.
2. **Prepare** — each backup, if it accepts, sends `<PREPARE, view, seq, digest>` to all peers. Wait for `2f` prepares matching.
3. **Commit** — each replica that prepared sends `<COMMIT, view, seq, digest>` to all peers. Wait for `2f+1` commits. Then execute and reply to client.

Three rounds. Network O(N²) messages per request. Heavy but feasible.

### View change
If primary is faulty (timeout / detected bad message), nodes initiate **view change** — elect new primary. The protocol guarantees safety across view changes.

### Properties
- **Safety always**: never agree on conflicting values.
- **Liveness in synchrony**: if message delays are bounded, progress is made.
- **Tolerates f Byzantine failures** in N = 3f+1 cluster.

### Real implementations
PBFT was the foundation; many improvements followed. Direct production use is rare; descendants (HotStuff, Tendermint) dominate.

---

## 5. Modern BFT: HotStuff, Tendermint, Algorand

### HotStuff (Yin et al., 2019)
Used by Facebook's Diem/Libra. Linear message complexity per phase (vs PBFT's O(N²)). Pipelined.

Three-phase voting: prepare → pre-commit → commit. A "leader" rotates each round.

**Strengths**
- Linear communication.
- Simpler view change than PBFT.
- Pipeline-able for throughput.

**Used by**: Diem (defunct), Aptos blockchain.

### Tendermint
Used in Cosmos blockchain ecosystem. Similar to PBFT with simpler design. Block-by-block consensus.

**Strengths**
- Stops on Byzantine fault (no fork).
- Mature, deployed widely.
- Plug into any state machine.

**Used by**: Cosmos, Binance Chain (originally), many Cosmos zones.

### Algorand
Cryptographic sortition: random nodes selected to vote each round. Probabilistic finality. No designated leader.

**Strengths**
- Truly decentralized leader selection.
- Massive scale (thousands of validators).
- Fast finality.

**Used by**: Algorand blockchain.

### Nakamoto Consensus (Bitcoin)
Not strictly BFT in the classical sense — uses **proof-of-work** for probabilistic agreement. Tolerates < 50% malicious hashpower. Bitcoin, Ethereum (pre-merge).

### Proof-of-Stake variants
Most modern blockchains use BFT-style consensus with stake-weighted votes: Ethereum 2.0 (Casper FFG), Solana, Avalanche.

---

## 6. When You Actually Need BFT

### You need BFT when
- **You can't trust the peers' code.** Open blockchain validators. Multi-organization consortiums.
- **You can't trust peer operators.** Public/permissionless networks.
- **A node compromise must not cascade.** High-security finance, government, sensitive multi-party computation.
- **Trustless coordination is the value proposition.** Decentralized everything.

### You don't need BFT when
- **All nodes are your own.** Internal infrastructure. Same admin team. Crash-fault is enough.
- **Operator trust is sufficient.** Enterprise cluster, all under one company.
- **Performance matters more than malicious tolerance.** BFT is 3–10× slower than CFT.

For typical SaaS / internal services, **Raft is the answer**. BFT is for blockchain and trust-minimized systems.

---

## 7. The Cost of BFT

### Latency
3+ communication rounds per commit. Cross-region: 100s of ms.

### Throughput
PBFT: hundreds of TPS at small N. HotStuff/Tendermint: thousands.

### Message complexity
PBFT: O(N²) per request. HotStuff: O(N). Algorand: O(N) with sortition.

### Cluster size
3f+1 minimum. To tolerate ~30 Byzantine validators, need 100+. Each node coordinates with all others.

### Operations
Harder to operate. Validators must be globally distributed, secured, monitored.

For an internal database: Raft costs ~5 ms commit. PBFT costs 50+ ms. Worth it only if you face actual Byzantine threats.

---

## 8. Bitcoin / Proof-of-Work as BFT

Bitcoin solves Byzantine consensus using **proof-of-work**:
- Miners compete to extend the chain.
- Honest miners follow the longest chain.
- An attacker needs > 50% of hashpower to fork.

Strictly: PoW is **probabilistic**. There's no "final" finalized block — older blocks are more secure but never absolutely final.

PoW solved a different problem from PBFT: **open membership**. Anyone can join as a miner without authorization. PBFT assumes a known set of validators.

This is why Bitcoin (open) uses PoW; consortium chains (known validators) use PBFT/HotStuff.

---

## 9. Comparison Table

| Aspect | Raft / Paxos (CFT) | PBFT | HotStuff | Bitcoin PoW |
|---|---|---|---|---|
| Fault model | Crash | Byzantine | Byzantine | Byzantine (incl. open) |
| Min nodes | 2f+1 | 3f+1 | 3f+1 | Open |
| Throughput | High (10k+ TPS) | Hundreds | Thousands | ~7 TPS (Bitcoin) |
| Latency | ms | 100s ms | 100s ms | minutes (probabilistic finality) |
| Finality | Immediate | Immediate | Immediate | Probabilistic |
| Membership | Permissioned | Permissioned | Permissioned | Permissionless |
| Used by | etcd, Spanner, CockroachDB | research, Hyperledger | Diem, Aptos | Bitcoin |
| Complexity | Manageable | Hard | Hard | Different paradigm |

---

## 10. BFT in Permissioned Blockchains

Most enterprise / consortium blockchains use BFT consensus because:
- Validators are known (regulatory, business reasons).
- Throughput matters (PoW too slow).
- Finality is desired (no chain reorgs).

Examples:
- **Hyperledger Fabric** — PBFT-like consensus (older versions used Kafka).
- **R3 Corda** — uses notaries with BFT or CFT options.
- **Quorum** (banking) — BFT.
- **Cosmos** — Tendermint.

These systems trade decentralization for throughput and known governance.

---

## 11. Cryptographic Building Blocks

BFT consensus typically uses:
- **Digital signatures** — sign every message; receivers verify the sender.
- **MACs** (Message Authentication Codes) — faster alternative to signatures.
- **Threshold signatures** — m-of-n signature aggregation (HotStuff uses this for efficient aggregation).
- **VRFs** (Verifiable Random Functions) — Algorand-style random leader selection.

Crypto turns "I trust this message came from node X" into mathematical certainty, even from untrusted peers.

---

## 12. Failure Modes Specific to BFT

### Eclipse attacks
An attacker controls all peers a node communicates with. The node has a distorted view. Mitigations: diverse peer sets, gossip topology with random links.

### Long-range attacks
In proof-of-stake systems: an attacker buys keys from past validators (no longer have anything at stake). They can rewrite history. Mitigations: checkpoints, weak subjectivity.

### Nothing-at-stake
PoS validators can sign multiple competing chains for free. Mitigated by slashing (deduct stake on misbehavior).

### Selfish mining
PoW miner withholds blocks to extend their chain in private. Captures more than their fair share. Theoretical attack with limited practical effect.

### Validator collusion
> 1/3 of validators colluding can break safety in PBFT. Mitigations: diverse validator set, economic incentives, slashing.

---

## 13. Worked Example: Why an Internal Database Doesn't Need BFT

You're building a 5-node Postgres replica cluster across 3 AZs in one cloud account.

### Threat model
- AZ failures (network).
- Hardware failures (disks, NICs).
- Server crashes (OOM, kernel panics).
- Operator mistakes (bad config).

### Not in the threat model
- A replica node sending different messages to different peers.
- A replica node deliberately corrupting writes.
- A replica node colluding with another to violate consensus.

Why? Because **you operate all 5 nodes**. Their behavior depends on your code and configuration. If they misbehave, it's a bug you fix — not an adversary you defend against.

Raft (CFT) is sufficient. BFT would add 5× cost for protection against a threat that doesn't exist.

### Compare to a public blockchain
Validators are operated by strangers. Some may be incentivized to attack. BFT is essential.

The threat model decides.

---

## 14. Where the Field Is Going

- **Faster BFT**: HotStuff and variants reduce communication. Algorand-style sortition scales validator set.
- **Hybrid**: PoS + BFT (Ethereum, Cosmos).
- **Asynchronous BFT**: HoneyBadgerBFT, DispersedLedger — works in fully async networks.
- **Confidential computing**: TEEs (Intel SGX, AMD SEV) reduce trust required, lowering BFT cost.
- **Optimistic rollups**: assume honest behavior; punish exceptions. Used by Optimism, Arbitrum.

For most production engineers, BFT remains "a thing blockchains do." For blockchain engineers, it's the core skill.

---

## 15. Common Mistakes

- **Reaching for BFT when CFT suffices.** Massive overkill for internal systems.
- **Assuming PoW is "BFT".** Probabilistic, open-membership — different family.
- **Forgetting the 3f+1 bound.** Tolerating 2 Byzantine faults needs at least 7 nodes.
- **No cryptographic signatures in BFT.** Without them, an attacker can fake messages from honest peers.
- **Trusting blockchain consensus as "secure"** without understanding the threat model.
- **Skipping eclipse / long-range / collusion analysis.** BFT's "f" only counts overt failures; subtle attacks may use fewer compromised nodes.
- **Choosing BFT for performance.** It's slower than CFT; you pay for malicious-tolerance.

---

## 16. Cheat Card

```
BFT          consensus tolerating arbitrary (Byzantine) node behavior
              not just crashes — lies, conflicts, collusion

WHY          when peers can't be trusted (blockchain, consortium,
              multi-org, open networks)

VS CFT       Raft/Paxos: crash-fault, 2f+1 nodes
              BFT:        3f+1 nodes for f Byzantine failures

GENERALS     classic problem (Lamport 1982); 3f+1 result

PBFT         3-phase commit (pre-prepare, prepare, commit)
              first practical BFT (1999)
              O(N²) messages per request

MODERN       HotStuff (linear messages), Tendermint, Algorand
              used by blockchain ecosystems

POW          Bitcoin-style; probabilistic; open membership
              different family from classical BFT

WHEN USE     trustless / multi-organization
              public blockchains
              high-security where node compromise must be tolerated

WHEN SKIP    your-own internal systems → use CFT (Raft)
              CFT is 3–10× faster

CRYPTO       digital signatures essential
              MACs, threshold sigs, VRFs for performance

PITFALLS     BFT when CFT enough; ignoring eclipse / long-range;
              fewer than 3f+1 nodes; no signatures

RULE         Threat model decides. Trust your peers? Use Raft.
              Don't trust them? Pay the BFT cost.
```

---

## 17. Resources

### Papers
- "The Byzantine Generals Problem" — Lamport, Shostak, Pease, 1982.
- "Practical Byzantine Fault Tolerance" — Castro & Liskov, OSDI 1999.
- "HotStuff: BFT Consensus with Linearity and Responsiveness" — Yin et al., 2019.
- "Algorand: Scaling Byzantine Agreements for Cryptocurrencies" — Gilad et al., 2017.
- "Tendermint: Byzantine Fault Tolerance in the Age of Blockchains" — Buchman, 2016.
- "Bitcoin: A Peer-to-Peer Electronic Cash System" — Nakamoto, 2008.

### Books
- *Bitcoin and Cryptocurrency Technologies* — Narayanan et al.
- *Mastering Bitcoin* — Andreas Antonopoulos.
- *Designing Data-Intensive Applications* — Kleppmann (covers Byzantine generals briefly).

### Articles
- "On the difficulty of Byzantine Generals" — Lamport's own blog.
- "Comparing BFT protocols" — various academic surveys.
- "What's the difference between Raft and PBFT" — Ethereum-research forums.

### Videos
- ByteByteGo — "Byzantine Generals Problem".
- Strange Loop talks on consensus.
- Stanford CS 251 — Bitcoin and Cryptocurrencies course.

### Tools
- **Hyperledger Fabric** — permissioned BFT-ish blockchain.
- **Cosmos SDK + Tendermint** — BFT framework.
- **HotStuff implementations** — Diem (archived), Aptos.

### Adjacent reading
- [Consensus →](./consensus.md)
- [CAP Theorem →](./cap-theorem.md)
- [Quorum-Based Replication →](./quorum.md)
- [Split-Brain →](./split-brain.md)
- [Blockchain & Distributed Ledger Basics →](../19-advanced/blockchain.md)
- [Peer-to-Peer Systems & DHTs →](../19-advanced/p2p-dht.md)

---

*Previous:* [← CRDTs](./crdts.md)  |  *Next:* [Split-Brain Problem →](./split-brain.md)
