# Designing WhatsApp (Real-Time Messaging): A Senior Interview Guide

> A practical, interview-focused reference for designing a real-time messaging system at WhatsApp scale — persistent connections, message routing, storage, presence, read receipts, group fan-out, end-to-end encryption, and media handling — with capacity math, trade-offs, and a deep bank of senior follow-up questions.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Architecture](#4-core-architecture)
5. [The Persistent Connection Model](#5-the-persistent-connection-model)
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
Messages/sec = 50 × 10⁹ / 10⁵ s/day  ≈ 500,000–600,000 msgs/sec   (avg)
   (peak is higher — call it ~1M/sec at peak, ~2× average)

Storage/day  = 50 × 10⁹ × 60 bytes    = 3 × 10¹² B  ≈ 3 TB/day
   (≈ ~1 PB/year of text+metadata; media is far larger and goes to object storage, not here)
```

**Takeaways that shape the design:**
- ~600K msgs/sec is enormous — the system is **write-heavy**, pushing us toward a write-optimized store (Cassandra/LSM).
- 3 TB/day of message data means **tiered storage** (hot recent data vs. cold archive) and partitioning are mandatory.
- The connection layer must hold **hundreds of millions of concurrent WebSocket connections** — its own significant engineering challenge.

---

## 4. Core Architecture

```
[Client] ──(WebSocket)──► [Gateway] ──► [Chat Server] ──► [Message DB (Cassandra)]
                              │              │
                              │              ├──► [Presence Service]
   connection registry        │              ├──► [Notification Service] ──► APNs / FCM
   user_id → gateway_node      │              └──► [Media Service] ──► S3 + CDN
   (Redis)  ◄─────────────────┘
```

- **Gateway** — terminates the client's WebSocket; holds the live connection. Maps `user_id → gateway_node` in a fast registry (Redis) so any server can find where a user is connected.
- **Chat Server** — handles message logic: persist, route, trigger receipts/notifications.
- **Message DB (Cassandra)** — write-optimized, time-series-friendly store for messages.
- **Presence Service** — tracks online/last-seen.
- **Notification Service** — sends push notifications via APNs (iOS) / FCM (Android) when a recipient is offline.
- **Media Service** — handles uploads to S3 and delivery via CDN; media never flows through chat servers.

---

## 5. The Persistent Connection Model

The defining design choice. Because the server must push messages instantly, each client opens a **long-lived WebSocket** to a gateway and keeps it open (see the protocols guide for why WebSocket over polling — lower latency, far less overhead, true bidirectional push).

The challenge this creates: with hundreds of millions of clients connected across thousands of gateway nodes, **how do you find which gateway holds a given user's connection?** The answer is a **connection registry**: when a user connects, the gateway records `user_id → its own node` in **Redis**. To deliver a message to that user, any chat server looks up their gateway in Redis and forwards the message there, which pushes it down the open socket.

Practical concerns:

- **Heartbeats / keepalives** detect dead connections (a client that vanished without closing cleanly) so stale registry entries are cleaned up.
- **Connection state is the thing you shard by `user_id`** — each user's connection lives on one gateway.
- **Reconnection** must be cheap and frequent (mobile networks drop constantly); on reconnect the client re-registers and pulls any queued messages.

---

## 6. Message Routing (Online & Offline)

The message path, step by step:

```
1. User A sends a message  →  Gateway A  →  Chat Server
2. Chat Server persists the message (Cassandra) and looks up User B in Redis
3a. User B ONLINE  →  find B's gateway  →  push down B's WebSocket  →  B receives (<200ms)
3b. User B OFFLINE →  message stays stored / queued  →  send a push notification (APNs/FCM)
4. When B reconnects → B pulls undelivered messages → marked delivered
```

The split between **online (push)** and **offline (store + notify)** is the core routing logic. Offline support is essential because mobile clients are *usually* disconnected — so the durable store is the source of truth, and the live socket is just a fast-path optimization for when the recipient happens to be online.

**Ordering within a conversation** is preserved by sequencing messages per `conversation_id` (e.g., a per-conversation sequence number or the server-assigned timestamp used as the sort key in storage), so the client can order/display correctly even if delivery races.

---

## 7. Message Storage

### Tiering

- **Hot data** — recent messages (e.g., last 30 days) in **Cassandra**, chosen because it's **write-heavy and time-series-friendly** (wide-column + LSM engine + consistent hashing — see the key-value store and SQL-vs-NoSQL guides). At ~600K writes/sec, write throughput is the dominant requirement, and Cassandra is built for exactly that.
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

Crucially, **status updates are written to the DB asynchronously** — they're high-volume and not on the critical path of delivering the actual message, so you don't want them slowing it down. Batching receipt writes is a common optimization given their volume.

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

> E2E wrinkle (see next section): in an encrypted group, the sender's device must encrypt the message **once per recipient** (each has different keys), so group fan-out also has a client-side encryption cost, not just a server-side delivery cost. This is another reason to cap group size.

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
- **Media off the hot path** — upload to S3, deliver via CDN, send only a URL.
- **Push via APNs/FCM** — for background/offline notifications, hand off to the platform push services rather than holding a socket open in the background.
- **Tiered storage** — hot in Cassandra, cold in S3.
- **Presence kept approximate** — heartbeat-based and pull-on-view to avoid fan-out storms.

---

## 14. Senior Follow-Up Questions (with Answers)

**Q1. How do you deliver a message to a user connected to a different server?**
A connection registry (Redis) maps `user_id → gateway_node`. The chat server looks up the recipient's gateway and forwards the message there; that gateway pushes it down the open WebSocket. If the user is offline, the message stays in durable storage and a push notification is sent.

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
Cassandra (wide-column, LSM, consistent hashing) because the workload is overwhelmingly write-heavy (~600K msgs/sec) and time-ordered, and partitioning by `conversation_id` makes "last N messages" a fast single-partition read. Hot data stays in Cassandra; cold archives to S3.

**Q9. How do you handle hundreds of millions of concurrent connections?**
A horizontally-scaled fleet of stateless-ish gateway nodes, each holding many WebSocket connections, fronted by load balancing; the Redis registry decouples "who's connected where" from any single node. Heartbeats reap dead connections. Connection state shards by `user_id`.

**Q10. What's the consistency model, and is that OK?**
It's effectively AP/eventually-consistent for presence and receipts (best-effort), with per-conversation ordering for messages. That's the right call (CAP trade-off): users prefer the app to always be available and fast over perfectly consistent presence/receipts, and message ordering only needs to hold within a conversation.

---

## 15. Quick Glossary

- **WebSocket** — persistent, full-duplex connection enabling server push (vs. request/response).
- **Gateway** — server that terminates client WebSocket connections.
- **Connection registry** — `user_id → gateway_node` map (Redis) used to route messages to the right node.
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
