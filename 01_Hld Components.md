# HLD Document 2: Building Blocks & Components

Covers topics 11–20 — the **system components** you'll mix and match when designing any large-scale system. Most "Design X" questions reuse these components.

---

## 11. Design WhatsApp (Real-Time Messaging)

### Functional Requirements
- 1-to-1 chat, group chat (up to 256 members).
- Read receipts (sent, delivered, read).
- Online/last-seen status.
- Media sharing (images, videos, voice).
- End-to-end encryption.
- Offline message queuing.

### Non-Functional
- Low latency (<200 ms message delivery).
- High availability.
- Strong message ordering within a conversation.
- Massive scale (2B+ users).

### Capacity Estimation
- 2B users, 50B messages/day → ~600K messages/sec.
- Storage: 60 bytes/msg × 50B = 3 TB/day.

### Core Architecture
```
[Client] ──(WebSocket)──► [Gateway] ──► [Chat Server] ──► [Message DB]
                                            │
                                            ├──► [Presence Service]
                                            ├──► [Notification Service (push)]
                                            └──► [Media Service (S3 + CDN)]
```

### Key Design Decisions

#### Persistent Connection (WebSocket)
Each client maintains a long-lived WebSocket to a gateway. Gateway maps `user_id → gateway_node` in Redis.

#### Message Routing
1. User A sends msg → Gateway A → Chat Server.
2. Chat Server looks up User B's gateway in Redis → routes msg.
3. If User B offline → store in Message Queue → push notification.

#### Message Storage
- **Hot data**: last 30 days in Cassandra (write-heavy, time-series friendly).
- **Cold data**: archived to S3.
- Partition by `conversation_id`; sort by `timestamp`.

#### Message Schema (Cassandra)
```
PRIMARY KEY (conversation_id, message_timestamp)
columns: sender_id, message_text, media_url, status
```
- Query "last 50 messages in chat X" = single partition scan, fast.

#### End-to-End Encryption (Signal Protocol)
- Each user has identity keys + pre-keys.
- Double Ratchet algorithm → new key per message.
- Server never sees plaintext.

#### Read Receipts
Send small ACK messages over the same channel. Update DB asynchronously.

#### Group Chat
- Fan-out: server replicates message to N members' inboxes (push).
- For huge groups: pull-based — clients fetch on open.

### Scaling Tips
- **Sharding**: by `user_id` for connection state; by `conversation_id` for messages.
- **Media**: never through chat servers — upload to S3, send URL.
- **Push**: integrate APNs (iOS) / FCM (Android) for background notifications.

---

## 12. Design Rate Limiter (HLD Perspective)

(Full LLD covered separately; this is the HLD layer.)

### Where to Put It
1. **Client-side**: easy to bypass; only good for UX hints.
2. **Server-side middleware**: each service enforces; consistent.
3. **API Gateway**: central enforcement; reduces load on services.
4. **Sidecar proxy** (Istio): same as above but per-pod.

### Distributed Challenges
- **State sharing**: counters must be coordinated.
- **Solution**: centralized store (Redis) with atomic operations (`INCR`, Lua scripts).

### Algorithms (recap)
| Algorithm | Distributed Implementation |
|-----------|----------------------------|
| Fixed Window | Redis `INCR` + `EXPIRE` |
| Sliding Window | Sorted set in Redis with timestamps |
| Token Bucket | Redis Lua script (atomic refill + consume) |
| Leaky Bucket | Redis list + worker draining at fixed rate |

### Architecture
```
[Client] ──► [API Gateway w/ Rate Limiter] ──► [Service]
                       │
                       ▼
                  [Redis Cluster]
```

### Response Strategy
- Soft: warning header (`X-RateLimit-Remaining`).
- Hard: `429 Too Many Requests` + `Retry-After` header.

---

## 13. Design Search Autocomplete / Typeahead System

### Functional
- As user types, return top-K matching suggestions.
- Personalize (history, location, language).
- Update suggestions as fresh data arrives.

### Non-Functional
- <100 ms latency.
- High throughput (every keystroke = 1 query!).
- Fresh popular terms within minutes.

### Naive Approach (won't scale)
- Run `SELECT * FROM terms WHERE term LIKE 'prefix%'` on every keystroke. Dies at scale.

