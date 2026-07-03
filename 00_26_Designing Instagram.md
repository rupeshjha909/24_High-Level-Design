# Designing Instagram: A Senior Interview Guide

> A practical, interview-focused reference for designing a media-centric social platform — sharing the social-feed skeleton of Twitter but dominated by a heavy media pipeline, ML-ranked feeds, and ephemeral Stories. Covers the media upload/processing flow, CDN delivery, feed generation, ranking, counters at scale, and Stories. With capacity math, trade-offs, and a senior follow-up bank.

> 💡 **The question this guide now answers in depth (see §4A):** *You record a Reel on Android and upload it — how does it play perfectly on an iPhone, a cheap Android, a tablet, and a slow 3G phone?* Short version: the raw file you upload is a **master copy** that almost no other device would play well as-is. Instagram **transcodes** it into a standard *ladder* of versions (many resolutions, bitrates, and codecs), cut into small segments, described by a **manifest** file. Any device downloads the manifest, **automatically picks the version that fits its screen and current network**, and streams it from a nearby **CDN** — so the phone that *recorded* it is irrelevant. Full, plain-language walkthrough below.

---

## Table of Contents

1. [How to Approach This (and the Twitter Connection)](#1-how-to-approach-this-and-the-twitter-connection)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Distinctive Part: The Media Pipeline](#4-the-distinctive-part-the-media-pipeline)
   - [4A. How an Android-Uploaded Reel Plays on iPhone & Everywhere (in detail)](#4a-how-an-android-uploaded-reel-plays-on-iphone--everywhere-in-detail)
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
- **Device- and network-agnostic playback** — the same content must play on any OS, screen size, and connection speed (the focus of §4A).

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
- **Transcoding multiplies stored bytes:** each upload becomes *several* renditions, so you store *more* than raw — a cost you pay to serve every device the right size (quantified in §4A).

---

## 4. The Distinctive Part: The Media Pipeline

This is where Instagram diverges from Twitter and where you should go deep. The upload-and-process flow:

```
1. Client requests a PRE-SIGNED S3 upload URL from the Media Service.
2. Client uploads the file DIRECTLY to S3 (bypassing our app servers).
3. The S3 upload fires an event → a Lambda/worker TRANSCODES it:
      photos → thumbnail, 480p, 1080p variants
      videos → multiple bitrates + codecs + HLS/DASH segments
4. Post Service stores the post: {post_id, user_id, caption, [media_urls], ts}.
5. Fan-out Service updates followers' feed caches.
```

Three senior points embedded here:

- **Pre-signed direct-to-S3 upload** — the client uploads straight to object storage using a short-lived signed URL, **bypassing your app servers entirely** (the pattern from the WhatsApp, Pastebin, and Dropbox guides). This protects the app tier and saves enormous bandwidth — at 200 TB/day you absolutely cannot route media through your servers.
- **Asynchronous transcoding** — processing (resize/transcode) is slow, so it's done **off the request path** (triggered by an S3 event), decoupled via the same queue/async principle as the scaling guide. The upload returns immediately; the post may briefly show a "processing" state.
- **Multiple resolutions** — generate variants so you serve the **right size for the context**: a tiny thumbnail for the grid, 1080p for full view, adaptive bitrate for video. This slashes bandwidth and speeds load — critical given the media volume.

The *why* and *how* of that third point — the thing that makes an Android upload play on an iPhone — is detailed next.

---

## 4A. How an Android-Uploaded Reel Plays on iPhone & Everywhere (in detail)

This is the part interviewers love and most candidates wave away. Let's walk the full journey of one Reel, in plain language, with the numbers verified in code.

### Step 0 — Why the raw Android file can't just be sent to the iPhone

When you record a Reel on Android, your phone produces **one video file** with specific properties: a resolution (say 1080×1920), a bitrate (maybe 12 Mbps — big), a **container** (like `.mp4`), and a **codec** (the compression method — e.g., H.264, HEVC, or VP9). Now think about who has to play it back:

- A **budget Android on 3G** can't download a 12 Mbps video fast enough — it would buffer endlessly.
- A **small phone screen** doesn't need 1080p — sending it wastes bandwidth for no visible benefit.
- An **older iPhone** might not decode the exact codec your Android recorded in.
- A **tablet on WiFi** *could* take full quality — but only if the network stays fast.

So there is **no single file** that is simultaneously right for all of them. If Instagram just stored your upload and served it as-is, playback would be broken or wasteful for most viewers. **The uploader's device and format must be made irrelevant.** That's the job of transcoding.

> 💡 **In simple words:** The file your Android recorded is fine for *your* phone, but wrong for most others (too big, wrong codec, wrong size for the screen/network). The system has to *convert* it into forms every device can use.

### Step 1 — Upload the "master" (platform-neutral raw material)

Using the **pre-signed URL** (§4), your Android uploads the original file straight to object storage (S3). Call this the **master** — the high-quality source. Nothing plays from the master directly; it's the raw material the pipeline processes. (Uploading direct-to-S3 is why 200 TB/day doesn't melt the app servers.)

### Step 2 — Transcoding: one master → a "ladder" of standard versions

An S3 event triggers a **transcoding** job. "Transcode" just means *re-encode the video into different forms.* The worker produces a **bitrate ladder** — the same Reel at several resolutions and bitrates:

```
Rendition   Resolution   Video bitrate
  240p          240p         400 kbps
  360p          360p         800 kbps
  480p          480p        1400 kbps
  720p          720p        2800 kbps
  1080p        1080p        5000 kbps
```

Each row is a **rendition** (a version). Low renditions are for small screens / slow networks; high renditions for big screens / fast networks. Crucially, these are **standardized formats** — the wild variety of what phones record is normalized into the *same fixed menu* that every device knows how to consume. **After this step, it no longer matters that the source came from Android.**

The pipeline also generates **poster/thumbnail** images (the still frame you see before playback).

### Step 3 — Segmentation: cut each rendition into little time-slices

Each rendition is chopped into short **segments** (typically 2–6 seconds each). A 30-second Reel at 4-second segments = **8 segments per rendition** (verified). With 5 renditions that's **40 small files** for one Reel.

Why bother? Because segments are what make **switching quality mid-playback** possible — the player can grab segment 1 in 720p, then segment 2 in 360p if the network dips, then segment 3 back in 720p, seamlessly. You can only switch at segment boundaries, so small segments = more responsive quality changes.

### Step 4 — The manifest: a "menu" that lists every version

Transcoding writes one more thing: a **manifest** (also called a playlist) — a tiny text file that lists all the renditions, their bitrates/resolutions, and where the segments live. Two standard formats:

- **HLS** (`.m3u8`) — created by Apple; the **native** choice for iPhone/Safari.
- **DASH** (`.mpd`) — an open standard; common on Android and web.

Instagram typically produces **both** (or uses formats playable by all), so every platform gets a manifest it understands. The manifest is the key handshake: **the player reads the menu, then decides what to order.**

### Step 5 — Store + distribute via CDN

The manifest, all renditions, and all segments are stored in object storage and pushed out through the **CDN** (edge servers worldwide — see the CDN guide). So when *any* viewer plays the Reel, the little segment files come from a server **near them**, not from the origin — fast everywhere, and the origin is protected from the load.

### Step 6 — Playback on the iPhone (and any device), step by step

Now your friend opens the Reel on their iPhone. Here's exactly what happens:

```
1. iPhone asks for the Reel → gets the MANIFEST (HLS .m3u8) from the CDN.
2. iPhone reads the menu: "available in 240p / 360p / 480p / 720p / 1080p."
3. iPhone measures its screen size and current network speed.
4. It PICKS the best rendition that fits, and requests those segments from the nearby CDN edge.
5. Its hardware video decoder plays each segment; it keeps measuring the network.
6. If the network changes, it switches rendition at the next segment boundary — smoothly.
```

The magic is Step 4: **the client chooses**, using two limits — never send more pixels than the screen shows, and never pick a bitrate higher than the network can sustain (with a safety margin). This is **Adaptive Bitrate streaming (ABR)**. Verified selections for the *same* Reel:

```
iPhone on fast WiFi (8 Mbps, 1080p screen) → serves 1080p
iPhone on 4G        (2 Mbps, 1080p screen) → serves 480p
budget Android      (0.6 Mbps, 720p screen)→ serves 240p
tablet, medium WiFi (3.5 Mbps, 1080p)      → serves 720p
```

One upload, and every device automatically gets the version that's right for *it* — with no coordination and nothing special from the uploader.

### Step 7 — Adaptive switching in action (why it never freezes)

Because the player re-checks the network every segment, quality **follows the connection** instead of freezing when it drops. Verified timeline (network fluctuating during one playback):

```
t= 0s  8000 kbps → 1080p
t= 8s  1500 kbps → 360p     ← network dipped, quality dropped to keep playing
t=12s   700 kbps → 240p     ← dipped more
t=24s  8000 kbps → 1080p    ← recovered, quality restored
```

The viewer sees a brief quality dip instead of the dreaded spinning buffer. This is the streaming version of "degrade gracefully."

### Step 8 — Codecs: give each device the most efficient format it can play

Beyond size, devices differ in which **codec** (compression method) they can decode. Newer codecs (AV1, HEVC/H.265) squeeze the same quality into fewer bytes but aren't supported everywhere; **H.264** is older, slightly bigger, but plays on *everything*. So the pipeline transcodes to a few codecs and each device gets the best it supports, with H.264 as the universal fallback (verified):

```
new iPhone     (AV1/HEVC/H264) → AV1   (smallest, best)
older iPhone   (HEVC/H264)     → HEVC
budget Android (H264 only)     → H264  (universal fallback)
```

### The key insight to say out loud

> 💡 **The senior one-liner:** *"The uploader's platform is made irrelevant by transcoding. Android uploads one master file; the pipeline re-encodes it into a standard ladder of resolutions, bitrates, and codecs, chops each into small segments, and writes a manifest. Any device — iPhone included — fetches the manifest, adaptively picks the rendition that fits its screen and live bandwidth, streams the segments from a nearby CDN, and switches quality on the fly. So one upload plays optimally on every OS, screen, and network, with the client doing the choosing."* Mention transcoding, the bitrate ladder, ABR (HLS/DASH), segmentation, CDN, and codec fallback and you've covered everything an interviewer wants here.

### The costs (name these too)

- **Storage multiplies** — one 30s Reel becomes ~**39 MB** across all renditions (verified), more than the original. You trade storage for universal, bandwidth-efficient playback.
- **Transcoding is compute-heavy** — a big async worker fleet (often GPU-accelerated); it's why the post shows "processing" briefly.
- **Popular Reels are pre-warmed** to CDN edges so the first viewers in each region don't wait.

---

## 5. Architecture

```
[Client] ─► [LB] ─► [API Gateway] ─┬─► [User Service]
                                    ├─► [Post Service]     ─► [Cassandra]  (posts)
                                    ├─► [Feed Service]     ─► [Redis]      (feed cache)
                                    ├─► [Media Service]    ─► [S3] ─► [CDN]
                                    │        └─► [Transcoding workers] ─► renditions + segments + manifest
                                    ├─► [Ranking Service]  ─► (ML feed scoring)
                                    ├─► [Search Service]   ─► [Elasticsearch]
                                    └─► [Notification Service]
```

Mirrors Twitter's structure plus a **Media Service** (pre-signed uploads + a transcoding worker fleet that builds the rendition ladder, segments, and manifest) and a **Ranking Service** (ML feed scoring) — the two Instagram-specific additions.

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

Partitioning by `user_id` makes a user's posts a single-partition scan, and a **Snowflake-style time-ordered `post_id`** gives sortable, globally-unique IDs — the same wide-column reasoning as Twitter. Note the post stores **media *URLs***, not media — the bytes are in S3/CDN. For a video, the stored URL points to the **manifest** (the menu from §4A), not to a single video file.

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

The ephemerality is the distinctive bit: a separate, TTL-governed store and a lighter (poll-based) delivery model than the main feed. (Video stories go through the same transcode/ABR pipeline as Reels — see §4A.)

---

## 10. Search, DMs & CDN Strategy

- **Search** — Elasticsearch indexes users, hashtags, and captions (inverted index, as in the Twitter and autocomplete guides). **Autocomplete** uses the trie-based typeahead design from the autocomplete guide.
- **Direct messages** — essentially a chat system; reuse the **WhatsApp design** (WebSocket connections, message store, delivery/read receipts) rather than reinventing it.
- **CDN strategy** — *all* media flows through the **CDN** (CDN guide): low-latency global delivery, origin offload, and DDoS protection. Additional media-specific tactics:
  - **Pre-warm popular content** to edges during off-peak hours.
  - **Adaptive bitrate streaming (HLS/DASH)** for video/Reels — the video is segmented at multiple bitrates and the client picks based on its current bandwidth, so playback adapts smoothly to network conditions (the full mechanism is §4A). This is the streaming analog of the multi-resolution photo strategy.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. How is Instagram different from Twitter at the design level?**
The feed core is the same (hybrid push/pull fan-out, Cassandra posts, Redis feeds). The differences: it's media-dominated (a heavy upload/transcode/CDN pipeline that handles ~200 TB/day), the feed is ML-ranked rather than chronological, and it has ephemeral Stories/Reels. The hard scaling is media, not feed metadata.

**Q2. Walk through the media upload pipeline.**
Client gets a pre-signed S3 URL and uploads directly to S3 (bypassing app servers). The S3 event triggers async transcoding into multiple resolutions and codecs (thumbnail/480p/1080p; video → a bitrate ladder + HLS/DASH segments + a manifest). Then the post record is created and fan-out updates feeds. Pre-signed upload + async transcoding keep heavy media work off the request path.

**Q3. I upload a Reel from Android — how does it play on an iPhone and other devices?**
The raw Android file is a *master*; nothing plays from it directly. Transcoding re-encodes it into a standard **bitrate ladder** (240p–1080p at set bitrates) and several **codecs** (AV1/HEVC/H.264), cuts each into short **segments**, and writes a **manifest** (HLS for Apple, DASH for others). All of it goes to the **CDN**. The iPhone fetches the manifest, **adaptively picks** the rendition matching its screen and live bandwidth, streams segments from a nearby edge, and switches quality at segment boundaries as the network changes — choosing the best codec it supports (H.264 as universal fallback). So the uploader's platform is made irrelevant by transcoding + adaptive streaming. (Full walkthrough in §4A.)

**Q4. Why not just serve the file the user uploaded?**
Because no single file suits all viewers: too big for slow networks, wrong resolution for small screens, and possibly a codec some devices can't decode. Serving the raw upload would buffer, waste bandwidth, or fail to play. Transcoding into a standard ladder + adaptive streaming is what makes playback universal and efficient.

**Q5. Why multiple resolutions and adaptive bitrate?**
To serve the right size for the context (thumbnail in the grid, full-res on tap) and to adapt video to the viewer's live bandwidth (HLS/DASH segments at multiple bitrates; the client downshifts on congestion and upshifts on recovery). This drastically cuts bandwidth, avoids buffering, and speeds loads — vital at this media volume.

**Q6. Why upload directly to S3 instead of through your servers?**
At 200 TB/day, routing media through app servers would saturate them and blow bandwidth costs. A pre-signed URL lets the client upload straight to object storage; your servers only handle small metadata. The same media-off-the-hot-path rule as WhatsApp/Dropbox.

**Q7. How does ML ranking change the feed vs a chronological one?**
Instead of merging candidates by timestamp, a ranking model scores them by recency, affinity, post type, and engagement signals, returning the top-ranked. It boosts engagement but adds read-path compute, so you pre-narrow candidates and use precomputed features to fit the ~200 ms budget.

**Q8. How do you handle like/comment counts on a viral post?**
A single counter row would be a hot partition under millions of likes/sec. Use sharded counters: split the count across N sub-counters, increment a random shard per like, and sum on read. Trades a slightly costlier read and approximate real-time counts for spread-out writes.

**Q9. How do Stories work and why are they handled differently?**
They're ephemeral (24h TTL), stored separately with explicit expiration (TTL + lifecycle cleanup), pre-aggregated per user, and delivered via ~2-minute client polling rather than push — because casual ephemeral content doesn't need real-time delivery, which is cheaper.

**Q10. How do you build the feed read to hit <200 ms with ranking?**
Serve the precomputed feed (push) from Redis, merge in celebrity pulls, then rank a pre-narrowed candidate set with precomputed features. Keep media out (only IDs + URLs/manifests in the feed; bytes from CDN). The expensive parts (fan-out, feature computation, transcoding) happen off the read path.

**Q11. What's the consistency model?**
Read-heavy and eventually consistent (AP-leaning), like Twitter: feeds, counts, and story availability can lag slightly. Acceptable because users prefer fast, always-available browsing over instant consistency, and it enables async fan-out, transcoding, and caching.

---

## 12. Quick Glossary

- **Media pipeline** — upload → transcode → segment → store → serve flow for photos/videos.
- **Master** — the original high-quality file the user uploads; the source for transcoding (not served directly).
- **Pre-signed URL** — short-lived URL letting clients upload directly to S3.
- **Transcoding** — re-encoding media into multiple resolutions, bitrates, and codecs.
- **Rendition / variant** — one version of the media (a specific resolution + bitrate).
- **Bitrate ladder** — the set of renditions from lowest to highest quality.
- **Codec** — the compression method (AV1, HEVC/H.265, H.264); newer = smaller but less universally supported; H.264 is the universal fallback.
- **Container** — the file wrapper (e.g., .mp4) holding the encoded video/audio.
- **Segment** — a short (2–6s) slice of a rendition; the unit of streaming and quality-switching.
- **Manifest / playlist** — the file listing all renditions and their segments (HLS `.m3u8`, DASH `.mpd`).
- **HLS / DASH** — adaptive-streaming standards; HLS is Apple-native, DASH is the open standard.
- **Adaptive bitrate (ABR)** — the client picking (and switching) rendition based on screen size and live bandwidth.
- **CDN** — edge servers delivering segments from near the viewer.
- **Hybrid fan-out** — push for regular users, pull for celebrities (see Twitter guide).
- **ML feed ranking** — scoring posts by predicted engagement instead of chronological order.
- **Affinity** — how strongly a user engages with a given author (a ranking feature).
- **Candidate set** — the narrowed list of posts the ranking model scores.
- **Sharded counter** — splitting a count across sub-counters to avoid a hot partition.
- **Hot partition** — an overloaded partition from concentrated writes (e.g., a viral post).
- **Stories** — ephemeral 24h-TTL content, stored separately and polled.
- **Reels** — short-form video; goes through the full transcode/segment/ABR pipeline (§4A).
- **Snowflake ID** — time-ordered, globally-unique ID used as the post clustering key.

---

*Reference document. Contributions and corrections welcome.*
