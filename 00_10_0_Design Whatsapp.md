# Designing WhatsApp (Real-Time Messaging): A Senior Interview Guide

> A practical, interview-focused reference for designing a real-time messaging system at WhatsApp scale — persistent connections, message routing, storage, presence, read receipts, group fan-out, end-to-end encryption, and media handling — with capacity math, trade-offs, and a deep bank of senior follow-up questions.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a message](#44-the-end-to-end-life-of-a-message)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Persistent Connection Model](#5-the-persistent-connection-model)
   - [5.1 Cross-Gateway Delivery: How Server A Reaches Server B](#51-cross-gateway-delivery-how-server-a-reaches-server-b)
6. [Message Routing (Online & Offline)](#6-message-routing-online--offline)
7. [Message Storage](#7-message-storage)
8. [Read Receipts & Delivery States](#8-read-receipts--delivery-states)
9. [Presence / Last-Seen](#9-presence--last-seen)
10. [Group Chat & Fan-Out](#10-group-chat--fan-out)
11. [End-to-End Encryption](#11-end-to-end-encryption)
12. [Media Handling](#12-media-handling)
13. [Scaling Summary](#13-scaling-summary)
14. [Senior Follow-Up Questions (with Answers)](#14-senior-follow-up-questions-with-answers)
15. [Quick Glossary](#15-quick-glossary)

---

## 1. How to Approach This in an Interview

This is one of the most common large-scale design questions, and the depth expected is high. The thing that makes messaging distinctive — and where you should spend your time — is the **persistent connection model**: unlike a typical request/response web service, the server must be able to **push** a message to a recipient *at any moment*, which means tracking who is connected where. That single requirement drives most of the architecture.

A good structure:

1. **Clarify requirements** — 1:1 vs groups, receipts, presence, media, encryption, offline support.
2. **Estimate scale** — messages/sec and storage/day; this justifies the connection layer and storage choice.
3. **Establish the connection model** — WebSockets + a gateway layer + a connection registry. This is the heart.
4. **Walk the message path** — send → route → deliver (online) or queue → push (offline).
5. **Go deep on the hard parts** — presence fan-out, group fan-out, ordering, and E2E encryption's implications.
6. **Cover media, scaling, and failure modes.**

Senior signal: recognizing that **presence and group fan-out are the genuinely hard scaling problems** (not the 1:1 message path), and that **E2E encryption constrains the design** (the server can't read messages, so no server-side search or content-based features).

---

## 2. Requirements

### Functional

- **1:1 chat** and **group chat** (up to ~256 members).
- **Read receipts** — sent → delivered → read.
- **Online / last-seen** presence.
- **Media sharing** — images, video, voice notes.
- **End-to-end encryption** — the server never sees plaintext.
- **Offline message queuing** — deliver when the recipient reconnects.

### Non-Functional

- **Low latency** — message delivery under ~200 ms.
- **High availability** — a messaging app that's down is useless and erodes trust instantly.
- **Strong ordering within a conversation** — messages must appear in the order intended; out-of-order chat is broken chat.
- **Massive scale** — 2B+ users.

> Note these don't require *global* strong consistency — ordering only needs to hold **within a single conversation**, which is a far cheaper guarantee than ordering everything. That observation simplifies the whole design.

---

## 3. Capacity Estimation

**Assumptions:** 2B users; 50B messages/day; ~60 bytes per message (text + metadata).

```
Messages/sec = 50 × 10⁹ / 86,400 s/day  ≈ 580,000 msgs/sec   (avg)
   (peak is higher — call it ~1.16M/sec at peak, ~2× average)

Storage/day  = 50 × 10⁹ × 60 bytes    = 3 × 10¹² B  ≈ 3 TB/day
   (≈ ~1.1 PB/year of text+metadata; media is far larger and goes to object storage, not here)
```

**Takeaways that shape the design:**
- ~580K msgs/sec (peak ~1.16M) is enormous — the system is **write-heavy**, pushing us toward a write-optimized store (Cassandra/LSM).
- 3 TB/day of message data means **tiered storage** (hot recent data vs. cold archive) and partitioning are mandatory.
- The connection layer must hold **hundreds of millions of concurrent WebSocket connections** — its own significant engineering challenge (see §4.6 for the node math).

---

## 4. Core Architecture (In Depth)

This section builds the architecture up piece by piece: the one-paragraph mental model, the diagram, then every box explained, then a full trace of a message moving through the system, and finally *why* it's split this way and where the real load lands.

### 4.1 The big picture in one paragraph

A messaging system is fundamentally different from a normal web app because of one requirement: **the server must be able to push a message to you at any instant, even though you never asked for it right then.** A normal web server only speaks when spoken to (you request, it responds). Here, when Alice messages Bob, *the server* must reach out to Bob. To do that it must (a) keep a live connection open to every online user, (b) know *which* server holds *which* user's connection, and (c) durably store every message so offline users (the common case on mobile) still get it later. Everything in the architecture exists to serve those three needs: the **Gateway** holds the live connections, the **Connection Registry** records who's connected where, the **Chat Server** runs the message logic, the **Message DB** is the durable source of truth, and the supporting services (**Presence**, **Notification**, **Media**) handle the cross-cutting concerns.

### 4.2 The diagram

```
                                            ┌───────────────────────┐
                                            │  Connection Registry  │
                                            │   (Redis)             │
                                            │  user_id → gateway     │
                                            └───────────▲───────────┘
                                                        │ lookup / register
                                                        │
 [Client A] ══(WebSocket)══►┌─────────┐                │
                            │ Gateway │────────────────┤
 [Client B] ══(WebSocket)══►│  nodes  │                │
                            └────┬────┘                │
                                 │ (message in)        │ (push out)
                                 ▼                     │
                            ┌──────────────┐           │
                            │ Chat Server  │───────────┘
                            │ (msg logic)  │
                            └──┬───┬───┬───┬──────────────────────────┐
                               │   │   │   │                          │
              persist ◄────────┘   │   │   └──► [Media Service] ──► [S3] ──► [CDN]
                  │                 │   │
                  ▼                 │   └──► [Notification Svc] ──► APNs / FCM (offline push)
        ┌──────────────────┐       │
        │   Message DB      │       └──► [Presence Service] ──► (Redis last-seen)
        │   (Cassandra)     │
        │  hot recent msgs  │──(archive)──► [S3 cold storage]
        └──────────────────┘
```

The key visual idea: a message comes **in** through a Gateway, goes to a **Chat Server** which **persists** it and **looks up** the recipient in the Registry, then goes **out** through the recipient's Gateway (if online) or to the **Notification Service** (if offline). Media and presence hang off to the side so they never clog the message path.

### 4.3 Each component, in detail

**① Client (the phone app).** Opens one long-lived **WebSocket** to a Gateway on launch and keeps it open. It does the E2E encryption/decryption locally (the server only ever sees ciphertext). On reconnect (constant on mobile) it re-registers and pulls anything it missed. It also sends **ACKs** (delivered/read receipts) back over the same socket.

**② Gateway (the connection layer) — the box people under-explain.** Its *only* job is to **hold the live WebSocket connections** and shuttle bytes between the client and the chat servers. Think of it as the "switchboard." Key properties:
- It is **horizontally scaled** — hundreds of nodes, each holding a large number of open sockets (see §4.6). A user lands on *one* gateway for the life of their connection.
- When a client connects, the gateway **writes `user_id → this_gateway_node` into the Connection Registry** so the rest of the system can find that user.
- It keeps a **local in-memory table** of `user_id → the actual socket object` for the connections *it personally holds* — this is what it uses to do the final push (see §5.1).
- It runs **heartbeats** to detect dead connections (a phone that vanished without a clean close) and removes the stale registry entry when one dies.
- It is **dumb on purpose** — it holds no business logic, so it can crash/restart and clients just reconnect to another node. Keeping logic *out* of the gateway is what lets the connection layer scale independently of the message logic.

**③ Connection Registry (Redis) — "who is connected where."** A fast in-memory map: `user_id → gateway_node`. This is the linchpin that makes cross-server delivery possible. Without it, a chat server holding Alice's message would have no idea which of 200 gateways is holding Bob's socket. It's small (tens of GB even for hundreds of millions of connections — see §4.6) and lives in a Redis cluster. Entries are created on connect, refreshed by heartbeat, and removed on disconnect/timeout. **Important:** the registry stores the *address* (which node) — it does **not** move messages. The actual A→B transport is a separate mechanism, detailed in §5.1.

**④ Chat Server (the brain).** This is where the actual message logic runs. For each incoming message it:
1. **Persists** the message to the Message DB (durability first — this is the source of truth).
2. **Looks up the recipient** in the Connection Registry.
3. **Routes** it: if the recipient is online, forward to their gateway to push down the socket (how this forward physically happens is §5.1); if offline, leave it stored and tell the Notification Service to send a push.
4. **Triggers receipts/notifications** and updates message status as ACKs come back.
Chat servers are **stateless** with respect to connections (they don't hold sockets — gateways do), so they scale horizontally and any chat server can handle any message.

**⑤ Message DB (Cassandra) — the durable source of truth.** Write-optimized (LSM engine), time-series-friendly, partitioned by `conversation_id` and sorted by timestamp (see §7). At ~580K writes/sec, raw write throughput dominates, which is exactly Cassandra's strength. The live socket is just a *fast path*; this store is what guarantees a message is never lost and can be delivered whenever the recipient finally reconnects.

**⑥ Presence Service.** Tracks online/last-seen, backed by Redis (last-heartbeat timestamps). Deliberately kept **approximate** to avoid fan-out storms (see §9). Split out as its own service because its traffic pattern (constant connect/disconnect flapping) is totally different from messaging and you don't want it contending with the message path.

**⑦ Notification Service → APNs / FCM.** When a recipient is **offline**, there's no socket to push to — so the chat server hands off to this service, which sends a push notification through Apple's (APNs) or Google's (FCM) platform services to wake the app. This is essential because on mobile, users are *usually* disconnected, so "store + notify" is the normal path, not the exception.

**⑧ Media Service → S3 → CDN.** Handles images/video/voice. The firm rule: **media never flows through the chat servers** (it would saturate them). Clients upload directly to S3 via a pre-signed URL and download via CDN; the chat message carries only a tiny URL+key reference (see §12).

### 4.4 The end-to-end life of a message

Here is exactly what happens when **Alice sends "hi" to Bob**, step by step:

```
1.  Alice's app encrypts "hi" (E2E) and sends the ciphertext over her WebSocket
        → arrives at Gateway-7 (the gateway holding Alice's connection).

2.  Gateway-7 forwards it to a Chat Server (any one — they're stateless).

3.  Chat Server PERSISTS the message to Cassandra
        partition = conversation_id(Alice,Bob), sort key = timestamp.
        (Durability FIRST — if everything else fails, the message is safe.)

4.  Chat Server looks up Bob in the Connection Registry (Redis):
        "Where is Bob connected?"

5a. Bob is ONLINE → Registry says "Bob is on Gateway-12."
        Chat Server forwards the message to Gateway-12   ← HOW this A→B hop works: §5.1
        → Gateway-12 finds Bob's socket in its LOCAL table and pushes it down
        → Bob's app receives & decrypts it.            (<200ms total)

5b. Bob is OFFLINE → Registry has no entry for Bob.
        Message simply stays stored in Cassandra (already done in step 3).
        Chat Server tells the Notification Service → APNs/FCM sends a push
        → Bob's phone shows "Alice: new message" and wakes the app.

6.  Bob's device sends a "delivered" ACK back over its socket → Chat Server
        updates status (async) → notifies Alice (one grey tick → two ticks).

7.  When Bob opens the chat, his device sends a "read" ACK
        → status → "read" → Alice sees blue ticks.

8.  (If Bob was offline) On reconnect, Bob's app re-registers in the Registry
        and PULLS all undelivered messages from Cassandra, which then get
        marked delivered.
```

The single most important thing to notice: **steps 3 (persist) and 5a (push) are independent.** The push is an optimization for "recipient happens to be online right now"; the durable store is what actually guarantees delivery. That separation is the whole design.

### 4.5 Why this split? (the design rationale)

Each separation exists for a concrete reason — be ready to justify them:

- **Gateway separate from Chat Server** — connections and message-logic scale on *different axes*. You might need 200 gateway nodes (to hold sockets) but far fewer chat servers (to run logic), or vice-versa. Splitting them lets each scale independently, and lets a gateway crash without losing message logic (clients just reconnect elsewhere).
- **Registry separate from everything** — "who's connected where" changes constantly and must be readable by *any* chat server in O(1). A shared fast map (Redis) decouples connection location from any single node.
- **Durable store as source of truth, socket as fast path** — because mobile clients are usually offline, you *cannot* treat the live socket as the delivery guarantee. Persist first, push best-effort.
- **Presence / Notification / Media as separate services** — each has a wildly different traffic shape (flapping presence, bursty pushes, huge media bytes). Isolating them stops one from starving the message path and lets each scale and fail independently.

### 4.6 Where the load actually goes

A senior is expected to know *which* part is hard. The math (verified):

- **Connections:** if ~10% of 2B users are online at once = **200M concurrent sockets**. At ~1M sockets/node that's **~200 gateway nodes** (~400 at 500K/node). This connection fleet is a real engineering problem in its own right (file descriptors, memory per socket, load balancing long-lived connections).
- **Registry:** 200M entries × ~50 bytes ≈ **~10 GB** — trivially fits across a Redis cluster. The registry is *not* the bottleneck.
- **Message writes:** ~580K/sec (peak ~1.16M) — this is why the store must be write-optimized (Cassandra).
- **Receipts:** up to 2 ACKs per message → on the order of **~1M ACK-writes/sec**, which is why receipt writes are **async and batched** (§8), never on the critical path.
- **Presence:** potentially *larger* than message traffic if done naively (§9) — the genuinely deceptive scaling trap.

> 💡 **The senior framing:** "The 1:1 message path is the easy part — persist, look up, push. The hard parts are (1) holding hundreds of millions of live connections, (2) presence fan-out, and (3) group fan-out under E2E. I'd spend my time there." Saying this up front signals you know where the difficulty actually lives.

---

## 5. The Persistent Connection Model

The defining design choice. Because the server must push messages instantly, each client opens a **long-lived WebSocket** to a gateway and keeps it open (see the protocols guide for why WebSocket over polling — lower latency, far less overhead, true bidirectional push).

The challenge this creates: with hundreds of millions of clients connected across thousands of gateway nodes, **how do you find which gateway holds a given user's connection?** The answer is the **connection registry** (§4.3 ③): when a user connects, the gateway records `user_id → its own node` in **Redis**. To deliver a message to that user, any chat server looks up their gateway in Redis and forwards the message there, which pushes it down the open socket.

Practical concerns:

- **Heartbeats / keepalives** detect dead connections (a client that vanished without closing cleanly) so stale registry entries are cleaned up.
- **Connection state is the thing you shard by `user_id`** — each user's connection lives on one gateway.
- **Reconnection** must be cheap and frequent (mobile networks drop constantly); on reconnect the client re-registers and pulls any queued messages.

> Why not just keep a giant in-memory table on one server? Because no single machine can hold 200M sockets or be a single point of failure for the whole app. The gateway fleet + shared registry pattern is precisely how you spread that state across many nodes while still being able to find any user in one Redis lookup.

### 5.1 Cross-Gateway Delivery: How Server A Reaches Server B

This is the step most designs hand-wave, and a favorite interview drill: the registry tells you *where* the recipient is, but **it does not move the message.** Re-read the routing step — "look up Bob's gateway in Redis and forward the message there." *Forward how?* Redis just returned an address; **something still has to physically carry the bytes from the sending gateway to the receiving one.** This section fills that gap.

#### Name the two connections precisely

The confusion dissolves the moment you separate two *different* WebSockets:

- **Alice's** WebSocket is held by **Gateway A**.
- **Bob's** WebSocket is held by **Gateway B**.

Alice's message enters through A, but it must leave through B — because that's where Bob's socket physically lives. So the real question is: *how does the message travel from process A to process B, two separate machines that may never have talked before?*

#### The key realization: a socket is pinned to its process

A WebSocket (and the TCP connection under it) is a **live, stateful, kernel-level object owned by the single process that accepted it.** Gateway A **cannot** write to Bob's socket — A doesn't hold it; only B does. You also cannot "move" or "teleport" the socket to A. Therefore delivery to Bob **must physically happen on Gateway B.** A's only job is to get the message *to* B; **B does the final socket write.** That single constraint is the whole reason the registry + an internal transport exist.

#### Two maps, not one (this is the crux)

You actually need *two* lookup structures, and conflating them is what causes the confusion:

| Map | Where it lives | Scope | Answers |
|:--|:--|:--|:--|
| **Connection Registry (Redis)** | shared, global | all users | "*which gateway* holds Bob?" |
| **Local socket table** | in each gateway's RAM | only that gateway's own connections | "which actual *socket object* on THIS machine is Bob's?" |

Gateway A uses the **global** registry to discover "Bob is on B." Gateway B uses its **local** in-memory table to find Bob's actual socket object and write to it. The registry is the directory of *nodes*; the local table is the directory of *sockets on this node*. You need both.

#### How A physically reaches B — two standard mechanisms

Gateways are ordinary backend servers on the same internal network. Once A knows "Bob is on B," it delivers over an **internal server-to-server connection** — never over Bob's WebSocket. Two standard approaches, and you should know both:

**Option 1 — Direct RPC (a mesh of internal connections).** The registry stores B's network *address*; A opens/reuses a persistent internal connection (gRPC / plain TCP) to B and calls it.

```
Gateway A → Redis: "Bob → Gateway B (10.0.0.12:7000)"
Gateway A → internal gRPC/TCP call to Gateway B:  deliver(msg, Bob)
Gateway B → finds Bob's socket in its LOCAL table → writes bytes down the socket
```

Every gateway can talk to every other gateway (an N×N mesh of persistent internal links). Simplest and lowest-latency; downsides are managing the mesh and handling B being down/moved.

**Option 2 — Pub/Sub backplane (the more common answer at scale).** Route through an internal message bus (Redis Pub/Sub, NATS, or Kafka). A doesn't need B's address or a direct link — it just **publishes**, and whichever gateway holds Bob is **subscribed** and receives it.

```
Bob connects to B  → B SUBSCRIBES to a channel for Bob
Alice's msg at A   → PUBLISH the message to Bob's channel
Bus routes to the subscriber (B) → B finds Bob's socket LOCALLY → pushes down the socket
```

Two channel-granularity styles:
- **Per-user channel** (`user:Bob`) — A publishes to `user:Bob` without even consulting the registry; B subscribed to it on Bob's connect. Simple, but you have millions of channels/subscriptions.
- **Per-gateway channel** (`gateway:B`) — A first reads the registry ("Bob → B"), then publishes to `gateway:B`, which B alone subscribes to. Far fewer channels; the registry does the user→gateway resolution. **This is the usual production choice.**

Trade-off: direct RPC is lower-latency but **couples** gateways (mesh + address management); the backplane **decouples** them (A never needs to know B exists) at the cost of an extra hop through the bus. Large systems usually pick the backplane for the decoupling and to absorb bursts.

#### The corrected end-to-end path

```
1. Alice's msg → Gateway A → Chat Server
2. Chat Server PERSISTS to Cassandra (durability first)
3. Chat Server looks up Redis registry: "Bob → Gateway B"
4. Route the message to Gateway B via EITHER
     (a) a direct gRPC/TCP call to B's address, OR
     (b) publishing to a channel B is subscribed to (pub/sub backplane)
5. Gateway B receives it on its internal connection / subscription
6. Gateway B finds Bob's WebSocket in its LOCAL in-memory table
7. Gateway B writes the bytes down Bob's socket → Bob's app receives & decrypts
```

Step 6 resolves the puzzle: the message never "moves to A," and Bob's socket never moves to A either. The message is **carried to the node that owns Bob's socket**, and that node does the write.

> 💡 **The senior one-liner:** *"Redis tells me **which** gateway holds Bob, but it doesn't move the message. The gateways form an internal cluster, so Gateway A forwards to Gateway B — either a direct gRPC call to B's address or by publishing to a pub/sub channel B subscribes to — and B looks up Bob's socket in its own local in-memory table and writes to it. A WebSocket is pinned to the process that owns it, so delivery must physically happen on that node."*

---

## 6. Message Routing (Online & Offline)

The message path, step by step (the full narrated version is in §4.4; the cross-gateway A→B hop is detailed in §5.1):

```
1. User A sends a message  →  Gateway A  →  Chat Server
2. Chat Server persists the message (Cassandra) and looks up User B in Redis
3a. User B ONLINE  →  find B's gateway  →  forward to that gateway (§5.1)  →  push down B's WebSocket  →  B receives (<200ms)
3b. User B OFFLINE →  message stays stored / queued  →  send a push notification (APNs/FCM)
4. When B reconnects → B pulls undelivered messages → marked delivered
```

The split between **online (push)** and **offline (store + notify)** is the core routing logic. Offline support is essential because mobile clients are *usually* disconnected — so the durable store is the source of truth, and the live socket is just a fast-path optimization for when the recipient happens to be online.

**Ordering within a conversation** is preserved by sequencing messages per `conversation_id` (e.g., a per-conversation sequence number or the server-assigned timestamp used as the sort key in storage), so the client can order/display correctly even if delivery races.

---

## 7. Message Storage

### Tiering

- **Hot data** — recent messages (e.g., last 30 days) in **Cassandra**, chosen because it's **write-heavy and time-series-friendly** (wide-column + LSM engine + consistent hashing — see the key-value store and SQL-vs-NoSQL guides). At ~580K writes/sec, write throughput is the dominant requirement, and Cassandra is built for exactly that.
- **Cold data** — older messages archived to cheaper storage (**S3**), since most reads are recent.

### Schema (Cassandra)

```
PRIMARY KEY ( conversation_id, message_timestamp )
columns: sender_id, message_text (ciphertext), media_url, status
```

The design trick: **partition by `conversation_id`, sort by `message_timestamp`.** This means all messages for a conversation live together on one partition, ordered by time — so the dominant query, *"give me the last 50 messages in chat X,"* is a **single-partition, single-disk-locality scan**: fast and cheap. Picking the partition key to match your dominant access pattern is the whole art of wide-column modeling.

---

## 8. Read Receipts & Delivery States

A message moves through a small **state machine**: **sent** (left sender) → **delivered** (reached recipient's device) → **read** (recipient opened the chat). These are the familiar one/two/blue checkmarks.

Implementation: receipts are just **small ACK messages** sent back over the *same* WebSocket channel. When B's device receives a message, it sends a "delivered" ACK; when B opens the chat, a "read" ACK. The server updates message status and notifies A.

Crucially, **status updates are written to the DB asynchronously** — they're high-volume (on the order of ~1M ACK-writes/sec, per §4.6) and not on the critical path of delivering the actual message, so you don't want them slowing it down. Batching receipt writes is a common optimization given their volume.

---

## 9. Presence / Last-Seen

Presence ("online", "last seen 5 minutes ago") looks trivial but is a **deceptively hard scaling problem** — and a great senior talking point.

The naive approach — broadcast "User A is online" to all of A's contacts the instant A connects — causes a **fan-out storm**: a user with 1,000 contacts triggers 1,000 notifications on every connect/disconnect, and people connect/disconnect constantly on mobile. Multiply across 2B users and presence traffic can **dwarf actual message traffic.**

Mitigations:
- **Don't broadcast eagerly** — update presence on a heartbeat interval, and only push presence changes to contacts who currently have that user's chat *open* (subscribe-on-view), not all contacts.
- **Pull on demand** — compute "last seen" from the user's last heartbeat timestamp (stored in Redis) when someone actually looks, rather than pushing proactively.
- **Coalesce / debounce** — don't react to every flicker of connectivity; smooth it.

The honest answer in an interview: presence is best-effort, eventually consistent, and deliberately *approximate* to keep it affordable.

---

## 10. Group Chat & Fan-Out

A group message must reach every member, which is a **fan-out** problem with two strategies — the same push-vs-pull trade-off seen in feeds.

### Fan-out on write (push)

When someone posts, the server **replicates the message to every member's inbox/delivery path** immediately.

- **Pros:** recipients get messages instantly; read path is simple.
- **Cons:** a message to a large group means many writes/pushes at once; wasteful if many members are inactive.

### Fan-out on read (pull)

The message is stored once per conversation; **clients fetch it when they open the chat.**

- **Pros:** cheap on write; no wasted work for inactive members.
- **Cons:** more work and latency at read time.

### The hybrid (the strong answer)

Use **fan-out on write for small/active groups** (instant delivery, the common case at ≤256 members) and **pull-based for very large or mostly-inactive groups** (avoid the write amplification). WhatsApp's 256-member cap keeps fan-out-on-write tractable for most groups — a deliberate product decision that bounds the engineering problem.

> E2E wrinkle (see next section): in an encrypted group, the sender's device must encrypt the message **once per recipient** (each has different keys), so a single message to a 256-member group means up to **256 client-side encryptions and 256 delivery pushes** — group fan-out has a client-side encryption cost, not just a server-side delivery cost. This is another reason to cap group size.

---

## 11. End-to-End Encryption

WhatsApp uses the **Signal Protocol** for E2E encryption, so **the server never sees plaintext** — it only relays ciphertext.

- **Identity keys + pre-keys.** Each user has a long-term identity key pair plus a set of one-time **pre-keys** uploaded to the server, so a sender can establish a secure session with a recipient *even while the recipient is offline* (the server hands out a pre-key on the sender's behalf).
- **Double Ratchet algorithm.** The session generates a **new encryption key for every message**, "ratcheting" forward. This gives **forward secrecy** (compromising one key doesn't expose past messages) and post-compromise security (the chain heals).
- **The server's role shrinks to routing ciphertext** — it stores and forwards encrypted blobs and manages key distribution, but can't read content.

**Design implications a senior should name:**
- **No server-side search or content features** — the server can't index what it can't read; search must happen on-device.
- **Key distribution & trust** — the server distributes public keys, so it could theoretically perform a man-in-the-middle; "safety number" verification lets users detect this.
- **Multi-device** is harder — each device is a separate cryptographic identity that must be brought into sessions.
- **Group encryption** requires per-recipient encryption (the fan-out cost noted above).

---

## 12. Media Handling

The firm rule: **media never flows through the chat servers.** Routing images and videos through the messaging path would saturate it and blow the latency budget. Instead:

1. The sender **uploads media directly to object storage (S3)** — often via a **pre-signed URL** the server hands out, so the upload bypasses application servers entirely.
2. The media is encrypted client-side (to preserve E2E) before upload.
3. The chat message carries only a **reference (URL/key) plus a decryption key** — a tiny ~60-byte-class message, not the megabytes of media.
4. The recipient **downloads from S3 via a CDN** (edge-cached, close to the user — see the scaling guide) and decrypts locally.

This keeps the message path lightweight and fast, and offloads bulk bytes to storage and CDN infrastructure built for them.

---

## 13. Scaling Summary

- **Shard connection state by `user_id`** — each user's live connection and registry entry live on one gateway.
- **Shard messages by `conversation_id`** — keeps a conversation's messages co-located for fast single-partition reads (Cassandra + consistent hashing handles this).
- **Cross-gateway delivery via RPC or a pub/sub backplane** — the sender's gateway reaches the recipient's gateway over the internal cluster; the recipient's gateway does the final socket write (§5.1).
- **Media off the hot path** — upload to S3, deliver via CDN, send only a URL.
- **Push via APNs/FCM** — for background/offline notifications, hand off to the platform push services rather than holding a socket open in the background.
- **Tiered storage** — hot in Cassandra, cold in S3.
- **Presence kept approximate** — heartbeat-based and pull-on-view to avoid fan-out storms.
- **Connection fleet** — ~200 gateway nodes for ~200M concurrent sockets (§4.6); scales independently of the chat-logic tier.

---

## 14. Senior Follow-Up Questions (with Answers)

**Q1. How do you deliver a message to a user connected to a different server?**
A connection registry (Redis) maps `user_id → gateway_node`. The chat server looks up the recipient's gateway and forwards the message there; that gateway pushes it down the open WebSocket. If the user is offline, the message stays in durable storage and a push notification is sent. (Full trace in §4.4; the *physical* A→B forwarding mechanism is §5.1 and Q13.)

**Q2. How do you guarantee message ordering?**
Order only needs to hold **within a conversation**, not globally — a much cheaper guarantee. Use a per-conversation sequence (or server-assigned timestamp) as the sort key; clients order by it, so display is correct even if network delivery races. Cassandra's `(conversation_id, timestamp)` clustering enforces this at storage time.

**Q3. Why is presence harder than it looks, and how do you scale it?**
Eagerly broadcasting online/offline to all contacts creates a fan-out storm that can exceed message traffic, especially with flaky mobile connections. Mitigate by making presence heartbeat-based and pull-on-view (only push to contacts actively viewing the chat), coalescing flapping, and treating it as approximate/eventually-consistent.

**Q4. Fan-out on write vs read for groups?**
Write (push to each member's path on send) gives instant delivery but amplifies work for large/inactive groups; read (store once, fetch on open) is cheap to write but slower to read. Hybrid: push for small/active groups (the ≤256 common case), pull for large/inactive ones. The 256-member cap deliberately keeps fan-out-on-write affordable.

**Q5. What does E2E encryption prevent you from building?**
Anything that needs message content server-side: search, content moderation, smart replies, server-side spam filtering on text. Search must run on-device. It also complicates multi-device and makes group messaging require per-recipient encryption. The trade is privacy for feature constraints.

**Q6. How do you support offline users?**
The durable store (Cassandra) is the source of truth; the live socket is just a fast path. Messages for offline users persist; a push notification (APNs/FCM) wakes the app; on reconnect the client pulls undelivered messages and they're marked delivered. Because most mobile clients are usually offline, this path is the norm, not the exception.

**Q7. How do you handle media at scale without choking the chat path?**
Never route media through chat servers. Client uploads directly to S3 (pre-signed URL), encrypted client-side; the message carries only a URL + key; recipient downloads via CDN and decrypts. This keeps messages tiny and offloads bulk bytes to storage/CDN.

**Q8. What store do you use for messages and why?**
Cassandra (wide-column, LSM, consistent hashing) because the workload is overwhelmingly write-heavy (~580K msgs/sec) and time-ordered, and partitioning by `conversation_id` makes "last N messages" a fast single-partition read. Hot data stays in Cassandra; cold archives to S3.

**Q9. How do you handle hundreds of millions of concurrent connections?**
A horizontally-scaled fleet of gateway nodes (~200 nodes for ~200M sockets, §4.6), each holding many WebSocket connections, fronted by load balancing; the Redis registry decouples "who's connected where" from any single node. Heartbeats reap dead connections. Connection state shards by `user_id`, and gateways hold no business logic so they restart freely.

**Q10. What's the consistency model, and is that OK?**
It's effectively AP/eventually-consistent for presence and receipts (best-effort), with per-conversation ordering for messages. That's the right call (CAP trade-off): users prefer the app to always be available and fast over perfectly consistent presence/receipts, and message ordering only needs to hold within a conversation.

**Q11. Why separate the Gateway from the Chat Server?**
They scale on different axes — connections (memory/file-descriptors per socket) vs. message logic (CPU). Splitting lets each scale and fail independently: a gateway can crash and clients simply reconnect to another node without losing the message logic tier, and you can add gateways for more connections without touching chat servers. Keeping the gateway logic-free is what makes it cheap to operate at the connection scale.

**Q12. What happens at the exact moment a recipient is reconnecting (race between push and offline)?**
The durable store resolves it. If the chat server pushes while the socket is mid-reconnect, the push may be lost — but the message is already persisted, so on reconnect the client pulls undelivered messages and gets it anyway. Delivery is idempotent on the client (dedupe by message id), so a message that arrives both via late push and via pull is shown once.

**Q13. Concretely, how does a message get from the sender's gateway to the recipient's gateway (different servers)?**
The registry only returns an *address*, not a transport — it doesn't move the message. The gateways form an internal cluster, and the sending gateway reaches the recipient's gateway in one of two ways: (a) a **direct gRPC/TCP call** to the recipient gateway's address (a mesh), or (b) **publishing to a pub/sub channel** (Redis Pub/Sub / NATS / Kafka) that the recipient's gateway is subscribed to (the common choice at scale, because it decouples the gateways). The recipient's gateway then finds the recipient's socket in its **own local in-memory `user_id → socket` table** and writes the bytes to it. Delivery *must* happen on the node that owns the socket, because a WebSocket is pinned to the process that accepted it — you can't write to it, or move it, from another server. (Full detail in §5.1.)

---

## 15. Quick Glossary

- **WebSocket** — persistent, full-duplex connection enabling server push (vs. request/response).
- **Gateway** — server that terminates client WebSocket connections; holds live sockets, holds no business logic.
- **Connection registry** — global `user_id → gateway_node` map (Redis) used to find which node holds a user; it locates, it does not transport.
- **Local socket table** — a per-gateway in-memory `user_id → socket object` map for the connections that gateway personally holds; used to do the final push (distinct from the global registry — see §5.1).
- **Pub/Sub backplane** — an internal message bus (Redis Pub/Sub, NATS, Kafka) that routes a message from the sender's gateway to the recipient's gateway, so the sender's node needn't know the recipient's node directly.
- **Chat Server** — the stateless brain that persists, routes, and updates message status.
- **Fan-out on write / read** — pushing a message to all recipients on send vs. having them pull on open.
- **Presence** — online/last-seen status; kept approximate to avoid fan-out storms.
- **Read receipt** — sent/delivered/read state of a message, sent as small ACKs.
- **Signal Protocol** — the E2E encryption protocol WhatsApp uses.
- **Double Ratchet** — algorithm generating a fresh key per message (forward secrecy).
- **Pre-keys** — pre-uploaded keys enabling secure session setup with an offline recipient.
- **APNs / FCM** — Apple / Google push-notification services for waking offline apps.
- **Pre-signed URL** — a time-limited URL letting clients upload/download directly to/from S3.
- **Tiered storage** — hot recent data (Cassandra) vs. cold archived data (S3).
- **Wide-column store** — Cassandra-style DB, partitioned by a key, sorted within the partition.

---

*Reference document. Contributions and corrections welcome.*
