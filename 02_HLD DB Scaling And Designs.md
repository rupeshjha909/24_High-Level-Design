# HLD Document 3: Database Scaling & System Designs (Part 1)

Covers topics 21–25 — database scaling techniques + four classic system designs that exercise them.

---

## 21. How to Scale a Database

When a single database can't handle the load, you must scale. Here are the main techniques.

### 21.1 Vertical Scaling (Scale-Up)
- Bigger CPU, more RAM, faster SSD.
- Easiest; no code change.
- Hits hardware ceiling fast.
- Single point of failure.

### 21.2 Read Replicas (Replication)
```
[Writer] ──► [Master DB]
                 │ replicates
                 ▼
[Reader 1] ◄── [Slave 1]
[Reader 2] ◄── [Slave 2]
[Reader 3] ◄── [Slave 3]
```
- Writes go to master; reads spread across replicas.
- Async replication → small replication lag (eventually consistent reads).
- Use for **read-heavy workloads**.

#### Replication Modes
| Mode | Pros | Cons |
|------|------|------|
| **Async** | Fast writes | Possible data loss on master crash |
| **Semi-sync** | At least one replica confirmed | Slightly slower writes |
| **Sync** | No data loss | Slow; replica failure blocks writes |

### 21.3 Sharding (Horizontal Partitioning)

When write throughput maxes out the master, split data across multiple DBs (shards).

#### Sharding Strategies

**1. Range-based sharding**
```
shard1: user_id 1–1M
shard2: user_id 1M–2M
shard3: user_id 2M–3M
```
- Pros: simple; range queries efficient.
- Cons: hotspots if data is skewed (e.g., recent users).

**2. Hash-based sharding**
```
shard = hash(user_id) % N
```
- Pros: even distribution.
- Cons: range queries require scatter-gather; adding/removing shards reshuffles everything.

**3. Consistent hashing** (best for elastic clusters)
- Only ~1/N keys move when changing N.

**4. Geographic sharding**
- Shard by region: `us-east`, `eu-west`, `ap-south`.
- Pros: data locality, compliance (GDPR).
- Cons: cross-region joins slow.

**5. Directory-based sharding**
- Lookup table: `user_id → shard_id`.
- Most flexible; extra indirection.

#### Challenges
- **Joins across shards** → must do app-level joins or denormalize.
- **Transactions across shards** → distributed transactions (slow) or sagas.
- **Resharding** → painful; plan ahead.
- **Hotspot mitigation** → split hot shards, replicate hot keys.

### 21.4 Partitioning (a.k.a. Vertical Partitioning)
Split a single table by **columns** into smaller tables.
- e.g., User table → `user_basic` (id, name) + `user_profile` (bio, photo) + `user_preferences`.
- Useful when some columns are huge and rarely accessed.

#### Horizontal Partitioning
- Split table by **rows** within the same DB (e.g., partition by date).
- Different from sharding (sharding spans multiple DBs).

### 21.5 Leader Election

In replicated systems, only one node should be the **leader (master)** at a time. Need automatic failover.

#### Algorithms
- **Raft** (used by etcd, Consul, CockroachDB).
- **Paxos** (Google Chubby).
- **Zookeeper-based** (older Kafka, HBase).
- **Bully algorithm** (simple, used in many academic systems).

#### How Raft Works (High Level)
1. All nodes start as **followers**.
2. If no heartbeat from leader → become **candidate**; request votes.
3. Majority votes → become **leader**; send heartbeats.
4. New leader takes over; old leader steps down on rejoin.

### 21.6 Indexing

Speeds up reads at the cost of write performance and storage.

#### Index Types
- **B-Tree** (most common): balanced; range queries fast.
- **Hash index**: O(1) equality lookup; no range queries.
- **LSM tree**: write-optimized (Cassandra, RocksDB).
- **Inverted index**: term → list of docs (ElasticSearch).
- **Bitmap index**: low-cardinality columns (gender, status).
- **GIN/GIST** (PostgreSQL): full-text, geo, JSONB.

#### Index Design Rules
1. Index columns used in WHERE, JOIN, ORDER BY.
2. Composite indexes follow **leftmost prefix rule**.
3. More indexes = slower writes (every update modifies indexes).
4. Cover queries with composite indexes to avoid table lookups.

