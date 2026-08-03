# The OSI Model — A Practical Guide

> The OSI (Open Systems Interconnection) model is a 7-layer framework that describes how data moves from an application on one machine to an application on another. It is a **teaching and troubleshooting model**, not what literally runs on the wire — but the vocabulary is used everywhere in real engineering.

---

## Quick reference

| # | Layer | One-line job | Data unit (PDU) | Address used | Example protocols | Example devices |
|---|-------|--------------|-----------------|--------------|-------------------|-----------------|
| 7 | Application | Interface to the user's app | Data | URL / hostname | HTTP, DNS, SMTP, SSH, gRPC | API gateway, WAF |
| 6 | Presentation | Translate, encrypt, compress | Data | — | TLS/SSL, UTF-8, JPEG, gzip | TLS terminator |
| 5 | Session | Open, manage, close conversations | Data | — | NetBIOS, RPC, SOCKS | — |
| 4 | Transport | Reliability, ports, segmentation | Segment (TCP) / Datagram (UDP) | Port number | TCP, UDP, QUIC | L4 load balancer, firewall |
| 3 | Network | Route across networks | Packet | IP address | IP, ICMP, BGP, OSPF | Router, L3 switch |
| 2 | Data link | Move a frame to the next device | Frame | MAC address | Ethernet, ARP, PPP, 802.11 | Switch, bridge, NIC |
| 1 | Physical | Transmit raw bits | Bit | — | Ethernet PHY, DSL, Bluetooth | Cable, hub, repeater |

**Mnemonic (7 → 1):** *All People Seem To Need Data Processing*
**Mnemonic (1 → 7):** *Please Do Not Throw Sausage Pizza Away*

---

## Layer 1 — Physical

**Job:** Convert bits into a signal, and push that signal onto a physical medium.

This layer has no concept of an address, a message, or a destination. It knows only *"transmit a 1"* and *"transmit a 0."*

**Analogy:** The road, the truck, and the fuel. It has no idea what's in the boxes.

**What lives here**
- Copper cable (Cat5e/Cat6), fibre optic, coaxial
- Radio frequencies (Wi-Fi 2.4/5/6 GHz, Bluetooth, 5G)
- Voltage levels, light pulses, modulation schemes
- Connectors and pinouts (RJ45, SFP+, USB-C)
- Bit rate, signal timing, and synchronization

**Real-world touchpoints**
- Link speed negotiation (100 Mbps vs 1 Gbps vs 10 Gbps)
- Duplex mismatch (half vs full duplex) causing mysterious slowness
- Attenuation over long cable runs, EMI from nearby power lines
- Cable length limits (~100m for copper Ethernet)

**How it fails**
Unplugged or damaged cable, dead port, dead SFP module, bad NIC, weak Wi-Fi signal.

**How you debug it**
```bash
ip link show               # Is the interface even UP?
ethtool eth0               # Speed, duplex, link detected
cat /sys/class/net/eth0/carrier   # 1 = physical link present
```
If `carrier` is 0, nothing above Layer 1 matters. Start here.

---

## Layer 2 — Data link

**Job:** Deliver a frame **to the next physically-connected device**, and detect corruption.

Layer 2 operates only within a single local network segment. It has no concept of "the internet."

**Analogy:** The truck driver who only knows the next depot. He doesn't know or care about the final destination country.

**Key concepts**

*MAC address* — a 48-bit hardware identifier burned into the NIC, e.g. `a4:83:e7:1c:9d:0f`. Globally unique-ish, but only meaningful on the local segment.

*Frame* — the PDU. Structure:
```
[ Dst MAC (6B) | Src MAC (6B) | EtherType (2B) | Payload (46–1500B) | FCS (4B) ]
```

*ARP (Address Resolution Protocol)* — the bridge between L3 and L2. Given an IP, find the MAC:
```
Host broadcasts: "Who has 192.168.1.1? Tell 192.168.1.42"
Router replies:  "192.168.1.1 is at a4:83:e7:1c:9d:0f"
```