### Trie-Based Approach
```
       (root)
       /  |  \
      c   a   ...
     /     \
    a       p
    /       \
   t (3K hits)  p (1K hits)
```
- Each node stores **top-K terms** with their frequencies.
- Lookup: walk to prefix → return cached top-K.
- Updates: increment counters on writes; rebuild top-K periodically.

### Architecture
```
[Client keystroke] ──► [API] ──► [Trie Service (cached in RAM)]
                          │              ▲
                          ▼              │
                     [Analytics DB] ──► [Trie Builder Job]
                       (raw queries)
```

### Scaling
- Shard trie by **first character** (a-z → 26 shards).
- Each shard fits in RAM.
- Real systems: use **ElasticSearch's completion suggester** or **Redis sorted sets**.

### Build & Update Pipeline
- Stream user queries to **Kafka**.
- Aggregation job (Flink/Spark) counts terms over rolling 24h window.
- Rebuild trie/index every ~1 hour for popular terms.

### Real-World Implementations
- Google: highly personalized, uses ML rankings.
- Amazon: catalog-driven + popularity.
- ElasticSearch: built-in `completion suggester` with FST (Finite-State Transducer).

---

## 14. HLD Components: Message Queue (Kafka) & Proxy Servers

### Message Queue / Event Streaming

#### Why Use a Queue?
- **Decoupling**: producer doesn't wait on consumer.
- **Buffering**: absorb spikes.
- **Reliability**: durable storage.
- **Replay**: re-process events.
- **Fan-out**: one event → many consumers.

#### RabbitMQ vs Kafka
| Feature | RabbitMQ | Kafka |
|---------|----------|-------|
| Model | Broker pushes to consumer | Consumer pulls from log |
| Persistence | Optional, per-queue | Always, append-only log |
| Throughput | 10K–50K msg/s | 1M+ msg/s |
| Replay | No (consumed = gone) | Yes (offset-based) |
| Use Case | Task queues, RPC | Event streaming, analytics |

#### Kafka Architecture
```
[Producer] ──► [Topic: orders]
                  ├─ Partition 0 (leader: broker1, replicas: broker2,3)
                  ├─ Partition 1 (leader: broker2, replicas: broker1,3)
                  └─ Partition 2 ...
[Consumer Group] ──► reads from partitions (one consumer per partition)
```

#### Key Kafka Concepts
- **Topic**: logical channel.
- **Partition**: ordered, immutable log; unit of parallelism.
- **Offset**: position in partition.
- **Consumer Group**: load-balances partitions among consumers.
- **Replication factor**: replicas per partition for fault tolerance.
- **ISR (In-Sync Replicas)**: replicas caught up with leader.

#### Delivery Semantics
- At-most-once: fire and forget (may lose).
- At-least-once: retry until ACK (may duplicate).
- Exactly-once: Kafka transactions + idempotent producer.

### Proxy Servers

#### Forward Proxy
- Sits between **client and internet**.
- Hides client identity.
- Used for: corporate firewall, content filtering, VPN.

```
[Client] ──► [Forward Proxy] ──► [Internet]
```

#### Reverse Proxy
- Sits between **internet and servers**.
- Hides server topology, load-balances, caches, terminates SSL.
- Examples: Nginx, HAProxy, Envoy.

```
[Internet] ──► [Reverse Proxy] ──► [App Server 1]
                              ──► [App Server 2]
```

#### Common Functions
- **Load balancing** (round-robin, least connections, IP hash).
- **SSL termination** (decrypt once, forward HTTP internally).
- **Caching** static assets.
- **Compression** (gzip).
- **Rate limiting & WAF** (web application firewall).

#### Load Balancer Layers
- **L4 (Transport)**: routes by IP + port. Faster, simpler. Examples: AWS NLB.
- **L7 (Application)**: routes by URL, headers, cookies. Smarter. Examples: AWS ALB, Nginx.

---

## 15. CDN (Content Delivery Network)

### What Is It?
A geographically distributed network of edge servers that cache and serve content close to users.

### Why?
- Reduce latency (closer to user).
- Reduce origin load (95%+ requests served from edge).
- Absorb DDoS attacks (huge edge bandwidth).
- Better availability.