### Database Scaling Decision Tree
```
Is workload read-heavy? 
    ├─ YES → Add Read Replicas
    └─ NO → Continue
Is data set large?
    ├─ YES → Shard / Partition
    └─ NO → Continue
Is DB still struggling?
    ├─ YES → Add Caching Layer (Redis)
    └─ Still? → Move to NoSQL or specialized stores
```

---

## 22. Design Notification System

(LLD covered separately; this is the HLD layer.)

### Functional Requirements
- Send notifications via Email, SMS, Push, In-app.
- Support transactional + promotional traffic.
- User preferences per channel.
- Template-based messages.
- Retry on failure.
- Track delivery status.
- Schedule future notifications.

### Non-Functional
- High throughput (millions/day).
- Reliability (no lost notifications for critical messages).
- Low latency for transactional (OTPs <5s).
- Idempotent (no duplicate sends).

### Capacity Estimation
- 100M users × 5 notifs/day = 500M notifs/day = ~6K/sec average.
- Peak ~30K/sec.

### High-Level Architecture
```
                        ┌─► [Email Worker] → SendGrid
[Service] ──► [Notification ──► [Kafka] ──► [SMS Worker]   → Twilio
              Service API]      Topics    ─► [Push Worker]  → FCM/APNs
                  │                       └─► [In-App Worker]
                  ▼
         [Template Service]
         [User Preferences]
         [Rate Limiter]
```

### Design Decisions

#### Why Async (Kafka)?
- Sender shouldn't wait for delivery.
- Buffer spikes (promo campaign blasts).
- Replay on failure.
- Decouple senders from channels.

#### Priority Queues
Separate topics for **transactional** vs **promotional**:
- Transactional consumed first (OTP, password reset).
- Promotional dequeued during quiet hours.

#### Retry Strategy
- Exponential backoff: 1s, 2s, 4s, 8s...
- Max retries (e.g., 5).
- Failed permanently → Dead Letter Queue for investigation.

#### Idempotency
Producer includes `idempotency_key`; consumer checks before sending. Stored in Redis with 24h TTL.

#### User Preferences Service
- Stores per-channel opt-ins.
- Check before sending; skip if opted out.

#### Templates
- Stored in DB with placeholders: `Hi {{name}}, your OTP is {{otp}}`.
- Localization keyed by language.

#### Status Tracking
- After send: store `(notif_id, status, timestamp)` in Cassandra.
- Status: PENDING → SENT → DELIVERED / FAILED.
- Receivers (SendGrid, FCM) post delivery webhooks back.

### Scaling
- **Sharding by user_id**: distribute load across worker fleets.
- **Per-channel rate limiting**: respect Twilio/SendGrid quotas.
- **Geo-routing**: send via regional providers for latency.

---

## 23. Design Pastebin

### Functional
- Paste text → get short URL.
- Anyone with URL can read paste.
- Optional: expiration, syntax highlighting, password, private pastes, edit history.