*MTU* — Maximum Transmission Unit, typically 1500 bytes for Ethernet. Anything bigger must be fragmented at L3 or rejected.

**Critical insight**
MAC addresses are **rewritten at every hop**. A packet crossing 15 routers has 15 different MAC address pairs but the same source and destination IP throughout.

**Sublayers**
- **LLC** (Logical Link Control) — multiplexes upper-layer protocols
- **MAC** (Media Access Control) — decides who gets to transmit and when (CSMA/CD on old Ethernet, CSMA/CA on Wi-Fi)

**Real-world touchpoints**
- Switches build a MAC address table and forward frames only to the correct port
- VLANs (802.1Q) tag frames to isolate traffic on shared hardware
- MTU mismatch causing packet loss on VPNs and overlay networks (a very common Kubernetes CNI issue — the overlay adds encapsulation overhead and pushes frames over 1500 bytes)
- ARP cache poisoning as an attack vector

**How you debug it**
```bash
ip neigh show              # ARP table
arping 192.168.1.1         # Is the neighbour reachable at L2?
ip link show eth0 | grep mtu
```

---

## Layer 3 — Network

**Job:** Get a packet from source network to destination network, across any number of intermediate networks.

This is the first layer that understands *"somewhere else on the internet."*

**Analogy:** The postal sorting office that reads the country and city on the envelope and decides which route the letter takes.

**Key concepts**

*IP address* — a logical, assignable address.
- IPv4: `93.184.216.34` (32-bit)
- IPv6: `2606:2800:220:1:248:1893:25c8:1946` (128-bit)

*Subnet mask / CIDR* — splits an IP into network portion and host portion. `192.168.1.0/24` means the first 24 bits identify the network, leaving 256 addresses.

*Routing table* — the decision engine. For every packet: is the destination in a directly-connected subnet? If yes, deliver locally. If no, forward to the appropriate next hop or the default gateway.

*TTL (Time To Live)* — decremented by every router. At zero, the packet is dropped and an ICMP "time exceeded" is sent back. This is exactly how `traceroute` maps a path: send packets with TTL 1, 2, 3... and record who complains.

*Fragmentation* — splitting a packet that exceeds the next link's MTU.

**Supporting protocols**
- **ICMP** — control and error messaging (`ping`, `traceroute`, "destination unreachable")
- **BGP** — how autonomous systems on the internet exchange routes
- **OSPF / IS-IS** — interior routing within one organization
- **NAT** — rewrites addresses so many private IPs can share one public IP

**Real-world touchpoints**
- VPC route tables, subnets, and CIDR planning in AWS
- Security groups and NACLs filtering by IP and port
- Pod CIDR ranges and service CIDR ranges in Kubernetes
- Asymmetric routing causing dropped connections through stateful firewalls

**How you debug it**
```bash
ip route show              # Where do packets go?
ping 8.8.8.8               # Is L3 reachability working at all?
traceroute 8.8.8.8         # Where does the path break?
mtr 8.8.8.8                # traceroute + continuous packet loss stats
```
If `ping 8.8.8.8` works but `ping google.com` fails, your problem is DNS (L7), not routing.

---

## Layer 4 — Transport

**Job:** Deliver data to the correct **process** on the destination host, with the requested reliability guarantees.

Layer 3 gets you to the right machine. Layer 4 gets you to the right *program* on that machine.

**Analogy:** The mailroom inside an office building. The building address got the letter here; the room number gets it to the right person.

**Key concepts**

*Ports* — 16-bit numbers identifying a process. `0–1023` are well-known (80 HTTP, 443 HTTPS, 22 SSH, 53 DNS, 5432 Postgres, 6379 Redis). Ephemeral ports (typically 32768–60999 on Linux) are assigned to client connections.

*Socket* — the 5-tuple that uniquely identifies a connection:
```
(protocol, src IP, src port, dst IP, dst port)
```

### TCP — reliable, ordered, connection-oriented

