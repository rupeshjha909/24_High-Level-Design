# Designing a URL Shortener (bit.ly / TinyURL): A Senior Interview Guide

> A practical, interview-focused reference for designing a URL shortening service — requirements, capacity math, the key-generation trade-offs (explained in depth), architecture, caching, redirect semantics, analytics, abuse prevention, and a deep bank of senior-level follow-up questions with model answers.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (Show Your Work)](#3-capacity-estimation-show-your-work)
4. [The Core Problem: Generating Short Keys (In Depth)](#4-the-core-problem-generating-short-keys-in-depth)
   - [4.1 What problem are we even solving?](#41-what-problem-are-we-even-solving)
   - [4.2 Prerequisite: what "base62" actually means](#42-prerequisite-what-base62-actually-means)
   - [4.3 Algorithm 1 — Hash-based](#43-algorithm-1--hash-based)
   - [4.4 Algorithm 2 — Counter + Base62](#44-algorithm-2--counter--base62)
   - [4.5 Algorithm 3 — Key Generation Service (KGS)](#45-algorithm-3--key-generation-service-kgs)
   - [4.6 Side-by-side comparison](#46-side-by-side-comparison)
   - [4.7 How to explain this in an interview (a script)](#47-how-to-explain-this-in-an-interview-a-script)
   - [4.8 Which to choose](#48-which-to-choose)
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

## 4. The Core Problem: Generating Short Keys (In Depth)

This is the heart of the design and where senior depth shows. The sections below build the idea from scratch — what we're solving, the base62 prerequisite, the three algorithms with worked examples, a comparison, and exactly how to *explain* it in an interview.

### 4.1 What problem are we even solving?

When someone gives us a long URL, we must hand back something short like `bit.ly/aBc123`. That `aBc123` is the **short key**. The entire job of "key generation" is:

> **Produce a short string that no other URL is already using — every single time, even when many servers are doing it at once.**

Two requirements pull against each other:

- **Uniqueness** — if two different long URLs ever get the *same* short key, then `bit.ly/aBc123` is ambiguous and one of them is broken. This must *never* happen. A clash is called a **collision**.
- **Scale & speed** — we generate ~40 keys/sec (peak ~100/sec) across *many* app servers, and each must be fast. We can't have every server waiting in line for one slow "give me the next key" service.

The three algorithms are three different bargains on "unique + fast + distributed." Keep "no collisions, and don't create a bottleneck" in mind and each approach is just a different way to satisfy those two points.

### 4.2 Prerequisite: what "base62" actually means

Two of the three algorithms use base62, so be comfortable with it first.

You already know **base 10** (10 digits, `0–9`) and **base 2** (2 digits, `0`/`1`). **Base62** is the same idea with **62 "digits"**:

```
0 1 2 3 4 5 6 7 8 9   (the 10 normal digits)
a b c ... z           (26 lowercase letters)
A B C ... Z           (26 uppercase letters)
= 10 + 26 + 26 = 62 symbols
```

Converting a normal number to base62 works like converting to any base — **repeatedly divide by 62 and read the remainders**. Worked example for **125**:

```
125 ÷ 62 = 2 remainder 1   → least-significant "digit" = symbol #1 = '1'
  2 ÷ 62 = 0 remainder 2   → next "digit"             = symbol #2 = '2'
read remainders bottom-to-top → "21"
```

So `125` in base62 is `"21"`. A few more (verified):

| number | base62 |
|:-------|:-------|
| 0 | `0` |
| 10 | `a` |
| 61 | `Z` |
| 62 | `10` |
| 125 | `21` |
| 1,000,000,000 | `15FTGg` |

**Why bother?** Base62 packs a big number into very few characters:

```
62^6 ≈  56.8 billion   keys with 6 characters
62^7 ≈   3.5 trillion  keys with 7 characters
```

We expect ~6 billion URLs over 5 years — only **0.17%** of the 7-character space. That's why short URLs are 6–7 chars: base62 makes a 7-character string represent trillions of distinct values.

> 💡 **The senior point about base62:** it's not magic — it's just "write the number in a 62-symbol alphabet so it's short and URL-safe" (only letters/digits, unlike base64's `+` and `/`, which would break URLs). Whenever you see base62, translate it to "a compact, URL-safe way to write a big integer."

### 4.3 Algorithm 1 — Hash-based

**The idea:** hash the long URL (e.g., MD5), base62-encode it, take the first 7 characters.

```
long_url   = "https://example.com/some/long/path"
md5(...)   = "9e107d9d372bb6826bd81d3542a419d6"   (128-bit hash, hex)
base62(…)  = "kQ8fW2aZ9mL..."                     (re-encode that hash in base62)
short_key  = "kQ8fW2a"                            (first 7 characters)
```

A hash is **deterministic** (same input → same output) but *looks* random.

**Pros:**
- **Stateless** — no counter, no central service; any server computes a key alone with zero coordination.
- **Free deduplication** — the same URL always hashes to the same key, so identical URLs collapse to one short link (if you want that).

**The problem — collisions:** a full MD5 is 128 bits (essentially unique), but we **chop it to 7 characters** to keep the URL short. The moment you discard most of the hash, *different* URLs can produce the *same* 7-char key — a collision, which is fatal (two URLs sharing one link). By a **birthday-paradox** argument, across billions of inserts this *will* happen even though it's rare per-insert.

**Resolving collisions (and why it's awkward):** you must detect and fix them:
1. Compute the candidate key.
2. **Check the DB**: does this key already map to a *different* URL?
3. If yes, **change something** and retry — e.g., salt and re-hash (`md5(url + "1")`, `"2"`, …) or probe to the next free key.

This forces a **read-before-write on every insert** and turns the write path into a retry loop. And the "free dedup" property fights you when you *want* two distinct short URLs for the same destination (e.g., two campaigns) — the hash gives them the same key, needing extra logic to force them apart.

> **Verdict:** simple and stateless, but truncation collisions + read-before-write + retries make the "simple" approach quietly not simple.

### 4.4 Algorithm 2 — Counter + Base62

**The idea:** keep one ever-increasing number (a **counter**); each new URL gets the next number, base62-encoded.

```
URL #1  → counter = 1              → base62(1)          = "1"        → bit.ly/1
URL #2  → counter = 2              → base62(2)          = "2"        → bit.ly/2
URL #62 → counter = 62             → base62(62)         = "10"       → bit.ly/10
...      → counter = 1,000,000,000 → base62(…)          = "15FTGg"   → bit.ly/15FTGg
```

**The killer feature — zero collisions by construction:** every counter value is unique, so every key is unique. There's *nothing to check* — no read-before-write, no retry loop. Keys also stay short (6–7 chars for billions of URLs).

**Problem A — the single counter is a bottleneck and SPOF.** If there's literally one counter, every app server must ask it "what's next?" for every create. That central thing becomes a throughput bottleneck (network round-trip per write) and a single point of failure (down → nobody can create).

**The fix — range allocation (understand this cold):** instead of asking for **one** number per request, each server asks for a **block** (say 10,000) at once, then hands out keys from that block **locally, with no network call**, refilling when low.

```
Coordinator hands out ranges:
  Server A  ← gets range [1 .. 10,000]
  Server B  ← gets range [10,001 .. 20,000]

They now work INDEPENDENTLY — no per-request coordination:
  Server A: uses 1, 2, 3, ...        (all local, instant)
  Server B: uses 10,001, 10,002, ... (all local, instant)

When Server A nears 10,000, it asks the coordinator ONCE for a new block:
  Server A  ← gets range [20,001 .. 30,000]
```

Why it's great:
- **Bottleneck nearly vanishes** — coordinate once per 10,000 keys instead of once per key (10,000× less).
- **Tolerates coordinator hiccups** — a brief coordinator outage is invisible; servers keep serving from local blocks.
- **Only cost is harmless gaps** — if a server dies holding `[1..10,000]` after using 1–500, the rest are simply never used. With trillions of keys, skipping a few thousand is irrelevant.

The coordinator is typically **Redis** (`INCR` by the block size) or **ZooKeeper**. The mantra: **coordinate rarely (per block), serve instantly (per request).**

**Problem B — keys are guessable (enumeration).** Sequential keys (`1, 2, 3, … 10, 11`) let anyone count up and visit `bit.ly/1`, `bit.ly/2`, … discovering everyone's links, including private ones.

**The fix — permute before encoding:** apply a **reversible permutation** (a small Feistel cipher, or `encrypt(counter)` with a fixed key) to the counter *before* base62. Because the permutation is a bijection (one-to-one), you keep collision-free uniqueness, but `1, 2, 3` now map to scattered keys like `9Xk2`, `Bf7`, `q2Lp` — no longer walkable.

> **Verdict:** collision-free by construction (its killer feature); the bottleneck is solved by **range allocation**, the guessability by **permuting the counter**. The clean default.

### 4.5 Algorithm 3 — Key Generation Service (KGS)

**The idea:** **ahead of time** (offline), generate a pile of random, unique keys and store them; at request time, servers just **grab a pre-made key**.

```
Offline, the KGS pre-generates millions of random 7-char keys with a status flag:

  key        status
  --------   --------
  9Xk2mPq    unused
  Bf7aL0z    unused
  q2Lp8Wd    unused

At request time:
  1. Server pulls a BATCH (say 1,000) of unused keys.
  2. KGS marks those 1,000 'used' atomically so no one else gets them.
  3. Server holds them in memory, hands out one per new URL — instant, no per-request call.
  4. When low, server pulls another batch.
```

**Pros:**
- **No collisions** — keys were pre-validated unique, so hand-off is conflict-free (nothing to check).
- **Fast at request time** — a create is just "pop a key from my in-memory batch."
- **Random, not sequential** — *directly fixes guessability* without needing the permutation trick.

So KGS gives the counter's collision-free + fast benefits **and** non-enumerable keys in one design.

**Costs (name these unprompted):**
- **Another service** to build, operate, and monitor.
- **Atomic hand-off** — marking keys `used` must be atomic (transaction/atomic flag) so two servers never grab the same key, or you'd reintroduce collisions.
- **Crash loss** — if a server pulls 1,000 keys then crashes, those are marked used but never assigned (stranded, like the counter's gaps). Acceptable; small batches limit the loss.
- **Supply monitoring** — a background generator must top up the pool before it runs dry, with an alert if "unused key count" drops low. Run out → creates stop.

> **Verdict:** pre-bakes random unique keys so request-time is just "pop a key" — collision-free *and* unguessable at once. Price: an extra service with careful atomic hand-off and supply monitoring.

### 4.6 Side-by-side comparison

| | **Hash-based** | **Counter + base62** | **KGS** |
|:--|:--|:--|:--|
| Collisions? | **Yes** — truncation collides; detect & retry | **None** (each counter value unique) | **None** (pre-validated unique) |
| Needs coordination? | No (stateless) | Yes, but **rarely** (range allocation) | Yes, to pull batches (also rare) |
| Request-time speed | Slower (read-before-write) | Fast (local block) | Fast (pop from memory) |
| Keys guessable? | No (hash looks random) | **Yes** (sequential) unless permuted | No (random by design) |
| Dedup same URL? | **Automatic** | No (needs extra lookup) | No |
| Main downside | Collision handling | Bottleneck (→ ranges) + guessability (→ permute) | Extra service + atomic hand-off + supply monitoring |

The throughline: **hashing trades collision-pain for statelessness; the counter trades coordination for collision-freedom (and you engineer the coordination away with ranges); KGS trades an extra service for getting collision-freedom AND unguessable keys together.**

### 4.7 How to explain this in an interview (a script)

**Opening frame (say first):**
> "The core challenge isn't storing URLs — 3 TB is small. It's generating a unique short key on every write, across many servers, without collisions or a central bottleneck. Three approaches trade off differently on *collisions*, *coordination*, and *guessability*."

**Hash-based:**
> "Hash the long URL, take the first 7 base62 chars. Stateless, and dedupes identical URLs for free. But truncating the hash causes collisions — inevitable at billions of URLs by a birthday-paradox argument — so I'd need read-before-write to detect them and re-hash-with-salt to resolve. That collision handling is why it's not my default."

**Counter + base62:**
> "A global counter, each URL gets the next integer, base62-encoded. Huge advantage: collision-free by construction — nothing to check. The objection is the single counter as bottleneck and SPOF, which I'd solve with **range allocation** — each server pulls a block of ~10,000 IDs and serves them locally, touching the coordinator once per block. That cuts coordination 10,000× and tolerates coordinator blips, costing only harmless gaps if a server dies. The remaining issue is guessable sequential keys, which I'd fix by running the counter through a reversible permutation before encoding."

**KGS:**
> "If unguessable keys are a hard requirement, pre-generate random unique keys offline in a Key Generation Service; servers pull a batch, mark them used atomically, and hand them out from memory. Collision-free *and* non-enumerable in one design. The cost is an extra service, airtight atomic hand-off so two servers never grab the same key, and monitoring the unused-key supply."

**Closing recommendation:**
> "I'd default to **counter + base62 with range allocation** — collision-free and scales cleanly — and add the permutation, or switch to KGS, if unguessable keys are required. I'm leading with the trade-off rather than a single dogmatic pick."

> 💡 **What makes this a senior answer:** you (1) name the *real* problem up front (unique keys at scale, not storage), (2) give each approach an honest pro *and* con, (3) show you can *fix* the cons (range allocation, permutation, atomic hand-off), and (4) recommend while acknowledging the trade-off. Reciting only "use a counter" is mid-level; *reasoning through why* is senior.

### 4.8 Which to choose

For a senior answer: **counter + base62 with range allocation** is the clean default (collision-free, scalable, easy to reason about); add **randomization via a permutation** (or switch to **KGS**) if non-enumerable keys are a hard requirement; reach for **hashing** only if you specifically want automatic dedup of identical URLs and accept the collision-handling cost. Lead with the trade-off, not a dogmatic pick.

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
- **Base62** — encoding using `[a-z A-Z 0-9]` (62 symbols); 7 chars ≈ 3.5 trillion values; a compact, URL-safe way to write a big integer.
- **Read:write ratio** — here ~100:1; the system is overwhelmingly read (redirect) traffic.
- **Collision** — two different long URLs producing the same short key; must never happen.
- **Distributed counter** — a globally unique, monotonically increasing source of IDs across servers.
- **Range allocation** — handing each server a block of IDs to serve locally, avoiding per-request coordination.
- **Reversible permutation** — a bijective scramble of the counter (e.g., Feistel) so sequential ids map to non-guessable keys while staying collision-free.
- **KGS (Key Generation Service)** — pre-generates unique random keys offline; servers consume them in batches.
- **Cache-aside** — check cache, fall back to DB on miss, then populate the cache.
- **301 / 302** — permanent (browser-cached) vs. temporary (server-hit-every-time) redirect.
- **Cache stampede / thundering herd** — many concurrent misses on the same hot key overwhelming the DB.
- **Enumeration** — guessing valid keys by walking sequential values; a privacy risk for sequential counters.
- **Open redirect / SSRF** — abuse where a redirect target is crafted to attack users or internal systems.

---

*Reference document. Contributions and corrections welcome.*