### Non-Functional
- High read:write ratio (~100:1).
- Low write latency.
- Durability (don't lose pastes).
- Decent search? (out of scope usually).

### Capacity Estimation
- 1M new pastes/day → ~12 writes/sec.
- 100M reads/day → ~1.2K reads/sec.
- Avg paste = 10 KB → 10 GB/day → 3.6 TB/year.

### Similar to URL Shortener
**Major difference**: store large text content, not a URL.

### Architecture
```
[Client] ──► [LB] ──► [API Server] ──► [Cache (Redis)]
                          │                  │
                          ▼                  ▼
                     [Key Gen]         [Object Store (S3)]
                          │
                          ▼
                     [Metadata DB]
```

### Key Generation
- Base62 short key (like URL shortener).
- 7 chars × 62 = 3.5 trillion possible.
- Pre-generate via KGS (key generation service).

### Storage Decisions
| Data | Where | Why |
|------|-------|-----|
| Short key → S3 URL mapping | DynamoDB / Cassandra | Fast KV lookup |
| Paste content | S3 / Blob storage | Cheap, scalable |
| User pastes index | PostgreSQL | Relational queries |

### Why S3 for content?
- Cheap ($23/TB/month).
- Highly durable (11 9's).
- Infinitely scalable.
- DB stores only metadata + S3 URL.

### Read Flow
1. GET /AbC1234 → cache hit? → return.
2. Cache miss → DB lookup → S3 fetch → fill cache → return.

### Write Flow
1. POST /paste with content.
2. Generate short key (KGS).
3. Upload content to S3 at `s3://pastes/AbC1234`.
4. Insert metadata: `{key: AbC1234, s3_url, expires_at, owner}`.
5. Return short URL.

### Expiration Cleanup
- Background job scans DB for expired pastes; deletes from S3 + DB.
- Or use S3 lifecycle policies (auto-expire by TTL tag).

### Optional Features
- **Syntax highlighting**: client-side (highlight.js).
- **Diffing**: store versions; UI shows diff.

---

## 24. Design Twitter

### Functional Requirements
- Post tweets (280 chars, optional media).
- Follow users.
- Home timeline (tweets from followed users, latest first).
- User timeline (own + retweets).
- Like, retweet, reply, search.
- Trending topics, notifications.

### Non-Functional
- **Read-heavy** (~100:1 read:write).
- **Low latency** (<200 ms home timeline).
- Eventual consistency OK (some replies appear slightly late).
- Scale: 500M users, 500M tweets/day.

### Capacity Estimation
- Writes: 500M tweets/day ≈ 6K/sec; peak 18K/sec.
- Reads: home timeline views ~30B/day ≈ 350K/sec.
- Storage: 280 bytes × 500M = 140 GB/day text; 1 PB/year with media.

### Core Challenge: Home Timeline Generation

#### Pull Model (compute on read)
```
On user visit: SELECT * FROM tweets 
               WHERE user_id IN (followees of X) 
               ORDER BY ts DESC LIMIT 50
```
- Pros: no precomputation.
- Cons: too slow at scale; users with 1000 followees → huge query.

#### Push Model / Fan-out on Write (precompute on tweet)
```
When user posts:
  for each follower of user:
    insert tweet into follower's timeline cache (Redis list)
```
- Pros: home timeline read = single cache lookup, fast.
- Cons: celebrity problem — user with 100M followers triggers 100M writes!

#### Hybrid (Twitter's actual approach)
- **Regular users**: push (fan-out on write).
- **Celebrities (>1M followers)**: pull on read for celeb's tweets, merge with cached timeline.

### Architecture
```
[Client] ──► [LB] ──► [API] ──┬──► [Tweet Service] ──► [Cassandra (tweets)]
                              │
                              ├──► [Timeline Service] ──► [Redis (timeline cache)]
                              │
                              ├──► [User Service] ──► [User DB]
                              │
                              └──► [Search Service] ──► [ElasticSearch]
```

### Data Model

#### Tweets (Cassandra)
```
PRIMARY KEY ((user_id), tweet_id DESC)
columns: text, media_url, like_count, retweet_count, ts
```
Partition by user_id → user timeline = single partition scan.

#### Followers/Following (graph)
- PostgreSQL or graph DB.
- Indexed both ways: who I follow, who follows me.

#### Home Timeline (Redis)
```
ZADD timeline:user_X <score=tweet_ts> <tweet_id>
```
Sorted set per user; trim to last 800 tweets.

### Fan-out Service
```
On new tweet T by user U:
  followers = get_followers(U)
  if len(followers) < 1M:
    for f in followers:
      redis.zadd("timeline:" + f, score=T.ts, value=T.id)
  else:
    # celebrity — pull model
    mark T as celebrity tweet
```

### Reading Home Timeline
```
def home_timeline(user):
  cached = redis.zrevrange("timeline:" + user, 0, 49)
  celeb_tweets = get_recent_celeb_tweets(user.celeb_followees)
  return merge_sorted(cached, celeb_tweets)[:50]
```

### Search
- Index tweets in ElasticSearch on write.
- Trending: stream of tweets → real-time aggregator (Flink) → top hashtags by recent count.

### Media
- Upload to S3 → URL stored in tweet record.
- CDN for serving (Cloudflare/Akamai).

### Optimization Notes
- Pre-warm Redis caches based on user activity (push during early-morning quiet hours).
- Tiered storage: hot (Redis) → warm (Cassandra) → cold (S3).
- Asymmetric: top 1% users get 90% of read traffic — special handling.

---

## 25. Design Dropbox (File Storage & Sync)

### Functional
- Upload, download, delete files.
- Sync across devices.
- Share files/folders (with permissions).
- Version history.
- Offline edits sync when reconnected.
- Real-time collaboration optional.

### Non-Functional
- Durable (never lose user data — 11 9's).
- Available (99.99%).
- Fast sync on small changes (don't re-upload entire file).
- Scale to billions of files / exabytes.

### Capacity
- 500M users, 1 GB/user avg = 500 PB.
- Daily uploads/downloads: petabytes.

### Key Design Insights

#### 1. Chunking (most important!)
Files split into **4 MB chunks**. Each chunk hashed (SHA-256).

Benefits:
- **Delta sync**: only changed chunks re-uploaded.
- **Deduplication**: same chunk on multiple users → store once.
- **Resumable uploads**: failed upload? Resume from last good chunk.
- **Parallel uploads**.

#### 2. Metadata vs Content Separation
- **Metadata** (file name, owner, hash, chunks list) → PostgreSQL (relational, ACID).
- **Content** (chunks) → Object storage (S3-like).

#### 3. Client-side caching + sync
- Local file system index.
- Watcher detects changes → diff with last sync → upload changed chunks.

### Architecture
```
[Client (Watcher)] ──► [Metadata Service] ──► [Metadata DB]
        │
        ├─► [Block Service] ──► [Object Store: S3 / GCS]
        │
        ├─► [Notification Service] ──► [Sync notifs to other devices]
        │
        └─► [Sharing Service]
```

### Upload Flow
```
1. Client splits file into 4 MB chunks; hashes each.
2. Client → Block Service: "do you have hash H1, H2, H3?"
3. Server returns missing hashes.
4. Client uploads only missing chunks to S3.
5. Client → Metadata Service: "file F = [H1, H2, H3]".
6. Metadata Service updates DB; notifies other devices via Notification Service.
```

### Download Flow
```
1. Client → Metadata: get file F's chunks.
2. Client → Block Service: fetch H1, H2, H3 from S3.
3. Reassemble locally.
```

### Sync
- Client listens (long-poll or WebSocket).
- On change: server pushes notification; client fetches diff.
- For offline: client queues local changes; on reconnect, push and conflict-resolve.

### Conflict Resolution
- Multiple devices edit same file → both versions saved with `(filename) (deviceA's conflicting copy)`.
- True real-time collab (Google Docs) needs OT (Operational Transform) or CRDTs.

### Deduplication
Crucial! For 1 PB raw uploads, real storage might be 200 TB.
- Block-level dedup using chunk hashes.
- Optional: cross-user dedup (privacy concern in some cases — encrypted blocks prevent it).

### Sharing & Permissions
- Share table: `(file_id, shared_with_user_id, permission)`.
- Generate shareable URL with token; access controlled by server check.

### Encryption
- **At rest**: server-side encryption in S3.
- **In transit**: HTTPS.
- **E2E (advanced)**: client encrypts before upload — server can't dedup (Mega.nz model).

### Optimizations
- **Bandwidth Delta Sync**: only transfer changed bytes within chunks (rsync algorithm).
- **CDN for popular shared files**.
- **Compression** for text/code files.

---

## Summary Cheatsheet

| Topic | Key Insight |
|-------|-------------|
| **DB Scaling** | Replicate (read), Shard (write), Partition (rows/cols), Leader Election (failover), Index (read speed) |
| **Notification System** | Async via Kafka, priority topics, retry + DLQ, idempotency |
| **Pastebin** | Like URL shortener but content in S3 (not DB) — meta in DB |
| **Twitter** | Push for normal users, pull for celebrities; Redis sorted sets for timelines |
| **Dropbox** | Chunking + dedup + metadata/content split; chunked sync = bandwidth efficient |

---

*Continue to Document 4 for the remaining system designs.*
