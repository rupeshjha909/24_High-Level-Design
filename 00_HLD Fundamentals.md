# HLD Document 1: Foundations & Core Concepts

Covers topics 1–10 from your notes — the building blocks every system designer must master before tackling specific designs.

---

## 1. Network Protocols (TCP, WebSocket, HTTP, etc.)

### TCP (Transmission Control Protocol)
- **Connection-oriented, reliable, ordered, byte-stream protocol** at OSI Layer 4.
- 3-way handshake: SYN → SYN-ACK → ACK.
- Guarantees: delivery, order, no duplicates.
- Used for: HTTP/1.1, HTTPS, SSH, FTP, databases.
- Overhead: ~20-byte header, handshake cost.

### UDP (User Datagram Protocol)
- **Connectionless, unreliable, unordered, datagram protocol.**
- No handshake; "fire and forget".
- Used for: video streaming, gaming, DNS, VoIP, real-time analytics.
- Lower latency than TCP; no congestion control.

### HTTP/1.1
- **Stateless request-response** protocol over TCP.
- One request per connection (pipelining rarely works).
- Headers + body, methods: GET, POST, PUT, DELETE, PATCH.
- Issues: head-of-line blocking, repeated headers.

### HTTP/2
- Multiplexed streams over a single TCP connection.
- Binary framing, header compression (HPACK), server push.
- Still suffers TCP head-of-line blocking.

### HTTP/3 (QUIC)
- Runs over **UDP** instead of TCP.
- No TCP head-of-line blocking; faster connection setup (0-RTT).
- Used by YouTube, Facebook, Cloudflare.

### WebSocket
- **Full-duplex, bidirectional** persistent connection over TCP.
- Starts as HTTP upgrade (101 Switching Protocols), then becomes WS.
- Used for: chat, live notifications, multiplayer games, stock tickers.
- Alternatives: Server-Sent Events (one-way), Long polling.

### gRPC
- HTTP/2-based RPC framework using **Protocol Buffers**.
- Strongly-typed, code-generated clients/servers in many languages.
- Streaming support (client, server, bidi).
- Used for service-to-service communication in microservices.

### Comparison Table
| Protocol | Layer | Connection | Use Case |
|----------|-------|------------|----------|
| TCP      | L4    | Reliable   | Most internet traffic |
| UDP      | L4    | Unreliable | Gaming, streaming, DNS |
| HTTP/1.1 | L7    | Req-Resp   | Web pages, REST APIs |
| HTTP/2   | L7    | Multiplexed | Modern web, gRPC base |
| HTTP/3   | L7    | over UDP   | Low-latency web |
| WebSocket| L7    | Full-duplex| Chat, real-time |
| gRPC     | L7    | RPC        | Service-to-service |

### Interview-Ready Insights
- "Why WebSocket over polling?" — Lower latency, less overhead.
- "Why HTTP/3 over UDP?" — Avoid TCP HOL blocking, faster handshake.
- "When TCP vs UDP?" — Reliability needed (TCP) vs latency-critical (UDP).

---

## 2. Client-Server vs Peer-to-Peer Architecture

### Client-Server
```
[Client A] ──┐
[Client B] ──┼──► [Central Server] ──► [Database]
[Client C] ──┘
```
- **Centralized control.**
- Pros: Consistent state, easy to secure & monitor, simpler logic.
- Cons: Single point of failure, scaling cost, server bottleneck.
- Examples: Web apps, banking, email.

### Peer-to-Peer (P2P)
```
[Peer A] ──── [Peer B]
   │  \      /  │
   │   \    /   │
[Peer D] ── [Peer C]
```
- No central server; each node is both client and server.
- Pros: Highly resilient, infinite horizontal scale, no single bottleneck.
- Cons: Hard to enforce consistency/security/auditing, complex routing.
- Examples: BitTorrent, blockchain (Bitcoin, Ethereum), Skype (originally), IPFS.

