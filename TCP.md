# TCP — Transmission Control Protocol, In Depth

> A companion to `osi-model-explained.md`. TCP is the Layer 4 protocol that makes the unreliable, packet-losing, out-of-order internet behave like a clean, ordered pipe between two programs.

---

## What TCP is

**TCP = Transmission Control Protocol** (RFC 793, updated by RFC 9293).

Four defining properties:

| Property | Meaning |
|---|---|
| **Connection-oriented** | Both sides must agree a connection exists before any data moves |
| **Reliable** | Every byte is acknowledged; anything lost is retransmitted |
| **Ordered** | Bytes arrive in the order sent, regardless of the order packets arrived |
| **Byte-stream** | There are no message boundaries. `send()` twice may arrive as one `recv()`, or three |

That last point trips people up constantly. **TCP has no concept of a "message."** If your application needs message framing, you must build it yourself — length prefixes, delimiters, or a framing protocol like HTTP's `Content-Length`.

### TCP vs UDP at a glance

| | TCP | UDP |
|---|---|---|
| Full form | Transmission Control Protocol | User Datagram Protocol |
| Connection | Handshake required | None |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Guaranteed | None |
| Header size | 20–60 bytes | 8 bytes |
| Speed | Slower (handshake + ACK overhead) | Faster |
| Congestion control | Yes | No |
| Use for | Web, SSH, databases, email, file transfer | DNS, video, voice, gaming, metrics, DHCP |

---

## The TCP header

Minimum 20 bytes, maximum 60 bytes with options.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source Port          |       Destination Port        |
+---------------------------------------------------------------+
|                        Sequence Number                        |
+---------------------------------------------------------------+
|                     Acknowledgment Number                     |
+-------+-------+-----------------+-----------------------------+
| Data  |  Rsvd | C E U A P R S F |          Window Size        |
| Offset|       | W C R C S S Y I |                             |
|       |       | R E G K H T N N |                             |
+-------------------------------+-------------------------------+
|           Checksum            |        Urgent Pointer         |
+-------------------------------+-------------------------------+
|                    Options (0–40 bytes)                       |
+---------------------------------------------------------------+
|                            Data                               |
+---------------------------------------------------------------+
```

### Field by field

| Field | Size | Purpose |
|---|---|---|
| **Source Port** | 16 bits | Sending process. Client side is an ephemeral port, typically 32768–60999 on Linux (`net.ipv4.ip_local_port_range`) |
| **Destination Port** | 16 bits | Receiving process — 80, 443, 22, 5432, 6379… |
| **Sequence Number** | 32 bits | Byte offset of the first data byte in this segment, within the overall stream |
| **Acknowledgment Number** | 32 bits | Next byte expected. Implies everything before it was received |
| **Data Offset** | 4 bits | Header length in 32-bit words (5 = 20 bytes, 15 = 60 bytes). Marks where data begins |
| **Reserved** | 3 bits | Must be zero |
| **Flags** | 9 bits | Control bits — see next section |
| **Window Size** | 16 bits | Bytes the sender of this segment can still buffer. Flow control |
| **Checksum** | 16 bits | Error detection over header, data, and an IP pseudo-header |
| **Urgent Pointer** | 16 bits | Offset to the end of urgent data. Only valid when URG is set |
| **Options** | 0–40 bytes | MSS, window scale, SACK-permitted, timestamps, TFO cookie |

### Options worth knowing

| Option | Full form | Why it matters |
|---|---|---|
| **MSS** | Maximum Segment Size | Largest payload the sender will accept. Negotiated in the SYN. Typically 1460 = 1500 MTU − 20 IP − 20 TCP |
| **Window Scale** | — | The 16-bit window field caps at 65535 bytes, far too small for modern links. This option shifts it left by up to 14 bits |
| **SACK-Permitted** | Selective Acknowledgment | Enables selective ACKs so a single loss doesn't force retransmission of everything after it |
| **Timestamps** | — | Improves RTT measurement and protects against sequence number wraparound (PAWS) |
| **TFO** | TCP Fast Open | Lets data ride along with the SYN on repeat connections, saving one RTT |

---

## The control flags — full forms

Nine bits, each a boolean switch.

| Flag | Full form | Meaning |
|---|---|---|
| **SYN** | **Syn**chronize sequence numbers | "Open a connection. Here is my initial sequence number." Appears only in the first two segments of a connection's life |
| **ACK** | **Ack**nowledgment | "The Acknowledgment Number field is valid." Set on nearly every segment after the initial SYN |
| **PSH** | **Push** | "Deliver this to the application immediately, don't sit in the buffer waiting for more." Used for interactive traffic and at the end of a logical message |
| **FIN** | **Fin**ish | "I have no more data to send." Begins a graceful, half-duplex shutdown |
| **RST** | **Res**e**t** | "Abort now." Sent when a segment arrives for a nonexistent connection or a closed port. This produces `Connection refused` |
| **URG** | **Urg**ent | "Urgent data is present; the Urgent Pointer marks its end." Effectively obsolete — middleboxes routinely strip or ignore it |
| **ECE** | **E**CN-**E**cho | "I received a congestion signal from a router." Part of ECN |
| **CWR** | **C**ongestion **W**indow **R**educed | "I saw your ECE and have reduced my sending rate" |
| **NS** | **N**once **S**um | Experimental ECN integrity check (RFC 3540). Deprecated |

**Supporting acronym:** **ECN = Explicit Congestion Notification** — routers mark packets to signal congestion *before* they have to drop them, letting TCP slow down without incurring loss.

### Combinations you will actually see

| Combination | Situation |
|---|---|
| `SYN` | Handshake packet 1 — connection request |
| `SYN, ACK` | Handshake packet 2 — connection accepted |
| `ACK` | Handshake packet 3, or any plain acknowledgment |
| `PSH, ACK` | Data delivery — by far the most common segment in a live connection |
| `FIN, ACK` | Graceful close request |
| `RST` | Hard abort — nothing listening, or state lost |
| `RST, ACK` | Abort in response to activity on an existing connection |

### Reading tcpdump notation

`tcpdump` compresses flags into single characters:

| Notation | Flags |
|---|---|
| `[S]` | SYN |
| `[S.]` | SYN, ACK |
| `[.]` | ACK only |
| `[P.]` | PSH, ACK |
| `[F.]` | FIN, ACK |
| `[R]` | RST |
| `[R.]` | RST, ACK |

The `.` always means ACK.

---

## The three-way handshake

**Why three steps?** Both directions need synchronized sequence numbers, and each side needs proof the other received its number. Two steps would synchronize only one direction.

```
Client                                          Server
  |                                               |
  |  ── 1. SYN, seq=x ──────────────────────────► |
  |                                               |
  |  ◄── 2. SYN + ACK, seq=y, ack=x+1 ─────────── |
  |                                               |
  |  ── 3. ACK, ack=y+1 ────────────────────────► |
  |                                               |
  |          [ connection ESTABLISHED ]           |
