# The Connection Gateway (WebSocket Fleet) — WhatsApp-Scale HLD (Detailed)

> Every real-time chat system has a layer most designs gloss over: the **Gateway** — the fleet of servers that hold the **long-lived WebSocket connections**. At WhatsApp scale (~200M concurrent sockets) this layer is a serious engineering problem in its own right: connections are **stateful and sticky**, bounded per node by **file descriptors and memory**, routed across nodes by a **registry + pub/sub backplane**, and load-balanced as **long-lived** (not request/response) traffic. This doc covers what the Gateway is, how it's used end-to-end, the per-node limits and the tech that hits them, and the cross-node routing, scaling, and failure handling.

> 💡 **The one idea that frames everything:** a Gateway connection is **stateful** — a specific user's socket lives in the memory of one specific node. That single fact drives the whole design: you can't load-balance it like stateless HTTP, you need a registry to find a user's node, a backplane to route messages between nodes, and special handling for deploys/reconnects. The Gateway's job is to turn "millions of pinned, stateful sockets" into something the rest of the system can treat as "send event to user X."

---

## Table of Contents
1. [What the Gateway is and why it exists](#1-what-the-gateway-is-and-why-it-exists)
2. [The connection-fleet sizing (the numbers)](#2-the-connection-fleet-sizing-the-numbers)
3. [What actually caps a single node](#3-what-actually-caps-a-single-node)
4. [The I/O model — the architectural decision](#4-the-io-model--the-architectural-decision)
5. [Which tech to use (and capacity per node)](#5-which-tech-to-use-and-capacity-per-node)
6. [OS / kernel tuning that unlocks the numbers](#6-os--kernel-tuning-that-unlocks-the-numbers)
7. [How a connection is used — the lifecycle](#7-how-a-connection-is-used--the-lifecycle)
8. [The stateful problem: registry + backplane](#8-the-stateful-problem-registry--backplane)
9. [Message flow through the Gateway (online & offline)](#9-message-flow-through-the-gateway-online--offline)
10. [Load-balancing long-lived connections](#10-load-balancing-long-lived-connections)
11. [Deploys, reconnect storms, draining](#11-deploys-reconnect-storms-draining)
12. [Backpressure & flow control](#12-backpressure--flow-control)
13. [The Architecture](#13-the-architecture)
14. [Capacity estimation](#14-capacity-estimation)
15. [Failure modes](#15-failure-modes)
16. [How to present this in the interview](#16-how-to-present-this-in-the-interview)
17. [Common mistakes to avoid](#17-common-mistakes-to-avoid)
18. [TL;DR](#18-tldr)

---

## 1. What the Gateway is and why it exists

The **Gateway** (a.k.a. edge / connection / WebSocket server) is the **front door that holds clients' persistent connections.** On launch, each client opens **one long-lived WebSocket** to a Gateway and keeps it open for the whole session; *all* real-time traffic — messages, receipts, presence, typing — multiplexes over that one connection.

The Gateway's responsibilities:
- **Terminate** the WebSocket (TLS, auth on connect).
- **Hold** the live socket in memory for the session.
- **Heartbeat** the client to detect dead connections.
- **Route** inbound events from the client to backend services, and deliver outbound events from the backend to the right client socket.

The rest of the backend (chat service, presence, etc.) is **stateless and request/response**; the Gateway is the **stateful** layer that absorbs the complexity of persistent connections so the backend can just say "deliver this to user X."

> 💡 **Why a separate Gateway layer at all?** Because connection-holding and business logic have opposite scaling profiles. The chat service is CPU/DB-bound and stateless (scale by requests). The Gateway is **memory-bound and stateful** (scale by *connections*). Splitting them lets you scale each independently and keeps the messy connection lifecycle out of business code.

---

## 2. The connection-fleet sizing (the numbers)

If ~10% of 2B users are online at once:

| Quantity | Value |
|:---------|:------|
| Concurrent sockets | **200,000,000** |
| Gateway nodes @ 1M sockets/node | **~200 nodes** |
| Gateway nodes @ 500K/node | **~400 nodes** |
| Gateway nodes @ 250K/node | ~800 nodes |
| Memory @ ~30 KB/socket | **~29 GB/node** (1M sockets) → **~5.6 TB fleet** |
| File descriptors | **~1M per node** (default limit is 1024 → must raise) |
| Message throughput | ~1.16M msgs/sec fleet → **~6K msgs/sec/node** |
| Heartbeats | ~6.7M/sec fleet (every 30s) |

The striking number: **~6K messages/sec/node but 10–48 GB RAM/node.** The fleet is **memory-bound, not CPU-bound** — most of those millions of connections are *idle most of the time* (people aren't typing every second). Sizing is dominated by "how many idle sockets can one box hold," which is RAM + file descriptors + I/O model — not message-processing CPU.

> 💡 **The sizing insight to state:** "Chat connections are mostly idle, so the fleet is memory-bound, not CPU-bound. Each node holds ~0.5–1M sockets limited by RAM and file descriptors; at 200M concurrent that's a few hundred nodes. The hard part isn't processing messages — it's *holding* the connections." That reframes the problem correctly.

---

## 3. What actually caps a single node

Three real limits per node (the "C10K → C10M" story):

**1. File descriptors.** Every socket is an FD. Linux defaults to `ulimit -n = 1024` — you'll stall instantly. Raise per-process limits and `fs.file-max` to the millions. One connection ≈ one FD.

**2. Memory — usually the binding constraint.** Each connection costs kernel socket buffers (send+recv, several KB, tunable) **plus** the app's per-connection object (the WebSocket state, user/session). At ~10–50 KB/socket, **1M sockets ≈ 10–50 GB**. Shrinking buffers and trimming the per-connection object is the lever that raises density.

**3. The myth to debunk: it is NOT 65,535.** People think a server caps at ~65K connections because of the port range. **Wrong for a server.** A TCP connection is the 4-tuple `(client IP, client port, server IP, server port)`; the Gateway listens on *one* port (443), and connections are distinguished by the *client's* (IP, port). Inbound capacity is bounded by FDs/RAM, not your single listen port. (The 65K limit only bites a single *client* opening many sockets to one server — which is why load-testing from one box misleads — or your *outbound* backplane connections.)

> 💡 **Bust the 65K myth proactively.** "A server isn't limited to 65K connections — that's the 4-tuple; the server uses one port and distinguishes clients by their (IP, port). The real caps are file descriptors and memory." Saying this signals you actually understand TCP, not just repeat folklore.

---

## 4. The I/O model — the architectural decision

The single most important per-node choice:

- **Thread-per-connection (blocking I/O).** Each socket needs a thread (~1 MB stack + context-switch cost). Caps you at a **few thousand** connections — the original **C10K problem**. Avoid for a connection fleet.
- **Event-driven / async non-blocking** (`epoll` on Linux, `kqueue` on BSD). A handful of threads run an event loop multiplexing *huge* numbers of mostly-idle sockets. **This is the only way to reach hundreds of thousands to millions per node.** Every viable tech below uses it.

The proof point: **WhatsApp ran ~2–3M connections per server** (Erlang on FreeBSD/`kqueue`). The industry moved from "C10K" to "C10M" precisely because of event-driven I/O.

> 💡 **The line:** "I'd never use thread-per-connection for the Gateway — that's C10K. An event-driven loop on epoll/kqueue serves millions of mostly-idle sockets with a few threads. That choice, plus FD/memory tuning, is what makes a million-connection node possible."

---

## 5. Which tech to use (and capacity per node)

| Tech / runtime | How it scales | Realistic conns/node* | Notes / who uses it |
|:---------------|:--------------|:----------------------|:--------------------|
| **Erlang / Elixir (BEAM)** | lightweight preemptive processes | **1–2M+** | Gold standard. **WhatsApp** (Erlang), **Discord** (Elixir/Phoenix Channels). Built-in distribution + fault tolerance — ideal for connection fleets. |
| **Go** | goroutines on the epoll netpoller | **100K–~1M** | Best simplicity/performance balance. Libs: `gorilla/websocket`, `coder/websocket`, `gobwas/ws` (zero-alloc, built for millions). Or **Centrifugo** (ready-made). |
| **Java/JVM — Netty** | async NIO (epoll) | **100K–~1M** | High perf; Netty directly, or Vert.x, Reactor Netty / Spring WebFlux, **Akka/Pekko** (actor model ≈ Erlang on JVM). Watch GC at scale. |
| **Node.js** | single-thread libuv event loop | **tens of K (`ws`) – ~1M (`µWebSockets.js`)** | Easiest to build; `ws`, `Socket.IO` (rooms + Redis adapter), `µWebSockets.js` (C++ core). Scale via clustering. |
| **Rust — tokio** | async runtime, no GC | **millions feasible** | `tokio-tungstenite`, `axum`. Lowest mem/conn, no GC pauses; steeper curve. |
| **C/C++ — µWebSockets** | raw epoll | **millions** | Max performance, max effort. |
| **Managed** | provider holds the sockets | (their problem) | **Ably, Pusher, PubNub**, **AWS API Gateway WebSocket / AppSync**, **Azure SignalR** — pay per connection/message. |

\*Order-of-magnitude; depends on hardware, message rate, payload size, and OS tuning.

> 💡 **Pick with a reason, name the trade-off.** "I'd use Erlang/Elixir or Go for the Gateway — event-driven, high connection density, and (BEAM) built-in distribution and fault isolation so one crashed connection process doesn't take others down. Rust if memory/GC is critical, managed (Ably/API-Gateway-WS) if I don't want to operate the fleet." The *why* (density + fault isolation + no GC pauses) beats naming a library.

---

## 6. OS / kernel tuning that unlocks the numbers

Choosing Go or Erlang isn't enough — you must tune the kernel, or you'll cap at the defaults:

| Knob | Why |
|:-----|:----|
| `ulimit -n`, `fs.file-max` | raise FD limits to millions (one FD per socket) |
| `net.ipv4.tcp_rmem` / `tcp_wmem` / `tcp_mem` | shrink per-socket buffers → pack more connections per GB |
| `net.core.somaxconn`, backlog | larger accept queue for connection bursts |
| `SO_REUSEPORT` | multiple worker threads/processes accept on the same port → spread accept load |
| `net.ipv4.ip_local_port_range`, conntrack limits | matter for **outbound** (backplane) connections |
| TCP keepalive / app heartbeats | detect and reap dead sockets so they free FDs/RAM |

> 💡 **Mention tuning unprompted.** "A million-connection node needs kernel tuning — raise file-descriptor limits, shrink TCP buffers to fit more sockets per GB, `SO_REUSEPORT` to spread accepts. The tech choice sets the ceiling; the kernel settings let you reach it." It shows you've operated such systems, not just drawn them.

---

## 7. How a connection is used — the lifecycle

A single connection's life, end to end:

```
1. CONNECT      client opens WebSocket to Gateway (via LB); TLS handshake.
2. AUTH/HELLO   client sends token; Gateway authenticates; "hello" returns
                heartbeat interval + initial state; Gateway REGISTERS user→node.
3. HEARTBEAT    client pings every ~30s; Gateway tracks liveness (and refreshes
                presence). Missed heartbeats → connection considered dead → reap.
4. SEND (up)    client sends a message over the socket → Gateway hands it to the
                chat service (persist, then route to recipient).
5. RECEIVE (down) backend has an event for this user → routed to THIS Gateway via
                the backplane → Gateway pushes it down the user's socket.
6. DISCONNECT   socket closes (clean) or heartbeats stop (dead) → Gateway
                DEREGISTERS user→node; presence flips.
7. RECONNECT    client reconnects (backoff+jitter) and RESUMES: sends last-seen
                sequence id; backend replays missed events instead of full resync.
```

The two mechanics that make "keeps it open" real:
- **Heartbeats** detect dead/half-open sockets (a broken TCP connection looks alive otherwise) and keep NAT/firewall mappings warm.
- **Reconnect + resume** — connections *will* drop (wifi blips, deploys). The client reconnects with **exponential backoff + jitter** and **resumes** from a sequence number so the server replays only the gap, not the whole state.

> 💡 **Resume is the detail that separates juniors from seniors.** "On reconnect the client sends the sequence id of the last event it saw, and the Gateway/backend replays only what was missed — not a full re-sync. Without resume, every reconnect is a heavy cold start, and at this scale reconnects are constant." Always bring up resume.

---

## 8. The stateful problem: registry + backplane

Here's the crux. A user's socket lives on **one specific node's memory**. When the chat service wants to push a message to **Bob**, it has no idea *which* of the 200 nodes holds Bob's socket. Two pieces solve this:

**1. The Registry (`user → gateway node`).** When Bob connects, his Gateway records "Bob is on node-47" (in Redis, or a distributed map). Lookups route outbound events.

**2. The Pub/Sub backplane.** Rather than every service knowing the registry, the common pattern: the backend **publishes** "event for Bob" to a channel; the node holding Bob's socket is **subscribed** and delivers it. Backplanes: **Redis pub/sub**, **NATS**, **Kafka** (durable), or a service mesh.

```
chat service: "deliver msg to Bob"
        │ publish to channel (e.g. user:Bob  or  node:47)
        ▼
   Pub/Sub backplane  ───────────────► node-47 (subscribed; holds Bob's socket)
                                                │ push down Bob's WebSocket
                                                ▼
                                               Bob's client
```

Two routing styles:
- **Publish per-user channel** (`user:{id}`) — node holding the user subscribes to that user's channel. Simple; many channels.
- **Registry lookup → publish per-node channel** (`node:{id}`) — look up Bob's node, publish to that node. Fewer channels; needs the registry.

> 💡 **This is THE WebSocket-scaling answer.** "Connections are stateful and pinned to a node, so I keep a `user→node` registry and route outbound messages over a pub/sub backplane: the backend publishes, the node holding the socket is subscribed and delivers. That's how a stateless backend reaches a stateful connection fleet." If you say nothing else about the Gateway, say this.

---

## 9. Message flow through the Gateway (online & offline)

**Alice → Bob, Bob ONLINE:**
```
Alice's socket → Gateway A → Chat Service (persist to Cassandra, get seq id)
   → ack to Alice (one tick) → look up Bob in registry → publish to Bob's channel
   → Gateway B (holds Bob's socket) → push to Bob → Bob's client acks delivery
   → relay delivery ack to Alice (two ticks)
```

**Alice → Bob, Bob OFFLINE (no registry entry):**
```
Alice's socket → Gateway A → Chat Service persists message (Cassandra) → ack Alice (one tick)
   → no Bob in registry → Notification Service → APNs/FCM push ("Alice: new message")
   → message waits in store; when Bob reconnects, RESUME/sync replays missed messages
   → Bob's device acks delivery on receipt → two ticks
```

The Gateway is the pivot: inbound it forwards to the chat service; outbound it's the *only* component that can actually push to a live client. When the client is offline, delivery falls back to **store + push-notification + sync-on-reconnect** (the Gateway plays no part until reconnect).

> 💡 **Connect it to durability:** "The Gateway only delivers to *live* sockets. Messages are durable in Cassandra regardless; if the recipient is offline, the registry has no entry, so we fall back to a push notification and replay on reconnect. The socket is the fast path, not the source of truth."

---

## 10. Load-balancing long-lived connections

You **can't** balance WebSockets like stateless HTTP (round-robin each request) — a connection is one long-lived TCP stream pinned to a node.

- **Use an L4 (TCP) load balancer** (e.g. **AWS NLB**, HAProxy in TCP mode) in front of the Gateway fleet: it balances *connections*, not requests, and passes the long-lived stream straight through. (An L7/ALB also supports WebSocket upgrades, but L4 is leaner for raw connection volume.)
- **Connections are sticky by nature** — once established, a socket stays on its node for its lifetime; you can't reshuffle a live socket.
- **Balance on connect** — spread *new* connections across nodes (least-connections / consistent hashing). Existing ones don't move.
- **DNS / anycast + multiple regions** in front for geo-routing and the first hop.

> 💡 **The LB nuance:** "WebSockets are balanced at L4 on *connect* and then pinned — you balance the act of connecting, not each message. So I watch connection-count skew across nodes (least-connections) and handle node loss by mass-reconnect, since you can't migrate a live socket." Naming "balance on connect, sticky thereafter" is the senior detail.

---

## 11. Deploys, reconnect storms, draining

The stateful fleet's nastiest operational problem: **restarting a node drops all its connections at once** — up to a million clients reconnect simultaneously → a **thundering herd / reconnect storm** that can cascade (the surge overloads the remaining nodes, which then fail, dropping *more* connections...).

Mitigations:
- **Graceful connection draining** — on deploy, stop accepting new connections, let existing ones migrate/close gradually, roll nodes slowly (not all at once).
- **Reconnect with exponential backoff + JITTER** — jitter is essential; without it, all clients retry in lockstep and re-create the storm.
- **Resume (cheap reconnect)** — replay-from-sequence makes each reconnect light, so a storm is survivable.
- **Capacity headroom** — run the fleet below max so it can absorb a node's worth of reconnects.
- **Spread/staggered rollouts** — deploy a few nodes at a time.

> 💡 **The reconnect-storm answer:** "Killing a Gateway node mass-disconnects ~1M clients that all reconnect at once — a thundering herd. I mitigate with graceful draining (roll slowly), **jittered** exponential backoff (so they don't retry in lockstep), cheap resume (replay from a sequence id), and capacity headroom to absorb a node's reconnects." This is a classic senior follow-up — have it ready.

---

## 12. Backpressure & flow control

A **slow consumer** (a client on bad network, or one subscribed to a busy channel) must not make the Gateway buffer events unboundedly — that's a memory blowup across millions of connections.

- **Bounded per-connection send queues** — cap buffered bytes/messages per socket.
- **Drop / coalesce policy on overflow** — for droppable data (presence, typing) drop oldest; for messages, slow the producer or disconnect the laggard (it'll resync on reconnect).
- **Apply backpressure to the backplane** — don't pull faster than you can push to clients.

> 💡 **Flow control matters at fleet scale:** "Each connection has a bounded send buffer; a slow client gets coalesced/dropped presence and, past a limit, disconnected to resync later — never an unbounded server-side buffer. Multiplied by millions of sockets, an unbounded queue is an OOM waiting to happen."

---

## 13. The Architecture

```
        clients (mobile/web)  ── 1 long-lived WebSocket each ──┐
                                                               ▼
                                                    ┌──────────────────┐
                                                    │  L4 Load Balancer │  (NLB; balance on CONNECT)
                                                    └─────────┬────────┘
                            ┌─────────────────────────────────┼─────────────────────────────────┐
                            ▼                                  ▼                                  ▼
                  ┌──────────────────┐              ┌──────────────────┐              ┌──────────────────┐
                  │  Gateway node 1   │   …          │  Gateway node 47  │   …          │ Gateway node 200 │
                  │ ~1M sockets       │              │ holds Bob's socket│              │                  │
                  │ epoll event loop  │              │ heartbeats        │              │                  │
                  └───┬───────┬──────┘              └───┬───────┬──────┘              └───────┬──────────┘
        register user→node│   │ pub/sub                 │       │                              │
                          ▼   ▼                         ▼       ▼                              ▼
                  ┌──────────────┐            ┌────────────────────────┐          ┌────────────────────┐
                  │   Registry    │            │   Pub/Sub backplane     │          │  Chat / Presence    │
                  │ user→node     │◄──────────►│ (Redis/NATS/Kafka)      │◄────────►│  / Notification svc │
                  │ (Redis)       │            │  route msg → right node │          │  (stateless)        │
                  └──────────────┘            └────────────────────────┘          └─────────┬──────────┘
                                                                                              ▼
                                                                                   Cassandra (messages),
                                                                                   APNs/FCM (offline push)
```

- **Gateway fleet** — stateful, memory-bound, event-driven; holds the sockets, heartbeats, registers users.
- **Registry** — `user → node` for outbound routing.
- **Backplane** — delivers cross-node events to the node holding a given socket.
- **Backend services** — stateless; persist, decide, and publish "deliver to user X" to the backplane.

---

## 14. Capacity estimation

| Quantity | Value | Note |
|:---------|:------|:-----|
| Concurrent sockets | 200M | 10% of 2B |
| Gateway nodes | ~200 (@1M) / ~400 (@500K) | choose density vs blast-radius |
| RAM/node | ~10–48 GB | at 10–50 KB/socket × 1M — **the binding constraint** |
| Fleet RAM | ~1.9–9.3 TB | for connection state alone |
| FDs/node | ~1M | raise `ulimit`/`fs.file-max` |
| Msgs/sec/node | ~6K | **memory-bound, not CPU-bound** |
| Heartbeats | ~6.7M/sec fleet | keepalive + presence refresh |

**Density trade-off:** fewer, bigger nodes (1M/node) = cheaper but each failure mass-disconnects 1M clients (bigger reconnect storm, bigger blast radius). More, smaller nodes (250–500K) = more headroom and smaller blast radius, higher cost. State the trade.

> 💡 **The estimation tells:** memory-bound (size by RAM/FDs, not CPU); blast-radius vs density (node size = how many clients a failure drops); idle-heavy (provision for *holding* connections, with headroom for reconnect surges).

---

## 15. Failure modes

| Failure | Effect | Handling |
|:--------|:-------|:---------|
| Gateway node crash | ~1M clients disconnect | jittered reconnect + resume; headroom; registry entries expire (TTL) |
| Reconnect storm | surge overloads survivors | draining, staggered deploys, backoff+jitter, cheap resume |
| Registry stale (node died) | outbound routed to dead node | TTL on registry entries; re-register on reconnect; backplane drops undeliverable |
| Backplane lag/outage | delivery delayed | durable backplane (Kafka) for at-least-once; clients resync on reconnect |
| Slow consumer | memory growth | bounded send queues; drop/coalesce; disconnect laggard |
| Half-open socket | "ghost" connection wastes FD/RAM | heartbeat timeout reaps it |
| Hot channel (huge group) | fan-out spike on one node | shard large groups; fan-out service; rate-limit |

> 💡 **Registry TTL is the subtle one.** "Registry entries expire by TTL and are re-written on (re)connect, so a crashed node's stale `user→node` entries self-clean and outbound routing doesn't keep targeting a dead node." It mirrors the presence heartbeat-TTL idea.

---

## 16. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Frame | "Connections are stateful and pinned — that drives the whole Gateway design." |
| Sizing | 200M sockets → ~200–400 nodes; **memory-bound, not CPU-bound**; bust the 65K myth. |
| Per-node | event-driven I/O (epoll), FDs, memory, kernel tuning; name a tech + why. |
| Lifecycle | connect→auth→heartbeat→send/recv→disconnect→**reconnect+resume**. |
| Cross-node | **registry + pub/sub backplane** — the core answer. |
| LB + ops | L4, balance-on-connect+sticky; reconnect storms (drain, jitter, resume); backpressure. |

### What to say
- *"The fleet is memory-bound — millions of mostly-idle sockets; size by RAM/FDs, not CPU."*
- *"Event-driven I/O (epoll), not thread-per-connection — that's C10K; I'd use Erlang/Elixir or Go for density and fault isolation."*
- *"Connections are stateful and pinned, so I route outbound via a `user→node` registry over a pub/sub backplane."*
- *"L4 LB, balance on connect, sticky thereafter; reconnect with jittered backoff and resume from a sequence id."*
- *"Killing a node mass-disconnects ~1M clients — drain gracefully, jitter reconnects, keep headroom."*

### Order
Frame stateful → size the fleet → per-node limits + tech → connection lifecycle → registry+backplane → LB → deploy/reconnect/backpressure.

---

## 17. Common mistakes to avoid

- ❌ **"Capped at 65K connections"** — that's the 4-tuple myth; servers are bounded by FDs/memory.
- ❌ **Thread-per-connection** — C10K; use an event loop (epoll/kqueue).
- ❌ **Treating WebSockets like stateless HTTP behind an L7 round-robin** — they're pinned; balance on connect, route cross-node via a backplane.
- ❌ **No registry/backplane** — then the backend can't find which node holds a user's socket.
- ❌ **No heartbeats** — half-open sockets leak FDs/RAM and presence goes wrong.
- ❌ **Reconnect without jitter** — synchronized retries recreate the storm.
- ❌ **No resume** — every reconnect is a heavy cold start at a scale where reconnects are constant.
- ❌ **Unbounded per-connection buffers** — slow clients OOM the node; bound + drop/coalesce.
- ❌ **All-at-once deploys** — mass-disconnect/thundering herd; drain and stagger.
- ❌ **Sizing by CPU** — it's memory/FD-bound; provision RAM + headroom for reconnects.
- ❌ **Forgetting kernel tuning** — the right tech still hits the default `ulimit 1024` wall.

---

## 18. TL;DR

### The sizing
```
200M concurrent sockets (10% of 2B) → ~200 nodes @1M, ~400 @500K
~10–48 GB RAM/node, ~1M FDs/node, but only ~6K msgs/sec/node
→ MEMORY-BOUND, not CPU-bound; idle-heavy; size by RAM + file descriptors.
```

### The per-node recipe
```
event-driven I/O (epoll/kqueue) + raised FD limits + tuned TCP buffers + SO_REUSEPORT
tech: Erlang/Elixir (WhatsApp/Discord, 1–2M/node) | Go | Netty | Rust | managed (Ably/API-GW-WS)
```

### The system recipe (stateful fleet)
```
L4 LB (balance on CONNECT, sticky)  →  Gateway fleet (holds sockets, heartbeats)
Registry (user→node, Redis, TTL)  +  Pub/Sub backplane (Redis/NATS/Kafka)
  → backend publishes "deliver to user X"; the node holding X's socket pushes it down
client: heartbeat + reconnect (backoff+JITTER) + RESUME from a sequence id
ops: graceful drain on deploy, headroom, bounded send queues (backpressure)
```

### The four things that score points
1. **Connections are stateful & pinned** → registry + pub/sub backplane to route cross-node (the core answer).
2. **Memory-bound, not CPU-bound** → size by RAM/FDs; event-driven I/O; bust the 65K myth.
3. **Reconnect = backoff + jitter + resume**; deploys need graceful draining (reconnect storms).
4. **L4 balance-on-connect + sticky**; bounded buffers for backpressure; right tech (Erlang/Go) with kernel tuning.

> **One-line philosophy:** *The Gateway turns millions of stateful, memory-bound, node-pinned sockets into a simple "deliver to user X" for the rest of the system — by holding connections on an event-driven fleet, tracking who-is-where in a registry, routing across nodes over a pub/sub backplane, and surviving the constant churn of mobile with heartbeats, jittered reconnect, resume, and graceful draining.*
