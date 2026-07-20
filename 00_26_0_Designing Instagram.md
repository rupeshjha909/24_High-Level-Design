# Designing Instagram: A Senior Interview Guide

> A practical, interview-focused reference for a media-centric social platform — the social-feed skeleton of Twitter, dominated by a heavy **media pipeline**, **ML-ranked feeds**, and **ephemeral Stories**. This guide builds the architecture up piece by piece, traces the full life of a post (upload → transcode → fan-out → ranked feed read → adaptive playback), goes deep on the two genuinely hard mechanisms (how an Android-uploaded Reel plays perfectly on an iPhone and everywhere, and how a feed is assembled and ML-ranked under a 200 ms budget), nails down the data contracts, and covers counters at scale, Stories, CDN, sharding, and failure modes — with verified capacity, transcoding, ABR, and fan-out math and a senior follow-up bank.

> 💡 **The one idea (see §5):** the feed metadata is the *easy, already-solved* part (it's Twitter's hybrid fan-out); the real scale is the **media pipeline** — ~200 TB/day of uploads that must play on every OS, screen, and network. The uploader's device and format are made **irrelevant** by transcoding into a standard bitrate/codec ladder that any client adaptively streams from a CDN.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified)](#3-capacity-estimation-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a post](#44-the-end-to-end-life-of-a-post)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [Hard Problem 1: How an Android Reel Plays on iPhone & Everywhere](#5-hard-problem-1-how-an-android-reel-plays-on-iphone--everywhere)
6. [Hard Problem 2: How the Feed Is Assembled and ML-Ranked](#6-hard-problem-2-how-the-feed-is-assembled-and-ml-ranked)
7. [Counters at Scale (likes/comments on viral posts)](#7-counters-at-scale-likescomments-on-viral-posts)
8. [Stories (ephemeral content)](#8-stories-ephemeral-content)
9. [Data Contracts: Request Fields, Payloads & DB Schemas](#9-data-contracts-request-fields-payloads--db-schemas)
10. [Search, DMs & CDN Strategy](#10-search-dms--cdn-strategy)
11. [Scaling Summary](#11-scaling-summary)
12. [Failure Modes & Handling](#12-failure-modes--handling)
13. [Senior Follow-Up Questions (with Answers)](#13-senior-follow-up-questions-with-answers)
14. [Quick Glossary](#14-quick-glossary)

---

## 1. How to Approach This in an Interview

Instagram's **social-feed core is essentially Twitter's**: a read-heavy feed of posts from followed accounts, built with **hybrid push/pull fan-out**, Cassandra for posts, Redis for precomputed feeds. So the efficient move is to **state that parallel quickly** ("feed generation is the same hybrid fan-out as Twitter") and then **spend your time on what makes Instagram different:**
1. **It's media-dominated** — posts are photos/videos, so the **media pipeline** (upload → transcode → store → serve via CDN) is the bulk of the engineering (~200 TB/day).
2. **The feed is ML-ranked**, not chronological — a model scores posts by predicted engagement, under a tight latency budget.
3. **Ephemeral Stories** and **Reels** add content types with their own handling.

> 💡 **Senior signal:** *"The feed metadata is the easy, already-solved part — it's Twitter's hybrid fan-out. The real scale is the media: ~200 TB/day of uploads that must play on any device and network. I'll go deep on the media pipeline (transcoding + adaptive streaming) and on ML ranking, and reference Twitter for the fan-out."*

---

## 2. Requirements

### Functional
- **Upload** photos/videos; **feed** of posts from followed users; **like, comment, follow.**
- **Stories** (24h expiry), **Reels**; **direct messages**; **search** (users, hashtags, places).

### Non-Functional
- **Heavily read-skewed** — people browse far more than they post (~100:1).
- **Low-latency feed** — under ~200 ms, *including* ranking.
- **Media served from CDN** — non-negotiable at this scale.
- **Highly available** (AP-leaning; slightly stale feeds/counts are fine).
- **Device- and network-agnostic playback** — the same content plays on any OS, screen, and connection (§5).

---

## 3. Capacity Estimation (verified)

```
Users        = 1B (500M DAU)
Uploads      = 100M/day → ~1,157/s
Media volume = 2 MB × 100M = ~200 TB/day               ← the dominant number
Feed reads   = 10B/day → ~115,700/s   (read:write ≈ 100:1, read-heavy)
Transcoding  = one 30s Reel → 5 renditions × 8 segments = 40 files; ~39 MB stored (vs 1 master)
```

**Takeaways that shape the design:**
- **~200 TB/day of media** is the headline — orders of magnitude more than Twitter's text. **The media pipeline + CDN dominate; feed metadata is comparatively tiny.**
- **Reads ~115K/s (100:1)** → precomputed feeds + caching (as in Twitter).
- **The feed payload is small** (post IDs + media *URLs/manifests*); the bytes are media, served from **CDN**, never through app servers.
- **Transcoding multiplies stored bytes** — each upload becomes several renditions (~39 MB for a 30s Reel vs one master); you trade storage for universal, bandwidth-efficient playback.

> 💡 The numbers dictate: **media direct-to-S3 + async transcode + CDN** (never through app servers); **precomputed ranked feeds** for the read-heavy path; feed carries **URLs/manifests, not bytes.**

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

Instagram is two systems bolted together. The **first** is a Twitter-style **social feed**: a read-heavy timeline of posts from accounts you follow, assembled with **hybrid fan-out** (precompute regular users' feeds into Redis on write; pull celebrities' posts at read time), then **ranked by an ML model** (predicted engagement, not time) — all serving small metadata (post IDs + media URLs), so it's fast and cache-friendly. The **second**, and the part that actually dominates the engineering, is the **media pipeline**: because a post is a 2 MB photo or a multi-megabyte Reel and there are ~200 TB of uploads a day, media never touches the app servers — the client uploads **directly to object storage via a pre-signed URL**, an event triggers **asynchronous transcoding** into a standardized **ladder** of resolutions, bitrates, and codecs (cut into small segments with a **manifest**), and everything is served from a **CDN** so any device can **adaptively stream** the version that fits its screen and live bandwidth. Around these sit counters (sharded to survive viral posts), Stories (ephemeral, TTL'd, polled), search (Elasticsearch), and DMs (a WhatsApp-style chat system). The feed is the easy half; the media pipeline is where the scale and the interesting mechanisms live.

### 4.2 The diagram

```
[Client] ─► [LB] ─► [API Gateway] ─┬─► [User Service]
                                    ├─► [Post Service]     ─► [Cassandra]  (posts: id, caption, media URLs)
                                    ├─► [Feed Service]     ─► [Redis]      (precomputed feed cache)
                                    ├─► [Ranking Service]  ─► [feature store] (ML feed scoring)
                                    ├─► [Media Service] ── pre-signed URL ──► [S3] ─► [CDN]
                                    │        └─(S3 event)─► [Transcoding workers] ─► renditions+segments+manifest
                                    ├─► [Fan-out Service]  ─► (update followers' Redis feeds)
                                    ├─► [Search Service]   ─► [Elasticsearch]
                                    └─► [Notification / DM (WhatsApp-style)]
```

The key visual idea: **metadata** (post IDs, URLs) flows through Post/Feed/Ranking and is small and fast; **media bytes** flow **client → S3 → transcode → CDN**, never through app servers. The feed carries a video's **manifest URL**, not the video.

### 4.3 Each component, in detail

**① Client.** Uploads media **directly to S3** via a pre-signed URL (not through app servers); renders the feed; for video, fetches a **manifest** and does **adaptive bitrate** playback (picks the rendition matching its screen + live bandwidth). It also polls for Stories and holds a WebSocket only for DMs.

**② Media Service.** Issues **pre-signed upload URLs** and records media metadata. It never carries the bytes — at 200 TB/day that would melt the app tier.

**③ Transcoding workers (the heavy, async fleet).** Triggered by the S3 upload event. Re-encode the **master** into a **bitrate ladder** (240p→1080p) across **codecs** (AV1/HEVC/H.264), cut each into **segments**, and write a **manifest** (HLS + DASH). Compute-heavy (often GPU) — which is why a new post briefly shows "processing" (§5).

**④ Post Service.** Stores the post record `{post_id, user_id, caption, media_urls/manifest, ts}` in **Cassandra** (partition by user_id, Snowflake-time-ordered post_id). Stores **URLs, not bytes.**

**⑤ Fan-out Service.** On a new post, pushes the post_id into followers' **Redis feed caches** — for regular users. Celebrities are **not** fanned out (pull at read time). (§6)

**⑥ Feed Service.** On a feed read, gathers candidate post IDs (from Redis push feed + celebrity pulls).

**⑦ Ranking Service.** Scores the candidate set with an **ML model** (recency, affinity, engagement signals) using **precomputed features** from a feature store, returning the top-ranked — within the ~200 ms budget. (§6)

**⑧ CDN.** Serves all media (renditions, segments, manifests, thumbnails) from edges near the viewer; the origin (S3) is protected. Popular content is **pre-warmed** to edges.

**⑨ Counters / Search / DM.** Sharded counters for likes/comments on viral posts (§7); Elasticsearch for search; a WhatsApp-style chat system for DMs.

### 4.4 The end-to-end life of a post

Here is exactly what happens from **recording a Reel to a follower watching it in their ranked feed**, step by step:

```
UPLOAD (media off the hot path):
1.  Creator records a Reel → client asks Media Service for a PRE-SIGNED S3 URL.
2.  Client uploads the raw file (the MASTER) DIRECTLY to S3 — bypassing app servers.
        (This is why 200 TB/day doesn't melt the app tier.)

TRANSCODE (async):
3.  The S3 upload fires an event → Transcoding workers re-encode the master into a
        bitrate/codec LADDER (240p–1080p; AV1/HEVC/H264), cut each into ~4s SEGMENTS,
        and write a MANIFEST (HLS .m3u8 + DASH .mpd). All pushed to the CDN.   ← §5
4.  Post Service stores the post: {post_id, user_id, caption, manifest_url, ts} in Cassandra.
        (The post shows "processing" until transcode completes.)

FAN-OUT (write-time, for regular users):
5.  Fan-out Service pushes post_id into the creator's followers' Redis feed caches.
        (If the creator is a celebrity, SKIP fan-out — followers pull at read time. §6)

FEED READ (the ~200 ms path):
6.  A follower opens their feed → Feed Service gathers candidates:
        their precomputed Redis feed  +  fresh pulls from any celebrities they follow.
7.  Ranking Service scores the (pre-narrowed) candidate set with the ML model
        (recency, affinity, engagement) using precomputed features → returns the top-ranked IDs.
8.  Client gets the ranked list of post IDs + a MANIFEST URL per video (small metadata, no bytes).

PLAYBACK (adaptive, from the CDN):
9.  For the Reel, the follower's device fetches the MANIFEST from the nearby CDN edge,
        reads the menu of renditions, measures its screen + live bandwidth, and
        ADAPTIVELY picks the best rendition — streaming segments from the edge,
        switching quality at segment boundaries as the network changes.   ← §5
```

The single most important thing to notice: **media bytes and feed metadata travel completely separate paths.** The bytes go client→S3→transcode→CDN and are *pulled by the viewer's device*; the feed read only ever moves small IDs + URLs. Mixing them (routing media through app servers, or putting bytes in the feed) is the mistake that doesn't survive 200 TB/day.

### 4.5 Why this split? (the design rationale)

- **Media direct-to-S3, never through app servers** — at 200 TB/day, routing media through your tier would saturate it and blow bandwidth costs. Pre-signed upload keeps app servers handling only small metadata.
- **Transcoding asynchronous, off the request path** — re-encoding is slow/compute-heavy; triggering it by an S3 event lets the upload return immediately (post briefly "processing"). Same queue/async principle as any heavy background job.
- **Feed carries URLs/manifests, not bytes** — keeps the read path tiny and cache-friendly; the bytes come from the CDN, near the viewer.
- **Hybrid fan-out (push regular, pull celebrity)** — push precomputes cheap feeds for normal users; pull avoids the fan-out explosion for a celebrity's 100M followers (§6).
- **Ranking on a pre-narrowed candidate set with precomputed features** — full ML scoring of everything can't fit 200 ms; narrow first, score a few hundred with cached features.
- **Sharded counters** — a viral post's likes would hot-spot one row; spread across shards (§7).

### 4.6 Where the load actually goes

- **Media bytes:** ~**200 TB/day** — the whole cost center, handled by **S3 + CDN** (never app servers). Transcoding **multiplies** it (~39 MB per 30s Reel across renditions) — storage traded for universal playback.
- **Transcoding compute:** a large **async (often GPU) worker fleet** — the heaviest compute in the system; why posts show "processing."
- **Feed reads:** ~**115K/s** — served from **precomputed Redis feeds + a bounded ranking step**; small metadata, cache-friendly.
- **Fan-out writes:** cheap for regular users (~200 feed writes/post); **celebrities are the deceptive spike** (100M writes/post → use pull instead).
- **Counters:** viral posts create **hot partitions** on a single like-counter row → sharded counters (§7).
- **CDN egress:** the dominant serving cost; pre-warm popular content to edges.

> 💡 **The senior framing:** *"Feed metadata is small and Twitter-solved. The load is media — 200 TB/day served from S3+CDN with a heavy async transcode fleet, never through app servers. On the feed side, the traps are the celebrity fan-out explosion (use pull) and viral-post counters (shard them). Everything the viewer sees is IDs + URLs; the bytes come from a nearby edge."*

---

## 5. Hard Problem 1: How an Android Reel Plays on iPhone & Everywhere

This is the mechanism interviewers love and most candidates wave away. *You record a Reel on Android and upload it — how does it play perfectly on an iPhone, a cheap Android, a tablet, and a slow 3G phone?*

### Step 0 — Why the raw Android file can't just be sent to the iPhone
Your phone produces **one file** with a fixed resolution (1080×1920), bitrate (~12 Mbps, big), container (.mp4), and **codec** (H.264/HEVC/VP9). But: a **budget Android on 3G** can't download 12 Mbps (endless buffering); a **small screen** doesn't need 1080p (wasted bytes); an **older iPhone** might not decode that codec; a **tablet on WiFi** could take full quality only while the network holds. **There is no single file right for all of them.** So the uploader's device/format must be made **irrelevant** — that's transcoding's job.

### Step 1 — Upload the "master"
Via the pre-signed URL, Android uploads the original straight to S3 — the **master**. Nothing plays from it directly; it's raw material.

### Step 2 — Transcode: one master → a standardized ladder
An S3 event triggers transcoding into a **bitrate ladder** (verified):
```
Rendition  Resolution  Video bitrate
 240p         240p        400 kbps      (small screens / slow networks)
 360p         360p        800 kbps
 480p         480p       1400 kbps
 720p         720p       2800 kbps
 1080p       1080p       5000 kbps      (big screens / fast networks)
```
The wild variety of what phones record is **normalized into the same fixed menu** every device knows. Poster/thumbnail frames are generated too. **After this, the Android origin is irrelevant.**

### Step 3 — Segment: cut each rendition into ~4s slices
A 30s Reel at 4s segments = **8 segments/rendition** → **40 files** across 5 renditions (verified). Small segments are what allow **switching quality mid-playback** (you can only switch at a boundary).

### Step 4 — The manifest: the menu
Transcoding writes a **manifest** listing renditions, bitrates, and segment locations: **HLS** (`.m3u8`, Apple-native) and **DASH** (`.mpd`, open/Android/web). The player reads the menu, then decides what to fetch.

### Step 5 — Store + CDN
Manifest + renditions + segments go to S3 and out to the **CDN**, so every viewer's segments come from a nearby edge.

### Step 6 — Playback: the client chooses (Adaptive Bitrate)
```
1. iPhone fetches the MANIFEST (HLS) from the CDN.
2. Reads the menu: 240p/360p/480p/720p/1080p.
3. Measures screen size + current network speed.
4. PICKS the best rendition that fits (≤ screen, and bitrate ≤ network × safety margin).
5. Streams those segments from the nearby edge; keeps measuring.
6. On a network change, switches rendition at the next segment boundary — smoothly.
```
Verified selections for the *same* Reel (pick highest rendition ≤ network×0.8 and ≤ screen):
```
iPhone fast WiFi (8 Mbps, 1080p) → 1080p
iPhone 4G        (2 Mbps, 1080p) → 480p
budget Android   (0.6 Mbps,720p) → 240p
tablet WiFi      (3.5 Mbps,1080p)→ 720p
```

### Step 7 — Adaptive switching (why it never freezes)
Because the player re-checks the network every segment, quality **follows the connection**. Verified timeline: `t=0 8Mbps→1080p; t=8 1.5Mbps→360p; t=12 0.7Mbps→240p; t=24 8Mbps→1080p`. The viewer sees a brief quality dip, not a spinning buffer.

### Step 8 — Codecs: best each device supports, H.264 fallback
Newer codecs (AV1, HEVC) are smaller but not universal; H.264 plays everywhere. Verified: new iPhone → **AV1**, older iPhone → **HEVC**, budget Android → **H.264**.

> 💡 **The senior one-liner:** *"The uploader's platform is made irrelevant by transcoding. Android uploads one master; the pipeline re-encodes it into a standard ladder of resolutions, bitrates, and codecs, chops each into small segments, and writes an HLS/DASH manifest. Any device fetches the manifest, adaptively picks the rendition fitting its screen and live bandwidth, streams segments from a nearby CDN edge, and switches quality on the fly — best codec it supports, H.264 as fallback. One upload plays optimally everywhere, with the client doing the choosing."* Costs: storage multiplies (~39 MB/Reel), transcoding is compute-heavy (async GPU fleet), and popular Reels are pre-warmed to edges.

---

## 6. Hard Problem 2: How the Feed Is Assembled and ML-Ranked

The second hard mechanism: *when you open the app, how is your feed built from the people you follow — fast — and why is it ranked, not chronological?*

### The two ways to build a feed, and why neither alone works
- **Fan-out on write (push):** when someone posts, immediately push the post_id into every follower's precomputed feed (Redis). Reads are then trivial (read your ready-made list). **Great for regular users** (~200 followers → 200 cheap writes). **Terrible for celebrities:** one post → **100M feed writes** (verified explosion), a write storm every time a celebrity posts.
- **Fan-out on read (pull):** store the post once; assemble feeds by pulling followees' recent posts at read time. **Cheap on write, expensive on read** (gather from everyone you follow each time).

### The hybrid (the real answer)
**Push for regular users, pull for celebrities.** Most posts fan out on write (cheap, ready feeds); a handful of celebrities are **not** fanned out — their recent posts are **pulled at read time** and merged into the reader's feed. This avoids both the celebrity write-storm and the general read-time cost. (Same derivation as Twitter — reference it, don't re-derive.)

### Then: rank, don't sort by time
Modern feeds are **ML-ranked**, not chronological. Feed assembly becomes:
```
1. Gather candidates: reader's precomputed push feed  +  celebrity pulls.
2. Score each candidate with an ML model — features: recency, user↔author AFFINITY,
   post type, engagement signals (like/comment velocity, watch time).
3. Return the top-ranked posts.
```
The catch is **latency**: you can't ML-score thousands of candidates in 200 ms. So you **pre-narrow the candidate set** (score only a few hundred) and use **precomputed features** from a **feature store** (affinity, author stats computed offline), keeping the online step a fast scoring pass.

> 💡 **The senior one-liner:** *"Feed assembly is hybrid fan-out — push into Redis feeds for regular users, pull for celebrities to avoid the 100M-write explosion — then an ML ranking pass instead of a timestamp sort. To fit 200 ms I pre-narrow to a few hundred candidates and score them with precomputed features from a feature store; the expensive fan-out and feature computation happen off the read path."*

---

## 7. Counters at Scale (likes/comments on viral posts)

A **viral post** taking millions of likes/sec would hammer a single counter row — a **hot partition**. Fix: **sharded counters.**
```
- Split the count into N sub-counters (e.g., 100) across partitions.
- Each like increments a RANDOM shard.
- Read = SUM the N shards.
```
(Verified: 1,000,000 likes over 100 shards → each shard ~10,000 writes instead of 1,000,000 on one row; sum reconstructs the exact total.) The cost is a slightly more expensive read (sum N shards) and approximate real-time counts — an acceptable trade for viral content. Alternatively, aggregate counts asynchronously/approximately.

> 💡 A single counter row is a write hotspot on viral content; **sharded counters spread the writes**, trading a cheap N-shard sum on read for surviving millions of increments/sec.

---

## 8. Stories (ephemeral content)

Stories are **ephemeral, 24h-TTL** content, handled separately from permanent posts:
- **Explicit expiration** — stored with a TTL; auto-expire after 24h (TTL eviction + lifecycle cleanup, like Pastebin), then removed from active serving (optionally archived).
- **Pre-aggregated per user** — a user's active stories are grouped so the "stories bar" is one lookup per followed user.
- **Polling** — clients **poll every ~2 min** for new stories rather than push; casual/ambient content doesn't need real-time delivery, and polling is cheaper.
- Video stories go through the same **transcode/ABR pipeline** as Reels (§5).

> 💡 Ephemerality is the distinctive bit: a **TTL-governed store** + a **lighter poll-based delivery** than the main feed.

---

## 9. Data Contracts: Request Fields, Payloads & DB Schemas

### Part A — Key client↔server requests
**Get an upload URL** (client → Media Service): `{ user_id, content_type:"video/mp4", size }` → `{ upload_url (pre-signed), media_key }`. Client PUTs the master to `upload_url`.
**Create post** (client → Post Service): `{ client_post_id (idempotency), user_id, caption, media_key, type:"reel" }` → `{ post_id, status:"processing" }`.
**Get feed** (client → Feed Service): `GET /feed?cursor=…&limit=20` →
```json
{ "items":[ { "post_id":"p_9","author":"u_3","caption":"…",
              "media":{ "type":"video","manifest_url":"https://cdn/…/master.m3u8",
                        "poster_url":"https://cdn/…/poster.jpg" },
              "likes":10450,"comments":231,"ranked_score":0.92 } ],
  "next_cursor":"…" }
```
> The feed carries a **manifest URL**, not the video; the client streams segments from the CDN (§5).

### Part B — Inter-service payloads
- **S3 event → Transcoding:** `{ media_key, content_type, size }` → produces renditions/segments/manifest, then callback `{ media_key, manifest_url, renditions[], status:"ready" }`.
- **Post Service → Fan-out:** `{ post_id, author_id, followers_shard? }` (skip if author is a celebrity).
- **Feed Service → Ranking:** `{ user_id, candidate_post_ids[] }` → `{ ranked_post_ids[] }` (uses feature store).
- **Like → counter:** `INCR like_shard:{post_id}:{randomN}`.

### Part C — DB schema per store
**① Posts — Cassandra** (partition by author, time-ordered id):
```
posts( user_id PARTITION, post_id CLUSTERING (Snowflake, time-sortable),
       caption, media_manifest_url, media_type, likes_count, comments_count, ts )
-- stores URLs/manifest, NOT bytes
```
**② Feed cache — Redis** (precomputed push feed): `feed:{user_id} → [post_id, …]` (capped, most-recent).
**③ Media metadata — SQL/Cassandra:** `media( media_key PK, uploader_id, type, manifest_url, renditions(json), status )`.
**④ Likes / counters — Cassandra** (sharded): `like_counts( post_id, shard_no, count, PK(post_id,shard_no) )`; `likes( post_id PARTITION, user_id CLUSTERING )` (who liked).
**⑤ Comments — Cassandra** (partition by post): `comments( post_id PARTITION, comment_id CLUSTERING, user_id, text, ts )`.
**⑥ Stories — TTL store:** `stories( user_id, story_id, media_url, expires_at )` (24h TTL).
**⑦ Feature store — for ranking:** precomputed `affinity(user,author)`, author/post engagement stats.
**⑧ Search — Elasticsearch:** users, hashtags, captions. **DMs — WhatsApp-style** chat store.

> 💡 **The contract in one line:** *"Media uploads go direct-to-S3 via a pre-signed URL and are transcoded async into a manifest+renditions+segments on the CDN; the post record (Cassandra, partitioned by author, Snowflake id) stores the **manifest URL, not bytes**. The feed read returns ranked post IDs + manifest URLs (small metadata); the client streams from the CDN. Likes use **sharded counters**; Stories carry a TTL. Every field exists to keep bytes off the app path and metadata tiny."*

---

## 10. Search, DMs & CDN Strategy
- **Search** — Elasticsearch inverted index over users/hashtags/captions; **autocomplete** via the trie-based typeahead design.
- **Direct messages** — a chat system; reuse the **WhatsApp design** (WebSocket + message store + receipts) rather than reinventing it.
- **CDN** — *all* media flows through the CDN: global low-latency delivery, origin offload, DDoS protection. Media-specific tactics: **pre-warm popular content** to edges off-peak; **adaptive bitrate (HLS/DASH)** so the client picks the right rendition (§5) — the streaming analog of multi-resolution photos.

---

## 11. Scaling Summary
- **Media never touches app servers** — pre-signed direct-to-S3 upload; async transcode; **CDN** delivery. The 200 TB/day cost center lives in S3+CDN.
- **Feed = hybrid fan-out** (push regular, pull celebrity) + **ML ranking** on a pre-narrowed candidate set with precomputed features (fits ~200 ms).
- **Feed carries URLs/manifests, not bytes**; bytes stream from the CDN edge.
- **Sharded counters** for viral like/comment counts (avoid hot partitions).
- **Stories** — TTL store + ~2-min polling (lighter than push).
- **Cassandra** for posts (partition by author, Snowflake id); **Redis** for feeds; **Elasticsearch** for search; **WhatsApp-style** DMs.
- **AP/eventually-consistent** feeds/counts; pre-warm popular media to edges.

---

## 12. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Transcoding backlog/failure | post stuck "processing" | async retry; post visible with poster once ready; monitor queue depth |
| Media through app servers | app tier saturates | never — pre-signed direct-to-S3 (the rule) |
| Celebrity posts (fan-out explosion) | 100M feed writes | **pull** for celebrities; don't fan out on write |
| Viral post like storm | hot counter partition | **sharded counters** (verified); sum on read |
| Ranking too slow | feed misses 200 ms | pre-narrow candidates + precomputed features; degrade to recency if the model times out |
| CDN edge miss / cold content | first viewers wait | **pre-warm** popular content to edges; origin shielding |
| Device can't decode codec | playback fails | multi-codec ladder + **H.264 universal fallback** |
| Network drops mid-play | buffering | **ABR** downshifts at the next segment boundary |
| Redis feed cache loss | feed unavailable | rebuild from posts (pull); degrade gracefully |

---

## 13. Senior Follow-Up Questions (with Answers)

**Q1. How is Instagram different from Twitter at the design level?** Feed core is the same (hybrid fan-out, Cassandra posts, Redis feeds). Differences: media-dominated (~200 TB/day upload/transcode/CDN pipeline), **ML-ranked** feed (not chronological), and ephemeral Stories/Reels. The hard scaling is media, not feed metadata.

**Q2. I upload a Reel from Android — how does it play on an iPhone and everywhere?** The raw file is a *master*; nothing plays from it. Transcode → standard **bitrate ladder** + **codecs** (AV1/HEVC/H.264) → **segments** → **manifest** (HLS + DASH) → **CDN**. The iPhone fetches the manifest, **adaptively picks** the rendition fitting its screen + live bandwidth, streams from a nearby edge, switches quality at segment boundaries, best codec it supports (H.264 fallback). Transcoding + ABR make the uploader's platform irrelevant. (§5)

**Q3. Why not just serve the uploaded file?** No single file suits all viewers (too big for slow nets, wrong size for small screens, undecodable codec). It'd buffer, waste bandwidth, or fail. Transcode to a ladder + adaptive streaming = universal, efficient playback.

**Q4. Why upload directly to S3?** At 200 TB/day, routing media through app servers saturates them. Pre-signed URL → client uploads straight to object storage; servers handle only small metadata. Media-off-the-hot-path (WhatsApp/Dropbox rule).

**Q5. How does the feed get built and why ranked?** Hybrid fan-out (push regular, pull celebrity to avoid the 100M-write explosion) gathers candidates; an ML model scores them (recency, affinity, engagement) instead of a timestamp sort — boosting engagement. To fit 200 ms, pre-narrow candidates and use precomputed features. (§6)

**Q6. How do you handle a viral post's like count?** **Sharded counters** — split the count across N sub-counters, increment a random one per like, sum on read. Spreads the write storm; trades a cheap N-shard read + approximate real-time count. (§7)

**Q7. How do Stories differ?** Ephemeral (24h TTL), stored separately with explicit expiration, pre-aggregated per user, delivered by ~2-min polling (not push) — casual content doesn't need real-time.

**Q8. What's the consistency model?** Read-heavy, AP/eventually-consistent (like Twitter): feeds, counts, story availability can lag slightly. Enables async fan-out, transcoding, and caching — users prefer fast, always-available browsing.

**Q9. How do you hit <200 ms with ranking?** Serve the precomputed Redis feed + celebrity pulls, then rank a **pre-narrowed** candidate set with **precomputed features**; keep media out (IDs + URLs/manifests only). Expensive work (fan-out, features, transcode) is off the read path.

**Q10. Where does the video URL point?** For video, the post's stored URL is the **manifest** (the menu of renditions), not a single file — the client streams segments from it via ABR.

**Q11. What are the costs of the media pipeline?** Storage multiplies (~39 MB per 30s Reel across renditions); transcoding is compute-heavy (async GPU fleet, "processing" state); popular content must be pre-warmed to CDN edges.

---

## 14. Quick Glossary
- **Media pipeline** — upload → transcode → segment → store → CDN-serve flow.
- **Master** — the original uploaded file; source for transcoding (never served directly).
- **Pre-signed URL** — short-lived URL for direct-to-S3 upload (bypasses app servers).
- **Transcoding** — re-encoding into multiple resolutions, bitrates, and codecs.
- **Rendition / bitrate ladder** — a version / the set of versions low→high quality.
- **Codec** — compression method (AV1/HEVC/H.264); newer = smaller, less universal; H.264 = fallback.
- **Segment** — a short (2–6s) slice of a rendition; unit of streaming + quality-switching.
- **Manifest (HLS `.m3u8` / DASH `.mpd`)** — the menu listing renditions + segments; the feed stores its URL.
- **Adaptive bitrate (ABR)** — client picks/switches rendition by screen size + live bandwidth.
- **CDN** — edge servers delivering segments from near the viewer; pre-warm popular content.
- **Hybrid fan-out** — push for regular users, pull for celebrities (avoids the write explosion).
- **ML feed ranking** — scoring posts by predicted engagement instead of time.
- **Affinity** — how strongly a user engages with an author (a ranking feature).
- **Feature store** — precomputed features for fast online ranking.
- **Candidate set** — the pre-narrowed list of posts the ranking model scores.
- **Sharded counter / hot partition** — splitting a count across sub-counters to avoid an overloaded row on viral content.
- **Stories** — ephemeral 24h-TTL content, stored separately, polled.
- **Snowflake ID** — time-ordered globally-unique post id (clustering key).

---

*Reference document. Contributions and corrections welcome.*
