# Designing Instagram: A Senior Interview Guide

> A practical, interview-focused reference for designing a media-centric social platform — sharing the social-feed skeleton of Twitter but dominated by a heavy media pipeline, ML-ranked feeds, and ephemeral Stories. Covers the media upload/processing flow, CDN delivery, feed generation, ranking, counters at scale, and Stories. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This (and the Twitter Connection)](#1-how-to-approach-this-and-the-twitter-connection)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Distinctive Part: The Media Pipeline](#4-the-distinctive-part-the-media-pipeline)
5. [Architecture](#5-architecture)
6. [Feed Generation & ML Ranking](#6-feed-generation--ml-ranking)
7. [Data Model](#7-data-model)
8. [Likes, Comments & Counters at Scale](#8-likes-comments--counters-at-scale)
9. [Stories (Ephemeral Content)](#9-stories-ephemeral-content)
10. [Search, DMs & CDN Strategy](#10-search-dms--cdn-strategy)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This (and the Twitter Connection)

Instagram's **social-feed core is essentially Twitter's** (see that guide): a read-heavy feed of posts from followed accounts, built with the same **hybrid push/pull fan-out**, Cassandra for posts, and Redis for precomputed feeds. So the efficient interview move is to **state that parallel quickly** ("the feed generation is the same hybrid fan-out as Twitter") and then **spend your time on what makes Instagram different:**

1. **It's media-dominated** — posts are photos/videos, not 280 bytes of text, so the **media pipeline** (upload, transcode, store, serve via CDN) is the bulk of the engineering.
2. **The feed is ML-ranked**, not chronological — a ranking model scores posts by predicted engagement.
3. **Ephemeral Stories** (24h expiry) and **Reels** add content types with their own handling.

Senior signal: recognizing that the feed metadata is the *easy, already-solved* part (it's Twitter), while the **media pipeline and CDN** are where the real scale lives, and that modern feeds are *ranked*, not time-ordered.

---

## 2. Requirements

### Functional

- **Upload** photos/videos.
- **Feed** of posts from followed users.
- **Like, comment, follow.**
- **Stories** (24h expiry), **Reels.**
- **Direct messages.**
- **Search** (users, hashtags, places).

### Non-Functional

- **Heavily read-skewed** — people browse far more than they post.
- **Low-latency feed** — under ~200 ms.
- **Media served from CDN** — non-negotiable at this scale.
- **Highly available.**

---

## 3. Capacity Estimation

```
Users        = 1B (500M DAU)
Uploads      = 100M photos/day  → 100M / 10⁵ ≈ ~1,200 uploads/sec
Media volume = 2 MB × 100M      = ~200 TB/day media        ← the dominant number
Feed reads   = ~10B/day         → 10B / 10⁵  ≈ ~100,000 reads/sec
```

**Takeaways:**
- **200 TB/day of media** is the headline — orders of magnitude more than Twitter's text. The media pipeline and CDN dominate the design; the feed metadata is comparatively tiny.
- Feed reads (~100K/sec) confirm the read-heavy profile → precomputed feeds + caching (as in Twitter).
- The feed payload itself is small (post IDs + media *URLs*); the bytes are media, served from **CDN**, never through your app servers.

---

## 4. The Distinctive Part: The Media Pipeline

This is where Instagram diverges from Twitter and where you should go deep. The upload-and-process flow:

```
1. Client requests a PRE-SIGNED S3 upload URL from the Media Service.
2. Client uploads the file DIRECTLY to S3 (bypassing our app servers).
3. The S3 upload fires an event → a Lambda/worker TRANSCODES it:
      photos → thumbnail, 480p, 1080p variants
      videos → multiple bitrates + HLS/DASH segments
4. Post Service stores the post: {post_id, user_id, caption, [media_urls], ts}.
5. Fan-out Service updates followers' feed caches.
```

Three senior points embedded here:

- **Pre-signed direct-to-S3 upload** — the client uploads straight to object storage using a short-lived signed URL, **bypassing your app servers entirely** (the pattern from the WhatsApp, Pastebin, and Dropbox guides). This protects the app tier and saves enormous bandwidth — at 200 TB/day you absolutely cannot route media through your servers.
- **Asynchronous transcoding** — processing (resize/transcode) is slow, so it's done **off the request path** (triggered by an S3 event), decoupled via the same queue/async principle as the scaling guide. The upload returns immediately; the post may briefly show a "processing" state.
- **Multiple resolutions** — generate variants so you serve the **right size for the context**: a tiny thumbnail for the grid, 1080p for full view, adaptive bitrate for video. This slashes bandwidth and speeds load — critical given the media volume.

---

## 5. Architecture

```
[Client] ─► [LB] ─► [API Gateway] ─┬─► [User Service]
                                    ├─► [Post Service]    ─► [Cassandra]  (posts)
                                    ├─► [Feed Service]    ─► [Redis]      (feed cache)
                                    ├─► [Media Service]   ─► [S3] ─► [CDN]
                                    ├─► [Ranking Service] ─► (ML feed scoring)
                                    ├─► [Search Service]  ─► [Elasticsearch]
                                    └─► [Notification Service]
```

Mirrors Twitter's structure plus a **Media Service** (pre-signed uploads + transcoding) and a **Ranking Service** (ML feed scoring) — the two Instagram-specific additions.

---

## 6. Feed Generation & ML Ranking

### Fan-out (same as Twitter)

Feed assembly uses the **hybrid push/pull** model: **push (fan-out on write)** for regular users (precompute followers' feeds in Redis), **pull on read** for celebrities (avoid the fan-out explosion), merged at read time. See the Twitter guide for the full derivation — it's identical here, so don't re-derive it; reference it and move on.

### ML ranking (the modern difference)

Unlike a chronological timeline, **Instagram's feed is ranked by a machine-learning model**, not by time. During feed generation, a **Ranking Service** scores candidate posts using features like:

- **Recency**, **user-user affinity** (how much you interact with the author), **post type**, and **engagement signals** (likes/comments velocity, watch time).

The flow becomes: gather candidate posts (from the hybrid fan-out) → **score them with the ranking model** → return the top-ranked. The trade-off: ranking drives much higher engagement but adds **compute and latency** on the read path — so it must fit the ~200 ms budget, typically by **pre-narrowing the candidate set** (only score a few hundred candidates, often with precomputed features from a feature store) rather than scoring everything. This shift from "merge by timestamp" to "score and sort by predicted engagement" is the defining modern feed change.

---

## 7. Data Model

### Posts — Cassandra

```
PARTITION KEY:  user_id
CLUSTERING KEY: post_id          (timestamp-based, Snowflake-style)
COLUMNS:        caption, media_urls[], likes_count, comments_count, ts
```

Partitioning by `user_id` makes a user's posts a single-partition scan, and a **Snowflake-style time-ordered `post_id`** gives sortable, globally-unique IDs — the same wide-column reasoning as Twitter. Note the post stores **media *URLs***, not media — the bytes are in S3/CDN.

---

## 8. Likes, Comments & Counters at Scale

- **Likes** — a Cassandra wide row maps `post_id → set of user_ids` (who liked), plus a **counter** for the total.
- **Comments** — a separate table **partitioned by `post_id`** (a post's comments live together).
- **Counter sharding (the senior point)** — a **viral post** receiving millions of likes/sec would hammer a single counter row — a **hot partition** (the hot-key problem from the caching guide). The fix is **sharded counters**: split the count into N sub-counters across partitions, increment a random one per like, and **sum them on read**. This spreads write load; the cost is a slightly more expensive read (sum N shards) and approximate real-time counts — an acceptable trade for viral content. (Alternatively, aggregate counts asynchronously / approximately.)

---

## 9. Stories (Ephemeral Content)

Stories introduce **ephemeral content with a 24-hour TTL**, handled separately from permanent posts:

- **Explicit expiration** — stories are stored with a TTL and auto-expire after 24h (TTL-based eviction + lifecycle cleanup, like the Pastebin expiration design). After expiry they're removed from active serving (optionally archived).
- **Pre-aggregated per user** — a user's active stories are grouped so the "stories bar" loads as one lookup per followed user.
- **Polling** — clients **poll every ~2 minutes** for new stories rather than relying on instant push, since stories are casual/ambient and don't need real-time delivery — cheaper than maintaining push for everything.

The ephemerality is the distinctive bit: a separate, TTL-governed store and a lighter (poll-based) delivery model than the main feed.

---

## 10. Search, DMs & CDN Strategy

- **Search** — Elasticsearch indexes users, hashtags, and captions (inverted index, as in the Twitter and autocomplete guides). **Autocomplete** uses the trie-based typeahead design from the autocomplete guide.
- **Direct messages** — essentially a chat system; reuse the **WhatsApp design** (WebSocket connections, message store, delivery/read receipts) rather than reinventing it.
- **CDN strategy** — *all* media flows through the **CDN** (CDN guide): low-latency global delivery, origin offload, and DDoS protection. Additional media-specific tactics:
  - **Pre-warm popular content** to edges during off-peak hours.
  - **Adaptive bitrate streaming (HLS/DASH)** for video/Reels — the video is segmented at multiple bitrates and the client picks based on its current bandwidth, so playback adapts smoothly to network conditions. This is essential for video at scale and is the streaming analog of the multi-resolution photo strategy.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. How is Instagram different from Twitter at the design level?**
The feed core is the same (hybrid push/pull fan-out, Cassandra posts, Redis feeds). The differences: it's media-dominated (a heavy upload/transcode/CDN pipeline that handles ~200 TB/day), the feed is ML-ranked rather than chronological, and it has ephemeral Stories/Reels. The hard scaling is media, not feed metadata.

**Q2. Walk through the media upload pipeline.**
Client gets a pre-signed S3 URL and uploads directly to S3 (bypassing app servers). The S3 event triggers async transcoding into multiple resolutions (thumbnail/480p/1080p; video → bitrates + HLS/DASH). Then the post record is created and fan-out updates feeds. Pre-signed upload + async transcoding keep heavy media work off the request path.

**Q3. Why upload directly to S3 instead of through your servers?**
At 200 TB/day, routing media through app servers would saturate them and blow bandwidth costs. A pre-signed URL lets the client upload straight to object storage; your servers only handle small metadata. The same media-off-the-hot-path rule as WhatsApp/Dropbox.

**Q4. Why multiple resolutions and adaptive bitrate?**
To serve the right size for the context (thumbnail in the grid, full-res on tap) and to adapt video to the viewer's bandwidth (HLS/DASH segments at multiple bitrates). This drastically cuts bandwidth and speeds loads — vital at this media volume.

**Q5. How does ML ranking change the feed vs a chronological one?**
Instead of merging candidates by timestamp, a ranking model scores them by recency, affinity, post type, and engagement signals, returning the top-ranked. It boosts engagement but adds read-path compute, so you pre-narrow candidates and use precomputed features to fit the ~200 ms budget.

**Q6. How do you handle like/comment counts on a viral post?**
A single counter row would be a hot partition under millions of likes/sec. Use sharded counters: split the count across N sub-counters, increment a random shard per like, and sum on read. Trades a slightly costlier read and approximate real-time counts for spread-out writes.

**Q7. How do Stories work and why are they handled differently?**
They're ephemeral (24h TTL), stored separately with explicit expiration (TTL + lifecycle cleanup), pre-aggregated per user, and delivered via ~2-minute client polling rather than push — because casual ephemeral content doesn't need real-time delivery, which is cheaper.

**Q8. How do you build the feed read to hit <200 ms with ranking?**
Serve the precomputed feed (push) from Redis, merge in celebrity pulls, then rank a pre-narrowed candidate set with precomputed features. Keep media out (only IDs + URLs in the feed; bytes from CDN). The expensive parts (fan-out, feature computation) happen off the read path.

**Q9. How do you implement search and DMs?**
Search via Elasticsearch (users/hashtags/captions) with trie-based autocomplete. DMs are a chat system — reuse the WhatsApp design (persistent connections, message store, receipts) rather than building anew.

**Q10. What's the consistency model?**
Read-heavy and eventually consistent (AP-leaning), like Twitter: feeds, counts, and story availability can lag slightly. Acceptable because users prefer fast, always-available browsing over instant consistency, and it enables async fan-out, transcoding, and caching.

---

## 12. Quick Glossary

- **Media pipeline** — upload → transcode → store → serve flow for photos/videos.
- **Pre-signed URL** — short-lived URL letting clients upload directly to S3.
- **Transcoding** — converting media into multiple resolutions/bitrates.
- **Adaptive bitrate (HLS/DASH)** — segmented video at multiple bitrates; client picks by bandwidth.
- **Hybrid fan-out** — push for regular users, pull for celebrities (see Twitter guide).
- **ML feed ranking** — scoring posts by predicted engagement instead of chronological order.
- **Affinity** — how strongly a user engages with a given author (a ranking feature).
- **Candidate set** — the narrowed list of posts the ranking model scores.
- **Sharded counter** — splitting a count across sub-counters to avoid a hot partition.
- **Hot partition** — an overloaded partition from concentrated writes (e.g., a viral post).
- **Stories** — ephemeral 24h-TTL content, stored separately and polled.
- **Reels** — short-form video with transcoding and its own discovery/ranking.
- **Snowflake ID** — time-ordered, globally-unique ID used as the post clustering key.

---

*Reference document. Contributions and corrections welcome.*
