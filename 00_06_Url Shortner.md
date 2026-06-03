# Designing a URL Shortener (bit.ly / TinyURL): A Senior Interview Guide

> A practical, interview-focused reference for designing a URL shortening service — requirements, capacity math, the key-generation trade-offs, architecture, caching, redirect semantics, analytics, abuse prevention, and a deep bank of senior-level follow-up questions with model answers.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (Show Your Work)](#3-capacity-estimation-show-your-work)
4. [The Core Problem: Generating Short Keys](#4-the-core-problem-generating-short-keys)
5. [Architecture](#5-architecture)
6. [Data Model & Storage Choice](#6-data-model--storage-choice)
7. [The Redirect Path & 301 vs 302](#7-the-redirect-path--301-vs-302)
8. [Caching Strategy](#8-caching-strategy)
9. [Analytics Pipeline](#9-analytics-pipeline)
10. [Abuse, Security & Edge Cases](#10-abuse-security--edge-cases)
11. [Senior-Level Follow-Up Questions (with Answers)](#11-senior-level-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This in an Interview

URL shortening looks trivial ("it's just a hash map") but is a favorite precisely because a *senior* candidate is expected to surface the non-obvious depth: the key-generation trade-off, the read-heavy caching story, redirect semantics vs. analytics, and abuse prevention. The trap is jumping straight to a solution. The structure interviewers reward:

1. **Clarify requirements & scope** — pin down what's in and out before designing.
2. **Estimate scale** — derive QPS and storage from stated assumptions; this drives every later decision.
3. **Define the API & data model** — the contract first.
4. **Design the happy path** — write (shorten) and read (redirect).
5. **Identify the bottleneck and go deep** — for this problem it's almost always *read latency at scale* and *unique key generation*. Spend your time here.
6. **Address cross-cutting concerns** — availability, abuse, analytics, expiration.

Senior signal = you proactively name trade-offs ("301 saves server load but blinds analytics — here's how I'd decide") rather than waiting to be asked. The rest of this guide is organized to let you do exactly that.

---

## 2. Requirements

### Functional

- **Shorten:** given a long URL, return a short URL.
- **Redirect:** given a short URL, redirect to the original long URL.
- **Custom alias:** let users optionally choose their own short key (e.g., `/my-brand`).
- **Analytics:** track clicks (count, and ideally geo/referrer/time).
- **Expiration:** URLs may have an optional expiry (default + custom).

### Non-Functional

- **Read-heavy:** ~**100:1 read:write** ratio. This is the single most important fact — it makes the system a *redirect-serving* system far more than a *URL-storing* one, and it justifies aggressive caching.
- **High availability:** a shortener that's down breaks every link anyone has ever shared — links are permanent and embedded everywhere, so downtime is unusually damaging.
- **Low latency:** redirects should feel instant (**< 100ms**); a redirect sits in the critical path of loading someone else's page.
- **Not easily guessable** (for unlisted links) — keys shouldn't be trivially enumerable.

> **Clarifying questions to ask the interviewer:** How long are URLs retained (forever? expiry)? Do we need custom aliases? What analytics granularity? Is link-guessing a security concern? These shape the design and asking them is itself a senior signal.

---

## 3. Capacity Estimation (Show Your Work)

Doing the arithmetic out loud is expected. Starting assumption: **100M new URLs per month.**

**Write QPS:**
```
100,000,000 / (30 days × 24 h × 3600 s)
  = 100,000,000 / 2,592,000
  ≈ 39  ≈ ~40 writes/sec   (call peak ~2–3× → ~100/sec)
```

**Read QPS** (100:1 ratio):
```
40 × 100 = ~4,000 reads/sec   (peak ~10,000/sec)
```

**Storage** (retain 5 years):
```
100M/month × 12 months × 5 years = 6,000,000,000 URLs
6B × ~500 bytes/record ≈ 3,000,000,000,000 bytes ≈ ~3 TB
```

**Key takeaways from the math:**
- 3 TB is **modest** — it fits comfortably and isn't the hard part. *Storage is not the bottleneck.*
- ~4K (peak ~10K) reads/sec **is** the thing to engineer for — hence caching is central.
- Writes at ~40/sec are tiny, so the write path can be relatively simple; the read path cannot.

**Key space sanity check:** base62 with 7 characters = `62⁷ ≈ 3.5 trillion` combinations. At 6 billion URLs over 5 years we use < 0.2% of the space, so **7 characters is plenty** (6 chars = ~56B, also enough; 7 gives generous headroom).

---

## 4. The Core Problem: Generating Short Keys

This is the heart of the design and where senior depth shows. Every short URL is a unique key; the question is *how to generate unique keys at scale without collisions.* Three approaches, with honest trade-offs.

### Option 1 — Hash-based

```
short_key = base62(md5(long_url))[:7]
```

Hash the long URL, encode it, take the first 7 characters.

- **Pros:** stateless, simple; the *same* URL maps to the *same* key (free deduplication, if you want it).
- **Cons:**
  - **Collisions** — truncating a hash to 7 chars means different URLs can collide; you must detect and resolve (re-hash with a salt, or probe), which adds read-before-write complexity.
  - **No control over the key space** — you can't cleanly support "I want a *different* short URL for the same long URL" without extra logic.
- **Verdict:** workable but the collision handling makes it less clean than alternatives.

### Option 2 — Counter + Base62

Maintain a **global monotonic counter**; each new URL gets the next integer, which you **base62-encode** into the short key.

- **Pros:** **zero collisions by construction** (every counter value is unique), keys are short and grow predictably, encoding is trivial.
- **Cons / the real challenge:** a single global counter is a **bottleneck and a single point of failure**, and coordinating it across many app servers is the hard part. Also, sequential counters make keys **enumerable** (guessable) — a privacy concern (mitigations below).

**How to make the distributed counter work** (this is the senior depth):

- **Redis `INCR`** — atomic increment from a central Redis. Simple, fast, but Redis becomes a critical dependency (needs HA/failover).
- **Range allocation (the strong answer):** each app server requests a **block** of the counter range (e.g., 10,000 IDs) from a coordinator (ZooKeeper / a counter service), then hands out IDs from its local block *without any network round trip*. When the block runs low, it grabs the next block. This removes the per-request coordination bottleneck almost entirely and tolerates coordinator hiccups (servers keep serving from their local block). The minor cost: small gaps in the sequence if a server dies with IDs unused — which is harmless.
- **ZooKeeper** — can hand out ranges and coordinate which blocks are taken.

### Option 3 — Key Generation Service (KGS)

**Pre-generate** random unique keys offline and store them in a database with two states (`unused` / `used`). App servers pull a **batch** of unused keys, mark them used, and hand them out.

- **Pros:** **no collisions** (keys are pre-validated unique), **fast** at request time (just pop a key), and keys can be **random** (not enumerable) — fixing the counter's guessability problem.
- **Cons:** extra service to build and operate; the KGS DB and its concurrency (two servers must not grab the same key) need care; you must monitor the **remaining-key supply** and replenish before it runs out.
- **Concurrency detail:** mark keys used *atomically* when handed to a server (transaction or atomic flag), and have each server keep keys in memory and in a "given out but unconfirmed" state to avoid double-issuing if it restarts.

### Which to choose?

For a senior answer: **counter + base62 with range allocation** is the clean default (collision-free, scalable, simple to reason about), and you add **randomization** (or switch to **KGS**) if non-enumerable keys are a hard requirement. Lead with the trade-off, not a single dogmatic pick.

---

## 5. Architecture

```
                                   ┌──────────────► [ Cache (Redis) ] ─┐
                                   │  (read path)                      │
[Client] ──► [Load Balancer] ──► [App Server] ──► [Key Gen Service]    │
                                   │                                   ▼
                                   └──────────────────────────► [ URL Database ]
                                                                       │
                          (clicks) ──► [ Kafka ] ──► [ Analytics workers ] ──► [ Analytics store ]
```

- **Load balancer** spreads requests across stateless app servers.
- **App servers** are stateless (any server handles any request) so they scale horizontally — write requests call the **Key Gen Service**; read requests hit the **cache first**, then the **DB**.
- **Key Gen Service** issues unique keys (range-allocated counter or KGS).
- **Cache (Redis)** absorbs the read-heavy traffic — the linchpin of the < 100ms redirect goal.
- **URL DB** is the source of truth for short_key → long_url.
- **Analytics** is decoupled via a queue so click logging never slows the redirect.

---

## 6. Data Model & Storage Choice

### Schema

```
url_mappings
---------------------------------------------
  short_key    VARCHAR(7)   PRIMARY KEY
  long_url     TEXT
  user_id      BIGINT
  created_at   TIMESTAMP
  expires_at   TIMESTAMP NULL
  click_count  BIGINT          -- see note
```

### SQL vs NoSQL — the trade-off

The access pattern is almost entirely **key-value lookup by `short_key`** with no joins and no complex queries. That profile fits a **NoSQL key-value/wide-column store** (DynamoDB, Cassandra) very well: it scales horizontally, partitions cleanly by `short_key`, and gives fast point lookups — and it pairs naturally with **consistent hashing** for distributing keys across nodes.

A SQL database is perfectly fine at the 3 TB / 4K-QPS scale too, and gives you easy secondary indexes (e.g., list a user's URLs) and transactions. A reasonable senior answer: *"point lookups by key don't need a relational model, so a key-value store scales most naturally; but at this scale SQL also works, and I'd pick based on the team's operational familiarity and whether we need rich queries on the side."*

> **`click_count` caveat:** don't increment a counter column on the hot redirect path — that's a write on every read and will not scale. Either keep counts in the analytics pipeline (Stage 9) or in a separate counter store updated asynchronously.

---

## 7. The Redirect Path & 301 vs 302

**Flow for `GET /aBc123`:**

1. Look up `aBc123` in the **cache**.
2. **Hit** → return redirect immediately (the common case — read-heavy + caching).
3. **Miss** → read from **DB**, populate the cache, then return the redirect.
4. **Not found / expired** → return `404` (or a custom page).

**301 (Permanent) vs 302 (Found / Temporary)** — a classic senior trade-off:

| | 301 Permanent | 302 Temporary |
|---|---|---|
| Browser behavior | **Caches** the redirect; future clicks skip your server | Comes back to your server **every click** |
| Server load | Lower (fewer hits) | Higher (every click hits you) |
| Analytics | **Breaks** — cached clicks are invisible to you | **Accurate** — you see every click |
| Changing the target later | Hard (browsers cached the old target) | Easy (you control every redirect) |

**The decision:** if analytics and the ability to change/expire targets matter (they usually do for a product like bit.ly), use **302** — you *want* every click to reach you. If raw redirect performance and offloading your servers matter most and you don't need per-click data, **301** is cheaper. Most commercial shorteners use **302** precisely to retain analytics and control.

---

## 8. Caching Strategy

Caching is what makes the read-heavy workload tractable.

- **Pattern: cache-aside.** App checks Redis; on miss, reads DB and writes the result back to the cache.
- **What to cache:** the `short_key → long_url` mapping. These are immutable once created (a short key's target rarely changes), so cached entries stay valid for a long time — ideal for caching.
- **Eviction:** **LRU** works well because access follows a heavy **power-law** — a small set of "hot" links (a viral post, a campaign) gets the vast majority of clicks, so the working set that matters is small and stays hot in cache.
- **Hit ratio:** with that skew you can expect a very high cache hit rate, so most redirects never touch the database.
- **Invalidation:** mostly a non-issue since mappings are immutable; the cases to handle are **expiry** and **deletion** — evict (or check expiry on read) so expired links don't keep redirecting.
- **Memory sizing:** you don't cache all 6B URLs — you cache the hot subset. Even caching, say, the hottest 20% comfortably fits and covers the overwhelming majority of traffic.

---

## 9. Analytics Pipeline

Analytics must **never slow down the redirect.** So make it **asynchronous**:

1. On each click, the app emits a **click event** (short_key, timestamp, referrer, geo/IP, user-agent) to a **message queue / log (Kafka)** — a fast, fire-and-forget write.
2. **Consumer workers** read the stream and **aggregate** (per-link counts, time series, geo breakdowns), writing rolled-up results to an analytics store (a columnar/OLAP DB or a time-series store).
3. The redirect path itself does *zero* synchronous analytics work.

This decoupling is also why you should **not** do a synchronous `UPDATE click_count` on redirect — the queue handles counting off the critical path, and it naturally buffers traffic spikes.

> Note this only works with **302** redirects — if you use 301, browsers cache the redirect and many clicks never reach you, so your analytics undercount. Another reason product shorteners lean 302.

---

## 10. Abuse, Security & Edge Cases

A senior candidate is expected to raise these unprompted — shorteners are a known abuse vector.

- **Malicious links / phishing.** Shorteners hide the destination, so they're attractive for phishing and malware distribution. Mitigations: scan submitted URLs against threat-intelligence/safe-browsing blocklists, optionally show an interstitial preview page, and support takedown of flagged links.
- **Open-redirect & SSRF hygiene.** Validate and normalize submitted URLs; only allow `http`/`https` schemes (block `javascript:`, `data:`, etc.).
- **Enumeration / guessing.** Sequential counter keys are guessable, exposing private links. Mitigate by randomizing keys (KGS or a reversible permutation over the counter) so adjacent links aren't adjacent keys.
- **Rate limiting.** Throttle creation per user/IP to stop bulk-spam generation; rate-limit redirects per source to blunt abuse.
- **Custom alias collisions.** A user-chosen alias must be checked for uniqueness atomically (it shares the key namespace with generated keys); reject or suggest alternatives on conflict, and reserve a separate namespace or prefix if you want to avoid clashes with the generated space.
- **Expiration & cleanup.** Expired links should stop redirecting (check `expires_at` on read and/or a background sweeper that purges and frees keys). Decide whether expired keys can be **recycled** (adds complexity — usually not worth it given the huge key space).
- **Hot-key thundering herd.** When a viral link's cache entry expires/misses, many simultaneous requests can stampede the DB. Mitigate with request coalescing / "cache stampede" protection (single-flight) or longer TTLs for hot keys.

---

## 11. Senior-Level Follow-Up Questions (with Answers)

**Q1. How do you generate unique keys across many servers without a central bottleneck?**
Use **range allocation**: each app server pulls a block of counter values (e.g., 10K) from a coordinator and serves IDs locally, refilling when low. This removes per-request coordination, tolerates brief coordinator unavailability, and only costs small harmless gaps if a server dies. Alternatives: central `Redis INCR` (simpler, but a hot dependency) or a **KGS** pre-generating random keys (also collision-free and non-enumerable).

**Q2. The counter makes keys guessable — how do you prevent enumeration of private links?**
Decouple the *public key* from the *sequential id*: either generate **random** keys via a KGS, or apply a **reversible permutation / encryption** (e.g., Feistel/`encrypt(counter)`) before base62-encoding, so consecutive ids map to scattered keys. You keep collision-free generation while making keys unguessable.

**Q3. Should redirects be 301 or 302, and why?**
**302** for a product that needs analytics and the ability to change/expire targets — every click returns to your server. **301** if you prioritize offloading server load and don't need per-click data, since browsers cache it. Commercial shorteners almost always pick 302 to keep analytics and control.

**Q4. How do you keep redirects under 100ms at peak?**
Serve from an **in-memory cache** (Redis) on the read path; the workload's power-law skew gives a very high hit rate so most redirects skip the DB. Keep app servers stateless behind a load balancer, push static/edge concerns to a CDN/anycast, and protect against cache stampedes for hot keys.

**Q5. SQL or NoSQL for the store?**
The access pattern is pure point-lookup by key with no joins — a natural fit for a **key-value/wide-column NoSQL** store that partitions by `short_key` (with consistent hashing) and scales horizontally. SQL is also viable at this scale and offers easy secondary indexes/transactions; choose based on query needs and operational familiarity. Either way, **never** increment a click counter on the read path.

**Q6. How do you scale to multiple regions, and what breaks?**
Put read replicas/caches in each region and route users to the nearest (geo-DNS / anycast). Reads are easy to serve locally. The hard part is **key generation and write consistency across regions** — give each region its own disjoint counter range (or a region-prefixed key space) so two regions never mint the same key without cross-region coordination. This is the **CAP trade-off** in practice: during a cross-region partition you favor availability and reconcile, since minting independent keys is naturally conflict-free if ranges are disjoint.

**Q7. A link goes viral — how does the system cope?**
The cache absorbs it (one hot key, served from memory). Risks: a **cache stampede** when that hot entry expires — mitigate with single-flight/request coalescing and longer TTLs for hot keys; and **analytics burst** — the Kafka buffer levels the spike so workers drain it without backpressure on the redirect path.

**Q8. How do you handle custom aliases without collisions?**
Custom aliases share the key namespace, so reserve/check them **atomically** (unique constraint or conditional write). On conflict, reject and suggest alternatives. To avoid clashing with generated keys entirely, you can give custom aliases a distinct namespace/prefix or a reserved character class.

**Q9. What happens when the KGS runs low on keys?**
Monitor the **unused-key count** and replenish proactively (a background generator tops up the pool below a threshold). Size batches so a server outage doesn't strand many keys, and mark keys used atomically on hand-off so a restart can't reissue them.

**Q10. How do you delete/expire links and reclaim space?**
Store `expires_at`; check it on read (and/or run a background sweeper) so expired links return 404 and are purged. Given the enormous key space (3.5T for 7 chars vs. 6B used), **recycling keys is usually not worth the complexity** — let them retire.

**Q11. How would you deduplicate identical long URLs?**
Optional. With the **hash approach** dedup is automatic (same URL → same key). With a counter you'd add a lookup (e.g., index on a hash of the long URL) to return an existing key — but note many users *want* distinct short URLs (separate analytics per campaign) for the same destination, so dedup should usually be **opt-in**, not default.

**Q12. What are the main single points of failure and how do you remove them?**
The **key generator** (mitigate with range allocation so servers run off local blocks even if the coordinator blips), the **cache** (run Redis in HA/cluster mode; a miss falls through to the DB, just slower), and the **database** (replicas + sharding/partitioning by key). Stateless app servers behind a load balancer remove the app tier as a SPOF.

---

## 12. Quick Glossary

- **Short key** — the compact identifier in the short URL (e.g., `aBc123`).
- **Base62** — encoding using `[a-z A-Z 0-9]` (62 symbols); 7 chars ≈ 3.5 trillion values.
- **Read:write ratio** — here ~100:1; the system is overwhelmingly read (redirect) traffic.
- **Distributed counter** — a globally unique, monotonically increasing source of IDs across servers.
- **Range allocation** — handing each server a block of IDs to serve locally, avoiding per-request coordination.
- **KGS (Key Generation Service)** — pre-generates unique keys offline; servers consume them.
- **Cache-aside** — check cache, fall back to DB on miss, then populate the cache.
- **301 / 302** — permanent (browser-cached) vs. temporary (server-hit-every-time) redirect.
- **Cache stampede / thundering herd** — many concurrent misses on the same hot key overwhelming the DB.
- **Enumeration** — guessing valid keys by walking sequential values; a privacy risk for sequential counters.
- **Open redirect / SSRF** — abuse where a redirect target is crafted to attack users or internal systems.

---

*Reference document. Contributions and corrections welcome.*