```

**Step 1 — Client sends SYN**
The client picks an **ISN (Initial Sequence Number)**. This is randomized rather than starting at zero, to prevent sequence-prediction and connection-hijacking attacks. The SYN also carries the client's options: MSS, window scale, SACK-permitted.
Client state: `CLOSED` → `SYN_SENT`

**Step 2 — Server sends SYN + ACK**
The server picks its own independent ISN and acknowledges the client's with `ack = x + 1`. The `+1` exists because **the SYN flag itself consumes one sequence number**, even though it carries zero bytes of data.
Server state: `LISTEN` → `SYN_RECEIVED`

**Step 3 — Client sends ACK**
The client acknowledges the server's ISN with `ack = y + 1`. The connection is now open in both directions. This third packet may carry application data.
Client: `SYN_SENT` → `ESTABLISHED`. Server, on receipt: `SYN_RECEIVED` → `ESTABLISHED`

**Latency cost:** one full **RTT (Round Trip Time)** before any application byte moves. Layer TLS on top and it becomes 2–3 RTT. This is precisely the cost that QUIC and HTTP/3 were designed to eliminate.

---

## The four-way teardown

TCP connections are **full-duplex** — two independent byte streams. Each direction must be closed separately, which is why closing takes four segments rather than two.

```
Client                                          Server
  |                                               |
  |  ── 1. FIN, ACK ────────────────────────────► |
  |                                               |  app still
  |  ◄── 2. ACK ────────────────────────────────  |  sending?
  |                                               |  (CLOSE_WAIT)
  |  ◄── 3. FIN, ACK ───────────────────────────  |
  |                                               |
  |  ── 4. ACK ─────────────────────────────────► |
  |                                               |
  |  [ TIME_WAIT, 2×MSL ]                [ CLOSED ]