### How It Works
```
1. User in Mumbai requests image.com/logo.png
2. DNS resolves to nearest CDN PoP (Mumbai edge).
3. Edge has cached file → serves directly (cache HIT).
4. If miss → fetch from origin → cache → serve.
```

### Cache Strategies
- **Pull (lazy)**: CDN fetches on first miss. Simple. Cold cache slow.
- **Push (eager)**: origin pushes new content to CDN proactively. Used for new releases.

### Cache Invalidation
- **TTL-based**: expire after N seconds.
- **Versioned URLs**: change URL when content changes (e.g., `logo.v2.png`).
- **Purge API**: explicit invalidation per file.

### Static vs Dynamic Content
- **Static** (images, CSS, JS, videos): perfect for CDN.
- **Dynamic** (HTML for logged-in users): use **edge computing** (Cloudflare Workers, Lambda@Edge).

### Major Providers
Cloudflare, Akamai, Fastly, AWS CloudFront, Google Cloud CDN.

### Anycast Routing
Same IP advertised from multiple PoPs; BGP routes user to nearest one.

---

## 16. Storage Types: Block, File, Object Storage, RAID

### Block Storage
- Raw block devices (like a virtual hard drive).
- No filesystem on the storage itself — host attaches and formats.
- Low latency, high IOPS.
- Examples: **AWS EBS, Azure Disks, iSCSI SANs**.
- Use case: databases, VM disks.

### File Storage
- Hierarchical filesystem (folders + files) accessed via NFS/SMB.
- Multiple clients mount it concurrently.
- Examples: **AWS EFS, NFS, GlusterFS, Azure Files**.
- Use case: shared content, home directories, CMS uploads.

### Object Storage
- Flat namespace; objects = data + metadata + globally unique key.
- Access via HTTP (S3 API).
- Infinitely scalable; cheap; ideal for unstructured data.
- Examples: **AWS S3, GCS, Azure Blob, MinIO**.
- Use case: backups, media files, data lakes, static website hosting.

### Comparison
| | Block | File | Object |
|-|-------|------|--------|
| Access | Block device | NFS/SMB | HTTP/REST |
| Hierarchy | None | Tree | Flat (keys) |
| Performance | Highest | Medium | Lower (HTTP) |
| Scale | Per-disk limit | Multi-PB | Exabyte |
| Cost | High | Medium | Low |
| Use Case | DBs | Shared FS | Media, backups |

### RAID (Redundant Array of Independent Disks)
Combines multiple disks for performance and/or redundancy.

| Level | Description | Pros | Cons |
|-------|-------------|------|------|
| RAID 0 | Striping | Fast | No redundancy (1 disk fail = data loss) |
| RAID 1 | Mirroring | Redundancy | 50% capacity wasted |
| RAID 5 | Striping + parity (3+ disks) | Good balance | Slow writes |
| RAID 6 | Like 5, double parity | Survives 2 disk failures | More space overhead |
| RAID 10 | Stripe + mirror | Fast + safe | Expensive |

Modern systems prefer **distributed replication** over RAID (e.g., HDFS does 3x replication instead).

---

## 17. File Systems (Google File System, HDFS)

### Why Distributed File System?
- Single machine can't store petabytes.
- Need fault tolerance, parallel reads, throughput optimization.

### Google File System (GFS) — 2003 Paper

#### Architecture
```
[Client] ──► [Master] (single, metadata only)
   │             │
   │             └──► (assigns chunks to chunk servers)
   ▼
[Chunk Server 1] [Chunk Server 2] [Chunk Server 3] ... 
   (each chunk replicated 3x)
```

#### Design Choices
- **Append-mostly workload** (logs, web pages).
- Files split into **64 MB chunks**, each replicated 3x.
- **Single master** holds all metadata in memory (file → chunk mapping).
- Clients talk directly to chunk servers for data.

#### Reads
1. Client → Master: "where's file X chunk 5?"
2. Master returns chunk locations (cached).
3. Client → nearest chunk server → reads data.

#### Writes (Append)
1. Master picks primary replica.
2. Client sends data to all replicas (pipelined).
3. Primary serializes appends, replicates to others.

#### Failures
- Chunk server fails → master replicates to maintain 3x.
- Master fails → shadow masters take over (single point of failure originally).

