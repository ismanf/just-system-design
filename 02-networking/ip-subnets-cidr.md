# IP Addressing, Subnets, CIDR

> **TL;DR** — An **IP address** is the L3 identity of a host on a network. **IPv4** uses 32 bits (~4.3B addresses) written as dotted-decimal like `10.0.5.42`. **IPv6** uses 128 bits, written in hex. A **subnet** is a contiguous range of IPs that share a network prefix. **CIDR** notation (`10.0.0.0/16`) tells you exactly how many of the leading bits are the network part. CIDR math underlies every VPC, every Kubernetes cluster, every firewall rule — knowing it cold is non-negotiable for backend engineers.

---

## 1. What an IP Address Is

An IP address identifies an **interface** (not a host — a host can have many).

### IPv4
- 32 bits, written as four 8-bit decimal numbers separated by dots.
- Range: `0.0.0.0` to `255.255.255.255`.
- ~4.3 billion total addresses. Exhausted globally in ~2011 (RIRs stopped handing them out).

```
192.168.1.42
└┬─┘└┬─┘└┬┘└┬─┘
 8b   8b  8b  8b   = 32 bits total
```

### IPv6
- 128 bits → 3.4 × 10³⁸ addresses (enough for every grain of sand to have its own internet).
- Written as eight groups of 4 hex digits: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`.
- Leading zeros can be omitted; one run of zero groups can be `::`.
  ```
  2001:0db8:85a3:0000:0000:8a2e:0370:7334
  → 2001:db8:85a3::8a2e:370:7334
  ```

> Modern systems are increasingly IPv6-native (mobile networks especially). But cloud infra (AWS VPCs, K8s) is still IPv4-heavy. Know both, focus on IPv4 for now.

---

## 2. Network vs Host Bits

Every IPv4 address is split into:
- **Network part** — identifies *which network* the address belongs to.
- **Host part** — identifies *which device* within that network.

The split point is defined by a **subnet mask** (or, more modernly, by CIDR notation).

```
IP:        10  .  0  .  5  .  42
Binary:   00001010.00000000.00000101.00101010

Mask /24: 11111111.11111111.11111111.00000000   ← first 24 bits = network
          └────── 24 bits ──────────┘└── 8 ─┘
                  NETWORK              HOST

So 10.0.5.0/24 is the network; 10.0.5.42 is host 42 in it.
```

---

## 3. CIDR Notation: The Only Thing You Need to Memorize

**CIDR** (Classless Inter-Domain Routing) replaced the old "Class A/B/C" rules. The notation:

```
<address>/<prefix length>
e.g., 10.0.0.0/16
```

The number after the slash = how many leading bits are the **network** part. The remaining bits are the **host** part.

### The CIDR Math Table (memorize)

| Prefix | Subnet mask | # of addresses | # of usable hosts (–2) | Common use |
| --- | --- | --- | --- | --- |
| /32 | 255.255.255.255 | 1 | 1 (host route) | A single IP |
| /31 | 255.255.255.254 | 2 | 2 (RFC 3021 P2P) | Point-to-point links |
| /30 | 255.255.255.252 | 4 | 2 | Router-to-router links |
| /29 | 255.255.255.248 | 8 | 6 | Tiny subnets |
| /28 | 255.255.255.240 | 16 | 14 | Small subnets |
| /27 | 255.255.255.224 | 32 | 30 | Small AWS subnet |
| /26 | 255.255.255.192 | 64 | 62 | Small AWS subnet |
| /25 | 255.255.255.128 | 128 | 126 | Mid subnet |
| **/24** | **255.255.255.0** | **256** | **254** | **The classic "one office subnet"** |
| /23 | 255.255.254.0 | 512 | 510 | Larger subnet |
| /22 | 255.255.252.0 | 1,024 | 1,022 | Larger subnet |
| /21 | 255.255.248.0 | 2,048 | 2,046 |  |
| /20 | 255.255.240.0 | 4,096 | 4,094 |  |
| /19 | 255.255.224.0 | 8,192 | 8,190 |  |
| /18 | 255.255.192.0 | 16,384 | 16,382 |  |
| /17 | 255.255.128.0 | 32,768 | 32,766 |  |
| **/16** | **255.255.0.0** | **65,536** | **65,534** | **The classic "VPC" size** |
| /15 | 255.254.0.0 | 131,072 |  |  |
| /14 | 255.252.0.0 | 262,144 |  |  |
| /13 | 255.248.0.0 | 524,288 |  |  |
| /12 | 255.240.0.0 | 1,048,576 |  | Private-net block `172.16/12` |
| /11 | 255.224.0.0 | 2,097,152 |  |  |
| /10 | 255.192.0.0 | 4,194,304 |  |  |
| /9 | 255.128.0.0 | 8,388,608 |  |  |
| **/8** | **255.0.0.0** | **16,777,216** |  | Private-net block `10/8` |

### Cheap mental math
- Each step down in prefix length **doubles** the size.
- `/24` = 256 addresses, easy mental anchor.
- `/16` = 65,536, the other anchor.
- Lose 2 for usable hosts: network address (all-zero host bits) and broadcast (all-ones host bits).

### How to compute the network address
Take the IP and zero out the host bits.
```
192.168.5.130/26
mask = first 26 bits = 255.255.255.192
192.168.5.130 AND 255.255.255.192 = 192.168.5.128
              ──────────────────
                Network address