```

| Step | Sender | Effect |
|---|---|---|
| 1 | Client `FIN` | "I'm done sending." Client: `ESTABLISHED` → `FIN_WAIT_1` |
| 2 | Server `ACK` | Server: `ESTABLISHED` → `CLOSE_WAIT`. Client: `FIN_WAIT_1` → `FIN_WAIT_2` |
| 3 | Server `FIN` | Sent only when the server's application also calls `close()`. Server: `CLOSE_WAIT` → `LAST_ACK` |
| 4 | Client `ACK` | Client: `FIN_WAIT_2` → `TIME_WAIT` |

### TIME_WAIT — why it exists

`TIME_WAIT` lasts **2 × MSL (Maximum Segment Lifetime)**, typically 60 seconds on Linux. Two reasons:

1. **Stray segment protection.** A delayed duplicate from the old connection could otherwise arrive during a *new* connection reusing the same 5-tuple and corrupt its stream.
2. **Reliable final ACK.** If the last ACK is lost, the peer retransmits its FIN. Something must still be around to answer it.

`TIME_WAIT` accumulating on a **client** is normal under high connection churn — the fix is connection pooling and keep-alive, not tuning kernel parameters.

### CLOSE_WAIT — almost always your bug

Sockets stuck in `CLOSE_WAIT` mean the kernel acknowledged the peer's FIN but **your application never called `close()`**. The kernel cannot send its own FIN until the application says it's finished. This is an application-level file-descriptor leak, not a network problem. Left unchecked it exhausts the process's fd limit.

---

## The TCP state machine

| State | Meaning |
|---|---|
| `CLOSED` | No connection. The starting and ending state |
| `LISTEN` | Server is waiting for incoming SYNs |
| `SYN_SENT` | Client sent SYN, waiting for SYN-ACK |
| `SYN_RECEIVED` | Server got SYN, sent SYN-ACK, waiting for the final ACK |
| `ESTABLISHED` | Connection open, data can flow both ways |
| `FIN_WAIT_1` | We sent FIN, waiting for its ACK |
| `FIN_WAIT_2` | Our FIN was acknowledged; waiting for the peer's FIN |
| `CLOSE_WAIT` | Peer sent FIN; we acknowledged. Waiting for **our application** to close |
| `CLOSING` | Both sides sent FIN simultaneously. Rare |
| `LAST_ACK` | We sent our FIN after being closed by the peer; waiting for the final ACK |
| `TIME_WAIT` | Waiting 2×MSL before fully releasing the 5-tuple |

Each connection's state lives in a kernel structure called the **TCB (Transmission Control Block)**.

---

## How a TCP server works

### The system-call sequence

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);   // 1. Create endpoint
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
bind(fd, (struct sockaddr*)&addr, sizeof(addr));  // 2. Claim IP:port
listen(fd, backlog);                        // 3. Passive open

while (1) {
    int conn = accept(fd, NULL, NULL);      // 4. Pop a ready connection
    recv(conn, buf, sizeof(buf), 0);        // 5. Read
    send(conn, response, len, 0);           //    Write
    close(conn);                            // 6. Teardown
}
```

| Call | What the kernel does |
|---|---|
| `socket()` | Allocates a socket structure. No network activity yet |
| `bind()` | Reserves the IP:port pair. Returns `EADDRINUSE` if already taken |
| `listen()` | Enters `LISTEN` state and creates two queues. From this moment the kernel answers SYNs on its own, with no application involvement |
| `accept()` | Removes one fully-established connection from the accept queue and returns a **new** fd for it. The listening fd stays open. Blocks when the queue is empty |
| `recv()` / `send()` | Copy bytes between user space and kernel socket buffers |
| `close()` | Sends FIN and begins teardown |

**`SO_REUSEADDR`** ("Socket Option: Reuse Address") lets you `bind()` to a port that still has connections in `TIME_WAIT` — without it, restarting a server often fails with `Address already in use`.

**`SO_REUSEPORT`** goes further: multiple processes can bind the same port, and the kernel load-balances incoming connections between them. This is how modern multi-process servers scale without a single accepting thread becoming a bottleneck.

### The two queues — a common production failure

`listen()` creates **two** distinct queues, and confusing them makes outages hard to diagnose.

**1. SYN queue** (incomplete connection queue)
Connections that received a SYN and had a SYN-ACK sent, but whose final ACK hasn't arrived. State: `SYN_RECEIVED`.
Sized by `net.ipv4.tcp_max_syn_backlog`.

**2. Accept queue** (completed connection queue)
Connections that finished the handshake and are waiting for the application to call `accept()`. State: `ESTABLISHED`.
Sized by the `backlog` argument to `listen()`, hard-capped by `net.core.somaxconn`.

