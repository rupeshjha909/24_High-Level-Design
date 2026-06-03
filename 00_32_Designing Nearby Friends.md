# Designing Nearby Friends / Yelp (Geospatial Search): A Senior Interview Guide

> A practical, interview-focused reference for geospatial systems — two products (Yelp's business search and Nearby Friends' live location) that share one core problem: *given my location, find X within Y km* efficiently. Covers why naive distance filtering fails, the spatial-indexing techniques (geohash, quadtree, R-tree, H3, Redis GEO), and how the same index serves opposite workloads. With trade-offs and a senior follow-up bank.

---

## Table of Contents

1. [The Common Core: Geospatial Search](#1-the-common-core-geospatial-search)
2. [Requirements (Both Systems)](#2-requirements-both-systems)
3. [Why Naive Distance Filtering Fails](#3-why-naive-distance-filtering-fails)
4. [Geospatial Indexing Techniques](#4-geospatial-indexing-techniques)
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

### 1. Grid-based

Divide the world into fixed cells (e.g., 1 km × 1 km), store places per cell. To query, look up the user's cell **plus its 8 neighbors** (since nearby points may be just over a border).

- *Pros:* simple, intuitive.
- *Cons:* **fixed resolution** — dense cities and empty oceans get the same cell size, so some cells are overloaded and others empty (uneven load); **border edge cases** (hence checking neighbors).

### 2. Geohash

Encode `(lat, lng)` into a **base32 string** by recursively subdividing the world (interleaving latitude/longitude bits — a **space-filling curve**). The key property: **nearby points share a common prefix.** A longer prefix = a smaller cell = finer precision (e.g., `"dr5ru"` ≈ a ~4.9 km × 4.9 km cell). So "find things near here" becomes **"find geohashes with the matching prefix"** — turning 2D proximity into **1D prefix matching**, which any sorted index (Redis sorted set, B-tree, trie) handles efficiently.

- *Pros:* turns geo search into cheap prefix lookups; trivially indexable and shardable by prefix.
- *Caveat:* the converse isn't perfect — at cell **boundaries** two physically-close points can fall in different cells and share **no** prefix (space-filling curves have discontinuities). So you still query neighboring geohashes too.

### 3. Quadtree

Recursively partition space into **4 quadrants**, splitting any quadrant that exceeds a bounded number of points. Dense regions subdivide further; sparse regions stay coarse.

- *Pros:* **adaptive to density** — handles non-uniform distributions (cities vs. countryside) far better than a fixed grid.
- *Cons:* more complex; rebalancing as points move (an issue for live location).

### 4. R-Tree

Like a **B-tree for rectangles** — indexes bounding boxes hierarchically, so you can query "what's in this region" by descending the tree. Used by **PostGIS** and **MySQL spatial** extensions.

- *Pros:* mature, supports arbitrary shapes/regions, integrated into relational spatial DBs.

### 5. H3 (Uber's hexagonal grid)

A global grid of **hexagons** at multiple stackable resolutions. Hexagons solve the **"4-corner problem"** of square grids: a square cell has neighbors at *two different distances* (edge-adjacent vs. corner-adjacent), which distorts radius/adjacency logic, whereas a **hexagon has 6 uniformly-adjacent neighbors** — cleaner, consistent "nearby." Used by **Uber** for rider/driver matching.

- *Pros:* uniform neighbor distances, multi-resolution; great for matching/coverage problems.

---

## 5. The Unifying Insight

Despite their differences, every technique does the same thing: **avoid scanning everything by mapping 2D space to a structure that supports cheap proximity lookups.** Two broad families:

- **Space-filling curves (geohash)** — collapse 2D into a 1D key whose ordering preserves locality, so proximity = prefix/range match. Simple, indexable, shardable — but with boundary discontinuities.
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

1. **Elasticsearch** runs a combined query: `{ text: "pizza", geo_distance: { origin: [lat, lng], distance: 5km } }` — ES supports geo filtering alongside its inverted-index full-text search (the search engine from prior guides), so it handles both "matches *pizza*" and "within 5 km" in one query.
2. Apply **business filters** (open-now, rating).
3. **Rank** by a combined score — relevance + distance + rating.
4. Return the **top 20.**

Polyglot storage: business metadata in MySQL, reviews in **Cassandra** (write-heavy, partitioned by business), photos in **S3 + CDN** (the media pattern from prior designs). Because the data is relatively static and reads dominate, you **cache** heavily and can precompute popular searches.

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

Redis has **built-in geospatial commands** (`GEOADD` to store a point, `GEORADIUS` to query within a radius — backed internally by geohash in a sorted set):

- **Sub-millisecond latency** — essential under a constant stream of updates and queries.
- **TTL / auto-expiry** — stale locations expire automatically, so a friend who goes offline simply drops off (no manual cleanup). This matches the **ephemeral, best-effort** nature of live location: losing one update is harmless (the next 30s update replaces it), so you don't need durability — exactly what an in-memory store is good at.

The contrast with Yelp is the senior point: **same geospatial index, opposite workload** — Yelp needs durable, indexed, cached static data; Nearby Friends needs a fast, ephemeral, TTL'd in-memory geo store.

### Privacy (live location is sensitive)

Location is sensitive PII, so privacy is a first-class requirement:
- **Opt-in per friend** — sharing is explicit and granular.
- **Approximate location** — round to ~100 m so exact whereabouts aren't exposed.
- **Auto-disable after inactivity** — sharing turns off automatically when unused (and TTL drops stale data).

---

## 8. Scaling & Hot Regions

- **Shard by geohash prefix** — a city's data lands on one shard (geohash prefixes group nearby points), giving **locality**: a query for "near Mumbai" hits one shard, not all. This is the consistent-hashing/partitioning principle applied geographically.
- **Hot-region skew (the recurring problem)** — dense areas (Manhattan, downtown Mumbai) generate vastly more traffic than empty regions, so their shards become **hot partitions** (the hot-key theme from the caching and sharding guides). Mitigations: **replicate hot regions read-only** (spread the read load), **edge-cache common queries** (CDN-like, from the CDN guide), and **precompute popular searches**.
- **Cache top-queried cells** — the most-searched geohash cells stay hot in cache.

The geographic angle on an old theme: sharding by location gives locality but *concentrates* load on dense-area shards, so you replicate and cache exactly those hot regions.

---

## 9. Senior Follow-Up Questions (with Answers)

**Q1. What's the common core of Yelp and Nearby Friends?**
Geospatial search — find things within a radius of a location, efficiently. Both need a spatial index. They differ in workload: Yelp is read-heavy over static business data; Nearby Friends is write-heavy over ephemeral live location.

**Q2. Why can't you just filter by distance in SQL?**
`WHERE distance(...) < 5km` computes distance to every row — a full O(n) scan that can't use a normal index. You need a spatial index that maps 2D space to something queryable so you only examine the relevant region, not everything.

**Q3. How does geohash work and why is it useful?**
It encodes (lat, lng) into a base32 string via recursive subdivision (a space-filling curve), so nearby points share a common prefix. Proximity search becomes prefix matching — cheap and indexable in a sorted set/B-tree, and naturally shardable by prefix. Caveat: at cell boundaries close points can differ in prefix, so you also query neighbors.

**Q4. Compare the indexing techniques.**
Grid: simple but fixed resolution (uneven load). Geohash: 1D prefix lookups, shardable, boundary caveats. Quadtree: adaptive to density. R-tree: B-tree for rectangles (PostGIS/MySQL spatial). H3: hexagons with uniform neighbor distances (no 4-corner problem), multi-resolution (Uber matching). Two families: space-filling curves vs. hierarchical partitioning.

**Q5. Why hexagons (H3) over squares?**
Square cells have neighbors at two different distances (edge vs. corner adjacency), distorting radius/adjacency logic. Hexagons have 6 uniformly-adjacent neighbors, giving consistent "nearby" semantics — useful for matching/coverage (Uber).

**Q6. Why Redis GEO for Nearby Friends?**
Built-in geospatial commands (GEOADD/GEORADIUS, geohash-backed) with sub-millisecond latency and TTL auto-expiry. Live location is ephemeral and best-effort — losing an update is harmless since the next replaces it in ~30s — so an in-memory, TTL'd store is ideal; no durability needed.

**Q7. Same index, opposite workloads — explain.**
Both use geospatial indexing, but Yelp is read-heavy over durable static data (ES + caching + precomputed popular searches), while Nearby Friends is write-heavy over ephemeral data (Redis GEO + TTL). The durability and consistency needs differ accordingly.

**Q8. How does Yelp combine text and geo search?**
Elasticsearch runs a single query combining full-text (inverted index) with a geo_distance filter, then applies business filters (open-now, rating) and ranks by relevance + distance + rating. Reviews live in Cassandra, photos in S3/CDN.

**Q9. How do you scale geospatial queries and handle hot regions?**
Shard by geohash prefix for locality (a city on one shard). Dense regions become hot shards, so replicate them read-only, edge-cache common queries (CDN-like), and cache/precompute popular cells. Geographic sharding gives locality but concentrates load on dense areas — replicate exactly those.

**Q10. How do you protect location privacy?**
Opt-in per friend, approximate location (round to ~100 m), and auto-disable after inactivity (plus TTL dropping stale data). Location is sensitive PII, so sharing is explicit, coarse-grained, and self-expiring.

---

## 10. Quick Glossary

- **Geospatial search** — finding entities within a distance of a location.
- **Spatial index** — a structure enabling proximity queries without scanning all data.
- **Space-filling curve** — a 2D→1D mapping preserving locality (geohash basis).
- **Geohash** — base32 encoding of coordinates where nearby points share a prefix.
- **Grid index** — fixed-cell partition; query a cell plus its neighbors.
- **Quadtree** — recursive 4-way partition that adapts to point density.
- **R-tree** — a B-tree-like index for rectangles/bounding boxes (PostGIS, MySQL).
- **H3** — Uber's hexagonal multi-resolution grid (uniform neighbor distances).
- **Redis GEO** — Redis geospatial commands (GEOADD/GEORADIUS) over a geohash-backed sorted set.
- **GEORADIUS** — query points within a radius of a coordinate.
- **TTL auto-expiry** — stale locations dropping automatically (ephemeral live data).
- **Geohash-prefix sharding** — partitioning so a region's data stays on one shard (locality).
- **Hot region** — a dense area generating skewed load (a geographic hot partition).
- **geo_distance filter** — Elasticsearch's radius filter combined with full-text search.

---

*Reference document. Contributions and corrections welcome.*
