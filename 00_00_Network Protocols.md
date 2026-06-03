# Network Protocols: A Ground-Up Guide

> A practical reference covering TCP, UDP, HTTP/1.1, HTTP/2, HTTP/3 (QUIC), WebSocket, and gRPC — built from first principles, with comparisons and interview prep.

---

## Table of Contents

1. [Prerequisites: What Even Is a Protocol?](#1-prerequisites-what-even-is-a-protocol)
2. [The OSI & TCP/IP Models](#2-the-osi--tcpip-models)
3. [TCP — Transmission Control Protocol](#3-tcp--transmission-control-protocol)
4. [UDP — User Datagram Protocol](#4-udp--user-datagram-protocol)
5. [TCP vs UDP](#5-tcp-vs-udp)
6. [HTTP/1.1](#6-http11)
7. [HTTP/2](#7-http2)
8. [HTTP/3 (QUIC)](#8-http3-quic)
9. [The HTTP Evolution at a Glance](#9-the-http-evolution-at-a-glance)
10. [WebSocket](#10-websocket)
11. [Server-Sent Events & Long Polling](#11-server-sent-events--long-polling)
12. [gRPC](#12-grpc)
13. [Master Comparison Table](#13-master-comparison-table)
14. [Interview-Ready Insights](#14-interview-ready-insights)
15. [Quick Glossary](#15-quick-glossary)

---

## 1. Prerequisites: What Even Is a Protocol?

A **network protocol** is a shared set of rules that two machines agree on so they can exchange data and both make sense of it. Think of it like a language: if one computer "speaks" in a format the other doesn't understand, communication fails. Protocols define:

- **Format** — how bytes are laid out (headers, body, framing).
- **Order** — who speaks first, what the valid sequence of messages is.
- **Error handling** — what happens when data is lost, corrupted, or arrives out of order.
- **Addressing** — how to identify the sender and receiver.

Because networking is complex, protocols are organized into **layers**. Each layer solves one part of the problem and relies on the layer beneath it. This is the single most important idea to internalize before the rest of this guide makes sense.

A useful mental model: when you send a message, your data travels *down* the stack on your machine (each layer wraps it in its own header — this is called **encapsulation**), across the wire, and then *up* the stack on the receiving machine (each layer peels off its header — **decapsulation**).

---

## 2. The OSI & TCP/IP Models

The **OSI model** is a 7-layer conceptual framework. You rarely need all seven in practice, but two layers dominate the protocols in this guide: **Layer 4 (Transport)** and **Layer 7 (Application)**.

```
OSI Layer                   Example Protocols           What it does
-----------------------------------------------------------------------------
7  Application              HTTP, WebSocket, gRPC, DNS   App-level data & meaning
6  Presentation            TLS, encoding, compression   Format/encrypt data
5  Session                 (often folded into others)   Manage sessions
4  Transport               TCP, UDP, QUIC               End-to-end delivery
3  Network                 IP                           Routing across networks
2  Data Link               Ethernet, Wi-Fi (MAC)        Node-to-node on a link
1  Physical                Cables, radio, fiber         Raw bits on the medium
```

The simpler, real-world **TCP/IP model** collapses this into four layers (Application, Transport, Internet, Link). For this guide, just remember:

- **Layer 4 (Transport)** decides *how reliably* data moves between two endpoints. TCP and UDP live here.
- **Layer 7 (Application)** decides *what the data means*. HTTP, WebSocket, and gRPC live here — and they're all built on top of Layer 4.

Everything below is an application of this stack.

---

## 3. TCP — Transmission Control Protocol

**One-line summary:** A connection-oriented, reliable, ordered, byte-stream protocol at Layer 4.

### The problem TCP solves

The underlying network (IP) makes no promises. Packets can be lost, duplicated, delayed, or arrive out of order. TCP sits on top and turns this chaos into a clean, ordered stream of bytes that "just works" for the application above it.

### TCP guarantees

- **Delivery** — lost packets are detected and retransmitted.
- **Ordering** — bytes arrive in the same order they were sent.
- **No duplicates** — duplicate packets are discarded.
- **Integrity** — checksums detect corruption.

### The 3-Way Handshake

Before any data flows, both sides establish a connection and agree on starting sequence numbers:

```
Client                          Server
  |                               |
  |---------- SYN --------------->|   "I want to talk; my seq = x"
  |                               |
  |<------- SYN-ACK --------------|   "OK; my seq = y, ack = x+1"
  |                               |
  |---------- ACK --------------->|   "Got it; ack = y+1"
  |                               |
  |======= connection open =======|
```

This handshake is why TCP has **connection setup cost** — a full round trip before the first byte of real data. (Closing uses a similar 4-way teardown with FIN/ACK.)

### How reliability actually works

- **Sequence numbers** label every byte so the receiver can reorder and detect gaps.
- **ACKs (acknowledgments)** confirm what was received.
- **Retransmission** resends data that wasn't acknowledged within a timeout.
- **Flow control** (sliding window) stops a fast sender from overwhelming a slow receiver.
- **Congestion control** (algorithms like Reno, CUBIC, BBR) backs off when the network is congested, preventing collapse.

### Costs & trade-offs

- ~20-byte header (minimum) per segment.
- Handshake adds latency before data transfer.
- Head-of-line blocking: because it's a single ordered stream, one lost packet stalls everything behind it until it's retransmitted.

### Used by

HTTP/1.1, HTTPS, SSH, FTP, SMTP, and most database wire protocols — anywhere correctness matters more than shaving off milliseconds.

---

## 4. UDP — User Datagram Protocol

**One-line summary:** A connectionless, unreliable, unordered datagram protocol at Layer 4.

### The philosophy

UDP is deliberately minimal. It's essentially IP with port numbers and a checksum bolted on. There's **no handshake** — you just send a packet ("datagram") and hope it arrives. This is often called **"fire and forget."**

```
Client                          Server
  |------ datagram 1 ----------->|   (may arrive, may not)
  |------ datagram 2 ----------->|   (may arrive out of order)
  |------ datagram 3 --X         |   (lost — no retransmission)
```

### What UDP does NOT do

- No delivery guarantee, no retransmission.
- No ordering.
- No congestion control or flow control.
- No connection state.

### Why anyone would want that

For real-time data, **stale data is useless data**. In a video call, retransmitting a dropped frame from 200ms ago is pointless — by the time it arrives, the conversation has moved on. You'd rather just skip it and stay current. UDP's lack of overhead means **lower latency** and less protocol machinery.

If an application needs *some* reliability, it builds exactly the parts it needs on top of UDP (this is precisely what QUIC does — see HTTP/3).

### Costs & trade-offs

- 8-byte header (much lighter than TCP).
- Application must handle loss/ordering itself if it cares.

### Used by

Video/audio streaming, online gaming, DNS lookups, VoIP, real-time analytics, and QUIC/HTTP/3.

---

## 5. TCP vs UDP

| Dimension            | TCP                          | UDP                              |
|----------------------|------------------------------|----------------------------------|
| Connection           | Connection-oriented          | Connectionless                   |
| Reliability          | Guaranteed delivery          | Best-effort (no guarantee)       |
| Ordering             | Ordered                      | Unordered                        |
| Handshake            | Yes (3-way)                  | None                             |
| Header size          | 20+ bytes                    | 8 bytes                          |
| Speed/latency        | Higher latency               | Lower latency                    |
| Congestion control   | Yes                          | No (unless app adds it)          |
| Best for             | Correctness-critical traffic | Latency-critical / real-time     |

**The decision rule:** Do you need every byte to arrive correctly and in order? → **TCP**. Is being fast and current more important than being complete? → **UDP**.

---

## 6. HTTP/1.1

**One-line summary:** A stateless request-response protocol running over TCP (Layer 7).

### How it works

The client sends a request (method + path + headers + optional body); the server returns a response (status code + headers + body). Each exchange is independent — the server doesn't inherently remember previous requests, which is what **stateless** means. State (logins, carts) is layered on via cookies, tokens, and sessions.

```
GET /index.html HTTP/1.1          HTTP/1.1 200 OK
Host: example.com         --->    Content-Type: text/html
Accept: text/html                 Content-Length: 1024

                          <---    <html>...</html>
```

### Common methods

`GET` (read), `POST` (create/submit), `PUT` (replace), `PATCH` (partial update), `DELETE` (remove).

### The pain points

- **One request at a time per connection.** Browsers worked around this by opening ~6 parallel TCP connections per domain.
- **Pipelining** (sending multiple requests without waiting for responses) existed in theory but was poorly supported and rarely usable in practice.
- **Head-of-line (HOL) blocking** at the application layer: a slow response holds up everything queued behind it on that connection.
- **Repeated, verbose headers** sent in full text on every request — wasteful for many small requests.

These problems are exactly what HTTP/2 set out to fix.

---

## 7. HTTP/2

**One-line summary:** Multiplexed, binary HTTP over a single TCP connection.

### Key improvements over 1.1

- **Multiplexing** — many concurrent **streams** share one TCP connection. Requests and responses are broken into frames, interleaved on the wire, and reassembled. No more opening six connections.
- **Binary framing** — messages are encoded in a compact binary format instead of plain text, making parsing faster and less error-prone.
- **Header compression (HPACK)** — repeated headers are compressed and a shared table avoids re-sending the same values.
- **Server push** — the server could proactively send resources the client would likely need (now largely deprecated in practice, but historically notable).
- **Stream prioritization** — clients can hint which streams matter more.

### The remaining catch

HTTP/2 multiplexes at the *application* layer, but it still runs over a *single TCP* connection. TCP guarantees ordered delivery of the whole byte stream, so **if one packet is lost, TCP stalls all streams** until it's retransmitted — even streams that didn't need that packet. This is **TCP-level head-of-line blocking**, and it's the core limitation HTTP/3 was created to escape.

---

## 8. HTTP/3 (QUIC)

**One-line summary:** HTTP running over QUIC, a new transport built on UDP.

### The big idea

Instead of fixing TCP (which is baked into operating systems and middleboxes everywhere and hard to change), the designers built a brand-new transport protocol called **QUIC** on top of **UDP**, and reimplemented the reliability, ordering, and congestion control features in user space — but **per-stream** rather than for one giant stream.

### Why that matters

- **No TCP head-of-line blocking.** Because each stream is independently ordered within QUIC, a lost packet on one stream doesn't stall the others. Only the affected stream waits.
- **Faster connection setup.** QUIC integrates the transport handshake with TLS encryption. A fresh connection takes ~1 round trip, and a resumed one can be **0-RTT** — data on the very first packet.
- **Connection migration.** A QUIC connection is identified by a connection ID, not the IP/port 4-tuple, so it can survive a network change (e.g., switching from Wi-Fi to cellular) without re-establishing.
- **Encryption is mandatory and built in** (TLS 1.3).

### The trade-off

UDP is sometimes throttled or blocked by older networks/firewalls that don't expect heavy UDP traffic, so clients fall back to HTTP/2 over TCP when QUIC isn't available.

### Used by

YouTube, Google services, Facebook/Meta, Cloudflare, and a growing share of the web.

---

## 9. The HTTP Evolution at a Glance

```
HTTP/1.1   --->   HTTP/2          --->   HTTP/3
over TCP          over TCP                over QUIC (UDP)

1 req/conn        multiplexed             multiplexed
text headers      binary + HPACK          binary + QPACK
                  server push             0-RTT, migration
app-level HOL     TCP-level HOL           no HOL blocking
```

Each version chips away at latency and the head-of-line blocking problem. HTTP/2 solved it at the application layer; HTTP/3 solved it at the transport layer by abandoning TCP.

---

## 10. WebSocket

**One-line summary:** A full-duplex, bidirectional, persistent connection over TCP.

### The problem it solves

HTTP is fundamentally request-response: the client asks, the server answers. The server can't spontaneously push data. For things like chat, live notifications, or multiplayer games, you need the server to send data *whenever it wants*, and you need it to be cheap. WebSocket gives you a single long-lived, two-way pipe.

### How it starts: the HTTP upgrade

A WebSocket connection begins life as an ordinary HTTP request with an `Upgrade` header. If the server agrees, it responds with status **`101 Switching Protocols`**, and from that point the same TCP connection is "upgraded" to the WebSocket protocol — no more HTTP semantics, just bidirectional message frames.

```
Client:  GET /chat HTTP/1.1
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Key: ...

Server:  HTTP/1.1 101 Switching Protocols
         Upgrade: websocket
         Connection: Upgrade

         ===== full-duplex WS connection now open =====
         <--- server can push any time --->
         <--- client can send any time  --->
```

### Used by

Chat apps, live notifications, collaborative editors, multiplayer games, and stock/crypto tickers.

### Alternatives

- **Server-Sent Events (SSE)** — one-way, server → client only.
- **Long polling** — an older hack to simulate push (covered next).

---

## 11. Server-Sent Events & Long Polling

These are the main alternatives to WebSocket, useful for understanding *why* WebSocket exists.

### Long Polling

The client makes a request and the server **holds it open** until it has data (or a timeout), then responds. The client immediately makes another request. It simulates "push" but with significant overhead — each message means a full new request/response cycle and connection churn.

### Server-Sent Events (SSE)

A standardized, one-directional stream from server to client over a single long-lived HTTP connection. Lighter than WebSocket and auto-reconnects, but **only the server can send** — the client can't push back over the same channel. Great for news feeds, live scores, and notifications where the client doesn't need to talk back in real time.

### Choosing between them

| Need                                   | Best fit        |
|----------------------------------------|-----------------|
| True two-way, low-latency, high-volume | WebSocket       |
| Server → client only, simple           | SSE             |
| Must work on ancient infrastructure    | Long polling    |

---

## 12. gRPC

**One-line summary:** A high-performance RPC framework built on HTTP/2, using Protocol Buffers.

### What "RPC" means

**Remote Procedure Call** lets you call a function on another server as if it were a local function — you call `getUser(id)` and the framework handles serializing the request, sending it over the network, and deserializing the response. It hides the networking so service-to-service calls feel like normal code.

### What makes gRPC distinctive

- **Built on HTTP/2** — inherits multiplexing, binary framing, and efficient connections.
- **Protocol Buffers (Protobuf)** — a compact, strongly-typed *binary* serialization format. You define your API and message shapes in a `.proto` file:

  ```proto
  service UserService {
    rpc GetUser (UserRequest) returns (UserReply);
  }
  message UserRequest { int32 id = 1; }
  message UserReply   { string name = 1; }
  ```

- **Code generation** — from that `.proto`, gRPC generates strongly-typed client and server stubs in many languages (Go, Java, Python, C++, etc.), so different services in different languages stay in sync automatically.
- **Streaming** — supports unary (one request/one response), server-streaming, client-streaming, and **bidirectional** streaming.

### Why teams use it

It's fast (binary, HTTP/2), strongly typed (fewer integration bugs), and language-agnostic — ideal for **service-to-service communication in microservices**. The trade-off: it's binary and not human-readable like JSON/REST, and browser support requires a proxy (gRPC-Web), so it's most common *behind* the public API rather than directly facing browsers.

---

## 13. Master Comparison Table

| Protocol  | Layer | Transport | Connection Style | Direction      | Primary Use Case               |
|-----------|-------|-----------|------------------|----------------|--------------------------------|
| TCP       | L4    | —         | Reliable stream  | Bidirectional  | Most internet traffic          |
| UDP       | L4    | —         | Unreliable datagram | Bidirectional | Gaming, streaming, DNS, VoIP  |
| HTTP/1.1  | L7    | TCP       | Request-response | Client → Server| Web pages, REST APIs           |
| HTTP/2    | L7    | TCP       | Multiplexed      | Client → Server| Modern web, gRPC base          |
| HTTP/3    | L7    | QUIC/UDP  | Multiplexed      | Client → Server| Low-latency web                |
| WebSocket | L7    | TCP       | Persistent full-duplex | Bidirectional | Chat, real-time apps      |
| gRPC      | L7    | HTTP/2    | RPC (+streaming) | Bidirectional  | Service-to-service / microservices |

---

## 14. Interview-Ready Insights

**Q: When would you choose TCP over UDP?**
Use TCP when correctness and ordering matter (web pages, file transfer, databases) and UDP when low latency matters more than completeness (live video, gaming, DNS). The one-liner: *reliability vs. latency.*

**Q: Why use WebSocket instead of polling?**
A single persistent, full-duplex connection means **lower latency and far less overhead** — no repeated request/response cycles or reconnection churn, and the server can push instantly.

**Q: Why does HTTP/3 run over UDP instead of TCP?**
To eliminate **TCP head-of-line blocking** and get **faster connection setup** (0-RTT). QUIC reimplements reliability per-stream on top of UDP, so one lost packet only stalls its own stream — and integrating TLS into the handshake cuts round trips.

**Q: HTTP/2 already has multiplexing — what problem remains?**
HTTP/2 multiplexes at the application layer but still rides a single TCP stream, so a lost packet stalls *all* streams (TCP-level HOL blocking). HTTP/3 fixes this at the transport layer.

**Q: Why is gRPC popular for microservices?**
Binary Protobuf is compact and fast, the contract is strongly typed and code-generated across languages, it runs on efficient HTTP/2, and it supports streaming — ideal for high-volume internal service-to-service calls.

**Q: What does "stateless" mean for HTTP?**
Each request is independent; the server doesn't inherently remember prior requests. State is added on top via cookies, tokens, and sessions.

**Q: Walk me through the TCP handshake.**
SYN (client proposes) → SYN-ACK (server agrees, proposes back) → ACK (client confirms). Sequence numbers are exchanged so both sides can order and acknowledge bytes.

---

## 15. Quick Glossary

- **Datagram** — a self-contained packet with no connection context (UDP's unit).
- **Multiplexing** — sending many independent logical streams over one connection.
- **Head-of-line (HOL) blocking** — one stuck item delaying everything queued behind it.
- **Full-duplex** — both sides can send simultaneously.
- **RPC** — calling a remote function as if it were local.
- **0-RTT** — sending application data on the very first packet of a (resumed) connection, with zero round-trips of setup latency.
- **Stateless** — no memory of past requests between exchanges.
- **Encapsulation** — wrapping data with each layer's header as it moves down the stack.

---

*Reference document — protocols current as of 2026. Contributions and corrections welcome.*