### Hybrid
- **WhatsApp, Zoom**: P2P for media (when feasible), client-server for signaling/control.

### When to Choose
| Use | Choose |
|-----|--------|
| Centralized authority (banks, e-commerce) | Client-Server |
| Massive scale + censorship resistance | P2P |
| Real-time low-latency media | Hybrid |

---

## 3. CAP Theorem

> In a distributed system, you can guarantee **at most 2 of 3**:  
> **C**onsistency, **A**vailability, **P**artition Tolerance.

In real distributed systems, **P is non-negotiable** (networks WILL fail). So the real choice is between **C and A** during a partition.

### Definitions
- **Consistency**: Every read returns the most recent write or an error.
- **Availability**: Every request gets a (non-error) response.
- **Partition Tolerance**: System continues despite network failures.

### Examples
| System | Choice | Why |
|--------|--------|-----|
| **MongoDB, HBase, Redis** | CP | Reject writes when network splits; ensures consistency |
| **Cassandra, DynamoDB, CouchDB** | AP | Always accept writes; reconcile later (eventual consistency) |
| **Traditional RDBMS** (single-node) | CA | No partition concept (single node) |

### PACELC (Extension)
> If Partition (P) — choose Availability (A) or Consistency (C).  
> Else (E) — choose Latency (L) or Consistency (C).

Real systems trade off latency vs consistency even when there's no partition.

### Interview Tip
Never say "CAP allows only 2 of 3" without nuance. Real answer: "When partition happens, choose C or A. Most modern systems are AP with tunable consistency."

---

## 4. Microservices — Important Design Patterns

### What Are Microservices?
Architectural style where an application is composed of **small, independently deployable services**, each owning its data and a single business capability.

### Pros / Cons
| Pros | Cons |
|------|------|
| Independent scaling | Distributed system complexity |
| Independent deployments | Network latency overhead |
| Tech stack flexibility | Data consistency hard |
| Fault isolation | Debugging across services hard |

### 4a. Decomposition Pattern

#### Decompose by Business Capability
Identify domains (e.g., Orders, Inventory, Payments) → one service per domain.

#### Decompose by Subdomain (DDD)
Use Domain-Driven Design's bounded contexts.

#### Strangler Pattern
Incrementally replace a monolith by routing requests through a façade — slowly "strangling" the legacy app.
```
[Client] ──► [Facade/Proxy] ──┬──► [Legacy Monolith]
                              └──► [New Microservice]
```
Gradually shift more endpoints to the new service.

### 4b. SAGA Pattern