### HDFS (Hadoop Distributed File System)
- Open-source clone of GFS.
- **NameNode** = master, **DataNodes** = chunk servers.
- Block size = 128 MB (vs GFS 64 MB).
- High-throughput, batch-friendly (not low-latency).
- Used by: Hadoop MapReduce, Hive, Spark.

### Modern Distributed File Systems
- **Ceph** (object + block + file).
- **GlusterFS** (POSIX-compliant, scales linearly).
- **AWS S3** (object storage, dominant for cloud).

---

## 18. Bloom Filter

### What Is It?
A **probabilistic data structure** that tests whether an element is in a set:
- **False positives possible** ("maybe in set").
- **False negatives impossible** ("definitely not in set").

### How It Works
- Bit array of size `m`, all initially 0.
- `k` hash functions.

**Insert(x)**: For each of k hash funcs → set bit `hash_i(x) % m` to 1.

**Lookup(x)**: For each of k hash funcs → check bit. If ALL are 1 → "maybe". If ANY 0 → "definitely no".

### Example
```
m = 16, k = 3
Insert "apple": hashes give 2, 7, 13 → set those bits.
Lookup "banana": hashes give 4, 7, 11 → bit 4 = 0 → DEFINITELY NOT in.
Lookup "ape":    hashes give 2, 7, 13 → all 1 → MAYBE in (could be false positive).
```

### False Positive Rate
`(1 - e^(-kn/m))^k` where n = elements, m = bits, k = hash functions.

### Use Cases
- **Cassandra/RocksDB**: skip SSTables that don't contain a key (huge speedup).
- **CDNs**: check if URL is in cache before disk lookup.
- **Web crawlers**: avoid re-visiting URLs.
- **Spell checkers**: dictionary membership.
- **DB joins**: pre-filter rows.
- **Bitcoin SPV nodes**: detect relevant transactions without full chain.

### Variations
- **Counting Bloom Filter**: supports deletion (counters instead of bits).
- **Cuckoo Filter**: supports deletion + better space efficiency.
- **Scalable Bloom Filter**: grows as elements added.

### Why Use It?
Tiny memory + O(k) operations vs HashSet's O(n) memory.  
**Example**: 1B URLs at 1% FP → ~1.2 GB Bloom filter vs 100+ GB HashSet.

---

## 19. Merkle Tree & Gossip Protocol

### Merkle Tree

#### What Is It?
A binary tree where each leaf is the hash of a data block, and each non-leaf is the hash of its children. Root hash represents the entire dataset.

```
        H_root = hash(H_AB + H_CD)
        /              \
   H_AB                H_CD
  /    \              /    \
H_A    H_B          H_C    H_D
 |      |            |      |
data1 data2        data3  data4
```

#### Why?
- **Efficient verification**: compare root hashes — if same, datasets identical.
- **Pinpoint differences**: walk tree from root; only descend where hashes differ. O(log n) instead of O(n).

#### Use Cases
- **Git**: every commit is a tree of file hashes.
- **Bitcoin/Ethereum**: block headers contain a Merkle root of transactions.
- **Cassandra/DynamoDB**: anti-entropy (compare replicas, sync only diffs).
- **IPFS**: content-addressed storage.
- **BitTorrent**: verify chunks of large files.

### Gossip Protocol

#### What Is It?
A decentralized communication style where each node randomly picks peers and exchanges state — like a rumor spreading.

#### Why Use It?
- No central coordinator.
- Scalable to thousands of nodes.
- Eventually consistent membership / failure detection.
- Tolerates churn.

#### How It Works
```
Every 1 sec, each node:
  1. Pick random peer.
  2. Exchange (membership, version vector, heartbeat).
  3. Merge differences.
```

After O(log N) rounds, all nodes converge.

#### Used By
- **Cassandra**: cluster membership + schema propagation.
- **Consul**, **Serf**: service discovery.
- **Dynamo / DynamoDB**: failure detection.
- **Bitcoin**: peer discovery + tx propagation.

### Anti-Entropy = Merkle Tree + Gossip
Cassandra periodically:
1. Compare Merkle tree roots between replicas (via gossip).
2. Walk diffs.
3. Sync only mismatched chunks → bounded bandwidth use.

---

## 20. Caching — Invalidation & Eviction

### Why Cache?
- Reduce latency.
- Reduce load on origin (DB).
- Reduce cost.

