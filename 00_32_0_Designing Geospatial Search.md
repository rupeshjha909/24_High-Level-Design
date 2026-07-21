# Designing Geospatial Search (Yelp & Nearby Friends): A Senior Interview Guide

> A practical, interview-focused reference for geospatial systems — two products (Yelp's business search and Nearby Friends' live location) that share one core problem: *given my location, find X within Y km, fast*. This guide builds the architecture up piece by piece, traces the full life of a proximity query, goes deep on the two genuinely hard mechanisms (**how geolocation is *actually* found** — the geohash algorithm bit by bit, and **how one spatial index serves two opposite workloads**), nails down the data contracts, and covers the boundary problem, hot-region skew, privacy, and failure modes — with verified geohash, haversine, and capacity math and a senior follow-up bank.

> 💡 **The one idea (see §5):** you can't index 2D distance directly, so the trick behind *every* geo system is to **turn the 2D coordinate into a 1D key that preserves closeness** — a **geohash** built by repeatedly halving the map and recording "which half" as bits. Nearby points then share a **prefix**, so "find things near me" becomes a cheap **prefix lookup** any sorted index can do. Same recipe powers Yelp, Nearby Friends, Uber, and food delivery.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified)](#3-capacity-estimation-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a proximity query](#44-the-end-to-end-life-of-a-proximity-query)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [Hard Problem 1: How Geolocation Is Actually Found (step by step)](#5-hard-problem-1-how-geolocation-is-actually-found-step-by-step)
6. [Hard Problem 2: One Index, Two Opposite Workloads](#6-hard-problem-2-one-index-two-opposite-workloads)
7. [The Other Indexing Techniques (and when to pick them)](#7-the-other-indexing-techniques-and-when-to-pick-them)
8. [Data Contracts: Request Fields, Payloads & Store Schemas](#8-data-contracts-request-fields-payloads--store-schemas)
9. [Scaling & Hot Regions](#9-scaling--hot-regions)
10. [Privacy (live location is sensitive)](#10-privacy-live-location-is-sensitive)
11. [Failure Modes & Handling](#11-failure-modes--handling)
12. [Senior Follow-Up Questions (with Answers)](#12-senior-follow-up-questions-with-answers)
13. [Quick Glossary](#13-quick-glossary)

---

## 1. How to Approach This in an Interview

Yelp and Nearby Friends look like different products, but underneath they're **the same problem: geospatial search — given a location, efficiently find things within a radius.** Yelp finds *businesses* near you; Nearby Friends finds *friends* near you. So the strong interview move is to **identify the shared core first** ("both reduce to geospatial proximity search"), go deep on the **spatial index** that makes it fast, and then note how the two products apply that same index to **opposite workloads:**
- **Yelp** — **read-heavy** search over relatively **static** business data (geo + full-text + filters + ranking).
- **Nearby Friends** — **write-heavy** over **ephemeral** live-location data (constant updates, TTL, privacy-sensitive).

A good structure: requirements → recognize the common geospatial core → go deep on **how geolocation is found (the geohash mechanism)** → show the **same-index/opposite-workload** split → cover hot regions, privacy, failure.

> 💡 **Senior signal:** say up front — *"Both are geospatial proximity search. The core is a spatial index that turns 2D coordinates into a 1D key so 'near me' is a prefix lookup, not a full scan. Then Yelp and Nearby Friends put that index to opposite use — read-heavy static vs write-heavy ephemeral. I'll go deep on how the geohash actually works, then the two workloads and hot-region skew."*

---

## 2. Requirements

### Yelp (read-heavy, static)
- Search businesses by name, category, location; filter by rating, distance, open-now; show details/reviews/photos.

### Nearby Friends (write-heavy, ephemeral)
- Users share **live location** with friends; see friends within N km on a map; notify when a friend is nearby.

### Non-Functional (shared)
- **Low latency** (<200 ms). **Massive write rate** for live location. **High read rate** for search. **Privacy** for live location (sensitive PII).

---

## 3. Capacity Estimation (verified)

```
NEARBY FRIENDS (write-heavy):
  50M active sharers × 1 update / 30s = ~1,666,667 location writes/sec   ← the write firehose
  (ephemeral: only the latest position matters → in-memory, TTL'd, no durability needed)

YELP (read-heavy):
  500M searches/day → ~5,787/s avg, peak ~23,000/s
  ~50M businesses (static, indexed by geohash + full-text)               ← the read/index load
```

**Takeaways that shape the design:**
- **Nearby Friends is a location firehose (~1.67M writes/s)** → keep the latest position in an **in-memory geo store (Redis GEO, last-write-wins, TTL'd)**, never a durable DB per update.
- **Yelp is read-heavy over static data** → a **durable, indexed, cached** store (Elasticsearch geo + full-text), heavily cached, popular searches precomputed.
- **The spatial index is identical**; the **durability/consistency needs are opposite** — that's the senior insight (§6).

> 💡 The numbers force the split: same geohash index, but Nearby Friends wants *fast + ephemeral + TTL*, while Yelp wants *durable + indexed + cached*. One idea, two engineerings.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

Every geo feature runs into the same wall: you want "find things within 5 km of me," but a database indexes 1D values (numbers, strings) beautifully and 2D *distance* not at all — computing distance to every row is an O(n) full scan that no ordinary index can help. The escape, used by essentially every geo system, is to **convert the 2D coordinate into a 1D key that preserves closeness**: a **geohash**, built by repeatedly cutting the map in half (left/right, then top/bottom, alternating) and recording each "which half" as a bit, then base32-encoding the bits into a short string. The magic property is that **two nearby points share a leading prefix**, so "near me" becomes "rows whose geohash starts with the same characters" — a cheap prefix/range lookup any sorted index does instantly. A proximity query is then three cheap steps: **encode** your location to a geohash, **prefix-match** that cell *plus its 8 neighbors* (to catch points just across a cell border), and **haversine-rank** the small candidate set by exact distance. Yelp and Nearby Friends both run this recipe — but Yelp puts the index over **durable, static business data** (Elasticsearch, cached), while Nearby Friends puts it over an **ephemeral live-location firehose** (Redis GEO, TTL'd) — same index, opposite workload. The recurring headache in both is **hot regions** (downtown Manhattan generates vastly more load than the ocean), handled by sharding on geohash prefix and replicating/caching the dense cells.

### 4.2 The diagram

```
                          ┌──────────────────────────────────────┐
                          │   THE SHARED CORE: geohash spatial index │
                          │   encode(lat,lon) → 1D key;              │
                          │   nearby points share a PREFIX           │
                          └───────────────┬──────────────────────────┘
             ┌──────────────────────────────┴──────────────────────────────┐
   YELP (read-heavy, static)                                NEARBY FRIENDS (write-heavy, ephemeral)
   [Client]→[Search API]                                    [phone] --loc every ~30s--> [Location API]
      ├─►[Elasticsearch]  geo + full-text (BKD/geohash)          │
      ├─►[Business DB (MySQL)]                                   ▼
      ├─►[Reviews (Cassandra)]                          [Redis GEO]  GEOADD / GEOSEARCH
      ├─►[Photos (S3+CDN)]                              (geohash-backed sorted set, TTL'd, in-memory)
      └─►[Ranking]  relevance+distance+rating                    ▲
      (durable, indexed, HEAVILY CACHED)                [friend's app] --query nearby--┘
```

The key visual idea: **one geohash index in the middle**, feeding two products with opposite storage needs — Yelp durable/cached/indexed on one side, Nearby Friends ephemeral/in-memory/TTL'd on the other.

### 4.3 Each component, in detail

**① The geohash spatial index (the shared core).** Turns `(lat, lon)` into a 1D geohash so proximity = prefix match (§5). Not a single server — it's a *technique* realized differently per product (Elasticsearch's geo index for Yelp; Redis GEO's geohash-backed sorted set for Nearby Friends).

**② Yelp — Search API.** Accepts "pizza near Mumbai," fans out to Elasticsearch (geo + full-text in one query), applies filters (open-now, rating), ranks (relevance + distance + rating), returns top-K.

**③ Yelp — Elasticsearch.** Holds business docs with a **geo field + inverted index**, so one query does both "matches *pizza*" and "within 5 km" (its geo index does exactly the §5 prefiltering before scoring). The read workhorse.

**④ Yelp — polyglot stores.** Business metadata in **MySQL**; reviews in **Cassandra** (write-heavy, partitioned by business); photos in **S3 + CDN**. Static data → cached heavily, popular searches precomputed.

**⑤ Nearby Friends — Location API.** Ingests the ~1.67M updates/s firehose; for each, **upserts the friend's latest position** into Redis GEO (last-write-wins). Never a durable write per update.

**⑥ Nearby Friends — Redis GEO.** In-memory geospatial store: `GEOADD` to upsert a point, `GEOSEARCH` to query a radius — backed internally by the exact geohash-in-a-sorted-set mechanism from §5. Sub-ms latency + **TTL auto-expiry** (a friend who goes offline simply drops off).

**⑦ Ranking / haversine (both).** After the index narrows to a small candidate set (cell + neighbors), compute exact **haversine** distance to sort/filter (§5, Step 7).

### 4.4 The end-to-end life of a proximity query

Here is exactly what happens for **"find pizza within 5 km of me"** (Yelp) — the Nearby-Friends path is the mirror image (query friends instead of businesses):

```
1. Client sends { lat, lon, query:"pizza", radius:5km, filters:{open_now} }.
2. Pick a geohash LENGTH matching the radius: 5 km → length 5 (a ~5 km cell).   ← §5 Step 4
3. ENCODE the user's location to a geohash: encode(lat,lon,5) → e.g. "9q8yy".    ← §5 Steps 2–3
4. Compute the CELL + its 8 NEIGHBORS (the 3×3 block) so points just across a
      border aren't missed.                                                       ← §5 Step 6
5. PREFIX-MATCH: fetch all businesses whose geohash starts with any of those 9
      cell prefixes  →  a small candidate set (not the whole 50M).                ← §5 Step 5
      (Yelp: Elasticsearch geo filter + "pizza" full-text + open_now, in one query.)
6. HAVERSINE-RANK: compute exact distance to each candidate; keep ≤ 5 km.         ← §5 Step 7
7. Combine with relevance + rating → sort → return top 20.
```

The single most important thing to notice: **the index does cheap coarse filtering (steps 2–5, prefix lookups), and exact math runs only on the handful of survivors (step 6).** You never compute distance across all 50M businesses — that's the whole point of the spatial index.

### 4.5 Why this split? (the design rationale)

- **2D → 1D geohash instead of raw distance** — databases index 1D keys, not 2D distance; the geohash makes proximity a prefix lookup, turning O(n) into a cheap range scan.
- **Cell + 8 neighbors, not just the cell** — geohash has boundary discontinuities (two close points can straddle a cell edge and share no prefix); querying neighbors guarantees you never miss a nearby point (§5 Step 6).
- **Index filters coarse, haversine ranks fine** — prefix matching is cheap but approximate (a cell is a square, not a circle); haversine on the small candidate set gives exact distance without scanning everything.
- **Same index, different stores per product** — Yelp's data is static/durable/read-heavy → Elasticsearch + caching; Nearby Friends' is ephemeral/write-heavy → in-memory Redis GEO + TTL. The *index* is shared; the *durability/consistency* choices are opposite (§6).
- **Shard by geohash prefix** — groups a region onto one shard for locality (a "near Mumbai" query hits one shard), at the cost of hot-region skew (§9).

### 4.6 Where the load actually goes

- **Nearby Friends writes:** ~**1.67M location updates/s** — the firehose, absorbed by **in-memory Redis GEO upserts (last-write-wins)**, never a durable DB per update. Only the latest position matters.
- **Yelp reads:** ~**5.8K/s avg, ~23K/s peak** — served from **Elasticsearch + heavy caching**; popular searches precomputed.
- **Candidate-set size, not dataset size, drives query cost** — a query touches ~9 cells' worth of points, then haversines that handful; the 50M total is irrelevant per query *because* of the index.
- **Hot regions (the deceptive skew):** dense areas (downtown) generate far more traffic than sparse ones, so their shards become **hot partitions** — the real scaling challenge (§9).
- **Boundary correctness costs 9× lookups, not N×** — querying cell + 8 neighbors is a fixed small multiplier, not a scan.

> 💡 **The senior framing:** *"Per-query cost is set by the candidate set (~9 cells), not the dataset — that's what the geohash index buys. The write firehose (Nearby Friends, ~1.67M/s) goes to in-memory last-write-wins Redis GEO; the read load (Yelp) goes to cached Elasticsearch. The real scaling pain is hot regions — dense-city shards — handled by prefix-sharding plus replicating and caching those cells."*

---

## 5. Hard Problem 1: How Geolocation Is Actually Found (step by step)

This is the mechanical heart — follow it once and you can rebuild geo search for *any* app. Every number is verified in code.

### Step 0 — What latitude and longitude are
Earth is addressed by two numbers: **latitude** (north-south, −90° to +90°, equator = 0) and **longitude** (east-west, −180° to +180°, Greenwich = 0). San Francisco = `(37.7749, −122.4194)`. The task: given one such pair, quickly find others physically close. The problem: "close" is a 2D calculation, and databases index 1D values well but not 2D distance. **The geohash turns the 2D pair into a 1D string that still respects closeness.**

### Step 1 — The core idea: play "which half?" on the map
Treat the world as one rectangle and repeatedly cut it in half, recording which half your point is in:
- Cut **left/right** by longitude: right half → **1**, left → **0**. Keep that half.
- Cut **bottom/top** by latitude: top → **1**, bottom → **0**. Keep that half.
- Repeat, **alternating longitude and latitude**; each cut halves the area and adds one bit of precision.

Each answer is one **bit**. More cuts → smaller surviving box → more precise location. That bit-stream *is* the encoded location.

### Step 2 — The algorithm, with a verified trace
```
lon_range=[-180,180]; lat_range=[-90,90]; start with LONGITUDE
repeat:
   LON step: mid=(lon_lo+lon_hi)/2; if lon>=mid → bit 1, lon_lo=mid; else bit 0, lon_hi=mid
   LAT step: mid=(lat_lo+lat_hi)/2; if lat>=mid → bit 1, lat_lo=mid; else bit 0, lat_hi=mid
   append bit; switch LON<->LAT
```
First 10 cuts for SF `(37.7749, −122.4194)`, verified:
```
 1 LON  -122.4194 >=    0.000? → 0   (western hemisphere)
 2 LAT    37.7749 >=    0.000? → 1   (northern hemisphere)
 3 LON  -122.4194 >=  -90.000? → 0
 4 LAT    37.7749 >=   45.000? → 0
 5 LON  -122.4194 >= -135.000? → 1
 6 LAT    37.7749 >=   22.500? → 1
 7 LON  -122.4194 >= -112.500? → 0
 8 LAT    37.7749 >=   33.750? → 1
 9 LON  -122.4194 >= -123.750? → 1
10 LAT    37.7749 >=   39.375? → 0
```
The bits **interleave** longitude/latitude — that interleaving is what keeps *both* dimensions' closeness in the final string.

### Step 3 — Bits → base32 string
Group every **5 bits** into one character via base32 (2⁵ = 32 symbols), using the geohash alphabet (no `a,i,l,o`): `0123456789bcdefghjkmnpqrstuvwxyz`. Verified: `encode(37.7749, −122.4194)` = **`9q8yyk8yt`**. Proof it's the industry-standard algorithm: the classic example `encode(57.64911, 10.40744)` = **`u4pruydqqvj`** — the exact published geohash.

### Step 4 — Longer string = smaller cell (verified table)
Each extra character adds 5 bits (~5 halvings), shrinking the cell:
```
 len | cell size            | scale
  1  | ~5000 km             | continent
  3  | ~156 km              | large region
  4  | ~39 km × ~20 km      | city
  5  | ~5 km × ~5 km        | neighborhood   ← common for "near me"
  6  | ~1 km × ~610 m       | few blocks
  7  | ~152 m               | street
  9  | ~5 m                 | precise
```
Pick a length matching your radius: 5–6 for "within a few km," 7+ for "within a block."

### Step 5 — The payoff: nearby points share a prefix
Because each character encodes shared "which half" answers, close points made the **same early choices** → their geohashes **start with the same characters**. Verified:
```
SF A            = 9q8yyk8yt
SF B (~200 m)   = 9q8yym1bj   → shares first 5 chars with A
LA  (~560 km)   = 9q5ctr186   → shares only 2 chars with A
```
So **"find things near A" = "find geohashes starting with `9q8yy`"** — a plain prefix/range query any sorted index does instantly (Redis sorted set, B-tree, `WHERE geohash LIKE '9q8yy%'`). No distance math over the whole table.

### Step 6 — The catch: the boundary problem
Two points meters apart can sit on **opposite sides of a cell boundary** and share little/no prefix. **Fix:** query the user's cell **plus its 8 neighbors** (the 3×3 block), so a point just across a border is never missed.
```
   ┌────┬────┬────┐
   │ NW │ N  │ NE │   query the user's cell (C)
   ├────┼────┼────┤   PLUS all 8 neighbors → boundary-straddling
   │ W  │ C  │ E  │   points are never missed
   ├────┼────┼────┤
   │ SW │ S  │ SE │
   └────┴────┴────┘
```

### Step 7 — Final ranking: exact haversine distance
Prefix + neighbors gives a small **candidate set**; now compute true great-circle distance and sort/filter:
```
a = sin²(Δlat/2) + cos(lat1)·cos(lat2)·sin²(Δlon/2)
distance = 2·R·asin(√a),   R ≈ 6371 km
```
Verified: haversine NYC→LA = **3,936 km** (matches known ~3,936); SF A→B = **216 m**.

### Step 8 — The whole query, end to end
```
1. pick geohash length L so a cell ≈ radius R          (R≈5km → L=5)
2. g = encode(lat, lon, L)
3. cells = { g } ∪ neighbors(g)                        (3×3 block)
4. candidates = points whose geohash starts with any cell prefix
5. for each candidate: d = haversine(user, candidate)
6. keep d ≤ R, sort by d, return top-K
```

> 💡 **The senior one-liner:** *"You find geolocation by turning 2D (lat,lon) into a 1D geohash — a string built by repeatedly halving the map and recording 'which half' as bits, interleaving longitude and latitude, then base32-encoding every 5 bits. Nearby points share a prefix, so proximity search is a prefix/range lookup on a sorted index; you also scan the 8 neighbor cells for boundary cases, then rank the small candidate set by exact haversine distance. Encode → prefix-match + neighbors → haversine-rank powers Yelp, Nearby Friends, Uber, and food delivery."*

---

## 6. Hard Problem 2: One Index, Two Opposite Workloads

The elegant twist: Yelp and Nearby Friends use the **same geohash index** but put it to **opposite work.** Understanding *why the same index fits both* is the senior insight.

### Yelp — read-heavy over static, durable data
Businesses barely change, and reads dominate (~23K/s peak). So the index lives in **Elasticsearch**, which combines a **geo field** with a **full-text inverted index** — one query does both "matches *pizza*" and "within 5 km":
```
{ text: "pizza", geo_distance: { origin:[lat,lon], distance:"5km" }, filter:{ open_now:true } }
```
Under the hood ES's geo index (BKD-tree / geohash) does exactly the §5 prefiltering before scoring. Because data is static and read-heavy, you **cache aggressively** and **precompute popular searches**. Polyglot: MySQL (business metadata), Cassandra (reviews), S3+CDN (photos).

### Nearby Friends — write-heavy over ephemeral data
The inverse: a **flood of location writes** (~1.67M/s) over data that's only briefly relevant. So the index lives in **Redis GEO** (`GEOADD`/`GEOSEARCH`, geohash-backed sorted set):
- **Sub-millisecond latency** — essential under a constant update+query stream.
- **TTL auto-expiry** — stale positions drop automatically; a friend who goes offline just disappears, no cleanup.
- **No durability needed** — losing one update is harmless (the next 30s update replaces it), so in-memory + last-write-wins is exactly right.

### Why the same index fits both
The geohash index answers "what's near here?" regardless of *what* is stored in the cells or *how durably*. Swap "businesses" for "friends," "durable + cached" for "ephemeral + TTL'd," and the identical encode→prefix→neighbors→haversine machinery serves both.

| | **Yelp** | **Nearby Friends** |
|:--|:--|:--|
| Workload | read-heavy | write-heavy |
| Data | static businesses | ephemeral live location |
| Store | Elasticsearch (durable, cached) | Redis GEO (in-memory, TTL'd) |
| Consistency | indexed, cacheable | best-effort, last-write-wins |
| Same index? | **geohash** | **geohash** |

> 💡 **The senior one-liner:** *"Same geohash index, opposite workloads. Yelp is read-heavy over static, durable business data — Elasticsearch geo + full-text, heavily cached. Nearby Friends is write-heavy over ephemeral location — Redis GEO with sub-ms latency and TTL auto-expiry, no durability because a lost update is replaced in 30 seconds. The spatial index is shared; the durability and consistency choices are what differ."*

---

## 7. The Other Indexing Techniques (and when to pick them)

Geohash is the most important, but know the alternatives and *when* each wins:

- **Grid (fixed cells)** — simple, but **fixed resolution**: dense cities and empty oceans get the same cell size, so some cells overload. Query cell + 8 neighbors.
- **Geohash (space-filling curve)** — 1D prefix lookups, trivially shardable/indexable; **boundary discontinuities** (hence neighbor checks). What **Redis GEO** uses internally. *Default choice for shardable indexing.*
- **Quadtree** — recursively splits into 4, subdividing only dense regions → **adapts to density** (cities vs countryside). More complex; rebalancing hurts for moving points.
- **R-tree** — a B-tree for **rectangles/bounding boxes**; supports arbitrary shapes/regions. Used by **PostGIS** and **MySQL spatial**. *Pick when you're in a relational spatial DB.*
- **H3 (Uber's hexagons)** — a global hex grid at stackable resolutions. Hexagons fix the square-grid **"4-corner problem"**: a square has neighbors at two distances (edge vs corner), a **hexagon has 6 uniformly-adjacent neighbors** → cleaner "nearby." *Pick for matching/coverage (Uber).*

**Two families:** *space-filling curves* (geohash — 2D→1D, prefix match, boundary caveats) vs *hierarchical partitioning* (quadtree/R-tree/H3 — descend to the region; adaptive/shape-friendly/uniform-neighbor). All convert an O(n) distance scan into a bucket/prefix/tree lookup — that's the whole game.

---

## 8. Data Contracts: Request Fields, Payloads & Store Schemas

### Part A — Key client↔server requests
**Yelp search** (client → Search API):
```json
{ "lat":19.0760, "lon":72.8777, "query":"pizza", "radius_km":5,
  "filters":{ "open_now":true, "min_rating":4.0 }, "limit":20 }
→ { "results":[ { "business_id":"b_9","name":"…","distance_m":420,"rating":4.3,
                  "geohash":"te7ud","photo_url":"https://cdn/…" } ], "next_cursor":"…" }
```
**Nearby Friends — location update** (phone → Location API, every ~30s):
| Field | Type | Purpose |
|:--|:--|:--|
| `user_id` | string | whose position |
| `lat`,`lon` | double | current position (may be coarsened for privacy) |
| `ts` | epoch ms | drop stale/out-of-order updates |
> Fire-and-forget: returns **202**; only the latest matters (last-write-wins).

**Nearby Friends — query** (app → Location API): `{ user_id, radius_km:2 }` → `{ friends:[ { friend_id, distance_m, last_seen } ] }` (filtered to opted-in friends only).

### Part B — Inter-service payloads
- **Location API → Redis GEO:** `GEOADD friends:{region} lon lat user_id` (upsert) + set TTL; query `GEOSEARCH friends:{region} FROMMEMBER user_id BYRADIUS 2 km`.
- **Search API → Elasticsearch:** the combined `geo_distance` + full-text + filter query (Part A).
- **Both → haversine ranker:** `{ origin:{lat,lon}, candidates:[{id,lat,lon}] }` → sorted-by-distance list.

### Part C — Store schema per system
**① Yelp businesses — Elasticsearch (durable, indexed, cached):**
```
business { business_id, name, category, location:geo_point, geohash,
           rating, hours, ... }   -- geo_point enables geo_distance; inverted index for text
```
**② Yelp metadata/reviews/photos — polyglot:**
```
MySQL:     business( id PK, name, category, address, lat, lon, geohash )
Cassandra: reviews( business_id PARTITION, review_id CLUSTERING, user_id, stars, text, ts )
S3+CDN:    photos keyed by business_id
```
**③ Nearby Friends — Redis GEO (in-memory, ephemeral, TTL'd):**
```
GEOADD friends:{region}  lon lat  user_id      -- upsert latest (last-write-wins)
key TTL ~2–5 min  → stale sharers auto-drop
friend-graph + opt-in flags in a separate durable store (who may see whom)
```
**④ Sharding:** both shard by **geohash prefix** (a region → one shard) for locality.

> 💡 **The contract in one line:** *"A geo query carries (lat, lon, radius); the server encodes a geohash, prefix-matches the cell + neighbors, and haversine-ranks. Yelp stores businesses in Elasticsearch (geo_point + full-text, cached) plus polyglot metadata; Nearby Friends upserts positions into Redis GEO (last-write-wins, TTL'd) and filters to opted-in friends. Every field exists for proximity, ranking, freshness, or privacy."*

---

## 9. Scaling & Hot Regions

- **Shard by geohash prefix** — a city's data lands on one shard (nearby points share a prefix), so "near Mumbai" hits one shard, not all → **locality**. This is the consistent-hashing/partitioning principle applied geographically.
- **Hot-region skew (the recurring problem)** — dense areas (Manhattan, downtown Mumbai) generate vastly more traffic than empty regions, so their shards become **hot partitions**. Mitigations: **replicate hot regions read-only** (spread reads), **edge-cache common queries** (CDN-like), **precompute popular searches**, and for writes consider **sub-sharding** a hot cell further.
- **Cache top-queried cells** — the most-searched geohash cells stay hot in cache.

> 💡 The geographic twist on an old theme: sharding by location gives locality but *concentrates* load on dense-area shards — so you replicate and cache exactly those hot regions.

---

## 10. Privacy (live location is sensitive)

Location is sensitive PII, so for Nearby Friends privacy is first-class:
- **Opt-in per friend** — sharing is explicit and granular (who may see you).
- **Approximate location** — round to ~100 m (equivalently, store a **shorter/coarser geohash** — §5 Step 4) so exact whereabouts aren't exposed.
- **Auto-disable after inactivity** — sharing turns off when unused; **TTL** drops stale data automatically.

> 💡 Coarsening precision *is* just using a shorter geohash — the same length knob that controls cell size also controls how much you reveal.

---

## 11. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Naive distance filtering | O(n) full scan, slow | geohash index → prefix lookup on cell + neighbors |
| Boundary straddle | miss a nearby point | query cell + 8 neighbors (3×3), then haversine-rank |
| Redis GEO node loss (Nearby Friends) | stale positions | in-memory rebuilds from next updates in ~30s; replicate hot regions |
| Hot region overload | dense-city shard hot | replicate read-only, edge-cache, precompute, sub-shard |
| Stale live location | friend appears where they aren't | TTL auto-expiry; show `last_seen`; next update corrects |
| Elasticsearch slow under load (Yelp) | search latency | cache popular searches, precompute, scale read replicas |
| Privacy leak | exposing exact location | opt-in per friend, coarsen to ~100 m (shorter geohash), auto-disable |
| Cell too big/small for radius | too many/few candidates | pick geohash length ≈ radius; widen/narrow as needed |

---

## 12. Senior Follow-Up Questions (with Answers)

**Q1. What's the common core of Yelp and Nearby Friends?** Geospatial proximity search — find things within a radius, efficiently, via a spatial index. They differ in workload: Yelp read-heavy over static businesses; Nearby Friends write-heavy over ephemeral live location. (§1, §6)

**Q2. Why can't you just filter by distance in SQL?** `WHERE distance(...) < 5km` computes distance to every row — an O(n) scan a normal index can't help. You need a spatial index that maps 2D to a queryable 1D key so you examine only the relevant region. (§3, §5)

**Q3. Explain how a geohash is computed.** Convert (lat,lon) to bits by repeatedly halving the map (longitude: right=1/left=0; latitude: top=1/bottom=0), alternating, each halving the box. Group every 5 bits into a base32 char. Nearby points share a prefix. Verified: (37.7749,−122.4194)→`9q8yyk8yt`; (57.64911,10.40744)→`u4pruydqqvj`. (§5)

**Q4. Why does sharing a prefix mean two points are near?** Each char encodes shared "which half" decisions; sharing 5 chars means the same 25 halvings → the same ~5 km cell. More shared chars = smaller shared cell = closer. (§5 Step 5)

**Q5. What's the boundary problem and the fix?** Two points meters apart can straddle a cell edge and share little prefix. Fix: query the user's cell **plus 8 neighbors** (3×3), then haversine-rank. (§5 Step 6)

**Q6. How do you rank exactly?** Haversine great-circle distance on the small candidate set. The geohash prefilters; haversine gives the true distance to sort by. Verified NYC→LA = 3,936 km. (§5 Step 7)

**Q7. How do you choose geohash length?** Match it to the radius: length 5 ≈ 5 km, 6 ≈ 1 km, 7 ≈ 150 m. Pick the length whose cell ≈ your radius, then add neighbors. (§5 Step 4)

**Q8. Compare the indexing techniques.** Grid: simple, fixed resolution. Geohash: 1D prefix, shardable, boundary caveats. Quadtree: density-adaptive. R-tree: rectangles (PostGIS/MySQL). H3: uniform-neighbor hexagons (Uber). Families: space-filling curve vs hierarchical partition. (§7)

**Q9. Why hexagons (H3) over squares?** Squares have neighbors at two distances (edge vs corner); hexagons have 6 uniformly-adjacent neighbors → consistent "nearby," good for matching/coverage. (§7)

**Q10. Why Redis GEO for Nearby Friends, and same-index/opposite-workload?** Sub-ms latency + TTL auto-expiry suit ephemeral best-effort location (a lost update is replaced in 30s; no durability needed). Yelp uses the same geohash idea over durable static data with ES + caching. Same index, opposite durability/consistency. (§6)

**Q11. How do you scale and handle hot regions?** Shard by geohash prefix for locality (a city → one shard); dense regions become hot shards → replicate read-only, edge-cache, precompute, sub-shard. (§9)

**Q12. How do you protect location privacy?** Opt-in per friend, coarsen to ~100 m (a shorter geohash), auto-disable on inactivity, TTL stale data. Sharing is explicit, coarse, self-expiring. (§10)

---

## 13. Quick Glossary
- **Latitude / longitude** — north-south (−90..+90) / east-west (−180..+180) coordinates.
- **Geospatial search** — finding entities within a distance of a location.
- **Spatial index** — a structure enabling proximity queries without scanning all data.
- **Space-filling curve** — a 2D→1D mapping preserving locality (geohash's basis).
- **Geohash** — base32 encoding of coordinates (repeated halving + bit interleaving) where nearby points share a prefix.
- **Bit interleaving** — alternating longitude/latitude bits so both dimensions' locality survives in one string.
- **Base32** — the 32-symbol alphabet (`0-9`,`b-z` minus `a,i,l,o`) turning every 5 bits into a char.
- **Prefix match** — "share the same leading characters," the proximity query on geohashes.
- **Neighbor cells (3×3)** — the user's cell + 8 surroundings, queried to cover boundary cases.
- **Haversine** — great-circle distance formula used to rank candidates exactly.
- **Grid / Quadtree / R-tree / H3** — fixed-cell / density-adaptive / rectangle / hexagon spatial indexes.
- **Redis GEO** — Redis geospatial commands (GEOADD/GEOSEARCH) over a geohash-backed sorted set.
- **TTL auto-expiry** — stale locations dropping automatically (ephemeral live data).
- **Geohash-prefix sharding** — partitioning so a region's data stays on one shard (locality).
- **Hot region** — a dense area generating skewed load (a geographic hot partition).
- **geo_distance filter** — Elasticsearch's radius filter combined with full-text search.
- **Last-write-wins** — keeping only a friend's latest position, overwriting the previous.

---

*Reference document. Contributions and corrections welcome.*