For **distributed transactions** across services (you can't use a global ACID transaction).

**Saga** = sequence of local transactions, each publishing an event to trigger the next. If one fails, run **compensating transactions** to undo prior steps.

#### Two Styles
1. **Choreography** — services publish/listen events; no central coordinator.
2. **Orchestration** — a central orchestrator service tells each service what to do.

#### Example: E-commerce order
```
Order → Reserve Inventory → Charge Payment → Ship
       (if payment fails: release inventory, cancel order)
```

### 4c. Strangler Pattern
(Described above under Decomposition.) Critical for migration; used by every team moving off a monolith.

### 4d. CQRS (Command Query Responsibility Segregation)

Separate the **write model** (commands) from the **read model** (queries).
```
                  ┌─► [Write DB] (normalized, transactional)
[Client] ──► API ─┤        │ replicates
                  │        ▼
                  └─► [Read DB(s)] (denormalized, optimized for queries)
```
- Write model: enforce business rules, transactions.
- Read model: pre-aggregated views, fast reads.
- Often paired with **Event Sourcing** (store events, not state).

### Other Critical Microservice Patterns
| Pattern | Purpose |
|---------|---------|
| **API Gateway** | Single entry; routing, auth, rate-limiting |
| **Service Discovery** | Find dynamic service instances (Consul, Eureka) |
| **Circuit Breaker** | Stop cascading failures (Hystrix, Resilience4j) |
| **Bulkhead** | Isolate resource pools |
| **Sidecar** | Run helper container (logging, proxy) — basis of service mesh |
| **Service Mesh** | Istio, Linkerd — handles service-to-service comms |

---

## 5. Scaling from 0 to a Million Users

### Stage 1 — Single Server (1-100 users)
```
[Client] ──► [Web Server + DB on same box]
```
Simple. Cheap. Single point of failure.

### Stage 2 — Separate DB (1K users)
```
[Web Server] ──► [Database Server]
```
Reasoning: separate failure domains; scale each independently.

### Stage 3 — Vertical Scaling (10K users)
Buy a bigger machine. Limits: hardware ceiling, no fault tolerance.

### Stage 4 — Horizontal Scaling + Load Balancer (100K)
```
[Client] ──► [Load Balancer] ──► [Web Server 1]
                              ──► [Web Server 2]
                              ──► [Web Server N]
                                       │
                                       ▼
                                [Master DB] ◄─► [Slave DBs (reads)]
```
- Stateless app servers.
- DB read replicas for read-heavy workloads.

### Stage 5 — Caching Layer
Add Redis/Memcached between app and DB. Massive read latency reduction.

### Stage 6 — CDN
Push static assets (images, JS, CSS) to edge servers globally.

### Stage 7 — Database Sharding
When write throughput maxes out a single DB, **shard horizontally** by user_id or geo.

### Stage 8 — Message Queue + Async Processing
Decouple slow operations (email, image processing) from request path.
```
[Web] ──► [Queue (Kafka/RabbitMQ)] ──► [Workers]
```

### Stage 9 — Microservices
Decompose monolith. Independent scaling per service.

### Stage 10 — Multi-Region + Multi-DC
Geo-replicate data. Active-active or active-passive setup.

### Architecture at Million Users
```
                  ┌─► [CDN (static)]
[Client] ──► [DNS] ──► [GeoLB] ──► [Regional LB] ──► [App Cluster]
                                                       ├─► [Cache (Redis)]
                                                       ├─► [Message Queue]
                                                       └─► [Sharded DB + Replicas]
```

---

## 6. Consistent Hashing

### Problem It Solves
With N servers and `hash(key) % N` routing, **changing N reshuffles almost all keys**. With consistent hashing, only ~1/N keys move.

### How It Works
1. Imagine a **ring** of hash values (0 to 2^32 - 1).
2. Place each server on the ring at `hash(server_id)`.
3. Place each key at `hash(key)`.
4. Each key belongs to the **next server clockwise**.

```
              key1
           ↓ (clockwise)
   ServerA ─────────── ServerB
          │           │
          │           │
   ServerD ─────────── ServerC
```

When you add ServerE, only keys between ServerD and ServerE move.

### Virtual Nodes
Each physical server is placed at **multiple points** (vnodes) on the ring → better load distribution, smoother rebalancing.

### Used By
- **DynamoDB** — partition keys across nodes.
- **Cassandra** — token ring.
- **Memcached** clients (ketama).
- **Akamai CDN**, **Discord** (for routing).

### Interview-Ready Code Idea
```
1. Maintain a sorted map: ring[hash] = server
2. For a key: find first ring key >= hash(key) — wrap around if needed
3. Use a TreeMap (Java) or SortedDict (Python).
```

---

## 7. URL Shortening (bit.ly, tinyurl)

### Requirements
- Functional: Shorten long URL → short URL; redirect short → long; analytics; custom alias.
- Non-functional: 100:1 read:write ratio; high availability; fast redirect (<100ms).

### Capacity Estimation
- Writes: 100M new URLs/month → ~40 URLs/sec.
- Reads: 100 × writes → ~4K reads/sec.
- Storage: 100M × 12 months × 5 years × ~500 bytes = ~3 TB.

### Short URL Generation — Approaches

#### Option 1: Hash-based
```
short_url = base62(md5(long_url))[:7]
```
Issues: collisions; same URL → same short (might want different).

#### Option 2: Counter + Base62
- Increment a global counter; encode in base62 (a-z, A-Z, 0-9).
- 7 chars in base62 = ~3.5 trillion combinations.
- Use **distributed counter** (Zookeeper, Redis INCR) or **range allocation** (each server takes a 10K range).

#### Option 3: KGS (Key Generation Service)
- Pre-generate keys offline → store in DB (unused / used).
- Servers pull batches; mark as used.
- Pros: no collision, fast.

### Architecture
```
[Client] ──► [LB] ──► [App Server] ──► [Cache (Redis)] ─┐
                          │                              │
                          ▼                              ▼
                      [Key Gen]                       [URL DB]
```

### Schema
```
url_mappings:
  short_key   VARCHAR(7) PRIMARY KEY
  long_url    TEXT
  user_id     BIGINT
  created_at  TIMESTAMP
  expires_at  TIMESTAMP NULL
  click_count BIGINT
```

### Redirect Flow
1. GET /aBc123 → lookup `aBc123` in cache → miss → DB → cache fill → 301 redirect.
2. Use **301 (permanent)** or **302 (temporary)** redirect:
   - 301: browsers cache; saves server hits but breaks analytics.
   - 302: every click hits server (better analytics).

### Analytics
Async: write click events to Kafka → aggregate in batch.

---

## 8. Back-of-the-Envelope Estimation

### Why It Matters
Interviewers want to see you can roughly size the system without a calculator.

### Numbers Every Engineer Should Memorize (Jeff Dean's)
| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| Send 1KB over 1Gbps network | 10 μs |
| Read 1MB sequentially from memory | 250 μs |
| Round-trip in same DC | 500 μs |
| Read 1MB sequentially from SSD | 1 ms |
| Read 1MB sequentially from HDD | 20 ms |
| Round-trip across continents | 150 ms |

### Time Units
- 1 second = 10^3 ms = 10^6 μs = 10^9 ns

### Data Sizes
- 1 char = 1 byte, 1 int = 4 bytes, 1 long/timestamp = 8 bytes, 1 UUID = 16 bytes.
- 1 KB = 10^3 B, 1 MB = 10^6 B, 1 GB = 10^9 B, 1 TB = 10^12 B.

### Useful Numbers
- Seconds/day = 86,400 ≈ 10^5.
- Seconds/month ≈ 2.5 × 10^6.
- QPS to daily volume: `QPS × 10^5`.

### Example: Twitter Tweets
- 500M users, 50% post once/day → 250M tweets/day = ~3K writes/sec average.
- Peak ≈ 3× average = 9K writes/sec.
- 1 tweet ≈ 300 bytes → 75 GB/day → ~27 TB/year.

### Example: Storage for 5 years of users
- 1B users × 1KB/user metadata = 1 TB.

### Tips
- Don't be precise — use orders of magnitude.
- State assumptions clearly: "assuming 50% of users active daily..."
- Convert read/write ratios early.

---

## 9. Key-Value Store Design

### What Is It?
A NoSQL store mapping `key → value`. Like a giant distributed HashMap.  
Examples: **DynamoDB, Redis, Cassandra (with KV semantics), etcd, Riak**.

### API
```
put(key, value)
get(key)
delete(key)
```

### Design Goals
- Horizontal scale to petabytes.
- High availability (99.99%+).
- Low latency (<10ms p99).
- Tunable consistency.

### Architecture Pieces
1. **Partitioning** — consistent hashing splits keys across nodes.
2. **Replication** — each key replicated to N nodes (typically N=3).
3. **Consistency model**:
   - **Quorum**: `R + W > N` for strong consistency (e.g., N=3, R=2, W=2).
   - Eventual consistency = anti-entropy + read repair.
4. **Vector Clocks** — track causal version history per key (used in Dynamo, Riak).
5. **Conflict Resolution**:
   - Last-Write-Wins (LWW) using timestamps.
   - Application-level merge (CRDTs).
6. **Membership & Failure Detection** — Gossip protocol.
7. **Persistence** — write-ahead log (WAL) + memtable + SSTables (LSM tree).

### LSM Tree (Used by Cassandra, RocksDB, LevelDB)
- Writes go to memtable (in-memory) + WAL.
- When memtable fills → flush to immutable SSTable on disk.
- Background compaction merges SSTables.
- Writes are FAST; reads need to check multiple SSTables (use Bloom filters to skip).

### Read/Write Flow
```
Write: client → coordinator → hash(key) → forward to W replicas → ack
Read:  client → coordinator → hash(key) → query R replicas → reconcile → return
```

### When to Use vs SQL
Use KV when: schemaless, simple access patterns, horizontal scale needed, latency-critical.
Don't use KV for: complex joins, analytics, multi-row transactions.

---

## 10. SQL vs NoSQL — When to Use Which DB

### SQL (Relational)
Examples: MySQL, PostgreSQL, Oracle, SQL Server.

**Strengths:**
- ACID transactions.
- Joins, complex queries.
- Mature ecosystem, BI tools.
- Strong consistency.

**Weaknesses:**
- Vertical scale ceiling; horizontal scale (sharding) painful.
- Rigid schema; migrations costly.
- Not ideal for unstructured/semi-structured data.

### NoSQL — Four Types

#### 1. Key-Value (DynamoDB, Redis, Riak)
Simple `key → value`. Fastest. Best for sessions, caches, simple lookups.

#### 2. Document (MongoDB, Couchbase)
Store JSON/BSON documents. Flexible schema. Best for user profiles, catalogs.

#### 3. Wide-Column (Cassandra, HBase, ScyllaDB)
Rows have varied columns; tables sparse. Best for time-series, IoT, write-heavy workloads.

#### 4. Graph (Neo4j, Amazon Neptune)
Nodes + edges. Best for social networks, fraud detection, recommendations.

### Decision Matrix
| Need | Pick |
|------|------|
| Strong consistency, complex queries, transactions | SQL |
| Massive write throughput, eventual consistency OK | Cassandra |
| Flexible schema, fast prototyping | MongoDB |
| Sub-millisecond reads, cache | Redis |
| Many-to-many relationships, graph traversals | Neo4j |
| Multi-region, low-latency reads | DynamoDB |

### Polyglot Persistence
Real systems use **multiple databases**:
- User accounts: PostgreSQL (ACID needed).
- Session: Redis.
- Activity feed: Cassandra.
- Search: ElasticSearch.
- Recommendations: Neo4j.
- Object/blob: S3.

### Interview Framing
Never say "X is better than Y" — say: "For this requirement (e.g., 100K writes/sec with eventual consistency), I'd choose Cassandra because of its wide-column model + tunable consistency."

---

## Summary Cheatsheet

| Topic | Key Insight |
|-------|-------------|
| **Protocols** | TCP for reliability, UDP for speed; WebSocket for real-time bidirectional |
| **Client-Server vs P2P** | C-S for control, P2P for resilience/scale |
| **CAP** | In a partition, choose C or A — most modern systems lean AP |
| **Microservices** | Decomposition + Saga + CQRS + API Gateway + Circuit Breaker |
| **Scaling 0→1M** | Single → LB → Cache → Replica → Shard → Microservices → Multi-DC |
| **Consistent Hashing** | Only 1/N keys move when N changes; use virtual nodes |
| **URL Shortening** | Base62 counter or KGS; cache aggressively |
| **Estimation** | Memorize Jeff Dean's numbers; convert QPS→daily quickly |
| **Key-Value Store** | Partition + replicate + quorum + LSM tree + gossip |
| **SQL vs NoSQL** | ACID & joins → SQL; scale & schema flexibility → NoSQL |

---

*This is the **foundation document**. Master these 10 topics before moving to specific system designs.*