**Three-way handshake:**
```
Client ──── SYN (seq=x) ────────► Server
Client ◄─── SYN-ACK (seq=y, ack=x+1) ─ Server
Client ──── ACK (ack=y+1) ──────► Server
        [ connection established ]
```

**Four-way teardown:** `FIN` → `ACK` → `FIN` → `ACK`

**What TCP guarantees**
- **Ordering** — sequence numbers let the receiver reassemble out-of-order segments
- **Reliability** — unacknowledged segments are retransmitted
- **Flow control** — the receive window stops a fast sender from drowning a slow receiver
- **Congestion control** — slow start, congestion avoidance, and backoff prevent network collapse

**Cost:** handshake latency (1 RTT before any data), head-of-line blocking, and connection state on both ends.

### UDP — fast, connectionless, no guarantees

Send and forget. No handshake, no ordering, no retransmission, ~8-byte header vs TCP's 20+.

**Use when** loss is more acceptable than delay: DNS, video/voice streaming, gaming, metrics, DHCP.

### QUIC

Runs over UDP but rebuilds reliability, ordering, and TLS 1.3 into a single protocol — eliminating TCP's head-of-line blocking. This is what HTTP/3 uses.

**Real-world touchpoints**
- **L4 load balancers** (AWS NLB) forward TCP connections without decrypting or inspecting content. Fast, protocol-agnostic, cannot route by URL path.
- Connection pool exhaustion in a database driver is an L4 resource problem
- `TIME_WAIT` socket buildup on high-throughput servers
- Firewall rules almost always express themselves as "protocol + port" — pure L4

**How you debug it**
```bash
ss -tulpn                        # What's listening on which port?
ss -s                            # Socket summary by state
nc -zv example.com 443           # Can I open a TCP connection?
tcpdump -i any port 443 -nn      # Watch the handshake happen
```
If `ping` works but `nc -zv host 443` hangs, L3 is fine and something is blocking at L4 — usually a firewall or security group.

---

## Layer 5 — Session

**Job:** Establish, manage, synchronize, and gracefully terminate a logical conversation between two applications.

This is the vaguest layer in practice. In the modern TCP/IP world, most of its responsibilities were absorbed into L4 (TCP) and L7 (application protocols).

**Analogy:** The rules of a phone call — dialling, confirming both parties are present, taking turns, and hanging up properly.

**Responsibilities on paper**
- Session establishment and teardown
- Dialogue control (full-duplex vs half-duplex)
- **Checkpointing** — inserting markers in a long transfer so an interrupted job resumes from the last checkpoint rather than restarting
- Session recovery after a network interruption

**Where you actually see it today**
- HTTP cookies and session IDs
- HTTP keep-alive and connection reuse
- WebSocket connections held open across many messages
- Database connection pooling
- RPC frameworks managing call lifecycle
- SSH sessions and SOCKS proxies
- OAuth tokens holding authenticated state across requests

**How it fails**
Session timeouts, stale connections in a pool, WebSocket disconnects that the client never notices, sticky-session misconfiguration behind a load balancer.

---

## Layer 6 — Presentation

**Job:** Make sure the data is in a form the receiving application can actually understand. Sometimes called the **syntax layer**.

**Analogy:** The translator and the sealed envelope. You write in your language; the translator converts it, then locks it in a tamper-proof pouch.

**Three responsibilities**

**1. Translation / serialization**
- Character encoding: UTF-8, ASCII, ISO-8859-1
- Data formats: JSON, XML, Protocol Buffers
- Media formats: JPEG, PNG, MP4, GIF
- Endianness conversion between architectures

**2. Encryption**
- TLS/SSL sits here conceptually — it encrypts the payload before it reaches the transport layer
- The TLS handshake negotiates protocol version, cipher suite, and validates certificates
- After the handshake, everything above is plaintext to the app but ciphertext on the wire

**3. Compression**
- gzip, brotli, deflate — reduce payload size before transmission