**The failure mode:** if your application is slow to call `accept()` — blocked on a database query, a GC pause, an exhausted thread pool — the accept queue fills. New connections are then **silently dropped or reset**. The client sees a timeout or `connection reset by peer`; your application logs show nothing, because those requests never reached the application at all.

```bash
ss -ltn
# For LISTEN sockets:
#   Recv-Q = current accept queue depth
#   Send-Q = configured backlog limit

netstat -s | grep -i "listen queue"    # overflow counter
sysctl net.core.somaxconn
```

If `Recv-Q` sits near `Send-Q`, you are dropping connections. The fix is usually faster request handling or more workers — raising the backlog only buys a slightly longer queue before the same failure.

### SYN flood and SYN cookies

A **SYN flood** sends many SYNs and never completes the handshake, exhausting the SYN queue so legitimate connections can't get in.

The defense is **SYN cookies** (`net.ipv4.tcp_syncookies=1`). Instead of allocating state on the SYN, the server encodes the connection parameters into the sequence number it sends back. If a valid final ACK arrives, the state is reconstructed from it. The server keeps zero state until the handshake actually completes.

### Concurrency models

| Model | Approach | Trade-off |
|---|---|---|
| Process per connection | `fork()` after `accept()` | Simple, isolated, heavy |
| Thread per connection | Spawn a thread | Lighter than processes, still doesn't scale to tens of thousands |
| Thread pool | Fixed workers pulling from a queue | Bounded resources, queueing under load |
| Event loop | `epoll` / `kqueue`, single thread, non-blocking sockets | Scales to very high connection counts — this is Node.js and nginx |
| `SO_REUSEPORT` | N processes each with their own accept queue | Kernel-level load balancing, no accept bottleneck |

---

## Reliability mechanisms

### Sequence and acknowledgment numbers

Every byte in the stream has a number. Acknowledgments are **cumulative**: `ack=5000` means "I have received everything through byte 4999."

If segment 3 of 5 is lost, the receiver keeps re-acknowledging the end of segment 2 even as segments 4 and 5 arrive. Three duplicate ACKs trigger **fast retransmit** — the sender resends the missing segment immediately rather than waiting for a timeout.

### SACK — Selective Acknowledgment

Cumulative ACKs are wasteful when multiple segments are lost. SACK lets the receiver report exactly which ranges it holds:

> "Cumulative ACK 1000, and I also have 2001–3000 and 4001–5000."

The sender then retransmits only the actual gaps.

### RTO — Retransmission Timeout

If no ACK arrives within the RTO, resend. The value is computed from a smoothed RTT estimate plus variance, and **doubles on each consecutive failure** (exponential backoff), so a dead path doesn't get hammered.

### Checksum

A 16-bit checksum covers the TCP header, the data, and a pseudo-header containing the source and destination IP addresses. Corrupt segments are dropped silently, and the retransmission machinery recovers them.

### Nagle's algorithm and delayed ACK

**Nagle's algorithm** buffers small writes until either a full MSS accumulates or all outstanding data is acknowledged — it prevents flooding the network with tiny packets.

**Delayed ACK** waits up to ~200ms before acknowledging, hoping to piggyback the ACK on outgoing data.

Together these two optimizations can **interact badly**, producing a stall where the sender waits for an ACK and the receiver waits for data. Disable Nagle with `TCP_NODELAY` for latency-sensitive request/response traffic.

---

## Flow control vs congestion control

Two different problems, frequently conflated.

| | Flow control | Congestion control |
|---|---|---|
| Protects | The **receiver** | The **network** |
| Question | "Can the other end absorb more?" | "Can the path carry more?" |
| Mechanism | **RWND** (Receive Window) | **CWND** (Congestion Window) |
| Where it lives | Advertised explicitly in every segment | Maintained privately by the sender |
| Signal | The window field | Inferred from loss, delay, or ECN marks |

The sender may have at most `min(RWND, CWND)` unacknowledged bytes in flight at any moment.

### Congestion control phases

1. **Slow start** — CWND doubles every RTT until it reaches **SSTHRESH (Slow Start Threshold)** or loss occurs. Despite the name, this is exponential growth
2. **Congestion avoidance** — linear growth, roughly +1 MSS per RTT
3. **Fast recovery** — on three duplicate ACKs, halve CWND and continue rather than collapsing back to slow start

### Algorithms

| Algorithm | Basis |
|---|---|
| **Reno / NewReno** | Loss-based, classic |
| **CUBIC** | Loss-based with a cubic growth curve. Linux default |
| **BBR** | Models actual bottleneck bandwidth and RTT instead of treating loss as the only congestion signal. Much better on lossy or high-latency links |

