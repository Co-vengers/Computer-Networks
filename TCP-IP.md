# TCP/IP Networking Protocols — Complete Reference

A comprehensive guide to the **Transmission Control Protocol (TCP)** and **Internet Protocol (IP)** — the two foundational protocols of modern networking.

---

## Table of Contents

1. [TCP — Transmission Control Protocol](#1-tcp--transmission-control-protocol)
   - [What is TCP?](#what-is-tcp)
   - [Core Properties](#core-properties-of-tcp)
   - [TCP Segment Structure](#tcp-segment-structure)
   - [Three-Way Handshake](#the-three-way-handshake)
   - [Reliable Data Transfer](#reliable-data-transfer-sequence-numbers--acks)
   - [Flow Control](#flow-control-the-sliding-window)
   - [Congestion Control](#congestion-control)
   - [Connection Teardown](#connection-teardown-four-way-handshake)
   - [TCP States](#tcp-states)
   - [Key Mechanisms Summary](#key-mechanisms-summary)
   - [TCP vs UDP](#tcp-vs-udp)
   - [TCP in Modern Context](#tcp-in-modern-context)

2. [IP — Internet Protocol](#2-ip--internet-protocol)
   - [What is IP?](#what-is-ip)
   - [IPv4 Packet Structure](#ipv4-packet-structure)
   - [IPv4 Addressing](#ipv4-addressing)
   - [Subnetting](#subnetting)
   - [Special & Private Addresses](#special-and-private-address-ranges)
   - [Fragmentation](#fragmentation)
   - [How Routing Works](#how-routing-works)
   - [NAT](#nat--network-address-translation)
   - [ICMP](#icmp--internet-control-message-protocol)
   - [IPv6](#ipv6--the-successor)
   - [ARP](#arp--the-missing-link)
   - [The Full Stack in Action](#the-ip-stack-in-action--putting-it-all-together)
   - [IPv4 vs IPv6 Comparison](#ipv4-vs-ipv6--side-by-side)

---

## 1. TCP — Transmission Control Protocol

### What is TCP?

**Transmission Control Protocol (TCP)** is a **connection-oriented, reliable, ordered, and error-checked** transport-layer protocol (Layer 4 of the OSI model). It sits on top of IP and is responsible for ensuring data gets from point A to point B — completely, in order, and without corruption.

TCP is used by: HTTP/HTTPS, SSH, FTP, SMTP/IMAP, database connections — any application where data integrity matters.

---

### Core Properties of TCP

| Property | Description |
|---|---|
| Connection-oriented | Requires a handshake before data exchange |
| Reliable delivery | Every byte is acknowledged; unacknowledged data is retransmitted |
| Ordered delivery | Sequence numbers ensure correct reassembly even if packets arrive out of order |
| Error detection | Each segment carries a checksum; corrupted segments are discarded and retransmitted |
| Flow control | Prevents a fast sender from overwhelming a slow receiver (receive window) |
| Congestion control | Detects network congestion and backs off intelligently |

---

### TCP Segment Structure

Each TCP segment has a **header (minimum 20 bytes)** followed by a payload.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Offset | Rsv |   Flags (SYN ACK FIN RST PSH URG)  |   Window  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if Offset > 5)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Data (payload)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields:**

- **Sequence Number** — byte offset of the first byte in this segment within the overall stream
- **ACK Number** — the next byte the receiver expects (cumulative acknowledgment)
- **Flags** — single-bit control signals:
  - `SYN` — synchronize (connection setup)
  - `ACK` — acknowledge
  - `FIN` — finish (connection teardown)
  - `RST` — reset (abort connection)
  - `PSH` — push data to application immediately
  - `URG` — urgent pointer is valid
- **Receive Window** — how much buffer space the receiver has (flow control)
- **Checksum** — 16-bit error detection over header + data

---

### The Three-Way Handshake

Before any data flows, TCP establishes a connection by synchronizing sequence numbers on both sides.

```
Client                          Server
  |                               |
  |------- SYN (seq=x) ---------->|   Client: SYN-SENT
  |                               |   Server: SYN-RCVD
  |<------ SYN-ACK (seq=y,ack=x+1)|
  |                               |
  |------- ACK (ack=y+1) -------->|   Both: ESTABLISHED
  |                               |
```

1. **SYN** — Client sends SYN with its Initial Sequence Number (ISN = x), randomly chosen
2. **SYN-ACK** — Server acknowledges client's ISN (`ack = x+1`) and announces its own ISN (`seq = y`)
3. **ACK** — Client acknowledges server's ISN (`ack = y+1`). Connection established on both ends.

> The ISN is random to prevent conflicts with old connections and as a basic security measure against blind injection attacks.

---

### Reliable Data Transfer: Sequence Numbers & ACKs

Every byte of data has a position in the stream. The sender labels segments with their byte offset; the receiver replies with the next byte it expects.

```
Sender                              Receiver
  |---seq=1, len=1000 (bytes 1-1000)-->|
  |<-------------- ack=1001 -----------|  ✓ received
  |---seq=1001, len=1000 ------------->|
  |<-------------- ack=2001 -----------|  ✓ received
  |---seq=2001 (LOST) ~~~~~~~~~~~~~~~X |
  |<-------------- ack=2001 -----------|  still waiting
  |---RETRANSMIT seq=2001 ------------>|  retransmit after timeout / 3× dup-ACK
```

**Loss detection mechanisms:**

- **Timeout (RTO)** — if ACK doesn't arrive within the Retransmission Timeout, segment is resent. RTO is dynamically computed using Smoothed Round-Trip Time (SRTT).
- **Triple Duplicate ACK (Fast Retransmit)** — if the receiver gets 3 segments but is missing one in the middle, it keeps ACKing the last good byte. Three identical consecutive ACKs = "resend immediately", no waiting for timeout.

---

### Flow Control: The Sliding Window

TCP uses a **sliding window** to prevent a fast sender from overwhelming a slow receiver.

- The receiver advertises its available buffer space in the **receive window** field of every ACK.
- The sender may have at most `window size` bytes outstanding (unacknowledged) at any time.
- As ACKs arrive, the window slides forward — new data can be sent.
- If the receiver's buffer fills: `window = 0` — sender pauses entirely.
- A **Zero Window Probe** (1-byte segment) is sent periodically to check if the window has reopened.

---

### Congestion Control

TCP infers network congestion from packet loss and adjusts its **congestion window (cwnd)** accordingly. Four interacting algorithms:

#### 1. Slow Start
- Begin with `cwnd = 1 MSS` (Maximum Segment Size, typically ~1460 bytes on Ethernet)
- Double `cwnd` for every ACK received → exponential growth
- Continues until `cwnd` reaches `ssthresh` (slow-start threshold)

#### 2. Congestion Avoidance
- Once `cwnd >= ssthresh`: increase by 1 MSS per RTT (linear growth)
- Probes for available bandwidth carefully

#### 3. Fast Retransmit
- On triple duplicate ACK: retransmit the missing segment immediately (don't wait for timeout)

#### 4. Fast Recovery (TCP Reno / CUBIC)
- After fast retransmit: `ssthresh = cwnd / 2`, `cwnd = ssthresh`
- Resume linear growth (don't go back to slow start)
- On timeout: `cwnd = 1 MSS`, restart slow start from scratch

```
cwnd
 ^
 |    /|
 |   / |         /|
 |  /  |        / |      /
 | /   |  ___/ |     ___/
 |/    | /     |    /
 +-----|---------|--------> RTT
   Slow  Cong.   After   After
   Start  Avoid  dup-ACK timeout
```

The sawtooth pattern — exponential growth, sharp drop on loss, gradual climb — is TCP probing for bandwidth continuously.

---

### Connection Teardown: Four-Way Handshake

Each side closes independently (half-close).

```
Initiator                       Responder
  |------- FIN ------------------>|   Initiator: FIN-WAIT-1
  |<------ ACK -------------------|   Initiator: FIN-WAIT-2
  |                               |   (Responder can still send data)
  |<------ FIN -------------------|   Responder: LAST-ACK
  |------- ACK ------------------>|   Initiator: TIME-WAIT (2×MSL)
  |                               |   → CLOSED
```

- **TIME-WAIT** lasts 2×MSL (Maximum Segment Lifetime, typically ~60s). This ensures the final ACK is retransmitted if lost, and prevents old duplicate segments from confusing new connections.

---

### TCP States

```
CLOSED → LISTEN → SYN-RCVD → ESTABLISHED
                                    |
                              FIN-WAIT-1 → FIN-WAIT-2 → TIME-WAIT → CLOSED
                              (active close)

ESTABLISHED → CLOSE-WAIT → LAST-ACK → CLOSED
(passive close)
```

View live states on Linux/macOS:
```bash
ss -tn
# or
netstat -tn
```

---

### Key Mechanisms Summary

| Mechanism | Problem Solved | How |
|---|---|---|
| Three-way handshake | Connection setup, ISN sync | SYN → SYN-ACK → ACK |
| Sequence numbers | Ordering, duplicate detection | Byte-level numbering of stream |
| ACKs + RTO | Reliability (loss recovery) | Retransmit on timeout |
| Fast retransmit | Faster loss recovery | Triple dup-ACK trigger |
| Receive window | Flow control | Receiver limits sender |
| cwnd + slow start | Congestion control | Exponential then linear growth |
| Checksum | Error detection | 16-bit over header + data |
| TIME-WAIT | Stale segment cleanup | Wait 2×MSL after close |

---

### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | Ordered | No ordering |
| Flow control | Yes | No |
| Congestion control | Yes | No |
| Overhead | Higher (20-byte header min) | Lower (8-byte header) |
| Use cases | HTTP, SSH, FTP, DB | DNS, video streaming, gaming, VoIP |

> **Modern note:** QUIC (HTTP/3) runs over UDP but implements TCP-like reliability in user space, enabling faster evolution and better performance for web traffic.

---

### TCP in Modern Context

**`TCP_NODELAY`** — disables Nagle's algorithm (which batches small writes). Use for interactive apps (SSH, games) where latency matters more than bandwidth efficiency.

**`TCP_KEEPALIVE`** — sends periodic probes on idle connections to detect dead peers. Critical for long-lived connections (database pools, SSH tunnels).

**MSS (Max Segment Size)** — negotiated in SYN handshake. Typical: 1460 bytes (1500 Ethernet MTU − 20 IP − 20 TCP headers).

**SACK (Selective ACK)** — extension allowing the receiver to acknowledge non-contiguous byte ranges. Makes loss recovery far more precise than basic cumulative ACKs. Negotiate via TCP options in handshake.

**ECN (Explicit Congestion Notification)** — routers mark packets instead of dropping them to signal congestion. TCP backs off before loss occurs. Requires support from both endpoints and network.

---

## 2. IP — Internet Protocol

### What is IP?

The **Internet Protocol** operates at **Layer 3 (Network layer)** of the OSI model. Its job: take a chunk of data, wrap it in a header with source and destination addresses, and forward it one hop at a time toward the destination.

IP is:
- **Connectionless** — no handshake; each packet is independent
- **Best-effort** — no delivery guarantees, no ordering, no error recovery
- **Unreliable** — reliability is TCP's job; IP just routes

Two versions in active use: **IPv4** (32-bit, exhausted) and **IPv6** (128-bit, the future).

---

### IPv4 Packet Structure

The IPv4 header is **minimum 20 bytes** (5 × 32-bit rows), up to 60 bytes with options.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP   |ECN|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source IP Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination IP Address                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if IHL > 5)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Data (Payload)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Field descriptions:**

| Field | Bits | Description |
|---|---|---|
| Version | 4 | `4` for IPv4, `6` for IPv6 |
| IHL | 4 | Header length in 32-bit words (min 5 = 20 bytes) |
| DSCP | 6 | Differentiated Services — QoS marking |
| ECN | 2 | Explicit Congestion Notification |
| Total Length | 16 | Entire packet size in bytes (max 65,535) |
| Identification | 16 | Groups fragments of the same original packet |
| Flags | 3 | DF (Don't Fragment), MF (More Fragments) |
| Fragment Offset | 13 | Position of this fragment in the original packet |
| TTL | 8 | Decremented by 1 at each router; packet dropped at 0 |
| Protocol | 8 | Content type: TCP=6, UDP=17, ICMP=1, OSPF=89 |
| Header Checksum | 16 | Covers IP header only; recomputed at every hop |
| Source Address | 32 | Sender's IPv4 address |
| Destination Address | 32 | Recipient's IPv4 address |

---

### IPv4 Addressing

An IPv4 address is **32 bits**, written as four decimal octets separated by dots:

```
192  .  168  .   1   .  10
11000000.10101000.00000001.00001010
```

Total address space: 2³² = **~4.3 billion addresses** (now exhausted).

#### Classful Addressing (historical)

| Class | Range | Default Prefix | Hosts |
|---|---|---|---|
| A | 0.0.0.0 – 127.255.255.255 | /8 | ~16.7 million |
| B | 128.0.0.0 – 191.255.255.255 | /16 | ~65,534 |
| C | 192.0.0.0 – 223.255.255.255 | /24 | 254 |
| D | 224.0.0.0 – 239.255.255.255 | — | Multicast |
| E | 240.0.0.0 – 255.255.255.255 | — | Reserved |

#### CIDR — Classless Inter-Domain Routing

Replaced classful addressing in 1993. Allows arbitrary prefix lengths:

```
192.168.10.0/22   → 1022 usable hosts
10.0.0.0/8        → 16,777,214 usable hosts
203.0.113.0/30    → 2 usable hosts (point-to-point link)
```

The `/n` notation (CIDR notation) means the first `n` bits are the **network prefix**; the remaining bits identify hosts.

---

### Subnetting

A subnet mask divides an IP address into **network prefix** and **host** portions.

**Example: `192.168.10.50/26`**

```
IP:   192.168.10.50  =  11000000.10101000.00001010.00|110010
Mask: 255.255.255.192 =  11111111.11111111.11111111.11|000000
                                              26 bits  ^  6 bits
                                              (network)   (host)
```

| Value | Address |
|---|---|
| Network address | `192.168.10.0` |
| Broadcast address | `192.168.10.63` |
| Usable host range | `192.168.10.1` – `192.168.10.62` |
| Number of hosts | **62** (2⁶ − 2) |

**Subnet formula:**
- Hosts per subnet = 2^(host bits) − 2
- Number of subnets = 2^(borrowed bits)

---

### Special and Private Address Ranges

| Range | Purpose |
|---|---|
| `10.0.0.0/8` | Private — Class A (RFC 1918) |
| `172.16.0.0/12` | Private — Class B (RFC 1918) |
| `192.168.0.0/16` | Private — Class C (RFC 1918) |
| `127.0.0.0/8` | Loopback (`127.0.0.1` = localhost) |
| `169.254.0.0/16` | Link-local / APIPA (when DHCP fails) |
| `0.0.0.0/8` | "This network" — unroutable |
| `255.255.255.255` | Limited broadcast |
| `224.0.0.0/4` | Multicast (Class D) |
| `100.64.0.0/10` | Carrier-grade NAT (RFC 6598) |

Private addresses are not routed on the public internet. NAT translates them to a public IP at the network boundary.

---

### Fragmentation

When a packet exceeds the link's **MTU (Maximum Transmission Unit)** — typically **1500 bytes on Ethernet** — IP fragments it.

Each fragment becomes its own independent packet sharing the same **Identification** field. The destination reassembles them using:
- **Identification** — which original packet these fragments belong to
- **Fragment Offset** — byte position in the original
- **MF flag** — set on all fragments except the last

**Flags:**
- `DF = 1` (Don't Fragment) — if the packet is too big, drop it and send ICMP "Fragmentation Needed" back
- `MF = 1` (More Fragments) — more fragments follow
- `MF = 0` + offset > 0 — this is the last fragment

**Path MTU Discovery (PMTUD):** The preferred modern approach. The source sends DF=1 packets; if a router can't forward them, it returns ICMP Type 3 Code 4. The source reduces its packet size accordingly. Avoids fragmentation entirely.

> Fragmentation is expensive and makes filtering/inspection harder. Most modern protocols avoid it via PMTUD.

---

### How Routing Works

IP routing is **hop-by-hop** and **stateless**. Each router makes an independent forwarding decision based only on the destination address.

```
Your PC          Home Router       ISP Router       Backbone        Server
192.168.1.5  →  NAT Gateway   →   BGP peer    →   Tier-1 AS   →  8.8.8.8
              (hop 1, TTL 63)   (hop 2, TTL 62)  (hop 3, TTL 61) (hop 4, TTL 60)
```

At each router:
1. Read destination IP from packet header
2. Look up in routing table — **longest prefix match wins**
3. Decrement TTL by 1; if TTL = 0, drop packet and send ICMP Time Exceeded
4. Recalculate header checksum (TTL changed)
5. Use ARP to find next-hop's MAC address
6. Forward packet on the outgoing interface

#### Longest Prefix Match

```
Routing table:
0.0.0.0/0        → 10.0.0.1    (default route)
192.168.0.0/16   → 192.168.1.1
192.168.10.0/24  → 192.168.10.1  ← wins for 192.168.10.50

Packet to 192.168.10.50:
  Matches /16  (16 bits match)
  Matches /24  (24 bits match) ← more specific = chosen
```

#### Routing Protocols

**Interior Gateway Protocols (within an Autonomous System):**

| Protocol | Type | Algorithm | Notes |
|---|---|---|---|
| OSPF | Link-state | Dijkstra (SPF) | Fast convergence, full topology knowledge |
| RIP | Distance-vector | Bellman-Ford | Old, max 15 hops |
| EIGRP | Hybrid | DUAL | Cisco proprietary |
| IS-IS | Link-state | Dijkstra | Used by large ISPs |

**Exterior Gateway Protocol (between Autonomous Systems):**

| Protocol | Use |
|---|---|
| BGP-4 | The internet's routing protocol. Policy-based, not just shortest-path. Used by ISPs, cloud providers, enterprises. |

---

### NAT — Network Address Translation

Since IPv4 addresses are exhausted, most networks use **private addresses** internally with NAT at the border:

```
Inside (private)              NAT Router (public)         Outside
192.168.1.5:54321  ──────→  203.0.113.1:54321  ──────→  8.8.8.8:53
                     rewrite src IP+port
← 192.168.1.5:54321 ←────  203.0.113.1:54321  ←──────  8.8.8.8:53
                    reverse mapping from table
```

**NAT table entry:**
```
Private IP:Port       Public IP:Port        Remote IP:Port
192.168.1.5:54321  ↔  203.0.113.1:54321  ↔  8.8.8.8:53
```

NAT broke the **end-to-end principle** (any host should reach any other host directly) but extended IPv4's life by ~20 years. IPv6 eliminates the need for NAT.

---

### ICMP — Internet Control Message Protocol

ICMP (Protocol = 1) carries **error messages and diagnostics**. It rides inside IP packets but is integral to IP's operation.

**Common ICMP messages:**

| Type | Code | Meaning |
|---|---|---|
| 0 | 0 | Echo Reply (ping response) |
| 3 | 0 | Destination network unreachable |
| 3 | 1 | Destination host unreachable |
| 3 | 3 | Destination port unreachable |
| 3 | 4 | Fragmentation needed, DF set |
| 8 | 0 | Echo Request (ping) |
| 11 | 0 | TTL exceeded in transit (traceroute) |
| 12 | 0 | Parameter problem (malformed header) |

**`ping`** sends ICMP Echo Requests and measures round-trip time:
```bash
ping 8.8.8.8
# ICMP seq, TTL, and RTT per reply
```

**`traceroute`** exploits TTL by sending packets with TTL=1, 2, 3…:
```bash
traceroute google.com
# Each router that drops a TTL-expired packet sends back ICMP Type 11
# → reveals each hop along the path
```

---

### IPv6 — The Successor

IPv6 uses **128-bit addresses** — 2¹²⁸ ≈ **340 undecillion** addresses.

Written as 8 groups of 4 hex digits:
```
Full:        2001:0db8:85a3:0000:0000:8a2e:0370:7334
Compressed:  2001:db8:85a3::8a2e:370:7334
             (:: = one or more consecutive groups of 0000)
```

#### IPv6 Fixed Header — Always 40 Bytes

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Source Address                          +
|                      (128 bits)                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                    Destination Address                        +
|                      (128 bits)                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Fields removed vs IPv4:** Header checksum · IHL · Identification · Flags · Fragment Offset · Options (replaced by extension headers)

**Key improvements:**
- **No header checksum** — link layer handles it; routers forward faster
- **No router fragmentation** — source only; PMTUD is mandatory
- **Flow Label** — routers can treat a flow's packets identically for hardware acceleration
- **Extension headers** — optional features (fragmentation, routing, IPsec auth) are chained, not baked in
- **Built-in IPsec** — encryption/authentication is first-class
- **No broadcast** — replaced by multicast; more efficient

#### IPv6 Address Types

| Type | Prefix | Description |
|---|---|---|
| Global unicast | `2000::/3` | Routable on the internet |
| Link-local | `fe80::/10` | Automatically configured, single link only |
| Unique local | `fc00::/7` | Private, like RFC 1918 for IPv4 |
| Loopback | `::1/128` | Equivalent to 127.0.0.1 |
| Multicast | `ff00::/8` | One-to-many delivery |
| Anycast | (no special prefix) | Routed to nearest of several nodes |

#### IPv6 Auto-Configuration

Unlike IPv4 which requires DHCP, IPv6 hosts can **self-configure** their address using:

- **SLAAC (Stateless Address Autoconfiguration)** — combines the `fe80::/64` prefix (from router advertisements) with the interface's EUI-64 MAC-derived identifier
- **DHCPv6** — stateful configuration (still available when needed)

---

### ARP — The Missing Link

IP addresses are **logical**. To actually transmit a frame on Ethernet, you need the **MAC address** of the next hop. **ARP (Address Resolution Protocol)** resolves this mapping.

```
Sender                            LAN                         Target
  |                                |                            |
  |-- ARP Request (broadcast) ---→ |→→→→→→→→→→→→→→→→→→→→→→→→→ |
  |   "Who has 192.168.1.1?"       |                            |
  |                                |                            |
  |←-- ARP Reply (unicast) ------- | ←- "I have it, MAC=AA:BB" |
  |                                |                            |
  | (caches in ARP table)          |                            |
```

Check the ARP table:
```bash
arp -n         # Linux/macOS
arp -a         # Windows
ip neigh show  # Linux (modern)
```

**IPv6 equivalent:** NDP (Neighbor Discovery Protocol) uses ICMPv6 multicast — more efficient than ARP's broadcast.

---

### The IP Stack in Action — Putting It All Together

What happens when you open `https://google.com`:

```
1. DNS:     google.com  →  142.250.80.46
2. TCP SYN: your OS creates a TCP segment (port 443)
3. IP:      wraps in packet: src=192.168.1.5, dst=142.250.80.46
4. ARP:     resolves home router's MAC address
5. Ethernet: frames the packet with router's MAC as destination
6. NAT:     router rewrites src IP to your public IP, forwards to ISP
7. Routing: BGP routes the packet across the internet
            ~10-20 routers: longest-prefix match, TTL--, checksum recalculate, forward
8. Arrival: Google's router strips IP header, passes TCP segment up
9. Total:   ~15-40ms, ~10-20 hops
```

---

### IPv4 vs IPv6 — Side by Side

| Feature | IPv4 | IPv6 |
|---|---|---|
| Address size | 32 bits | 128 bits |
| Address space | ~4.3 billion | ~340 undecillion |
| Header size | 20–60 bytes (variable) | 40 bytes (fixed) |
| Fragmentation | At routers + source | Source only (PMTUD mandatory) |
| Header checksum | Yes (recomputed per hop) | No |
| Broadcast | Yes | No — multicast replaces it |
| Auto-configuration | DHCP | SLAAC + DHCPv6 |
| NAT required | Usually yes | No |
| IPsec | Optional | Built-in |
| ARP | Yes | Replaced by NDP (ICMPv6) |
| Loop prevention | TTL | Hop Limit |
| QoS | DSCP (6 bits) | Traffic Class + Flow Label |

---

## Quick Reference

### Common Protocol Numbers (IP Protocol field)

| Number | Protocol |
|---|---|
| 1 | ICMP |
| 6 | TCP |
| 17 | UDP |
| 41 | IPv6 (tunneled in IPv4) |
| 47 | GRE |
| 50 | ESP (IPsec) |
| 51 | AH (IPsec) |
| 89 | OSPF |
| 132 | SCTP |

### Common Port Numbers (TCP/UDP)

| Port | Protocol |
|---|---|
| 22 | SSH |
| 25 | SMTP |
| 53 | DNS (UDP + TCP) |
| 80 | HTTP |
| 110 | POP3 |
| 143 | IMAP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |

### Useful Diagnostic Commands

```bash
# Ping (ICMP echo)
ping 8.8.8.8
ping6 2001:4860:4860::8888

# Traceroute (TTL manipulation)
traceroute google.com          # Linux/macOS
tracert google.com             # Windows

# Show active TCP connections
ss -tn                         # Linux (modern)
netstat -tn                    # cross-platform

# Show routing table
ip route show                  # Linux
route -n                       # Linux (old)
netstat -rn                    # macOS/Windows

# ARP table
ip neigh show                  # Linux
arp -n                         # Linux/macOS

# DNS lookup
dig google.com
nslookup google.com

# TCP handshake + HTTP (raw)
curl -v https://google.com

# Capture packets
tcpdump -i eth0 tcp port 80
tcpdump -i eth0 icmp
```

---

## Design Philosophy

> The internet's robustness comes from two principles working together:
>
> **IP is dumb by design.** Routers just forward packets. No state, no sessions, no reliability logic — just the destination address and a forwarding table.
>
> **Intelligence lives at the edges.** TCP, TLS, HTTP — all the complex behavior runs at the endpoints. The network doesn't need to know what application is running.
>
> This **end-to-end principle** is why the internet could grow from a research network to planetary infrastructure without redesigning the core. New applications — the web, video streaming, VoIP, cloud computing — ran on top of the same dumb IP fabric without requiring changes to routers.

---

*Generated from a detailed TCP/IP networking discussion covering protocol internals, packet structures, addressing, routing, and modern considerations.*