**Real-world touchpoints**
- Certificate expiry breaking production at 3am
- TLS version mismatch (a client stuck on TLS 1.0 against a server requiring 1.2+)
- SNI (Server Name Indication) letting one IP serve many HTTPS hostnames
- TLS termination at a load balancer, so backends receive plaintext HTTP
- Mojibake — garbled text from an encoding mismatch

**How you debug it**
```bash
openssl s_client -connect example.com:443 -servername example.com
curl -vI https://example.com        # Shows TLS handshake details
```

---

## Layer 7 — Application

**Job:** Provide the interface that the user's application actually speaks.

Important distinction: **Layer 7 is not the application itself.** Chrome is not Layer 7. HTTP — the protocol Chrome speaks — is Layer 7.

**Analogy:** The letter itself, and the shared language you and your friend both read.

**Common protocols**

| Protocol | Purpose | Default port |
|----------|---------|--------------|
| HTTP / HTTPS | Web | 80 / 443 |
| DNS | Name → IP resolution | 53 |
| SMTP / IMAP / POP3 | Email send / retrieve | 25 / 143 / 110 |
| FTP / SFTP | File transfer | 21 / 22 |
| SSH | Remote shell | 22 |
| DHCP | Automatic IP assignment | 67 / 68 |
| NTP | Clock synchronization | 123 |
| MQTT | IoT messaging | 1883 |
| gRPC | RPC over HTTP/2 | varies |

**Real-world touchpoints**
- **L7 load balancers** (AWS ALB, nginx, Kubernetes Ingress) read the actual HTTP request and can route `/api/*` to one service and `/static/*` to another, rewrite headers, and terminate TLS. Far more capable than L4, and correspondingly more expensive per request.
- WAFs inspect HTTP bodies for SQL injection and XSS patterns
- API gateways doing rate limiting, auth, and request transformation
- HTTP status codes, headers, methods, and caching semantics

**How you debug it**
```bash
curl -v https://example.com          # Full request/response cycle
dig example.com                      # DNS resolution detail
http --verbose GET example.com       # httpie, friendlier output
```

---

## Encapsulation — how the layers wrap each other

Going **down** the stack, each layer adds its own header. Going **up**, each layer strips its own header.

```
L7  Application    │ HTTP request                                  │
L6  Presentation   │ TLS( HTTP request )                           │
L5  Session        │ TLS( HTTP request )                           │
L4  Transport      │ [TCP hdr][ TLS( HTTP request ) ]              │  → Segment
L3  Network        │ [IP hdr][TCP hdr][ TLS(HTTP) ]                │  → Packet
L2  Data link      │ [Eth hdr][IP hdr][TCP hdr][ TLS(HTTP) ][FCS]  │  → Frame
L1  Physical       │ 010110100011010101101001110101...             │  → Bits
```

Each layer treats everything handed down from above as **opaque payload**. TCP has no idea whether it's carrying HTTP or SSH. IP has no idea whether it's carrying TCP or UDP beyond a protocol number in the header.

**Overhead cost:** roughly 14 bytes Ethernet + 20 bytes IPv4 + 20 bytes TCP = ~54 bytes of headers per packet before any actual data.

---

## Walkthrough: `https://example.com` end to end

**Step 0 — DNS (L7)**
Browser cache → OS cache → `/etc/hosts` → configured resolver. Returns `93.184.216.34`. This lookup is itself a full trip down and back up the stack, typically over UDP/53.

**Going down**

| Layer | What happens |
|-------|--------------|
| 7 | Browser composes `GET / HTTP/1.1` with `Host:` and `User-Agent:` headers |
| 6 | TLS handshake completes; the request is encrypted. Content encoding negotiated |
| 5 | Logical session established; keep-alive intent recorded |
| 4 | TCP picks ephemeral source port `54321`, targets `443`. Three-way handshake. Data split into numbered segments |
| 3 | Each segment wrapped in an IP packet: src `192.168.1.42`, dst `93.184.216.34`, TTL 64. Routing table consulted → not local → send to default gateway |
| 2 | ARP resolves the gateway's MAC. Ethernet frame built with that MAC as destination |
| 1 | Frame converted to voltage / light / radio and transmitted |

