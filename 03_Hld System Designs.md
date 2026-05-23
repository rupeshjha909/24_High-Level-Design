# HLD Document 4: System Designs (Part 2)

Covers topics 26–32 — the remaining classic system designs.

---

## 26. Design Instagram

### Functional Requirements
- Upload photos/videos.
- Feed of posts from followed users.
- Like, comment, follow.
- Stories (24h expiry), Reels.
- Direct messages.
- Search (users, hashtags, places).

### Non-Functional
- **Heavily read-skewed** (people browse more than post).
- Low latency feed load (<200 ms).
- Media must be served from CDN.
- Highly available.

### Capacity
- 1B users, 500M DAU.
- 100M photos uploaded/day → ~1.2K uploads/sec.
- Avg photo 2 MB → 200 TB/day media.
- Feed reads ~10B/day → 100K/sec.

### Architecture
```
[Client] ──► [LB] ──► [API Gateway]
                      ├──► [User Service]
                      ├──► [Post Service] ──► [Cassandra (posts)]
                      ├──► [Feed Service] ──► [Redis (feed cache)]
                      ├──► [Media Service] ──► [S3 → CDN]
                      ├──► [Search Service] ──► [ElasticSearch]
                      └──► [Notification Service]
```

### Media Upload Flow
```
1. Client requests pre-signed S3 upload URL from Media Service.
2. Client uploads directly to S3 (avoids hitting our servers).
3. S3 event → Lambda → resize into multiple resolutions (thumbnail, 480p, 1080p).
4. Post Service stores: {post_id, user_id, caption, [media_urls], ts}.
5. Fan-out service updates followers' feed caches.
```

### Feed Generation
Same hybrid push-pull as Twitter:
- Normal users: push (fan-out on write).
- Celebrities: pull on read.

### Posts Data Model (Cassandra)
```
PARTITION KEY: user_id
CLUSTERING KEY: post_id (timestamp-based, like Twitter Snowflake)
COLUMNS: caption, media_urls[], likes_count, comments_count, ts
```

### Likes & Comments
- Likes: Cassandra wide row `post_id → set of user_ids`; counter for total.
- Comments: separate table partitioned by `post_id`.
- Counter sharding to avoid hot partitions on viral posts.

### Stories
- TTL of 24h.
- Stored separately with explicit expiration.
- Pre-aggregated per user; client polls every ~2 min.

### Newsfeed Ranking
Modern Instagram uses ML ranking (not chronological):
- Features: recency, user-user affinity, post type, engagement signals.
- Ranking service called during feed generation.