```
So 192.168.5.130 lives in the subnet 192.168.5.128/26, which covers 192.168.5.128 – 192.168.5.191.

---

## 4. Private vs Public IPs (RFC 1918)

The IETF reserved three ranges for "private" use — not routable on the public internet. Everyone uses them inside their own networks; **NAT** translates them to public IPs at the edge.

| Range | CIDR | Size | Typical use |
| --- | --- | --- | --- |
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | 16.7M | Big enterprise, cloud VPCs |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | 1M | Docker default |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | 65K | Home routers |

Other reserved blocks worth knowing:
- `127.0.0.0/8` — loopback (`127.0.0.1` = localhost).
- `169.254.0.0/16` — link-local (no DHCP fell back to this; AWS instance metadata at `169.254.169.254`).
- `224.0.0.0/4` — multicast.
- `100.64.0.0/10` — Carrier-Grade NAT (mobile networks).
- `0.0.0.0/0` — "everything" (default route).
- `255.255.255.255` — limited broadcast.

---

## 5. NAT: How Private IPs Reach the Public Internet

Your laptop has IP `192.168.1.42`. The internet has no idea what that means. **NAT** (Network Address Translation) on your router rewrites the source IP+port on the way out, and reverses it on the way in.

```
Laptop                  Router                       Internet
192.168.1.42:5500 ──►  rewrites src ──► 203.0.113.10:35012 ──► server
                       ◄── rewrites dst back ──────────────
```

NAT keeps a translation table mapping `internal_ip:port ↔ external_ip:port` so the return packet finds the right device.

Consequences:
- **You can have millions of devices behind one public IP** (this is why we haven't fully run out of IPv4).
- **Direct inbound connections from the internet don't work** without port forwarding — you have to initiate from inside.
- **Long-lived connections die** when the NAT table entry times out. WebSocket keepalives exist for this.
- **NAT traversal** for peer-to-peer (WebRTC, gaming) is non-trivial — STUN, TURN, ICE.

---

## 6. Special Addresses Inside a Subnet

For any subnet `X/Y`:
- **Network address** (all host bits 0) — the name of the subnet itself; *not assignable* to a host.
- **Broadcast address** (all host bits 1) — sends to every host in the subnet; *not assignable*.
- **Everything in between** — usable host addresses.

Hence "usable hosts = total – 2" (except `/31` and `/32`, where modern standards allow both addresses to be used).

Example for `10.0.5.0/24`:
- Network: `10.0.5.0`
- Broadcast: `10.0.5.255`
- Usable: `10.0.5.1` – `10.0.5.254` (254 addresses)

---

## 7. Subnetting in a Cloud VPC (the practical view)

You'll mostly meet CIDR when designing a VPC.

```
VPC:           10.0.0.0/16   (65,536 IPs)
 ├─ Subnet A:  10.0.0.0/20   (4,096 IPs)  — us-east-1a, public
 ├─ Subnet B:  10.0.16.0/20  (4,096 IPs)  — us-east-1b, public
 ├─ Subnet C:  10.0.32.0/20  (4,096 IPs)  — us-east-1a, private
 ├─ Subnet D:  10.0.48.0/20  (4,096 IPs)  — us-east-1b, private
 └─ Subnet E:  10.0.64.0/20  (4,096 IPs)  — DB tier