```bash
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
```

### Zero window

If the receiving application stops reading, its buffer fills and it advertises `window=0`. The sender halts and sends periodic **window probes** until the receiver reopens.

Seeing `TCP ZeroWindow` in a capture means the receiving **application** is the bottleneck — not the network, not the sender.

### BDP — Bandwidth-Delay Product

```
BDP = bandwidth × RTT
```

This is how many bytes can be "in the air" at once. If the window is smaller than the BDP, you cannot saturate the link no matter how fast it is. A 1 Gbps link with 100ms RTT has a BDP of ~12.5 MB — far beyond the unscaled 64 KB window limit, which is exactly why the window-scale option exists.

---

## Full-form glossary

| Acronym | Full form |
|---|---|
| TCP | Transmission Control Protocol |
| UDP | User Datagram Protocol |
| SYN | Synchronize sequence numbers |
| ACK | Acknowledgment |
| PSH | Push |
| FIN | Finish |
| RST | Reset |
| URG | Urgent |
| ECE | ECN-Echo |
| CWR | Congestion Window Reduced |
| NS | Nonce Sum |
| ECN | Explicit Congestion Notification |
| ISN | Initial Sequence Number |
| MSS | Maximum Segment Size |
| MTU | Maximum Transmission Unit |
| RTT | Round Trip Time |
| RTO | Retransmission Timeout |
| MSL | Maximum Segment Lifetime |
| RWND | Receive Window |
| CWND | Congestion Window |
| SSTHRESH | Slow Start Threshold |
| SACK | Selective Acknowledgment |
| BDP | Bandwidth-Delay Product |
| TFO | TCP Fast Open |
| TCB | Transmission Control Block |
| PAWS | Protection Against Wrapped Sequence numbers |
| PMTUD | Path MTU Discovery |
| PDU | Protocol Data Unit |
| QUIC | Quick UDP Internet Connections |

---

## Debugging cheatsheet

```bash
# Connection state
ss -tan                                  # All TCP sockets + state
ss -tan state established | wc -l        # Active connection count
ss -tan state time-wait | wc -l          # TIME_WAIT count
ss -tan state close-wait                 # App forgot close() — investigate
ss -tin                                  # Per-connection cwnd, rtt, retransmits

# Listen queue health
ss -ltn                                  # Recv-Q vs Send-Q on LISTEN sockets
netstat -s | grep -i listen              # Overflow counters

# Packet-level
tcpdump -i any -nn 'tcp port 443'
tcpdump -i any -nn 'tcp[tcpflags] & tcp-syn != 0'   # SYNs only
tcpdump -i any -nn 'tcp[tcpflags] & tcp-rst != 0'   # RSTs only

# Aggregate counters
netstat -s | grep -i retrans
nstat -az | grep -i tcp

# Tunables
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.ip_local_port_range
```

### Symptom → cause

| Symptom | Likely cause |
|---|---|
| `Connection refused` instantly | RST received — nothing listening on that port |
| Hangs, then times out | SYN never answered — firewall or security group dropping silently |
| `Connection reset by peer` mid-request | Peer sent RST — app crashed, timeout, or accept queue overflow |
| Many `CLOSE_WAIT` on your server | Your application isn't calling `close()` — fd leak |
| Many `TIME_WAIT` on your client | Normal with high churn. Use connection pooling / keep-alive |
| High retransmission counters | Path packet loss, or MTU mismatch causing silent drops |
| `TCP ZeroWindow` in captures | Receiving application too slow to drain its buffer |
| Intermittent failures at exactly ~1500 bytes | MTU / MSS mismatch — very common on VPNs and container overlay networks |
| Throughput plateaus far below link speed | Window smaller than the BDP, or window scaling disabled by a middlebox |

---

## Summary

TCP takes an unreliable packet network and presents it to applications as a clean, ordered, bidirectional byte pipe. It does this with four core mechanisms: a **handshake** to establish shared sequence numbers, **cumulative acknowledgments plus retransmission** for reliability, a **receive window** to avoid overwhelming the peer, and a **congestion window** to avoid overwhelming the network. The nine flag bits — SYN, ACK, PSH, FIN, RST, URG, ECE, CWR, NS — are simply the control signals that drive that machinery. Nearly every TCP problem you meet in production traces back to one of three things: a queue that filled up, a socket somebody forgot to close, or an MTU that didn't match.
