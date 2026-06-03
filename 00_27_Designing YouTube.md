# Designing YouTube: A Senior Interview Guide

> A practical, interview-focused reference for designing a video platform at YouTube scale — fundamentally a content-delivery and transcoding problem where ~95% of cost is storage and bandwidth. Covers the upload/transcoding pipeline, adaptive bitrate streaming, tiered storage, CDN optimization, view counts, recommendations, and live streaming. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Upload & Transcoding Pipeline](#4-the-upload--transcoding-pipeline)
5. [Adaptive Bitrate Streaming](#5-adaptive-bitrate-streaming)
6. [Storage Strategy & the Long Tail](#6-storage-strategy--the-long-tail)
7. [CDN: The Core Optimization](#7-cdn-the-core-optimization)
8. [Architecture](#8-architecture)
9. [Metadata, View Counts, Recommendations & Search](#9-metadata-view-counts-recommendations--search)
10. [Live Streaming](#10-live-streaming)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This in an Interview

The single most important framing — and the thing to say first — is that **YouTube is fundamentally a content-delivery and transcoding problem, not a metadata problem.** Roughly **95% of the cost is video storage and bandwidth**, so the architecture is judged on two things: **transcoding throughput** (the dominant compute cost) and **CDN cache hit rate** (the dominant bandwidth/cost lever). The social/metadata layer (titles, comments, likes, even recommendations) is comparatively small and reuses patterns from the Instagram and Twitter designs.

So the strong approach: establish the staggering scale, then go deep on (1) the **upload→transcoding pipeline**, (2) **adaptive bitrate streaming**, (3) **tiered storage for the long tail**, and (4) **CDN optimization**. Treat the metadata/social parts briefly by reference. Senior signal: keeping the conversation anchored on delivery and transcoding economics rather than over-designing the comments table.

---

## 2. Requirements

### Functional

- **Upload** videos (any length, any format).
- **Stream** with adaptive bitrate.
- **Search** videos.
- **View counts, likes, comments, subscriptions.**
- **Personalized recommendations.**
- **Live streaming** (optional).

### Non-Functional

- **Ingest petabytes of video/day.**
- **Serve hundreds of petabytes/day.**
- **Low buffering** — even on slow networks.
- **Highly available across regions.**

---

## 3. Capacity Estimation

```
Uploads   = 500 hours/min  → 30,000 hours/hour → ~720,000 hours/day uploaded
Views     = ~5B videos/day → 5B / 10⁵ ≈ ~58,000 video starts/sec
Storage   = exabytes (each video → many resolutions × formats × segments)
Bandwidth = hundreds of PB/day served   ← the dominant cost
```

**Takeaways:**
- **Transcoding** ~720,000 hours of new video *per day*, each into ~6 resolutions and multiple formats, is an immense, embarrassingly-parallel compute job — the dominant *processing* cost.
- **Bandwidth** (hundreds of PB/day served) is the dominant *operating* cost, which is why **CDN cache hit rate** is the number you optimize above all else.
- Storage is exabytes because each source video explodes into many derived files (resolutions × formats × segments) — driving tiered storage.

---

## 4. The Upload & Transcoding Pipeline

```
[Client] ─► [Upload Service] ─► [Raw video in S3]
                                      │ (S3 event → queue)
                                      ▼
                          [Transcoding cluster — parallel workers]
                                      │  → 240p, 360p, 480p, 720p, 1080p, 4K
                                      │  → segmented (~5–10s each) for HLS/DASH
                                      ▼
                              [S3 (variants)] ─► [CDN edges]
```

**Transcoding is the most expensive operation**, so the pipeline is built around doing it efficiently and asynchronously:

- **Async, queue-driven** — the raw upload lands in S3 and a job is queued; the upload returns immediately and the video shows as "processing." Transcoding never blocks the uploader (the async principle from the scaling and Instagram guides).
- **Massively parallel** — a long video is **split into chunks that are transcoded independently across many workers** (MapReduce-style), then reassembled. This makes transcoding a 2-hour video fast and scales horizontally with the cluster.
- **The combinatorial explosion** — each video produces `{formats} × {resolutions} × {segments}` files. A single video can become thousands of small segment files spanning many GB — which is *why* total storage reaches exabytes.

---

## 5. Adaptive Bitrate Streaming

For video, **adaptive bitrate (ABR) streaming is core, not optional** — it's how you deliver "low buffering even on slow networks." Using **HLS or DASH**:

- The video is **split into short segments** (~5–10s), each encoded at **multiple bitrates** (240p…4K).
- A **manifest file** lists the available bitrates/segments.
- The **client measures its bandwidth and picks the bitrate per segment**, switching dynamically — dropping to a lower bitrate when the network slows (to avoid buffering) and climbing back up when it improves.

So a viewer on a fast connection gets 1080p/4K; the same viewer entering a tunnel seamlessly drops to 360p without stalling. The buffering-avoidance logic lives in the client (choose a sustainable bitrate, buffer a few segments ahead). ABR is the streaming analog of Instagram's multi-resolution photos, but per-segment and dynamic — and it's the reason transcoding produces so many variants.

---

## 6. Storage Strategy & the Long Tail

Video popularity follows an **extreme power law**: a tiny fraction of videos get the overwhelming majority of views, while most videos are watched rarely (the long tail). The storage strategy exploits this:

- **Hot videos** — kept on **CDN edges and nearby S3**, fully transcoded and replicated widely for instant delivery.
- **Cold videos** (the long tail) — moved to **cheap cold storage** (S3 Glacier / GCS Coldline), pulled on demand when someone actually watches.
- **Popular videos** — pre-transcoded into all variants and **replicated to many edges**.

A senior nuance: because most uploads get few views, you can **prioritize transcoding** — generate the common resolutions immediately, but defer rare ones (e.g., 4K) or transcode lazily for unpopular videos — trading some first-view latency for huge savings in transcoding compute and storage. Matching storage tier and transcoding effort to a video's actual popularity is the key cost discipline (tiered-storage principle from the storage-types guide).

---

## 7. CDN: The Core Optimization

Since bandwidth dominates cost, **maximizing CDN cache hit rate is the central optimization** (CDN guide). Video segments are static and immutable — ideal for caching — and the power-law popularity means a relatively small edge cache can serve the vast majority of view traffic.

- **All video is served from the CDN**, never from origin directly on the hot path.
- **Pre-warm popular content** to many edges (often during off-peak), so a trending video is already cached close to viewers before the surge.
- **The long tail** stays in cheap origin/cold storage; a rare view incurs an origin pull (a cache miss) — acceptable because it's rare.
- Every percentage point of higher cache hit rate directly cuts expensive origin egress, so CDN tuning is where the money is.

This is why the "key insight" is worth repeating: optimize the architecture for CDN hit rate and transcoding throughput, because that's where ~95% of the cost lives.

---

## 8. Architecture

```
[Uploader] ─► [Upload Service] ─► [S3 raw] ─► [Transcoding Cluster] ─► [S3 variants]
                                                                          │
                                                                          ▼
[Viewer] ─► [CDN] ◄── pre-warmed by ── [Replication Service]   ◄── tiered storage
              │                                                     (hot S3 / cold Glacier)
              └─► [Video/Metadata Service] ─► [Recommendations, view counts, search]
```

The two halves: a **write/processing path** (upload → transcode → store → replicate to edges) and a **read/delivery path** (viewer → CDN → segments), with a comparatively lightweight **metadata/social service** alongside.

---

## 9. Metadata, View Counts, Recommendations & Search

### Metadata Service

Stores `video_id, title, description, uploader, tags, view_count, transcoded_urls` in a **relational DB (MySQL/PostgreSQL)** — it's structured, queryable, and modest in size compared to the video bytes.

### View counts (a hot-counter problem)

Incrementing a DB row on every play would create a **hot row** on viral videos. The fix (same pattern as Instagram's counters and the caching guide's hot key): **count in Redis with periodic flush** to the DB, and stream play events to **Kafka** for accurate analytics. You also dedup/throttle to avoid counting bots or double-counting a session. Counts are eventually consistent and approximate in real time — fine for a view counter.

### Recommendations

A **pull-based discovery feed** (unlike the follow-based feeds of Twitter/Instagram): **offline ML** (collaborative filtering + deep models) trained on watch history produces candidate scores, **blended online** with real-time signals, and **cached per user**, refreshed periodically. Recommendations drive most watch time, so this is a major (but separable) subsystem.

### Search

**Elasticsearch** indexes titles, descriptions, tags, and even **transcripts** (auto-generated captions make video content searchable), with query understanding (synonyms, typo correction) — the inverted-index search pattern from the autocomplete and Twitter guides.

### Comments & likes

Comments **partitioned by `video_id`** (a video's comments live together); likes as **sharded counters** to survive viral hot partitions — same patterns as Instagram.

---

## 10. Live Streaming

Live adds a real-time ingest path and a fundamental **latency-vs-scale trade-off**:

```
[Producer] ─(RTMP/SRT)─► [Ingest Server] ─► [Transcoder] ─► [CDN] ─► [Viewers]
```

The broadcaster pushes a stream (RTMP/SRT) to an ingest server, which transcodes it into ABR variants in real time and pushes segments to the CDN. The latency tier is a design choice:

- **Traditional HLS** — segment-based, **~10s** latency; simplest and most scalable (rides the same CDN/segment infrastructure as VOD).
- **LL-HLS (Low-Latency HLS)** — **~3s**; smaller/partial segments.
- **WebRTC** — **~1s** (sub-second); needed for true interactivity (auctions, video calls) but **harder to scale** to millions of viewers because it's connection-oriented rather than cacheable segments.

The trade-off: **lower latency costs scalability and complexity.** Pick the tier the use case actually needs — most live video tolerates a few seconds and rides the cacheable-segment path; only genuinely interactive cases justify WebRTC's cost.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. What's the fundamental nature of the YouTube design?**
A content-delivery and transcoding problem — ~95% of cost is video storage and bandwidth. So you optimize for transcoding throughput (the dominant compute) and CDN cache hit rate (the dominant bandwidth cost). The metadata/social layer is comparatively trivial.

**Q2. How do you transcode efficiently at scale?**
Asynchronously (queue-driven, so uploads return immediately and show "processing") and in massive parallel — split each video into chunks transcoded independently across a worker cluster (MapReduce-style), then reassemble. Each video produces resolutions × formats × segments files.

**Q3. How does adaptive bitrate streaming work and why is it essential?**
The video is segmented and encoded at multiple bitrates; a manifest lists them; the client measures bandwidth and picks a bitrate per segment, switching dynamically. This delivers low buffering on any network — dropping to 360p in a tunnel, climbing to 1080p on Wi-Fi. It's why transcoding generates so many variants.

**Q4. How do you handle storage given exabytes and a long tail?**
Tier it: hot/popular videos fully transcoded and replicated to CDN edges + nearby S3; the long tail in cheap cold storage (Glacier/Coldline), pulled on demand. Prioritize transcoding by popularity — generate common resolutions immediately, defer rare ones or transcode lazily for unpopular videos.

**Q5. Why is CDN cache hit rate the key metric?**
Because bandwidth is the dominant cost and CDN egress is expensive. Video segments are immutable (cache-friendly), and power-law popularity means a small edge cache serves most views. Pre-warm popular content; every point of higher hit rate cuts expensive origin pulls.

**Q6. How do you count views without melting the database?**
Don't increment a DB row per play (hot row on viral videos). Count in Redis with periodic flush to the DB, and stream play events to Kafka for accurate analytics. Dedup bots/sessions. Counts are eventually consistent and approximately real-time — fine for a view counter.

**Q7. How do recommendations differ from a Twitter/Instagram feed?**
They're a pull-based discovery feed, not follow-based: offline ML (collaborative filtering + deep models) on watch history produces scores, blended with real-time signals and cached per user. It drives most watch time but is a separable subsystem.

**Q8. How do you make search work over video?**
Elasticsearch indexes titles, descriptions, tags, and auto-generated transcripts (so spoken content is searchable), with synonym/typo handling — the inverted-index pattern. Autocomplete uses the trie-based typeahead design.

**Q9. What are the live-streaming latency options and trade-offs?**
HLS (~10s, simplest/most scalable, rides VOD CDN infra), LL-HLS (~3s), and WebRTC (~1s, interactive but hard to scale because it's connection-oriented, not cacheable segments). Lower latency costs scalability — choose the tier the use case needs.

**Q10. Why is the metadata/social layer the easy part?**
Because the bytes and cost are in video. Metadata is small relational data; comments/likes reuse partition-by-id and sharded-counter patterns; the feed/recs reuse Instagram/Twitter techniques. The genuinely hard, expensive engineering is transcoding and CDN delivery.

---

## 12. Quick Glossary

- **Transcoding** — converting a source video into multiple resolutions/formats/bitrates.
- **Segment** — a short (~5–10s) chunk of video, the unit of streaming and caching.
- **Adaptive bitrate (ABR)** — client picks per-segment bitrate based on measured bandwidth.
- **HLS / DASH** — segment-based ABR streaming protocols.
- **Manifest** — file listing available bitrates/segments for a video.
- **Parallel transcoding** — splitting a video into chunks transcoded independently across workers.
- **Tiered storage** — hot (CDN/S3) vs cold (Glacier/Coldline) by popularity.
- **Long tail** — the majority of videos that are watched rarely.
- **CDN cache hit rate** — fraction of view traffic served from edges; the key cost lever.
- **Pre-warming** — pushing popular content to edges before demand spikes.
- **Sharded counter** — splitting a count to avoid a hot partition (view/like counts).
- **Collaborative filtering** — recommendation technique using user-item interaction patterns.
- **RTMP / SRT** — protocols for ingesting a live stream from a broadcaster.
- **LL-HLS / WebRTC** — lower-latency live streaming options (~3s / ~1s).

---

*Reference document. Contributions and corrections welcome.*
