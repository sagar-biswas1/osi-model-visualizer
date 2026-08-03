# Sockets — What They Actually Are at the OS Level

> A companion to `osi-model-explained.md` and the TCP deep-dive. TCP describes what goes on the wire. This document describes what lives **inside the kernel** — the structures, queues, and syscalls that turn "network" into a file descriptor your program can `read()`.

---

# Part 1 — The simple version

Read this first. Everything in Part 2 onward is the same story with real names attached.

## A socket is a telephone

You want to talk to someone far away. You do **not** run a wire to their house. You pick up a phone, and the phone company handles the wires.

- **Your program** = you
- **The socket** = your phone
- **The kernel (OS)** = the phone company
- **The network** = the wires you never see

Your program never touches the network. It only ever talks to the phone. The kernel does everything else.

## The address has two parts

| Part | Phone version | Network version |
|---|---|---|
| Which building | Street address | **IP address** |
| Which person inside | Extension number | **Port** |

`142.250.x.x:443` means "that building, extension 443."

## The socket is a numbered ticket

When your program asks for a socket, the kernel builds the real thing (a complicated object full of buffers and counters), keeps it, and hands you back a small number — like a **coat-check ticket**.

```
fd = socket(...)   →   fd is 3
```

That number is called a **file descriptor**. Every time you want to do anything, you show the ticket:

```
send(3, "hello")
recv(3, buffer)
close(3)
```

You never see the actual object. You only ever hold ticket #3.

**Why "file" descriptor?** Because on Linux, sockets are deliberately made to look like files. Same `read()`, same `write()`, same `close()`. The kernel plays along so your program doesn't need to learn anything new.

## Every socket has two mailboxes

```
        ┌──────────── your program ────────────┐
        │                                      │
     write()                                recv()
        │                                      ▲
        ▼                                      │
   ┌─────────┐                          ┌─────────┐
   │ OUT box │                          │ IN box  │
   │ (send   │                          │ (receive│
   │  buffer)│                          │  buffer)│
   └────┬────┘                          └────▲────┘
        │                                    │
        ▼            the network             │
      out ──────────────────────────────── in
```

