# Designing Nearby Friends / Yelp (Geospatial Search): A Senior Interview Guide

> A practical, interview-focused reference for geospatial systems — two products (Yelp's business search and Nearby Friends' live location) that share one core problem: *given my location, find X within Y km* efficiently. Covers why naive distance filtering fails, the spatial-indexing techniques (geohash, quadtree, R-tree, H3, Redis GEO), and how the same index serves opposite workloads. With trade-offs and a senior follow-up bank.

> 💡 **The deep question this guide now answers (see §4A):** *How is "geolocation" actually found — mechanically, bit by bit — so I can understand it and replicate it for any location-based system?* Short version: you convert the 2D coordinate `(latitude, longitude)` into a **1D string called a geohash** by repeatedly asking "is my point in the left/right (or bottom/top) half?" and recording each answer as a bit. Nearby points end up with the **same leading characters**, so "find things near me" becomes "find rows whose geohash starts with the same prefix" — a cheap lookup any database can do. The full algorithm, worked out step by step with verified numbers, is in §4A.

---

## Table of Contents

1. [The Common Core: Geospatial Search](#1-the-common-core-geospatial-search)
2. [Requirements (Both Systems)](#2-requirements-both-systems)
3. [Why Naive Distance Filtering Fails](#3-why-naive-distance-filtering-fails)
4. [Geospatial Indexing Techniques](#4-geospatial-indexing-techniques)
   - [4A. How Geolocation Is Actually Found — Step by Step (the deep dive)](#4a-how-geolocation-is-actually-found--step-by-step-the-deep-dive)
5. [The Unifying Insight](#5-the-unifying-insight)
6. [Yelp: Read-Heavy Geo Search](#6-yelp-read-heavy-geo-search)
7. [Nearby Friends: Write-Heavy Live Location](#7-nearby-friends-write-heavy-live-location)
8. [Scaling & Hot Regions](#8-scaling--hot-regions)
9. [Senior Follow-Up Questions (with Answers)](#9-senior-follow-up-questions-with-answers)
10. [Quick Glossary](#10-quick-glossary)

---

## 1. The Common Core: Geospatial Search

Yelp and Nearby Friends look like different products, but underneath they're **the same problem: geospatial search — given a location, efficiently find things within a radius.** Yelp finds *businesses* near you; Nearby Friends finds *friends* near you. So the interview move is to **identify the shared core first** ("both reduce to geospatial proximity search") and go deep on the **spatial index** that makes it fast — then note how the two products apply that index to **opposite workloads:**

- **Yelp** — **read-heavy** search over relatively **static** business data (geo + full-text + filters + ranking).
- **Nearby Friends** — **write-heavy** over **ephemeral** live-location data (constant updates, TTL, privacy-sensitive).

Senior signal: recognizing the common geospatial-indexing core, choosing the right index for each workload, and handling the hot-region skew that geographic data always produces.

---

## 2. Requirements (Both Systems)

### Yelp
- Search businesses by name, category, location.
- Filter by rating, distance, open-now.
- Show details, reviews, photos.

### Nearby Friends
- Users share **live location** with friends.
- See friends within N km on a map.
- Notify when a friend is nearby.

### Non-Functional (shared)
- **Low latency** (<200 ms).
- **Massive write rate** for live location (Nearby Friends).
- **High read rate** for search (Yelp).

---

## 3. Why Naive Distance Filtering Fails

The obvious approach:

```sql
SELECT * FROM places
WHERE distance(user_location, place_location) < 5km;
```

This computes the distance from the user to **every single place** — a **full table scan, O(n)** — and can't use a normal index because distance is computed across two dimensions at query time. At millions of places (or millions of constantly-moving users), it's hopelessly slow.

The core need: a **spatial index** that lets you fetch "things near here" by looking only at the *relevant region*, not the whole dataset. The central trick of every technique below is to **map 2D space into something indexable** — either a 1D key (via a space-filling curve) or a hierarchical spatial partition — so a proximity query becomes a cheap bucket/prefix lookup instead of a scan.

---

## 4. Geospatial Indexing Techniques

*(Quick tour of the options; §4A then dissects geohash — the most important one — in full mechanical detail.)*

### 1. Grid-based
Divide the world into fixed cells (e.g., 1 km × 1 km), store places per cell. To query, look up the user's cell **plus its 8 neighbors** (since nearby points may be just over a border).
- *Pros:* simple, intuitive.
- *Cons:* **fixed resolution** — dense cities and empty oceans get the same cell size, so some cells are overloaded and others empty; **border edge cases** (hence checking neighbors).

### 2. Geohash
Encode `(lat, lng)` into a **base32 string** by recursively subdividing the world (interleaving latitude/longitude bits — a **space-filling curve**). The key property: **nearby points share a common prefix.** A longer prefix = a smaller cell = finer precision. So "find things near here" becomes **"find geohashes with the matching prefix"** — turning 2D proximity into **1D prefix matching**, which any sorted index (Redis sorted set, B-tree, trie) handles efficiently. **Dissected step by step in §4A.**
- *Pros:* turns geo search into cheap prefix lookups; trivially indexable and shardable by prefix.
- *Caveat:* at cell **boundaries** two physically-close points can fall in different cells and share **no** prefix. So you still query neighboring geohashes too.

### 3. Quadtree
Recursively partition space into **4 quadrants**, splitting any quadrant that exceeds a bounded number of points. Dense regions subdivide further; sparse regions stay coarse.
- *Pros:* **adaptive to density** — handles non-uniform distributions (cities vs. countryside) far better than a fixed grid.
- *Cons:* more complex; rebalancing as points move (an issue for live location).

### 4. R-Tree
Like a **B-tree for rectangles** — indexes bounding boxes hierarchically, so you can query "what's in this region" by descending the tree. Used by **PostGIS** and **MySQL spatial** extensions.
- *Pros:* mature, supports arbitrary shapes/regions, integrated into relational spatial DBs.

### 5. H3 (Uber's hexagonal grid)
A global grid of **hexagons** at multiple stackable resolutions. Hexagons solve the **"4-corner problem"** of square grids: a square cell has neighbors at *two different distances* (edge-adjacent vs. corner-adjacent), whereas a **hexagon has 6 uniformly-adjacent neighbors** — cleaner, consistent "nearby." Used by **Uber** for rider/driver matching.
- *Pros:* uniform neighbor distances, multi-resolution; great for matching/coverage problems.

---

## 4A. How Geolocation Is Actually Found — Step by Step (the deep dive)

This is the mechanical heart. Follow it once and you can rebuild geospatial search for *any* app (see "Replicating this" at the end). Every number below is verified in code.

### Step 0 — What latitude and longitude actually are

Earth's surface is addressed by two numbers:
- **Latitude** — how far north/south, from **−90°** (south pole) to **+90°** (north pole). The equator is 0.
- **Longitude** — how far east/west, from **−180°** to **+180°**. Greenwich is 0.

So any place is a pair like San Francisco = `(lat 37.7749, lon −122.4194)`. Our whole task: given one such pair, quickly find other pairs that are physically close. The problem (from §3) is that "close" is a 2D calculation, and databases index 1D values well but not 2D distance. **Geohash's job is to turn the 2D pair into a single 1D string that still respects closeness.**

### Step 1 — The core idea: play "higher or lower" on the map

Imagine the whole world as one big rectangle. To pin down a point, you repeatedly cut the rectangle in half and record which half your point is in:

- Cut the world **left/right** (by longitude). Is my point in the right half? Write **1**. Left half? Write **0**. Then keep only that half.
- Cut the remaining box **bottom/top** (by latitude). Top half → **1**, bottom half → **0**. Keep that half.
- Repeat, **alternating longitude and latitude**, each cut halving the area and adding one bit of precision.

Each answer is one **bit** (0 or 1). The more cuts you make, the smaller the surviving box and the more precisely you've located the point. That stream of bits **is** the location, encoded.

> 💡 **In simple words:** Encoding a location is just a game of "is it in this half or that half?" played over and over, alternating between the east-west and north-south directions. Each answer is a 0 or 1.

### Step 2 — The actual algorithm (with a real, verified trace)

Here is the exact procedure for `encode(lat, lon)`:

```
lon_range = [-180, 180];  lat_range = [-90, 90]
alternate = longitude first
repeat:
    if this step is LONGITUDE:
        mid = (lon_low + lon_high) / 2
        if lon >= mid:  bit = 1;  lon_low  = mid     # keep the right half
        else:           bit = 0;  lon_high = mid     # keep the left half
    else (LATITUDE):
        mid = (lat_low + lat_high) / 2
        if lat >= mid:  bit = 1;  lat_low  = mid     # keep the top half
        else:           bit = 0;  lat_high = mid     # keep the bottom half
    append bit
    switch longitude <-> latitude
```

The first 10 cuts for San Francisco `(37.7749, −122.4194)`, verified in code:

```
step 1: LON — is -122.4194 >= 0?        → 0   (western hemisphere)
step 2: LAT — is  37.7749  >= 0?        → 1   (northern hemisphere)
step 3: LON — is -122.4194 >= -90?      → 0
step 4: LAT — is  37.7749  >= 45?       → 0
step 5: LON — is -122.4194 >= -135?     → 1
step 6: LAT — is  37.7749  >= 22.5?     → 1
step 7: LON — is -122.4194 >= -112.5?   → 0
step 8: LAT — is  37.7749  >= 33.75?    → 1
step 9: LON — is -122.4194 >= -123.75?  → 1
step 10:LAT — is  37.7749  >= 39.375?   → 0
```

Notice the bits **interleave**: longitude, latitude, longitude, latitude… That interleaving is what keeps *both* dimensions' closeness in the final string.

### Step 3 — Turn the bits into a short string (base32)

A long run of 0s and 1s is awkward, so every **5 bits** are grouped and turned into **one character** using **base32** (32 symbols because 2⁵ = 32). Geohash uses this alphabet (note: no `a, i, l, o` — they're dropped to avoid confusion):

```
0123456789bcdefghjkmnpqrstuvwxyz
```

So the bitstream becomes a compact string. Verified: `encode(37.7749, −122.4194)` = **`9q8yyk8yt`**. And to prove the algorithm is exactly the industry-standard one, encoding the classic Wikipedia example `(57.64911, 10.40744)` produces **`u4pruydqqvj`** — the exact published geohash. (Both verified in code.)

### Step 4 — Longer string = smaller box = more precise (verified table)

Each extra character adds 5 more bits (≈ 5 more halvings), shrinking the cell. The verified length→size table:

```
 len | cell width | cell height
  1  |   ~5000 km |   ~5000 km      (continent-ish)
  2  |   ~1250 km |    ~625 km
  3  |    ~156 km |    ~156 km      (large region)
  4  |     ~39 km |     ~20 km      (city)
  5  |      ~5 km |      ~5 km      (neighborhood)   ← common for "near me"
  6  |      ~1 km |    ~610 m       (few blocks)
  7  |    ~152 m  |    ~152 m       (street)
  8  |     ~38 m  |     ~19 m       (building)
  9  |      ~5 m  |      ~5 m       (precise)
```

So you pick a geohash **length** to match your search radius: length 5–6 for "within a few km," length 7+ for "within a block."

### Step 5 — The payoff: nearby points share a prefix

Because every character encodes a *shared* sequence of "which half" answers, two points that are close have made the **same early choices**, so their geohashes **start with the same characters**. Verified:

```
SF point A            = 9q8yyk8yt
SF point B (~180 m)   = 9q8yykcvj   → shares first 6 chars with A
Los Angeles (~560 km) = 9q5ctr186   → shares only 2 chars with A
```

This is the whole trick: **"find things near A" becomes "find geohashes that start with `9q8yyk`"** — a plain prefix/range query that any sorted index does instantly (Redis sorted set, a B-tree, even `WHERE geohash LIKE '9q8yyk%'`). No distance math over the whole table; just a prefix lookup on a 1D string. **That is how geolocation is "found" efficiently.**

### Step 6 — The catch: the boundary problem (and why you check neighbors)

There's one honest wrinkle. Two points can be meters apart yet sit on **opposite sides of a cell boundary**, so they *don't* share the expected prefix. Verified:

```
p1 = 9q8yqxp43
p2 (~9 m away) = 9q8yqxp47   (here they still share 8 chars — but near an edge they might not)
```

Near a cell edge, a close point can fall into an adjacent cell entirely. **Fix:** when you query, don't look at only the user's cell — compute the **8 neighboring cells** too (the 3×3 block around the center) and search all nine. This guarantees you never miss a point that's close but just across a border. (This is the same "check neighbors" rule the grid index needs.)

```
   ┌────┬────┬────┐
   │ NW │ N  │ NE │     query the user's cell (C)
   ├────┼────┼────┤     PLUS all 8 neighbors,
   │ W  │ C  │ E  │     so boundary-straddling points
   ├────┼────┼────┤     are never missed
   │ SW │ S  │ SE │
   └────┴────┴────┘
```

### Step 7 — Final ranking: exact distance with haversine

Prefix+neighbors gives you a small **candidate set** (everything in ~9 cells). Now compute the **true distance** to each candidate and sort/filter by it. Straight-line distance on a sphere uses the **haversine formula**:

```
a = sin²(Δlat/2) + cos(lat1)·cos(lat2)·sin²(Δlon/2)
distance = 2 · R · asin(√a)          where R ≈ 6371 km (Earth's radius)
```

Verified: haversine NYC→LA = **3,936 km** (matches the known ~3,936 km); the SF A→B pair = **173 m**. So the pipeline is: **geohash prefix → narrow to a cell + neighbors (cheap) → haversine-rank the handful of candidates (exact).** The index does the coarse filtering; haversine does the fine ordering.

### Step 8 — Putting the whole query together

```
FIND places within R km of (lat, lon):
  1. pick geohash length L so a cell ≈ R across        (e.g., R≈5km → L=5)
  2. g = encode(lat, lon, L)                            (Step 2–3)
  3. cells = { g } ∪ neighbors(g)                       (Step 6, the 3×3 block)
  4. candidates = all places whose geohash starts with any cell prefix   (Step 5)
  5. for each candidate: d = haversine(user, candidate) (Step 7)
  6. keep d <= R, sort by d, return top-K
```

Steps 1–4 are cheap prefix lookups; step 5 runs exact math on only a small candidate set. That's the entire "how geo is found," end to end.

### Replicating this for other systems (the transferable recipe)

The exact same machinery powers almost every location feature — just change what you store in the cells:

- **Cab / ride matching (Uber/Ola):** store **drivers'** live positions by geohash; a rider's request queries their cell + neighbors for nearby drivers, then ranks by haversine (or road ETA). (Uber uses H3 hexagons for the uniform-neighbor benefit, but the principle is identical.)
- **Food delivery:** find restaurants within range (like Yelp) *and* nearby delivery agents (like Nearby Friends) — both are this recipe.
- **"Notify when a friend/driver is nearby" (geofencing):** encode the target area to a geohash prefix; when an incoming location update shares that prefix, it's inside the zone → fire the alert.
- **Store locator / ATM finder:** static points indexed by geohash; pure read-heavy prefix lookup (the Yelp pattern).
- **Ride-share pooling / coverage:** H3 hexagons to reason about adjacent cells uniformly.

The mental template to carry: **encode location → prefix-match the cell + neighbors → haversine-rank.** Swap "places" for drivers, restaurants, friends, scooters, or delivery zones and you've designed the geo layer for any of them.

> 💡 **The senior one-liner for this whole section:** *"You find geolocation by turning the 2D (lat, lon) into a 1D geohash — a string built by repeatedly halving the map and recording 'which half' as bits, interleaving longitude and latitude, then base32-encoding every 5 bits. Nearby points share a prefix, so proximity search is a prefix/range lookup on a sorted index; you also scan the 8 neighbor cells to cover boundary cases, then rank the small candidate set by exact haversine distance. Same recipe — encode, prefix-match + neighbors, haversine-rank — powers Yelp, Nearby Friends, Uber, and food delivery."*

---

## 5. The Unifying Insight

Despite their differences, every technique does the same thing: **avoid scanning everything by mapping 2D space to a structure that supports cheap proximity lookups.** Two broad families:

- **Space-filling curves (geohash)** — collapse 2D into a 1D key whose ordering preserves locality, so proximity = prefix/range match. Simple, indexable, shardable — but with boundary discontinuities (hence neighbor checks, §4A Step 6).
- **Hierarchical spatial partitioning (quadtree, R-tree, H3)** — recursively divide space, so a query descends to the relevant region. Adaptive (quadtree), shape-friendly (R-tree), or uniform-neighbor (H3).

The choice depends on workload: **geohash** for simple, shardable indexing (and what Redis GEO uses internally); **quadtree/H3** for skewed density or matching problems; **R-tree** when you're in a relational spatial DB. All convert an O(n) distance scan into a bucket/prefix/tree lookup — that's the whole game.

---

## 6. Yelp: Read-Heavy Geo Search

Yelp is **read-heavy search over mostly-static business data**, combining geo + full-text + filters + ranking.

```
[Client] ─► [LB] ─► [Search API]
                       ├─► [Elasticsearch]   (full-text + geo)
                       ├─► [Business DB (MySQL)]
                       ├─► [Reviews DB (Cassandra)]
                       ├─► [Photos (S3 + CDN)]
                       └─► [Ranking Service]
```

**Query flow for "pizza near Mumbai":**
1. **Elasticsearch** runs a combined query: `{ text: "pizza", geo_distance: { origin: [lat, lng], distance: 5km } }` — ES supports geo filtering alongside its inverted-index full-text search, so it handles both "matches *pizza*" and "within 5 km" in one query. (Under the hood ES uses a geo index — BKD-tree/geohash — doing exactly the §4A prefiltering before scoring.)
2. Apply **business filters** (open-now, rating).
3. **Rank** by a combined score — relevance + distance (haversine, §4A Step 7) + rating.
4. Return the **top 20.**

Polyglot storage: business metadata in MySQL, reviews in **Cassandra** (write-heavy, partitioned by business), photos in **S3 + CDN**. Because the data is relatively static and reads dominate, you **cache** heavily and can precompute popular searches.

---

## 7. Nearby Friends: Write-Heavy Live Location

Nearby Friends is the inverse workload: a **flood of location writes** over **ephemeral** data.

```
[Friend A's phone] ─(location update every ~30s)─► [Location API]
                                                       │
                                                       ▼
                                          [Geospatial index: Redis GEO]
                                                       │
[Friend B's app] ─(query nearby friends)─◄────────────┘
                       │
                       ▼
              [Returns A: distance, last_seen]
```

### Why Redis GEO fits
Redis has **built-in geospatial commands** (`GEOADD` to store a point, `GEOSEARCH`/`GEORADIUS` to query within a radius — backed internally by the exact geohash-in-a-sorted-set mechanism from §4A):
- **Sub-millisecond latency** — essential under a constant stream of updates and queries.
- **TTL / auto-expiry** — stale locations expire automatically, so a friend who goes offline simply drops off (no manual cleanup). This matches the **ephemeral, best-effort** nature of live location: losing one update is harmless (the next 30s update replaces it), so you don't need durability — exactly what an in-memory store is good at.

The contrast with Yelp is the senior point: **same geospatial index, opposite workload** — Yelp needs durable, indexed, cached static data; Nearby Friends needs a fast, ephemeral, TTL'd in-memory geo store.

### Privacy (live location is sensitive)
Location is sensitive PII, so privacy is a first-class requirement:
- **Opt-in per friend** — sharing is explicit and granular.
- **Approximate location** — round to ~100 m so exact whereabouts aren't exposed (equivalently, store a shorter/coarser geohash — §4A Step 4).
- **Auto-disable after inactivity** — sharing turns off automatically when unused (and TTL drops stale data).

---

## 8. Scaling & Hot Regions

- **Shard by geohash prefix** — a city's data lands on one shard (geohash prefixes group nearby points, §4A Step 5), giving **locality**: a query for "near Mumbai" hits one shard, not all. This is the consistent-hashing/partitioning principle applied geographically.
- **Hot-region skew (the recurring problem)** — dense areas (Manhattan, downtown Mumbai) generate vastly more traffic than empty regions, so their shards become **hot partitions**. Mitigations: **replicate hot regions read-only** (spread the read load), **edge-cache common queries** (CDN-like), and **precompute popular searches**.
- **Cache top-queried cells** — the most-searched geohash cells stay hot in cache.

The geographic angle on an old theme: sharding by location gives locality but *concentrates* load on dense-area shards, so you replicate and cache exactly those hot regions.

---

## 9. Senior Follow-Up Questions (with Answers)

**Q1. What's the common core of Yelp and Nearby Friends?**
Geospatial search — find things within a radius of a location, efficiently. Both need a spatial index. They differ in workload: Yelp is read-heavy over static business data; Nearby Friends is write-heavy over ephemeral live location.

**Q2. Why can't you just filter by distance in SQL?**
`WHERE distance(...) < 5km` computes distance to every row — a full O(n) scan that can't use a normal index. You need a spatial index that maps 2D space to something queryable so you only examine the relevant region.

**Q3. Explain in detail how a geohash is computed.**
Convert (lat, lon) to bits by repeatedly halving the map: for longitude, is the point in the right half (bit 1) or left (bit 0); for latitude, top (1) or bottom (0); alternate lon/lat each step, each halving the surviving box. Group every 5 bits into one base32 character. The result is a string where nearby points share a prefix. Verified: (37.7749, −122.4194) → `9q8yyk8yt`; the standard example (57.64911, 10.40744) → `u4pruydqqvj`. (Full trace in §4A.)

**Q4. Why does sharing a prefix mean two points are near each other?**
Each geohash character encodes a shared sequence of "which half" decisions. If two points share the first 6 characters, they made the same first 30 halvings, so they sit in the same small cell (~1 km at length 6). More shared characters = smaller shared cell = closer. (Caveat: boundary cases, Q5.)

**Q5. What's the boundary problem and how do you fix it?**
Two points meters apart can fall on opposite sides of a cell edge and thus share little/no prefix. Fix: query the user's cell **plus its 8 neighbors** (the 3×3 block), so a point just across a boundary is still captured. Then rank the candidates by exact distance.

**Q6. How do you get the exact distance for ranking?**
Haversine: `a = sin²(Δlat/2) + cos(lat1)cos(lat2)sin²(Δlon/2); d = 2R·asin(√a)`, R≈6371 km. The geohash prefilters to a small candidate set; haversine gives the true distance to sort/filter by. Verified NYC→LA = 3,936 km.

**Q7. How do you choose the geohash length?**
Match it to your search radius: length 5 ≈ 5 km cells (neighborhood), length 6 ≈ 1 km, length 7 ≈ 150 m (street). Longer = smaller cell = more precise. Pick the length whose cell is roughly your radius, then add neighbors.

**Q8. Compare the indexing techniques.**
Grid: simple but fixed resolution (uneven load). Geohash: 1D prefix lookups, shardable, boundary caveats. Quadtree: adaptive to density. R-tree: B-tree for rectangles (PostGIS/MySQL spatial). H3: hexagons with uniform neighbor distances, multi-resolution (Uber). Two families: space-filling curves vs. hierarchical partitioning.

**Q9. Why hexagons (H3) over squares?**
Square cells have neighbors at two different distances (edge vs. corner). Hexagons have 6 uniformly-adjacent neighbors, giving consistent "nearby" semantics — useful for matching/coverage (Uber).

**Q10. Why Redis GEO for Nearby Friends, and same-index/opposite-workload?**
Redis GEO (GEOADD/GEOSEARCH, geohash-backed) gives sub-ms latency and TTL auto-expiry — ideal for ephemeral, best-effort live location (a lost update is replaced in ~30s; no durability needed). Yelp uses the same geohash idea but over durable static data with ES + caching + precomputed searches. Same index, opposite durability/consistency needs.

**Q11. How do you scale geospatial queries and handle hot regions?**
Shard by geohash prefix for locality (a city on one shard). Dense regions become hot shards, so replicate them read-only, edge-cache common queries, and cache/precompute popular cells.

**Q12. How do you protect location privacy?**
Opt-in per friend, approximate location (round to ~100 m / store a coarser geohash), and auto-disable after inactivity (plus TTL dropping stale data). Location is sensitive PII, so sharing is explicit, coarse-grained, and self-expiring.

---

## 10. Quick Glossary

- **Latitude / longitude** — north-south (−90..+90) and east-west (−180..+180) coordinates of a point.
- **Geospatial search** — finding entities within a distance of a location.
- **Spatial index** — a structure enabling proximity queries without scanning all data.
- **Space-filling curve** — a 2D→1D mapping preserving locality (geohash basis).
- **Geohash** — base32 encoding of coordinates (built by repeated halving + bit interleaving) where nearby points share a prefix.
- **Bit interleaving** — alternating longitude and latitude bits so both dimensions' locality is preserved in one string.
- **Base32** — the 32-symbol alphabet (`0-9`, `b-z` minus `a,i,l,o`) that turns every 5 geohash bits into one character.
- **Prefix match** — the query "share the same leading characters" used to find nearby geohashes.
- **Neighbor cells (3×3)** — the user's cell plus its 8 surroundings, queried to cover boundary cases.
- **Haversine** — the great-circle distance formula used to rank candidates exactly.
- **Grid index** — fixed-cell partition; query a cell plus its neighbors.
- **Quadtree** — recursive 4-way partition that adapts to point density.
- **R-tree** — a B-tree-like index for rectangles/bounding boxes (PostGIS, MySQL).
- **H3** — Uber's hexagonal multi-resolution grid (uniform neighbor distances).
- **Redis GEO** — Redis geospatial commands (GEOADD/GEOSEARCH) over a geohash-backed sorted set.
- **TTL auto-expiry** — stale locations dropping automatically (ephemeral live data).
- **Geohash-prefix sharding** — partitioning so a region's data stays on one shard (locality).
- **Hot region** — a dense area generating skewed load (a geographic hot partition).
- **geo_distance filter** — Elasticsearch's radius filter combined with full-text search.

---

*Reference document. Contributions and corrections welcome.*