### Cache Levels
1. **Browser cache** (client-side).
2. **CDN** (edge).
3. **Application cache** (local in-memory).
4. **Distributed cache** (Redis, Memcached).
5. **Database query cache** (built-in or materialized views).

### Cache Patterns

#### Cache-Aside (Lazy Loading)
```
read(key):
  v = cache.get(key)
  if v is None:
    v = db.get(key)
    cache.set(key, v)
  return v
```
Most common. App owns logic.

#### Write-Through
Every write goes to cache + DB synchronously.
- Pro: cache always fresh.
- Con: extra write latency.

#### Write-Back (Write-Behind)
Write to cache; asynchronously persist to DB.
- Pro: fast writes.
- Con: data loss risk if cache crashes.

#### Write-Around
Write directly to DB, skip cache; cache populated only on read.
- Good for write-heavy + rarely-read data.

#### Refresh-Ahead
Cache proactively refreshes hot keys before TTL expires. Avoids cold-cache spikes.

### Cache Invalidation Strategies (HARD problem!)

#### 1. TTL (Time-To-Live)
Set expiry; cache evicts after N seconds. Simple, but possibly stale.

#### 2. Explicit Invalidation
On write: `cache.delete(key)` or `cache.set(key, new_value)`.
Risk: race conditions, stale reads in distributed cache.

#### 3. Write-Through (described above)
Cache and DB always in sync.

#### 4. Event-Driven
DB publishes change events (CDC — Change Data Capture); cache subscribes & invalidates.
Examples: Debezium → Kafka → cache invalidator.

### Eviction Policies (When Cache Full)
| Policy | Description | Use Case |
|--------|-------------|----------|
| **LRU** | Evict least recently used | General-purpose |
| **LFU** | Evict least frequently used | Stable workloads |
| **FIFO** | Evict oldest insertion | Simple |
| **Random** | Evict random | Surprisingly OK for some workloads |
| **TTL-based** | Evict expired first | Time-bound data |
| **W-TinyLFU** | Hybrid LFU + LRU windows (Caffeine) | Modern, best for most workloads |

### Common Pitfalls
- **Cache Stampede / Thundering Herd**: Many requests miss simultaneously → all hit DB.
  - Fix: request coalescing (single fetch + others wait), jittered TTL, probabilistic early refresh.
- **Hot Key**: One key gets 90% of traffic → single node overload.
  - Fix: replicate hot keys to multiple nodes; client-side fan-out.
- **Cache Avalanche**: Many keys expire at the same moment.
  - Fix: jittered TTLs.
- **Cache Penetration**: Lookups for non-existent keys bypass cache → hammer DB.
  - Fix: cache "negative" results (with short TTL) or use Bloom filter.

### Cache Sizing
- Working set ≈ data accessed in last 24h.
- Aim for 80%+ hit rate.
- Right-size based on memory cost vs DB load reduction.

### Tools
- **Redis** — KV + data structures, replication, persistence.
- **Memcached** — simpler KV, multi-threaded.
- **Caffeine** (Java) — in-process W-TinyLFU.
- **Hazelcast / Apache Ignite** — distributed in-memory grids.

---

## Summary Cheatsheet

| Topic | Key Insight |
|-------|-------------|
| **WhatsApp** | WebSocket + connection mapping + Cassandra + E2E encryption |
| **Rate Limiter** | API Gateway + Redis Lua scripts for distributed enforcement |
| **Typeahead** | Trie with cached top-K per node; shard by first char |
| **Kafka** | Append-only log, partitioned, replicated, replay-able |
| **Proxy** | Forward = hides client; Reverse = hides servers |
| **CDN** | Edge caching, pull/push, TTL or version-based invalidation |
| **Storage** | Block (DBs), File (shared FS), Object (media/backup) |
| **GFS/HDFS** | Master + chunk servers, 64-128MB blocks, 3x replication |
| **Bloom Filter** | Tiny memory, false positives only, used to skip lookups |
| **Merkle Tree** | Detect diffs in O(log n); used in Git, blockchain, Cassandra |
| **Gossip** | Decentralized state propagation; eventually consistent |
| **Caching** | Cache-aside default; TTL + explicit invalidation; beware stampede & hot keys |

---

*These components are the **Lego blocks** of any large system. Master each, then mix & match in design questions.*
