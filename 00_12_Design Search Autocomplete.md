# Designing Search Autocomplete / Typeahead: A Senior Interview Guide

> A practical, interview-focused reference for designing a typeahead suggestion system — why naive prefix search fails, the trie-with-cached-top-K approach, the crucial split between the data-gathering pipeline and the serving layer, sharding, freshness trade-offs, and real-world implementations. With trade-offs and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Why This Is Hard: The Keystroke Multiplier](#3-why-this-is-hard-the-keystroke-multiplier)
4. [The Naive Approach (and Why It Dies)](#4-the-naive-approach-and-why-it-dies)
5. [The Key Insight: Precompute, Don't Compute](#5-the-key-insight-precompute-dont-compute)
6. [The Trie-Based Serving Layer](#6-the-trie-based-serving-layer)
7. [The Build & Update Pipeline](#7-the-build--update-pipeline)
8. [Architecture End-to-End](#8-architecture-end-to-end)
9. [Sharding & Scaling](#9-sharding--scaling)
10. [Freshness vs Cost](#10-freshness-vs-cost)
11. [Personalization & Real-World Implementations](#11-personalization--real-world-implementations)
12. [Senior Follow-Up Questions (with Answers)](#12-senior-follow-up-questions-with-answers)
13. [Quick Glossary](#13-quick-glossary)

---

## 1. How to Approach This in an Interview

Typeahead looks simple ("just match a prefix") but is a favorite because the scale and latency constraints force a non-obvious architecture. The single most important realization — and the thing that separates a strong answer — is that this system has **two completely different halves with opposite requirements**, and you must design them separately:

1. **The serving path** — ultra-low-latency (<100 ms), extreme read volume, runs on *every keystroke*. This must be as close to "look it up" as physically possible.
2. **The data-gathering / ranking path** — figures out *which* suggestions are good (by popularity, freshness, etc.). This is heavy, batch/streaming, and runs offline, far from the request path.

The structure to follow: clarify requirements → note the keystroke-multiplier scale → reject the naive query → arrive at **precomputation** as the core idea → design the trie serving layer → design the offline build pipeline → cover sharding, freshness, and personalization.

Senior signal: separating "gather & rank data" from "serve suggestions," and recognizing that the serving path can't afford to *compute* rankings at request time — it must serve **precomputed** answers.

---

## 2. Requirements

### Functional

- As the user types, return the **top-K matching suggestions** for the current prefix.
- **Personalize** by history, location, language where applicable.
- **Reflect fresh data** — newly-popular terms should appear within minutes.

### Non-Functional

- **< 100 ms latency** — suggestions must appear faster than the user types the next character, or they're useless.
- **Very high throughput** — every keystroke can be a query, so QPS is enormous.
- **Freshness within minutes** for trending terms (but *not* instant — see freshness trade-off).

> Note the asymmetry: the system is **read-dominated to an extreme degree** and **tolerant of slightly stale rankings.** Those two facts together justify the entire precompute-and-cache design.

---

## 3. Why This Is Hard: The Keystroke Multiplier

The defining scale property: **autocomplete fires on every keystroke**, so its query volume is a *multiple* of actual search volume. If a search service handles S searches/sec and users type ~K characters before selecting, the autocomplete layer sees on the order of **K × S queries/sec** — often making typeahead the **highest-QPS service in the entire search stack.**

```
1 completed search ≈ several keystroke-queries
  → autocomplete QPS = (chars typed) × (search QPS)
  → typeahead is frequently the busiest service you run
```

Two consequences:

- **Debouncing** is standard — the client waits a few tens of milliseconds of typing pause before firing, collapsing bursts of keystrokes into fewer queries. This is the cheapest, first optimization.
- Even with debouncing, the volume is huge, so per-query work must be **minimal** — which rules out any heavy computation or database scan at request time.

---

## 4. The Naive Approach (and Why It Dies)

The obvious first idea:

```sql
SELECT term FROM terms WHERE term LIKE 'prefix%' ORDER BY frequency DESC LIMIT K;
```

This runs a prefix scan and ranking **on every keystroke**. It collapses at scale because:

- `LIKE 'prefix%'` scans potentially huge numbers of rows; even with an index it's far too slow for a 100 ms budget at high QPS.
- Sorting by frequency at query time is repeated, wasteful work — the same popular prefixes are ranked over and over.
- The database becomes a massive bottleneck under the keystroke multiplier.

The lesson that points to the real design: **you cannot afford to search and rank at request time.** The answer must already be computed and waiting.

---

## 5. The Key Insight: Precompute, Don't Compute

The central design move: **separate gathering/ranking the data (offline, expensive, occasional) from serving it (online, cheap, constant).**

- **Offline:** periodically crunch the firehose of user queries to compute, for *every prefix*, its **top-K suggestions**, and store that mapping in a fast structure.
- **Online:** at request time, the serving layer does essentially a **lookup** — "give me the precomputed top-K for this prefix" — with **no ranking computation at all.**

This is the same philosophy as caching and the read-model in CQRS: do the expensive work once, ahead of time, so the hot path is trivial. It works *because* autocomplete tolerates slightly stale rankings — you don't need the top-K to reflect the last second of traffic, so you can afford to compute it on a schedule.

---

## 6. The Trie-Based Serving Layer

A **trie** (prefix tree) is the natural structure: each path from the root spells a prefix, so walking the tree character-by-character is exactly what typeahead needs.

```
        (root)
        /   \
       c     a ...
      /       \
     a         p
    /           \
  cat (3K)     app (1K)     ← each node caches the TOP-K terms for its prefix
```

The crucial optimization: **store the precomputed top-K suggestions at each node** (not just leaves). Then a lookup is:

1. Walk from the root following the prefix characters — O(length of prefix), which is tiny.
2. At the node for that prefix, **return the cached top-K directly** — no scanning of descendants, no ranking.

So the read is "traverse a handful of nodes, return a cached list" — easily within 100 ms. The trie lives **in RAM** for speed, and because the top-K is precomputed, serving does zero ranking work.

The trade-off: caching top-K at every node uses more memory and makes updates more expensive (you must recompute affected nodes' top-K when frequencies change) — which is exactly why updates are **batched offline** rather than applied per-write.

---

## 7. The Build & Update Pipeline

This is the offline half — turning raw user queries into the ranked structure the serving layer uses.

```
[user queries] → [Kafka] → [Aggregation job (Flink/Spark)] → [Trie/Index Builder] → [Serving layer]
                 (stream)   (count over rolling 24h window)    (compute top-K)        (RAM, swapped in)
```

1. **Stream every query into Kafka** — a durable, high-throughput log of what people searched (decoupling collection from processing, as in the scaling guide's queue stage).
2. **Aggregate** with a stream/batch processor (Flink, Spark): count term frequencies over a **rolling window** (e.g., trailing 24 h) so rankings reflect recent popularity, not all-time.
3. **Build** the new trie/index with fresh top-K per prefix.
4. **Swap it into the serving layer** — rebuild popular terms roughly every ~hour (and trending terms more often), atomically replacing the in-RAM structure.

Because rebuilding is periodic and offline, the heavy ranking work never touches the latency-critical serving path.

---

## 8. Architecture End-to-End

```
                          ┌──────────────────────────────────────┐
[Client keystroke] ──► [API] ──► [Trie Serving Service (in RAM)]   │ serving (online, <100ms)
   (debounced)            │              ▲                         │
                          ▼              │ (periodic rebuild/swap)  │
                    [Analytics store]    │                         │
                          ▲              │                         │
   (log queries) ──► [Kafka] ──► [Aggregation (Flink/Spark)] ──► [Trie Builder]   gathering (offline)
```

The clean separation is the whole point: the **top half** serves precomputed suggestions at memory speed; the **bottom half** continuously gathers and ranks data and periodically refreshes what the top half serves. The two scale and fail independently.

---

## 9. Sharding & Scaling

A single global trie may not fit in one machine's RAM or handle the QPS, so shard it.

- **Shard by first character (a–z)** is the simplest idea — 26 shards, route by the prefix's first letter. **But beware the hot-shard problem:** prefixes are *not* uniformly distributed — far more terms start with "s" than "z" — so naive first-character sharding gives badly uneven load (this is exactly the uneven-distribution issue consistent hashing/vnodes address). 
- **Better:** shard by a **hash of the prefix** (e.g., first 2–3 characters) or use a routing layer that balances load, so traffic spreads evenly. Each shard holds a slice of the trie and fits in RAM.
- **Replicate** each shard for availability and read throughput.

**Caching on top:** the prefix→suggestions mapping is extremely cacheable. Popular prefixes can be cached aggressively (even at the edge/CDN), so most keystroke-queries are served from cache without touching a trie node.

**Real systems** often skip a hand-rolled trie entirely and use **Elasticsearch's completion suggester** (backed by an **FST** — Finite-State Transducer, a very compact prefix structure) or **Redis sorted sets** (score = frequency, range query for top-K by prefix). These give the same precomputed-prefix-lookup behavior with battle-tested infrastructure.

---

## 10. Freshness vs Cost

A core trade-off worth naming explicitly: **how fresh do suggestions need to be, and what does freshness cost?**

- Applying every query to the trie's top-K **in real time** is expensive — each update can ripple through many nodes' cached top-K lists, at enormous write volume.
- But autocomplete **tolerates staleness** — suggestions reflecting the last hour are perfectly fine for the vast majority of prefixes.

So the design **batches updates** (rebuild every ~hour) for most terms, and may run a **faster path for trending terms** (a lighter, more frequent job that bumps suddenly-spiking queries into suggestions within minutes). You spend freshness effort only where it matters, keeping the common case cheap. This freshness-vs-cost dial is the system's main tuning knob.

---

## 11. Personalization & Real-World Implementations

**Personalization** layers a per-user signal (search history, location, language) on top of the global popularity ranking. A common pattern: serve global top-K from the shared trie, then **re-rank or blend** with the user's personal signals at the edge/serving layer (personal history is small and per-user, so it's cheap to apply last). Full per-user precomputation doesn't scale, so personalization is usually a lightweight final re-rank, not a separate trie per user.

**Real-world systems:**
- **Google** — heavily personalized, ML-ranked suggestions blending popularity, personal history, location, and trends.
- **Amazon** — catalog-driven plus product popularity, tuned to drive purchases.
- **Elasticsearch** — built-in **completion suggester** using an **FST** for compact, fast prefix matching — the common off-the-shelf choice.

The takeaway: the precompute-and-serve skeleton is universal; what differs is the sophistication of the *ranking* (simple popularity → full ML) layered into the offline pipeline.

---

## 12. Senior Follow-Up Questions (with Answers)

**Q1. Why can't you just query the database with `LIKE 'prefix%'` on each keystroke?**
At the keystroke-multiplier QPS with a <100 ms budget, prefix scans plus per-query ranking are far too slow and overload the DB. You must serve **precomputed** top-K via an in-memory structure instead of searching/ranking at request time.

**Q2. What's the single most important architectural decision?**
Separating the **offline data-gathering/ranking pipeline** from the **online serving layer**. They have opposite requirements (heavy/batch/tolerant-of-latency vs. trivial/instant), and the serving path must do lookups only, never ranking.

**Q3. How does the trie make lookups fast?**
Walking the prefix is O(prefix length) — tiny — and each node **caches its precomputed top-K**, so you return a ready list with no scanning of descendants and no ranking. The trie sits in RAM.

**Q4. What's the cost of caching top-K at every node, and how do you manage it?**
More memory and expensive updates (changing frequencies ripples into many nodes' top-K). You manage it by **batching updates offline** (periodic rebuilds) rather than updating per query, accepting slightly stale rankings.

**Q5. How do you shard, and what's the pitfall?**
Sharding by first character is simple but creates **hot shards** because prefixes aren't uniform (many more "s" terms than "z"). Prefer hashing the prefix or a load-aware router so traffic spreads evenly; replicate shards for availability.

**Q6. How fresh are suggestions, and why is that acceptable?**
Most terms rebuild on a schedule (~hourly); trending terms get a faster path (minutes). Real-time top-K updates are too expensive at this write volume, and autocomplete **tolerates mild staleness**, so batching is the right trade. Freshness vs. cost is the main tuning dial.

**Q7. How do you handle personalization without a trie per user?**
Serve the global top-K, then **re-rank/blend** with the user's small personal signal (history, location) at the serving edge as a lightweight final step. Per-user precomputation doesn't scale; per-user re-ranking on top of shared results does.

**Q8. How do you keep latency under 100 ms at the highest QPS in the stack?**
Debounce on the client; serve from in-RAM tries; cache popular prefixes aggressively (even at the edge); shard and replicate for throughput; do zero ranking on the hot path. Most keystroke-queries should hit a cache or a single in-memory node.

**Q9. Would you build a trie or use off-the-shelf infrastructure?**
For most teams, use **Elasticsearch's completion suggester (FST)** or **Redis sorted sets** — they implement the same precomputed-prefix-lookup behavior with proven operations. Hand-rolling a trie is justified mainly at extreme scale or with special ranking needs.

**Q10. How do you support typo tolerance / fuzzy matching?**
Beyond exact prefixes, fuzzy matching (edit-distance) is layered into the index (e.g., Elasticsearch's fuzzy completion or an FST that tolerates edits). It's a ranking/index-side concern handled in the offline build, keeping the serving path a lookup.

---

## 13. Quick Glossary

- **Typeahead / autocomplete** — suggesting completions as the user types a prefix.
- **Top-K** — the K best-ranked suggestions for a given prefix.
- **Keystroke multiplier** — autocomplete QPS ≈ (chars typed) × search QPS; why it's so high-volume.
- **Debouncing** — waiting for a typing pause before firing a query, to collapse keystroke bursts.
- **Trie (prefix tree)** — tree where each path spells a prefix; natural fit for prefix matching.
- **Precomputation** — computing top-K per prefix ahead of time so serving is a lookup.
- **Serving layer** — the online, in-RAM component returning precomputed suggestions fast.
- **Build pipeline** — the offline Kafka → aggregation → index-build flow producing rankings.
- **Rolling window** — counting term frequency over a recent period (e.g., 24 h) for relevance.
- **Hot shard** — an overloaded partition from uneven key distribution (e.g., first-character sharding).
- **FST (Finite-State Transducer)** — a compact automaton for prefix lookups (Elasticsearch completion).
- **Completion suggester** — Elasticsearch's built-in autocomplete feature.

---

*Reference document. Contributions and corrections welcome.*