**Across the internet — at every router**
```
Receive frame → strip L2 → read L3 destination IP → consult routing table
→ decrement TTL → build a NEW L2 frame for the next hop → transmit
```
The router never opens L4–L7. IP addresses stay constant end-to-end; MAC addresses change at every single hop.

**Going up (at the server)**

| Layer | What happens |
|-------|--------------|
| 1 | Signal received, bits recovered |
| 2 | FCS checksum verified, destination MAC confirmed as own, Ethernet header stripped |
| 3 | Destination IP confirmed as own, IP header stripped |
| 4 | Segments reassembled by sequence number, ACKs sent, byte stream handed to the process listening on 443 |
| 5/6 | TLS decrypts back to plaintext |
| 7 | nginx reads `GET / HTTP/1.1` and generates a response |

Then the entire process runs again in reverse to carry the HTML back.

**The core rule:** each layer communicates only with its **peer layer** on the other machine. Your browser's L7 talks to the server's L7. Everything in between is plumbing neither side thinks about.

---

## OSI vs TCP/IP — what actually runs

OSI is the teaching model. TCP/IP is what the internet is actually built on.

| OSI | TCP/IP model | Reality |
|-----|--------------|---------|
| 7 Application | | |
| 6 Presentation | **Application** | HTTP, DNS, TLS — all collapsed together |
| 5 Session | | |
| 4 Transport | **Transport** | TCP, UDP, QUIC — maps cleanly |
| 3 Network | **Internet** | IP — maps cleanly |
| 2 Data link | **Link** | Ethernet, Wi-Fi — merged |
| 1 Physical | | |

**Why learn OSI at all if TCP/IP is what runs?**
Because the vocabulary is universal. "That's an L3 problem," "we need an L7 rule," "the L4 health check passes but the L7 one fails" — this is how network and infrastructure engineers actually talk. The numbers are shorthand everyone shares.

**Where OSI is a poor fit for reality**
- TLS spans L5, L6, and arguably L4 (in QUIC)
- HTTP/2 multiplexing does session-layer work inside an application-layer protocol
- Kubernetes service meshes operate at L4 and L7 simultaneously
- A "layer 3.5" is sometimes invoked for MPLS and ARP, which fit neither cleanly

Treat the layers as a useful abstraction, not a physical law.

---

## Troubleshooting: work bottom-up

The single most useful thing about OSI is that it gives you an ordered checklist. Start at Layer 1 and climb.

| # | Question | Command |
|---|----------|---------|
| 1 | Is the link physically up? | `ip link show`, `ethtool eth0` |
| 2 | Can I reach my local gateway? | `ip neigh`, `arping <gateway>` |
| 3 | Can I route to the internet? | `ping 8.8.8.8`, `traceroute`, `mtr` |
| 4 | Is the port open and accepting? | `nc -zv host 443`, `ss -tulpn` |
| 5 | Is the session/connection healthy? | Check pool metrics, timeouts |
| 6 | Is TLS valid and negotiable? | `openssl s_client -connect host:443` |
| 7 | Is the app responding correctly? | `curl -v`, `dig`, check app logs |

**The classic diagnostic split:**
`ping 8.8.8.8` works but `ping google.com` fails → L3 is fine, DNS (L7) is broken.
`ping host` works but `curl host` hangs → L3 is fine, something at L4 (firewall/port) or L7 (app down) is wrong.

---

## Summary in one paragraph

Data starts as a message in an application (L7), gets formatted and encrypted (L6), placed in a managed conversation (L5), split into numbered segments addressed to a port (L4), wrapped in packets addressed to an IP (L3), wrapped in frames addressed to the next device's MAC (L2), and finally turned into physical signals (L1). At the other end it climbs back up, each layer undoing exactly what its counterpart did. Every layer trusts the one below it to do its job and treats everything from above as an opaque blob — which is precisely why you can swap Wi-Fi for fibre without changing a single line of application code.