### Search
- ElasticSearch indexes: users, hashtags, captions.
- Autocomplete: trie-based (covered in Doc 2 #13).

### CDN Strategy
- All media goes through CDN.
- Pre-warm popular content during off-peak.
- Adaptive bitrate for videos (HLS / DASH).

---

## 27. Design YouTube

### Functional Requirements
- Upload videos (any length, any format).
- Stream videos with adaptive bitrate.
- Search videos.
- View counts, likes, comments, subscriptions.
- Personalized recommendations.
- Live streaming optional.

### Non-Functional
- Ingest petabytes of video/day.
- Serve hundreds of petabytes of video/day.
- Low buffering — even on slow networks.
- Highly available across regions.

### Capacity
- 500 hours uploaded per minute.
- ~5B videos watched/day.
- Storage: exabytes.

### Major Pieces

#### Upload Pipeline
```
[Client] ──► [Upload Service] ──► [Raw Video in S3]
                                       │
                                       ▼
                            [Transcoding Workers (parallel)]
                                       │
                                       ▼ multiple resolutions/formats
                            [S3] ──► [CDN edge caches]
```

#### Transcoding
- Most expensive operation.
- Convert original → multiple bitrates: 240p, 360p, 480p, 720p, 1080p, 4K.
- Split into segments (~5-10 sec each) for HLS/DASH streaming.
- Each video produces: {format} × {resolution} × {segment count} files.

#### Adaptive Bitrate Streaming (HLS / DASH)
- Video split into chunks; manifest file lists available bitrates.
- Client measures bandwidth → picks bitrate per segment → switches dynamically.

#### Storage Strategy
- **Hot videos**: CDN edge + nearby S3.
- **Cold videos**: cheap storage (S3 Glacier / GCS Coldline).
- **Pre-transcoded popular videos**: replicated to many edges.

### Architecture
```
[Uploader] ──► [Upload Service] ──► [S3 (raw)] ──► [Transcoding Cluster] ──► [S3 (variants)]
                                                                              │
                                                                              ▼
[Viewer] ──► [CDN] ◄── pre-warmed by ── [Replication Service]
              │
              └──► [Video Service] (for metadata, recommendations)
```

### Video Metadata Service
- Stores: video_id, title, description, uploader, tags, view_count, transcoded_urls.
- DB: MySQL or PostgreSQL for metadata (relational, queryable).

### View Count
- Incrementing on each play would hammer DB.
- Solution: count in Redis with periodic flush; Kafka stream for analytics.

### Recommendation System
- Offline: ML model (collaborative filtering + deep learning) on user history.
- Online: blend offline scores with real-time signals.
- Cached recommendations per user, refreshed periodically.

### Search
- ElasticSearch index over title, description, tags, transcripts.
- Query understanding (synonyms, typo correction).

### Live Streaming
- Producer pushes RTMP/SRT stream → ingest server → transcoder → CDN.
- Latency: traditional HLS (~10s); LL-HLS (~3s); WebRTC (~1s).

### Comments / Likes
- Comments partitioned by `video_id`.
- Likes as counters with sharded counter pattern.

### Key Insight
YouTube is fundamentally a **content delivery problem**. ~95% of cost is video storage + bandwidth. Architecture optimizes for CDN cache hit rate.

---

## 28. Design Google Drive

(Similar to Dropbox but with collaboration features.)

### Functional
- File upload/download/sync.
- Real-time collaboration on documents (Docs, Sheets, Slides).
- Sharing with permissions (view/comment/edit).
- Folder structure, search.
- Version history.

### Non-Functional
- Durable, available, low-latency.
- Real-time updates within milliseconds.

### Differences from Dropbox
1. **Collaboration**: multiple editors on same doc simultaneously.
2. **Online-first**: Drive opens docs in browser; Dropbox primarily syncs to local.
3. **Native editing**: Google's own document format (vs Dropbox storing user files as-is).

### Architecture (Storage Side — Similar to Dropbox)
```
[Client] ──► [Metadata Service] ──► [Spanner / Bigtable]
        ├──► [Block Service]    ──► [Object Store (Colossus)]
        ├──► [Sharing Service]
        └──► [Realtime Service] (for collab)
```

### Real-Time Collaboration

#### Operational Transformation (OT)
Original approach used by Google Docs.
- Each edit = operation (insert, delete).
- Server transforms operations to maintain consistency across clients.
- Example: A inserts "X" at position 5; B inserts "Y" at position 3 → server adjusts A's position to 6.

#### CRDTs (Newer Approach)
- Conflict-free Replicated Data Types.
- Operations commute → no transformation needed.
- Used by Figma, Notion, Yjs library.

### Document Storage
- Not stored as a flat file; stored as a **stream of operations**.
- Snapshot every N ops to allow fast load.
- Version history = list of ops.

### Sharing & Permissions
- ACL table: `(file_id, user_id_or_group, permission)`.
- Folder permissions inherit unless overridden.

### Search
- Index file contents (text extracted from PDFs, docs).
- ElasticSearch or proprietary index.

### Offline Support
- Service Worker caches docs locally.
- Edits queued; sync when online.

---

## 29. Design Web Crawler

### Functional
- Given seed URLs, crawl the web.
- Extract content; store for indexing.
- Respect robots.txt.
- Avoid duplicates.
- Configurable crawl depth.

### Non-Functional
- Scalable (billions of URLs).
- Polite (don't hammer servers).
- Fault-tolerant.
- Extensible (HTML, PDFs, images).

### Capacity
- Index 10B pages.
- Refresh every ~7 days → ~1.5K pages/sec needed.

### Architecture
```
[Seed URLs]
    │
    ▼
[URL Frontier (priority queue)] ◄────┐
    │                                │
    ▼                                │
[Fetcher Workers] ──► [HTML Storage] │
    │                                │
    ▼                                │
[Parser]                             │
    ├──► [Content Extractor] ──► [Search Index (ES)]
    ├──► [URL Extractor] ───► [Dedup Filter] ──► back to Frontier
    └──► [Indexer]
```

### URL Frontier
- Priority queue of URLs to crawl.
- Per-host queue for politeness — don't hit same host too fast.
- Priority based on PageRank, freshness, depth.

### Politeness
- Respect `robots.txt`.
- Crawl-delay header.
- Max N concurrent connections per host.
- Spread requests over time.

### Deduplication
- **URL dedup**: hash of normalized URL → check Bloom filter.
- **Content dedup**: MinHash / SimHash to detect near-duplicate pages.

### Storage
- Raw HTML → S3 (cheap, durable).
- Parsed content → search index.
- URL metadata → DB (last crawled, status, depth).

### Distributed Coordination
- URL Frontier partitioned by domain hash.
- Workers pull from local frontier; periodically rebalance.

### Avoiding Crawl Traps
- Detect spider traps (infinite calendars, session-based URLs).
- Limit URLs per host.
- Detect cycles via Bloom filter.

### Refresh Strategy
- Important sites (news) refreshed hourly.
- Static sites (docs) weekly.
- Adaptive: based on observed change frequency.

---

## 30. Design Facebook News Feed

### Functional
- Show feed of friends' posts.
- Personalized ranking (not strictly chronological).
- Post types: text, photo, video, link, share.
- Comments, reactions.
- Stories (24h ephemeral).

### Non-Functional
- Read-heavy (~50:1).
- Low latency (<200 ms).
- High availability.
- Massive scale (3B+ users).

### Core Problem: Feed Generation
Same fundamental issue as Twitter/Instagram: how to assemble a personalized feed at request time.

### Two Approaches

#### 1. Fan-out on Write (Push)
- When user posts, write to all friends' feed lists.
- Pro: read = single cache lookup.
- Con: heavy on write, especially for users with many friends.

#### 2. Fan-out on Read (Pull)
- On feed request, query each friend's recent posts.
- Pro: lightweight writes.
- Con: heavy reads; slow for users with many friends.

#### Hybrid (Facebook's actual)
- Push for users with <few thousand friends.
- Pull for highly connected users (celebs, public pages).

### Feed Ranking
This is the **secret sauce**. Facebook uses ML.
- Features per (user, candidate post): recency, affinity (interactions), post type, content quality, predicted CTR.
- Hundreds of signals fed into a deep learning model.
- Cached per user; refreshed every few minutes.

### Architecture
```
[Client] ──► [Edge / Load Balancer]
              ├──► [Feed Service]
              │      ├──► [Candidate Generation]
              │      │      ├──► [Friends' Recent Posts (cache)]
              │      │      └──► [Following Pages' Posts]
              │      ├──► [Ranking Service (ML)]
              │      └──► [Feed Aggregator]
              │
              ├──► [Post Service] ──► [TAO / Cassandra]
              ├──► [Reaction Service]
              ├──► [Comment Service]
              └──► [Ad Service] (interleave ads)
```

### TAO (Facebook's caching layer)
- Custom KV cache over MySQL.
- Optimized for the social graph.
- Stores objects (posts, comments) and edges (friend, like).

### Data Models

#### Post
```
post_id, user_id, content_type, text, media_url, ts, audience
```

#### Edge
```
(from_id, edge_type, to_id) — e.g., (alice, friend, bob), (alice, likes, post123)
```

### Edge Cases
- New post: needs to be in friends' feeds quickly (within seconds).
- Deleted post: must disappear from caches.
- Privacy: respect "Friends only", "Public", custom audiences.

### Caching Strategy
- Hot posts (viral) heavily cached at every layer.
- User affinity scores precomputed daily.
- Feed cache invalidation on new post for active users only (saves work on inactive users — they'll get fresh on next visit).

---

## 31. Design Ticketmaster (Event Ticket Booking)

(Similar to BookMyShow LLD but at HLD scale.)

### Functional
- Browse events by city, date, artist.
- View seat map and pricing.
- Reserve seat (hold) + checkout.
- Payment processing.
- E-ticket delivery.
- Resale marketplace optional.

### Non-Functional
- **Concurrency hot spot**: popular events draw millions of users at same minute.
- **No double-booking** (critical!).
- High availability.
- Fair queueing (anti-bot).

### Capacity
- During hot sale: 1M concurrent users; 100K bookings in first 5 minutes.

### Critical Challenges

#### 1. Preventing Double-Booking on Hot Events
The single most important problem.

**Approach: Seat-level distributed lock**
```
Redis: SETNX seat:show:S5 user_X EX 600
```
- Atomic; only one user can lock.
- TTL releases automatically if user abandons.

**Approach: Database row lock** (slower but more durable)
```
BEGIN;
SELECT * FROM seats WHERE show_id=X AND seat_id=S5 FOR UPDATE;
-- check if available
UPDATE seats SET status='HELD', user_id=Y WHERE seat_id=S5;
COMMIT;
```

#### 2. Waiting Room / Virtual Queue
For event launches with millions of users:
1. Users enter virtual queue.
2. Server admits N users/sec into actual booking flow.
3. Display position + estimated wait.
4. Prevents site crash; gives fairness.

#### 3. Bot Mitigation
- CAPTCHA on entry.
- Rate limit per IP/user.
- Behavioral analysis (mouse patterns).
- Token-based throttling.

### Architecture
```
[Client] ──► [DNS] ──► [GLB] ──► [WAF / Bot Filter]
                                       │
                                       ▼
                              [Virtual Queue Service]
                                       │ admit
                                       ▼
                              [Booking API]
                                ├─► [Seat Inventory Service] ──► [Redis + DB]
                                ├─► [Payment Service]
                                ├─► [Notification Service] (emails)
                                └─► [Search Service] ──► [ES]
```

### Seat Inventory
- Redis Cluster holds seat availability (hot data).
- DB is source of truth (durable).
- Background sync between Redis and DB.

### Hold Workflow
1. User picks seat → API → `SETNX seat:X` with 10-min TTL.
2. User completes payment within 10 min.
3. Success → confirm in DB → notify.
4. Timeout → seat auto-released.

### Payment Failure Handling
- 2-phase: hold seat + process payment.
- If payment fails: release hold.
- If payment succeeds but DB write fails: critical path — use saga pattern with compensating action.

### Data Model
```
events(id, name, venue_id, date, ...)
shows(id, event_id, time, ...)
seats(id, show_id, row, col, type, price)
holds(id, seat_id, user_id, expires_at)
bookings(id, user_id, seat_ids[], payment_id, status)
```

### Scaling
- Shard by `event_id` — each event's traffic isolated.
- Hot event (Taylor Swift) gets dedicated cluster.
- Geo-route to nearest region.

---

## 32. Design Nearby Friends / Yelp

### Common Underlying Problem
**Geospatial search**: given my location, find X within Y km.

### Functional (Nearby Friends)
- Users share live location with friends.
- See friends within N km on a map.
- Notify when friend nearby.

### Functional (Yelp)
- Search restaurants/businesses by name, category, location.
- Filter by rating, distance, open-now.
- Show details, reviews, photos.

### Non-Functional
- Low latency (<200 ms).
- Massive write rate for live location.
- High read rate for search.

### Core Challenge: Geospatial Indexing
Naive: `WHERE distance(user_loc, place_loc) < 5km` — full table scan, terrible.

### Geospatial Indexing Techniques

#### 1. Grid-based
- Divide world into cells (e.g., 1 km × 1 km).
- Store places per cell.
- Query: look up cell + 8 neighbors.
- Cons: edge cases at cell borders.

#### 2. Geohash
- Encode (lat, lng) as base32 string.
- **Nearby points share common prefix.**
- e.g., "dr5ru" → ~4.9 km × 4.9 km.
- Index in Redis sorted set or DB index.
- Query: find geohashes with matching prefix.

#### 3. Quadtree
- Recursive partition into 4 quadrants.
- Each leaf has bounded number of points.
- Dynamic — splits dense regions further.
- Used by traditional GIS.

#### 4. R-Tree
- Like B-Tree for rectangles.
- Used by PostGIS, MySQL spatial.

#### 5. H3 (Uber's Hexagonal Grid)
- Hexagons (no 4-corner edge problem).
- Multiple resolutions stackable.
- Used by Uber for ride matching.

### Yelp Architecture
```
[Client] ──► [LB] ──► [Search API]
                          │
                          ├──► [ElasticSearch] (full-text + geo)
                          ├──► [Business DB (MySQL)]
                          ├──► [Reviews DB (Cassandra)]
                          ├──► [Photos (S3 + CDN)]
                          └──► [Ranking Service]
```

#### Search Query Flow
1. User searches "pizza near Mumbai".
2. ElasticSearch query: `{text: pizza, geo_distance: {origin: [lat, lng], distance: 5km}}`.
3. Apply business filters (open now, rating).
4. Rank by combined score (relevance + distance + rating).
5. Return top 20.

### Nearby Friends Architecture
```
[Friend A's phone] ──► (location update every 30s) ──► [Location API]
                                                          │
                                                          ▼
                                            [Geospatial Index (Redis GEO)]
                                                          │
[Friend B's app] ──► (query nearby friends) ◄─────────────┘
                          │
                          ▼
                  [Returns A, distance, last_seen]
```

#### Why Redis GEO?
- Built-in geospatial commands (`GEOADD`, `GEORADIUS`).
- Sub-millisecond latency.
- Auto-expires stale locations.

#### Privacy
- Users opt in per friend.
- Approximate (round to ~100 m).
- Auto-disable after period of inactivity.

### Scaling Geospatial Queries
- Shard by **geohash prefix**: data for a city stays on its shard.
- Cache top-queried cells.
- Pre-compute popular searches.

### Hot Areas
- Popular regions (Manhattan, downtown Mumbai) have skewed load.
- Use replicated read-only shards for hot regions.
- Edge cache (CDN-like) for common queries.

---

## Summary Cheatsheet

| Topic | Key Insight |
|-------|-------------|
| **Instagram** | Pre-signed S3 URLs, push/pull hybrid for feed, CDN for media, ML ranking |
| **YouTube** | Transcoding pipeline + adaptive bitrate (HLS/DASH); CDN is the product |
| **Google Drive** | Same as Dropbox + OT/CRDT for collab; ops stream as storage model |
| **Web Crawler** | URL frontier (per-host queues), Bloom dedup, polite + adaptive refresh |
| **Facebook Feed** | Hybrid fan-out, ML ranking layer, TAO graph cache |
| **Ticketmaster** | Distributed locks (Redis SETNX), virtual queue, anti-bot, seat sharding |
| **Nearby Friends/Yelp** | Geohash / Quadtree / H3 for spatial indexing; Redis GEO for live locations |

---

## Master Cheatsheet — Component Reuse

The same patterns appear across designs. Internalize these reusable building blocks:

| Building Block | Used In |
|----------------|---------|
| **CDN** | YouTube, Instagram, Facebook, Dropbox media |
| **Object storage (S3)** | All media/file-heavy designs |
| **Cassandra (wide-column)** | Twitter, Instagram, Facebook posts |
| **Redis (cache + sorted sets)** | Feed timelines, leaderboards, rate limiters, geo |
| **Kafka** | Notifications, analytics, async fan-out |
| **ElasticSearch** | Search, autocomplete, geo queries |
| **Consistent hashing** | Caches, KV stores, load balancing |
| **Sharding** | Anything > 1 TB or > 10K writes/sec |
| **Fan-out on write vs read** | Feed-like systems |
| **Distributed lock (Redis SETNX)** | Booking, reservations, double-spend |
| **WebSocket** | Real-time chat, notifications, live tracking |
| **Bloom filter** | Dedup, cache penetration prevention |
| **Geospatial index (Geohash, H3)** | Maps, rides, location features |

---

*Master these 32 topics and you can attack any HLD interview. Start with Document 1 (foundations) — without those, the rest won't stick.*