- You `send()` → you drop a letter in the **out box** and walk away. **You are not waiting for it to be delivered.** `send()` returning successfully means "the kernel took it," not "they got it."
- You `recv()` → you check the **in box**. If it's empty, you wait.
- If your out box gets full (network is slow), `send()` makes you wait until there's room.
- If your in box gets full (you're reading too slowly), the kernel tells the other side **"stop sending"**. That's flow control — and it's your program's fault, not the network's.

## A server is a shop with a doorbell

| Syscall | Shop version |
|---|---|
| `socket()` | Buy a phone |
| `bind()` | Claim a specific address and extension |
| `listen()` | Put out a doorbell and hire a doorman |
| `accept()` | "Next customer, please" |
| `close()` | Say goodbye |

The crucial bit: after `listen()`, **the doorman answers the door without waking you up.** The kernel completes the whole TCP handshake by itself. Customers pile up in a **waiting room** (the accept queue). Your program calls `accept()` to fetch the next one.

If you're slow — stuck on a database query, a garbage-collection pause, no free threads — the waiting room fills. New customers get turned away at the door. **Your logs show nothing**, because those requests never reached your code.

Also important: `accept()` gives you a **brand-new phone** for that one customer. The doorbell keeps working. That's why one server handles thousands of connections.

## Waiting is sleeping, not looking

When your program waits for data, it does not sit in a loop asking "is it here yet?" That would burn a whole CPU.

Instead, the program **goes to sleep** and leaves a note: *wake me when something arrives on socket 3.* The kernel puts it on a wait list. When a packet lands, the kernel taps it on the shoulder. Zero CPU used while waiting.

**epoll** is the same idea, scaled up: *wake me when anything happens on any of these 10,000 sockets, and tell me which ones.*

## Hanging up leaves the line warm

`close()` doesn't make everything vanish instantly. The kernel still has to say goodbye politely, and then it keeps the line reserved for about a minute (`TIME_WAIT`) so a late letter from the old conversation can't get mixed into a new one on the same address.

And the reverse: if the *other* side hangs up but your program never calls `close()`, the connection sits half-dead forever in `CLOSE_WAIT`. The kernel can't finish the goodbye until you say you're done. Thousands of those and you run out of tickets.

## The one-line summary

> A socket is a **file descriptor** that points at a kernel object containing two buffers, a protocol state machine, and a wait list. Your program moves bytes in and out of the buffers; the kernel does everything else.

---

# Part 2 — What a socket really is

A socket is **not** fundamentally a network thing. It is a **file** whose read/write operations happen to be wired to the network stack.

```
Process                          Kernel
┌──────────────┐
│ fd table     │
│  0 → file    │
│  1 → file    │
│  3 → file ───┼──► struct file
└──────────────┘        │ f_op = socket_file_ops
                        │ private_data ──► struct socket   ← VFS / BSD layer
                                              │ state, type
                                              │ ops = inet_stream_ops
                                              │ sk ──► struct sock   ← generic net layer
                                                          │ sk_receive_queue
                                                          │ sk_write_queue
                                                          │ sk_backlog
                                                          │ sk_state
                                                          │ sk_rcvbuf / sk_sndbuf
                                                          │ sk_wq (wait queue)
                                                          ▼
                                                      struct tcp_sock   ← the TCB
```

Sockets live in **sockfs**, an in-memory pseudo-filesystem with no mount point. This is why `ls -l /proc/<pid>/fd` shows `socket:[4026531234]` — an inode number and no path. It is also why `read()`, `write()`, `close()`, `dup()`, `fcntl()`, and `epoll` all work on sockets unmodified: VFS never learns it isn't a file.

## The struct chain

C has no inheritance, so the kernel embeds structs at offset 0 and upcasts:

| Struct | Layer | Contains |
|---|---|---|
| `struct socket` | VFS / BSD | `type` (SOCK_STREAM…), `state`, `ops` vtable, back-pointer to `struct file` |
| `struct sock` | Protocol-agnostic | Queues, buffer limits, `sk_state`, wait queue, callbacks (`sk_data_ready`, `sk_state_change`) |
| `struct inet_sock` | IPv4 / IPv6 | saddr, daddr, sport, dport, TTL, tos |
| `struct inet_connection_sock` | Connection-oriented | Accept queue, retransmit/keepalive timers, congestion-control ops |
| `struct tcp_sock` | TCP | `snd_una`, `snd_nxt`, `rcv_nxt`, `snd_cwnd`, `snd_wnd`, `srtt_us`, `rttvar_us`, SACK state, `out_of_order_queue` |

`struct tcp_sock` **is** the TCB (Transmission Control Block) from the TCP doc. It's roughly 2 KB per connection, before buffers. The fd is 4 bytes; the connection is kilobytes. That's the real cost of a million-connection server.

`SOCK_STREAM` → `inet_stream_ops` → TCP. `SOCK_DGRAM` → `inet_dgram_ops` → UDP. Same `struct socket`, different function table. This is polymorphism in C.

---

## Part 3 — What each syscall does to those structures

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
bind(fd, (struct sockaddr *)&addr, sizeof(addr));
listen(fd, backlog);
int conn = accept(fd, NULL, NULL);
recv(conn, buf, sizeof(buf), 0);
send(conn, resp, len, 0);
close(conn);
```

| Syscall | Kernel work |
|---|---|
| `socket()` | Allocates `struct socket` + `struct tcp_sock`, allocates a sockfs inode, allocates `struct file`, installs it at the lowest free fd. **No network activity.** |
| `bind()` | Inserts the sock into the **bind hash** (`bhash`) keyed by port. `EADDRINUSE` is a bind-hash collision, resolved purely by the `SO_REUSEADDR` / `SO_REUSEPORT` flags on both the existing and new socket. |
| `listen()` | `sk_state` → `TCP_LISTEN`, inserts into the **listen hash** (`lhash2`), allocates `icsk_accept_queue`. From this instant the kernel answers SYNs autonomously. |
| `connect()` | Selects an ephemeral port from `ip_local_port_range`, does a route lookup (`ip_route_output`), builds and transmits the SYN, then sleeps in `TCP_SYN_SENT` until `sk_state_change` wakes it. |
| `accept()` | Pops an already-`ESTABLISHED` `struct sock` off the accept queue, wraps it in a **new** `struct socket` + `struct file` + fd. The listening socket is untouched and keeps accepting. |
| `send()` | Copies user bytes into `sk_buff`s, appends to `sk_write_queue`, *may* transmit. Returns once copied into kernel memory — **never** implies delivery or acknowledgment. |
| `recv()` | Copies from `sk_receive_queue` into user memory, frees the skbs, possibly sends a window update. |
| `close()` | Decrements the `struct file` refcount. **Only at zero** does it send FIN. The `struct sock` outlives the fd through `FIN_WAIT` / `TIME_WAIT`. |

### Two consequences worth internalizing

**1. The `close()` refcount explains `CLOSE_WAIT` leaks precisely.** A `fork()`ed child, a `dup()`ed fd, or a copy stashed in a connection pool holds an extra reference. Your `close()` returns 0 successfully, the refcount never reaches zero, the FIN is never sent, and the socket sits in `CLOSE_WAIT` forever.

**2. `send()` returning success means nothing about the peer.** It means "copied into `sk_write_queue`." The data may still be unsent, in flight, lost, or dropped by a peer that crashed. Application-level acknowledgment is the only real acknowledgment.

### The two queues, structurally

`listen()` creates two things people conflate:

```
        incoming SYN
              │
              ▼
    ┌──────────────────┐   handshake completes   ┌──────────────────┐
    │    SYN queue     │ ──────────────────────► │  accept queue    │
    │ (request_sock_   │                          │ (icsk_accept_    │
    │  queue)          │                          │  queue)          │
    │ state SYN_RECV   │                          │ state ESTABLISHED│
    └──────────────────┘                          └────────┬─────────┘
     net.ipv4.tcp_max_syn_backlog                          │ accept()
                                                            ▼
                                                     new fd for app
```

Accept-queue size = `min(backlog, net.core.somaxconn)`. When it overflows, behaviour depends on `net.ipv4.tcp_abort_on_overflow`: by default the final ACK is **silently dropped** (the client retransmits and eventually times out); set to 1, the kernel sends RST instead.

### `SO_REUSEADDR` vs `SO_REUSEPORT`

| Option | Effect |
|---|---|
| `SO_REUSEADDR` | Permits `bind()` over a port held only by `TIME_WAIT` sockets. Fixes "Address already in use" on server restart. |
| `SO_REUSEPORT` | Multiple *live* sockets bind the same port. The kernel forms a reuseport group and load-balances new connections across them by 4-tuple hash — or by an attached eBPF program (`SO_ATTACH_REUSEPORT_EBPF`). Each process gets its **own accept queue**, eliminating the thundering herd and the single-acceptor bottleneck. |

---

## Part 4 — The receive path

```
NIC DMA ──► RX ring ──► hardware IRQ ──► NAPI poll (NET_RX_SOFTIRQ)
                                              │
                                    netif_receive_skb
                                              │
                                          ip_rcv
                                              │
                                        tcp_v4_rcv
                                              │
                          ┌───────────────────┴──────────────────┐
                    socket lookup                           no match
                    ehash by 5-tuple,                            │
                    else lhash (listeners)                      RST
                              │                          → "Connection refused"
                  ┌───────────┴───────────┐
            socket locked            not locked
            by userspace                  │
                  │                 tcp_v4_do_rcv
             sk_backlog                   │
          (drained on unlock)     ┌───────┴────────┐
                              in-order         out-of-order
                                  │                  │
                          sk_receive_queue    out_of_order_queue
                                  │              (rb-tree)
                            sk_data_ready()
                                  │
                          wake_up on sk->sk_wq
```

### Demultiplexing

`__inet_lookup_established()` hashes the full 5-tuple (saddr, sport, daddr, dport, protocol) into `ehash`. Established connections are found in O(1) — this is why a server with 500k connections doesn't slow down per packet.

A miss falls through to `__inet_lookup_listener()`, which scores candidates: a socket bound to a specific IP beats one bound to `INADDR_ANY`; a specific device beats none. `SO_REUSEPORT` groups then select a member by hash or BPF.

A total miss → RST. That RST is exactly what produces `ECONNREFUSED` on the client, instantly, with no timeout.

### Three queues, not two

| Queue | When it's used |
|---|---|
| `sk_receive_queue` | Normal path — in-order data ready for the application |
| `out_of_order_queue` | An rb-tree of segments that arrived ahead of a gap. Drained into the receive queue once the hole is filled by a retransmission |
| `sk_backlog` | A userspace thread holds `lock_sock()` (it's inside `recv()`), so softirq context cannot touch the receive queue. Packets park here and are processed by `release_sock()` when the syscall returns |

`sk_backlog` is the one nobody knows about. It bounds how much softirq can defer while a slow reader holds the lock, and overflow there is a silent drop counted in `nstat` as `TCPBacklogDrop`.

---

## Part 5 — The transmit path

```
send() → tcp_sendmsg()
             │  copy user data into skbs, sized to MSS
             ▼
      sk_write_queue                     "not yet sent"
             │
             ▼
      tcp_write_xmit()
             │  gate: in_flight_allowed = min(cwnd, rwnd) − packets_out
             │  plus Nagle, TSQ (TCP Small Queues), pacing
             ▼
      ip_queue_xmit() → qdisc → NIC driver → wire
             │
             ▼
      tcp_rtx_queue (rb-tree)            "sent, unacknowledged"
             │
             │  cumulative ACK arrives
             ▼
        skb freed, sk_wmem_alloc drops, poll reports writable
```

Key points:

- Segments are **cloned** for transmission; the original stays in the retransmit queue until cumulatively ACKed. That queue is where retransmissions come from.
- `send()` blocks (or returns `EAGAIN` when non-blocking) once `sk_wmem_alloc` exceeds `sk_sndbuf`. On a lossy or high-RTT path, your application blocks in `send()` because the *retransmit* queue is full of unacknowledged data — not because the network refused to take more.
- **TSQ (TCP Small Queues)** additionally caps per-socket bytes sitting in the qdisc/driver, which is what keeps one heavy flow from bufferbloating the whole NIC.

---

## Part 6 — Blocking, wakeups, and how epoll actually works

Every `struct sock` carries a wait queue at `sk->sk_wq`.

### Blocking mode

```
task: add_wait_queue(sk_wq); set_current_state(TASK_INTERRUPTIBLE); schedule();
                                    ⋮  (off the runqueue, zero CPU)
softirq: data arrives → sk_data_ready(sk) → wake_up_interruptible_sync_poll()
task: back on the runqueue, copies data out
```

### epoll

`epoll_ctl(EPOLL_CTL_ADD)` does **not** put your task on the socket's wait queue. It registers an `epitem` whose wait-queue entry has the callback `ep_poll_callback`. Now when `sk_data_ready()` fires, that callback runs and links the `epitem` onto the eventpoll instance's **ready list**, then wakes whoever sleeps in `epoll_wait()`.

```
            ┌──────────────┐
socket A ──►│              │
socket B ──►│  eventpoll   │──► ready list ──► epoll_wait() returns
socket C ──►│  (rb-tree of │        ▲          only the ready fds
   ...      │   epitems)   │        │
socket N ──►└──────────────┘   ep_poll_callback
                                    ▲
                              sk_data_ready()
```

This is why epoll is **O(ready)**, not O(watched): readiness is *pushed* by the protocol stack. `select()` and `poll()` are O(watched) because they re-scan every fd on every call.

| Mode | Behaviour |
|---|---|
| Level-triggered (default) | Reports readiness as long as data remains. Safe; re-armed automatically |
| Edge-triggered (`EPOLLET`) | Reports only on a state *transition*. You **must** drain until `EAGAIN` or you will hang holding unread data |

This mechanism is the entire foundation of nginx, Node.js (via libuv), Netty, and Go's netpoller.

---

## Part 7 — Buffers and memory accounting

| Knob | Meaning |
|---|---|
| `sk_rcvbuf` / `sk_sndbuf` | Per-socket byte limits. Autotuned between the three values in `net.ipv4.tcp_rmem` / `tcp_wmem` (min / default / max) |
| `SO_RCVBUF` / `SO_SNDBUF` | Manual override. **Disables autotuning.** The kernel doubles your value internally to account for skb overhead |
| `net.ipv4.tcp_mem` | Global page thresholds (low / pressure / high). Under pressure the kernel shrinks every socket's buffers system-wide |
| `sk_rmem_alloc` / `sk_wmem_alloc` | Bytes actually charged — counts full `sk_buff` overhead (~232 B per skb), not just payload |

The advertised **RWND** is derived from free space in `sk_rcvbuf`. Two direct consequences:

- Setting `SO_RCVBUF` too small caps your window below the **BDP** and permanently limits throughput. This is the usual cause of "throughput plateaus far below link speed" — that, or a middlebox stripping the window-scale option.
- Many small packets can exhaust `sk_rcvbuf` while holding far less payload than you think, because the accounting charges skb overhead too.

---

## Part 8 — Network namespaces (why this matters in Kubernetes)

Every network namespace owns a `struct net`. **Every hash table described above is per-`net`**: separate `ehash`, separate bind hash, separate listen hash, separate port space, separate loopback device, separate routing table, separate sysctls.

That is the entire mechanism behind:

- Two pods both binding `:8080` on the same node with no conflict — different `struct net`, different bind hash, no collision possible.
- `ss` inside a pod showing only that pod's sockets.
- Containers in the same pod sharing a namespace and therefore reaching each other over `127.0.0.1` — and colliding on ports.
- `hostNetwork: true` placing the pod in the host's `struct net`, where real `EADDRINUSE` returns.

Container overlay networks also add encapsulation headers (VXLAN ≈ 50 bytes), reducing effective MTU. That's the source of the "intermittent failures at exactly ~1500 bytes" symptom.

---

## Part 9 — Unix domain sockets, for contrast

Same `struct socket`, different `ops` (`unix_stream_ops`). `unix_stream_sendmsg()` appends the skb **directly onto the peer's receive queue**. No IP header, no TCP header, no checksum, no routing, no qdisc, no loopback device, no congestion control.

| | TCP over loopback | Unix domain socket |
|---|---|---|
| Addressing | IP:port | Filesystem path or abstract name |
| Stack traversal | Full IP + TCP | Direct queue append |
| Throughput | Baseline | Roughly 2× |
| Access control | Firewall rules | **Filesystem permissions** |
| Bonus | — | `SCM_RIGHTS` — pass file descriptors between processes; `SO_PEERCRED` — verify peer's pid/uid |

If two processes are on the same host, a Unix socket is the correct choice. `SCM_RIGHTS` in particular is how zero-downtime restarts hand live listening sockets from an old process to a new one.

---

## Part 10 — Inspecting the structures

```bash
# Per-socket internals — the direct readout of the structs above
ss -tinme
#   skmem:(r<rmem_alloc>,rb<rcvbuf>,t<wmem_alloc>,tb<sndbuf>,f,w,o,bl,d)
#   plus cwnd, rtt/rttvar, retrans, pacing_rate, bytes_acked, timer state

# Listen queue health
ss -ltn                              # Recv-Q = current depth, Send-Q = backlog limit
nstat -az | grep -i -E 'ListenOverflow|ListenDrop|BacklogDrop'

# File descriptors
ls -l /proc/<pid>/fd | grep socket   # socket:[inode]
cat /proc/<pid>/limits | grep files  # fd ceiling

# Raw kernel tables
cat /proc/net/tcp                    # ehash dump: local, remote, state, tx/rx queue, inode
cat /proc/net/sockstat               # global sock/tcp memory accounting
cat /proc/net/unix                   # unix domain sockets

# Buffer tuning
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.ipv4.tcp_mem
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog

# Trace the kernel path live
sudo bpftrace -e 'kprobe:tcp_v4_rcv { @[comm] = count(); }'
sudo bpftrace -e 'kprobe:tcp_drop { @[kstack] = count(); }'
sudo perf trace -e 'syscalls:sys_enter_accept*' -p <pid>
sudo strace -f -e trace=network -p <pid>
```

### Symptom → structural cause

| Symptom | What's actually happening in the kernel |
|---|---|
| `ECONNREFUSED` instantly | `ehash` and `lhash` lookup both missed → RST generated |
| Connect hangs, then times out | SYN sent, no SYN-ACK; sock stuck in `TCP_SYN_SENT`, retrying with exponential backoff. Firewall dropping silently |
| Many `CLOSE_WAIT` | `struct file` refcount never hit zero → FIN never sent. An fd leak, usually a dup/fork/pool copy |
| Many `TIME_WAIT` on client | Normal. The sock is retained past `close()` for 2×MSL. Fix with connection pooling, not sysctl |
| `accept()` slow, clients time out | Accept queue at `min(backlog, somaxconn)`; final ACKs being dropped. Check `ListenOverflow` |
| `send()` blocking | `sk_wmem_alloc` ≥ `sk_sndbuf` — the retransmit queue is full of unacknowledged data |
| `TCP ZeroWindow` | `sk_rcvbuf` full because the receiving **application** isn't calling `recv()` fast enough |
| Throughput plateau | `sk_rcvbuf` (and thus RWND) smaller than the BDP, or window scaling stripped by a middlebox |
| `EADDRINUSE` on restart | Bind-hash collision against a `TIME_WAIT` sock. Set `SO_REUSEADDR` |
| `EMFILE` / "too many open files" | fd table hit `RLIMIT_NOFILE`. Almost always a leak, not a limit problem |
| Silent packet drops under load | `sk_backlog` or qdisc overflow. Check `nstat -az \| grep -i drop` |

---

## Glossary

| Term | Full form / meaning |
|---|---|
| fd | File Descriptor — small integer indexing the process's open-file table |
| VFS | Virtual File System — the abstraction that lets sockets act like files |
| sockfs | In-memory pseudo-filesystem where socket inodes live |
| TCB | Transmission Control Block — in Linux, `struct tcp_sock` |
| skb | `sk_buff` — the kernel's packet buffer structure |
| ehash | Established-connections hash table, keyed by 5-tuple |
| bhash | Bind hash table, keyed by port |
| lhash2 | Listener hash table |
| NAPI | New API — interrupt-mitigation polling in the NIC driver |
| softirq | Deferred kernel work context where packet processing runs |
| TSQ | TCP Small Queues — per-socket cap on bytes queued in the device layer |
| netns | Network namespace — the `struct net` that isolates the entire stack |
| SCM_RIGHTS | Ancillary message type for passing fds over a Unix socket |
| BDP | Bandwidth-Delay Product — bandwidth × RTT |

---

## Summary

A socket is a file descriptor pointing at a chain of kernel structures: `struct file` → `struct socket` → `struct sock` → `struct tcp_sock`. That chain holds two buffers, three receive queues, a write queue, a retransmit queue, a wait queue, and a state machine.

Your application only ever moves bytes between user space and those buffers. Everything else — handshakes, retransmission, ordering, congestion control, demultiplexing — happens in softirq context with no application involvement whatsoever.

Nearly every socket-level production bug is one of four things: **a queue that filled** (accept queue, receive buffer, backlog), **a refcount that never reached zero** (fd leak → `CLOSE_WAIT`), **a buffer smaller than the BDP** (throughput ceiling), or **a namespace/MTU mismatch** (silent drops in overlay networks).
