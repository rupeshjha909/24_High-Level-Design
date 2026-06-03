# Caching — Invalidation & Eviction: A Ground-Up Guide

> A practical reference for caching in distributed systems — why it works, the layered cache hierarchy, read vs write patterns, the genuinely hard problem of invalidation, eviction policies, and the four classic failure modes (stampede, hot key, avalanche, penetration) with their fixes. With trade-offs and interview prep.

---

## Table of Contents

1. [Why Caching Works](#1-why-caching-works)
2. [The Cache Hierarchy](#2-the-cache-hierarchy)
3. [Read Patterns vs Write Patterns](#3-read-patterns-vs-write-patterns)
4. [The Hard Problem: Invalidation](#4-the-hard-problem-invalidation)
5. [The Cache-Aside Race Condition](#5-the-cache-aside-race-condition)
6. [Eviction Policies](#6-eviction-policies)
7. [The Four Classic Failure Modes](#7-the-four-classic-failure-modes)
8. [Cache Sizing](#8-cache-sizing)
9. [Tools](#9-tools)
10. [Interview-Ready Insights](#10-interview-ready-insights)
11. [Quick Glossary](#11-quick-glossary)

---

## 1. Why Caching Works

A cache stores copies of data in a faster/closer location so future requests are served without redoing expensive work. The three motivations:

- **Reduce latency** — serve from fast storage instead of a slow source.
- **Reduce load on the origin** — fewer requests reach the database/backend.
- **Reduce cost** — less compute and fewer expensive operations.

The reason caching is so universally effective comes straight from the latency hierarchy (estimation guide): **reading from memory is ~100,000× faster than a cross-continent round trip and far faster than disk or a database query.** Caching exploits that enormous gap by keeping hot data in fast storage. The second enabler is **locality of reference** — real workloads are skewed (a small fraction of data gets the majority of requests — a power law), so caching a small "hot set" serves a large share of traffic. Those two facts — a huge speed gap and skewed access — are why caching works almost everywhere.

The cost is the theme of this guide: now you have **two copies of the truth** (cache and origin), and keeping them coherent is hard.

---

## 2. The Cache Hierarchy

Caches stack in layers, each catching requests before they reach the next (slower, more central) layer:

1. **Browser cache** (client-side) — assets cached on the user's device; zero network cost on a hit.
2. **CDN** (edge) — static content cached near users (see the CDN guide).
3. **Application cache** (local in-memory) — per-server in-process cache; fastest server-side, but not shared across servers.
4. **Distributed cache** (Redis, Memcached) — a shared cache cluster all app servers consult; the workhorse layer.
5. **Database query cache** (built-in caches, materialized views) — caching at the data layer itself.

A request ideally dies at the highest layer that can serve it. A common combination is a small **local cache** (L1, ultra-fast, absorbs the very hottest keys) backed by a **distributed cache** (L2, shared, larger) — a near/far hierarchy mirroring CPU caches.

---

## 3. Read Patterns vs Write Patterns

A key clarifying distinction: **read patterns** govern how data gets *into* the cache on reads; **write patterns** govern what happens to the cache on *writes*. They're largely **orthogonal** — you pick one of each and combine them.

### Read pattern: Cache-Aside (Lazy Loading) — the most common

```
read(key):
    v = cache.get(key)
    if v is None:                 # cache miss
        v = db.get(key)
        cache.set(key, v)         # populate for next time
    return v
```

The application owns the logic: check cache, fall back to DB on miss, populate. Simple and resilient (a cache outage just means more DB load, not failure). The downside is the first read of each key is always a miss (cold start). *(A close cousin, "read-through," pushes this logic into the cache library instead of the app.)*

### Write patterns

- **Write-Through** — every write goes to **cache and DB synchronously**. *Pro:* cache is always fresh. *Con:* added write latency (two writes on the critical path).
- **Write-Back (Write-Behind)** — write to **cache immediately**, persist to DB **asynchronously**. *Pro:* very fast writes, absorbs write bursts. *Con:* **data-loss risk** if the cache crashes before flushing — acceptable for tolerant data (metrics, counters), risky for critical data.
- **Write-Around** — write **straight to the DB, skip the cache**; the cache fills only on subsequent reads. *Good for* write-heavy, rarely-read data (don't waste cache space on data nobody reads back soon).
- **Refresh-Ahead** — proactively **refresh hot keys before their TTL expires**, so popular data never goes cold and you avoid the latency spike of a miss on a hot key.

**Combining them:** cache-aside + write-around is a frequent pairing (lazy population, writes bypass cache); write-through + read-through is another (always-fresh, library-managed). The choice depends on your read/write ratio and freshness needs.

---

## 4. The Hard Problem: Invalidation

"There are only two hard things in computer science: cache invalidation and naming things." Invalidation is hard because the cache and origin are **two copies of the same truth that can diverge** — the moment the origin changes, the cached copy is stale, and reconciling them across a distributed system is fundamentally a **consistency problem** (the same family as the replication consistency in the CAP guide). Four strategies, trading freshness against complexity:

### 1. TTL (Time-To-Live)

Each entry expires after N seconds, then is re-fetched. **Simple and self-healing**, but data can be **stale for up to the TTL**. The classic trade-off: long TTL = higher hit rate but staler; short TTL = fresher but more origin load. The right TTL depends on how much staleness the data tolerates.

### 2. Explicit Invalidation

On a write, **delete or update the cached key** (`cache.delete(key)`). Precise and immediate, but **race-prone** in a distributed cache (see Section 5) — concurrent reads and writes can interleave to leave a stale value behind.

### 3. Write-Through

Keep cache and DB in sync by writing both together (Section 3). Eliminates staleness for written keys at the cost of write latency.

### 4. Event-Driven (CDC — Change Data Capture)

The database **publishes change events**; a separate invalidator **subscribes and invalidates** affected cache keys. A common pipeline is **Debezium → Kafka → cache invalidator** (tying to the message-queue guide). This **decouples** invalidation from the write path and scales well, but adds infrastructure and a small propagation delay. It's the most robust approach for large systems where many services write the underlying data.

**Guidance:** TTL is the default baseline (always set one, even with other strategies, as a safety net). Add explicit or event-driven invalidation when staleness windows must be short. Write-through when freshness is paramount and write latency is acceptable.

---

## 5. The Cache-Aside Race Condition

A senior-level subtlety worth knowing cold, because it's the most common way cache-aside silently serves stale data. Consider concurrent operations:

```
Thread A (read)                    Thread B (write)
─────────────────                  ─────────────────
read miss for key
read v1 from DB
                                   write v2 to DB
                                   delete key from cache
set cache[key] = v1   ← STALE!     (cache now holds v1, but DB has v2)
```

Thread A read the *old* value, then — after B updated the DB and cleared the cache — wrote the **stale** value back. The cache now disagrees with the DB until the next invalidation.

Mitigations:
- **Delete-on-write, not update-on-write** — on a write, *delete* the key (let the next read repopulate) rather than writing the new value into the cache; this narrows the window (and avoids writing stale values during interleavings).
- **Versioning / CAS** — store a version with the value and only set the cache if the version is still current.
- **Short TTL as a backstop** — even if a race leaves a stale value, it self-corrects within the TTL.
- **Accept bounded staleness** — for many systems, a brief stale window is fine; don't over-engineer.

The lesson: cache-aside is simple but **not automatically consistent**; recognizing this race (and using delete-on-write + TTL) is the practical answer.

---

## 6. Eviction Policies

When the cache is full, something must be evicted. The policy determines *what*:

| Policy | Evicts | Best for |
|--------|--------|----------|
| **LRU** (Least Recently Used) | the item unused for longest | general-purpose default — assumes recently-used = soon-reused |
| **LFU** (Least Frequently Used) | the item accessed least often | stable workloads with persistent hot items |
| **FIFO** | the oldest-inserted item | simple; ignores access patterns |
| **Random** | a random item | surprisingly decent for some workloads; cheap |
| **TTL-based** | expired items first | time-bound data |
| **W-TinyLFU** | hybrid (LFU frequency + LRU recency windows) | modern best-general-purpose (Caffeine) |

**The intuition:** LRU bets on *recency* (good when access is bursty/temporal), LFU bets on *frequency* (good when some items are durably popular but maybe not recent). Each has a failure mode — LRU evicts a frequently-used item after one quiet stretch; LFU clings to once-popular items forever. **W-TinyLFU** combines both (a frequency sketch plus recency windows) and outperforms either alone on most real workloads, which is why modern caches like **Caffeine** use it.

---

## 7. The Four Classic Failure Modes

These are interview gold and frequently confused — the key is that they're *distinct* problems with distinct fixes.

### Cache Stampede / Thundering Herd

A **hot key expires or misses**, and **many concurrent requests** all miss simultaneously and hammer the DB to rebuild the same value at once — a spike that can overwhelm the origin (the problem flagged in the URL-shortener and autocomplete guides).
- **Fixes:** **request coalescing / single-flight** (one request fetches; the rest wait for its result), **jittered TTLs** (don't expire hot keys on a sharp boundary), and **probabilistic early refresh** (refresh slightly before expiry so the key never actually goes cold).

### Hot Key

A **single key receives a huge share of traffic** (e.g., a celebrity's profile), overloading the one cache node that owns it.
- **Fixes:** **replicate the hot key to multiple nodes** and spread reads across them; add a **local in-process cache** in front of the distributed cache for the hottest keys; client-side fan-out.

### Cache Avalanche

**Many keys expire at the same moment** (e.g., everything cached at startup with the same TTL), causing a broad simultaneous miss and a DB-wide spike.
- **Fix:** **jittered TTLs** — add randomness to expiry times so keys expire spread out, not all at once.

### Cache Penetration

Requests for **keys that don't exist anywhere** (often malicious) always miss the cache and fall through to the DB every time, since there's nothing to cache.
- **Fixes:** **cache the negative result** (store "not found" with a short TTL so repeats are absorbed), or put a **Bloom filter** in front (see the Bloom-filter guide) — "definitely not in the DB" lets you reject the lookup instantly without touching the DB.

> Distinguishing them clearly: **stampede** = one hot key, many concurrent rebuilds; **hot key** = one key, sustained overload of one node; **avalanche** = many keys expiring together; **penetration** = lookups for nonexistent keys bypassing the cache. (Jitter fixes stampede and avalanche; Bloom filters/negative caching fix penetration; replication fixes hot keys.)

---

## 8. Cache Sizing

You don't cache everything — you cache the **working set** (roughly, the data actively accessed in a recent window, e.g., the last 24 h). Because access follows a power law, a relatively small cache covers most traffic.

- **Target an 80%+ hit rate** — the hit rate is the cache's defining metric; each point higher means fewer origin trips.
- **Right-size by trade-off** — more cache memory costs money but reduces DB load; size where the marginal memory cost is worth the marginal load reduction. Past a point, doubling cache size barely moves the hit rate (diminishing returns), so there's a sweet spot.

This is the same skew that made caching effective in Section 1: a small hot set serves the bulk of requests, so you size for the hot set, not the whole dataset.

---

## 9. Tools

- **Redis** — in-memory key-value store with rich data structures, replication, persistence, and atomic operations; the most common distributed cache (and more — see the KV-store and rate-limiter guides).
- **Memcached** — simpler, multi-threaded key-value cache; lean and fast for plain caching.
- **Caffeine** (Java) — high-performance in-process cache using **W-TinyLFU**.
- **Hazelcast / Apache Ignite** — distributed in-memory data grids (caching plus compute/clustering).

---

## 10. Interview-Ready Insights

**Q: Why does caching work so well?**
The latency gap (memory is orders of magnitude faster than disk/DB/network) combined with skewed access (a small hot set serves most requests). Cache the hot set in fast storage and you serve the majority of traffic cheaply.

**Q: Cache-aside vs write-through vs write-back?**
Cache-aside (read pattern): app checks cache, falls back to DB on miss, populates — simple and resilient. Write-through: write cache+DB synchronously — always fresh, slower writes. Write-back: write cache now, DB async — fast writes, data-loss risk. Write-around: write DB only, cache on read — good for write-heavy rarely-read data. Read and write patterns are orthogonal; you combine one of each.

**Q: Why is cache invalidation hard?**
Because you have two copies of the truth (cache and origin) that diverge the moment the origin changes — it's fundamentally a distributed consistency problem. Strategies trade freshness vs complexity: TTL (simple, stale up to N s), explicit delete (precise, race-prone), write-through (always sync, slower), event-driven/CDC (decoupled, scalable, more infra).

**Q: Describe the cache-aside race condition.**
A reader gets a miss and reads v1 from the DB; meanwhile a writer sets v2 and clears the cache; then the reader writes v1 back — leaving a stale cache. Mitigate with delete-on-write (not update), versioning/CAS, and a short TTL backstop.

**Q: Stampede vs avalanche vs penetration vs hot key?**
Stampede: one hot key misses, many concurrent rebuilds hit the DB (fix: single-flight, jitter, early refresh). Avalanche: many keys expire together (fix: jittered TTLs). Penetration: lookups for nonexistent keys bypass the cache (fix: negative caching or a Bloom filter). Hot key: one key overloads one node (fix: replicate it / local cache). They're distinct problems with distinct fixes.

**Q: How do you pick an eviction policy?**
LRU (recency) is the general default; LFU (frequency) suits stable hot items; W-TinyLFU combines both and is the modern best-general-purpose choice (Caffeine). Match the policy to whether recency or frequency better predicts reuse in your workload.

**Q: How do you size a cache?**
Cache the working set (recently-accessed data), target 80%+ hit rate, and right-size by balancing memory cost against DB-load reduction — past a point, more cache barely improves the hit rate (diminishing returns).

---

## 11. Quick Glossary

- **Cache** — fast/near storage of data copies to avoid repeating expensive work.
- **Hit / miss** — request served from cache vs. requiring an origin fetch.
- **Hit rate** — fraction of requests served from cache; the key cache metric.
- **Working set** — the data actively accessed in a recent window; what you size the cache for.
- **Cache-aside (lazy loading)** — app checks cache, falls back to origin on miss, then populates.
- **Read-through** — like cache-aside but the cache library handles the fallback.
- **Write-through / write-back / write-around** — write to cache+DB sync / cache-then-async / DB-only.
- **Refresh-ahead** — proactively refresh hot keys before TTL expiry.
- **TTL** — time-to-live; how long a cached entry stays valid.
- **Invalidation** — removing/updating stale cache entries when the origin changes.
- **CDC (Change Data Capture)** — streaming DB change events (e.g., Debezium) to drive invalidation.
- **Eviction policy** — what to remove when the cache is full (LRU, LFU, FIFO, W-TinyLFU, …).
- **Cache stampede / thundering herd** — many concurrent misses rebuilding the same hot key.
- **Hot key** — a single key with disproportionate traffic overloading one node.
- **Cache avalanche** — many keys expiring simultaneously.
- **Cache penetration** — repeated lookups for nonexistent keys bypassing the cache.
- **Single-flight / request coalescing** — one fetch serves many waiting callers on a miss.
- **Jittered TTL** — randomized expiry times to avoid synchronized expirations.

---

*Reference document. Contributions and corrections welcome.*