```

Rules of thumb:
- VPC `/16` is enough for ~65k addresses — usually fine.
- Subnets typically `/20`–`/24` per AZ-tier combination.
- AWS reserves **5 IPs in each subnet** (network, broadcast, router, DNS, future-use).
- Subnets cannot overlap within a VPC.
- VPCs that you want to peer cannot overlap each other.

> **Senior trap:** if you pick the same CIDR (e.g., `10.0.0.0/16`) for two VPCs and later need them to peer, you have a painful renumbering ahead. Plan globally.

---

## 8. Routing 101

A **routing table** is a list of CIDRs paired with a "next hop":

```
Destination            Next hop
10.0.0.0/16            local
10.1.0.0/16            VPC peering connection X
0.0.0.0/0              internet gateway
```

**Longest-prefix match** wins: the most specific (longest prefix) entry that matches the destination IP is used. `10.0.1.5` matches `10.0.0.0/16` (length 16); it would also match `10.0.0.0/8` (length 8); the /16 wins.

This is *the* mental model for any routing decision. Cloud route tables, Linux `ip route`, BGP — all the same rule.

---

## 9. CIDR Tricks Engineers Use Daily

### Match individual host
`192.168.5.42/32` — exactly one IP. Used in firewall rules and ALLOW/DENY lists.

### Match the whole internet
`0.0.0.0/0` — used in route tables to mean "default route" and in security groups to mean "anywhere".

### Combine two subnets
Two adjacent `/24`s combine into one `/23`. Useful for shrinking firewall rules.
```
192.168.0.0/24 + 192.168.1.0/24  →  192.168.0.0/23
```

### Splitting a subnet
A `/24` can be split into two `/25`s:
```
10.0.0.0/24  →  10.0.0.0/25  (.0 – .127)
                10.0.0.128/25 (.128 – .255)
```

### Useful command-line checks
```bash
# What network does this IP belong to?
ipcalc 192.168.5.130/26

# Check subnet overlap
ipcalc 10.0.0.0/20 10.0.16.0/20

# Sanity check from netaddr (Python)
python3 -c "import ipaddress; print(ipaddress.ip_network('10.0.0.0/22'))"
```

---

## 10. IPv6 — The Short Tour

Eventually you'll deal with it.

### Format
- 128 bits, 8 groups of 4 hex digits.
- `2001:db8::1` (`::` means "the missing groups are all zero").
- One `::` per address, max.

### Prefix sizes (different intuition from v4)
- `/64` — the *standard* subnet size for an end-user network. Yes, 64 bits of host space (10^19 addresses) per network.
- `/56` or `/48` — common allocation to a customer site/enterprise.
- No NAT in IPv6 culture (mostly); every device gets a globally routable address.

### Special addresses
- `::1` — IPv6 loopback (= 127.0.0.1).
- `fe80::/10` — link-local.
- `fc00::/7` — "Unique Local Addresses" (private equivalent).
- `2000::/3` — current global unicast space.

---

## 11. Worked Examples

### Example 1 — VPC planning
You need a VPC supporting 3 AZs × 2 tiers (public, private) = 6 subnets. Each tier should hold ~1,000 instances.

- VPC: `10.10.0.0/16` (65k IPs — plenty).
- Subnets: `/22` each → 1,022 usable hosts.
- Layout:
  ```
  10.10.0.0/22   public  az-a
  10.10.4.0/22   public  az-b
  10.10.8.0/22   public  az-c
  10.10.12.0/22  private az-a
  10.10.16.0/22  private az-b
  10.10.20.0/22  private az-c
  ```

### Example 2 — Firewall rule
"Allow all internal Kubernetes pods (`10.244.0.0/16`) to reach the database, but no public IPs."
```
ALLOW src 10.244.0.0/16 dst <db> tcp 5432
DENY  src 0.0.0.0/0     dst <db>
```

### Example 3 — Pod CIDR collision
Your VPC is `10.0.0.0/16`. Your EKS cluster's default pod CIDR is `10.100.0.0/16`. Fine. But if you launch a *second* cluster with the same defaults, those clusters can never talk to each other directly. Choose distinct ranges up front.

---

## 12. Common Mistakes

- **Allocating /24s and running out.** Cloud apps grow fast — start with /16 VPCs at minimum.
- **Forgetting cloud reservations.** AWS reserves 5 IPs per subnet; GCP 4; don't size to the bone.
- **Overlapping VPCs.** Future-you will be furious when peering can't happen.
- **Confusing /24 with /16.** Off-by-8 errors leak across the entire range.
- **Hardcoding IPs in app config.** Always use DNS, service discovery, or env vars.
- **Trusting client IPs.** Behind NAT/CDN/proxies, the IP you see is the *last hop*. Use `X-Forwarded-For` (and validate it).
- **Subnet masks ≠ MTU ≠ NAT type ≠ firewall rules.** Different concepts; people confuse them.

---

## 13. Quick-Reference Card

```
IPv4:    32 bits, dotted decimal       e.g., 10.0.5.42
IPv6:    128 bits, hex groups          e.g., 2001:db8::1

CIDR:    address/prefix-length         e.g., 10.0.0.0/16
         prefix length = network bits  remaining = host bits

ANCHORS:  /32  →     1                 /24 →   256 (254 usable)
          /28  →    16                 /16 → 65,536 (65,534 usable)
          /22  → 1,024                 /8  → 16,777,216
          Each step DOWN in prefix DOUBLES the size.

PRIVATE RANGES (RFC 1918)
  10.0.0.0/8     172.16.0.0/12    192.168.0.0/16
  Loopback: 127.0.0.0/8           Link-local: 169.254.0.0/16
  AWS metadata: 169.254.169.254   "anywhere": 0.0.0.0/0

ROUTING:  longest-prefix match wins.
SUBNET:   network addr (all-0 hosts) + broadcast (all-1 hosts) not usable.
NAT:      one public IP can hide thousands of private IPs.
```

---

## 14. Resources

### Foundational
- *Computer Networks* — Tanenbaum & Wetherall, Ch. 5 (Network Layer).
- *TCP/IP Illustrated Vol. 1* — W. Richard Stevens.
- **RFC 4632** — Classless Inter-Domain Routing (CIDR): <https://datatracker.ietf.org/doc/html/rfc4632>
- **RFC 1918** — Private Address Space: <https://datatracker.ietf.org/doc/html/rfc1918>

### Tools
- `ipcalc` — CLI subnet calculator.
- `sipcalc` — alternative CLI.
- **subnet-calculator.com** — visual GUI.
- **cidr.xyz** — beautiful interactive CIDR explorer: <https://cidr.xyz/>
- Python's `ipaddress` module — built-in, very clean: <https://docs.python.org/3/library/ipaddress.html>

### Practice
- **subnetting.org / subnetipv4.com** — drill problems till you can do them in your head.
- **Cisco subnetting practice** — searchable.

### Articles
- Cloudflare Learning — "What is an IP address?", "What is CIDR?": <https://www.cloudflare.com/learning/network-layer/what-is-an-ip-address/>
- AWS VPC docs — practical CIDR planning: <https://docs.aws.amazon.com/vpc/latest/userguide/configure-your-vpc.html>
- Julia Evans' "Networking Acronyms" zine.

### Videos
- PowerCert Animated Videos — "Subnetting Mastery": <https://www.youtube.com/@PowerCertAnimatedVideos>
- Practical Networking — full subnetting series on YouTube.

---

*Previous:* [← OSI Model & TCP/IP Stack](./osi-tcp-ip.md)  |  *Next:* [TCP vs UDP →](./tcp-vs-udp.md)
