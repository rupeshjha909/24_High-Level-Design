# Design Typeahead / Autocomplete (Search Suggestions) — Detailed HLD

> A complete, interview-ready high-level design for a **search typeahead** system: as a user types, return the top suggestions in <100 ms. The design splits cleanly into an **online serving layer** (an in-RAM trie with **precomputed top-K per node**) and an **offline build/update pipeline** (Kafka → stream aggregation → trie builder → atomic swap). This doc details the **request/response contract at every hop** (client → gateway → suggestion service, plus the async logging path), the data structures, ranking, sharding, caching, capacity math (verified), failure modes, and senior Q&A.

> 💡 **The one idea:** *do all the ranking work offline, so the read path is just "walk a few nodes and return a cached list."* Serving does **zero ranking**; the trie caches the top-K terms at every prefix node, so a lookup is O(prefix length) with no descendant scan.

---

## Table of Contents
1. [Requirements & assumptions](#1-requirements--assumptions)
2. [Capacity estimation (verified)](#2-capacity-estimation-verified)
3. [API design — the contract at every layer](#3-api-design--the-contract-at-every-layer)
4. [The client (debounce, params, behavior)](#4-the-client-debounce-params-behavior)
5. [The serving layer — trie with precomputed top-K](#5-the-serving-layer--trie-with-precomputed-top-k)
6. [The read path, step by step](#6-the-read-path-step-by-step)
7. [The logging path (collecting queries)](#7-the-logging-path-collecting-queries)
8. [The build & update pipeline (offline)](#8-the-build--update-pipeline-offline)
9. [Ranking (frequency + recency + more)](#9-ranking-frequency--recency--more)
10. [End-to-end architecture](#10-end-to-end-architecture)
11. [Sharding the trie](#11-sharding-the-trie)
12. [Caching & the latency budget](#12-caching--the-latency-budget)
13. [The atomic swap (updating without downtime)](#13-the-atomic-swap-updating-without-downtime)
14. [Trending / real-time terms](#14-trending--real-time-terms)
15. [Extensions (personalization, spell-correct, i18n)](#15-extensions-personalization-spell-correct-i18n)
16. [Failure modes & handling](#16-failure-modes--handling)
17. [Senior-level Q&A](#17-senior-level-qa)
18. [TL;DR](#18-tldr)

---

## 1. Requirements & assumptions

**Functional**
1. As the user types a prefix, return the **top-K suggestions** (e.g., 5–10) for that prefix.
2. Suggestions ranked by **popularity** (query frequency), biased to **recent** trends.
3. Match on **prefix** (optionally infix/fuzzy as an extension).
4. Support **locale/region** (suggestions differ by language/country).

**Non-functional**
1. **Latency < 100 ms** end-to-end (it must feel instant as you type).
2. **Very read-heavy** — many suggest calls per typed query; ranking updates are periodic.
3. **High availability** — degrade gracefully (stale suggestions are fine; no suggestions is acceptable, errors are not).
4. **Eventual consistency of rankings** — a new trending term appearing minutes later is fine.
5. **Scale** — hundreds of thousands of QPS; tens of millions of distinct terms.

**Assumptions to state aloud**
- We serve suggestions only for the **top ~10M popular terms** (drop the long tail — nobody needs a suggestion that's only ever been typed once).
- Prefix length ≥ 1–2 chars before we suggest (avoid returning for empty input).
- Rankings refresh ~hourly for popular terms, faster for trending.

---

## 2. Capacity estimation (verified)

**Traffic** (100M DAU × 10 searches/day):
- **Searches/day:** 1,000,000,000 (~11.6K/sec).
- Each search = ~5 debounced *suggest* calls as the user types → **5B suggest requests/day**.
- **Suggest QPS: ~57.9K avg, ~289K peak** (5×).
- **Log/write events:** ~11.6K/sec (one per submitted search) → **read:write ≈ 5:1**, heavily read-dominated (and caching pushes it higher).

**Caching effect** (popular prefixes dominate): a 70% edge-cache hit rate drops the trie fleet to **~87K QPS at peak** (60%→116K, 80%→58K).

**Trie memory** (top 10M terms, avg length 20, top-K=10 cached per node):
- ~100M nodes, ~**20 GB** total → **must shard across machines**.
- Shard by first **1 char** (~40 buckets) → ~0.5 GB/shard; by first **2 chars** (~1,500 buckets) → ~0.01 GB/shard.

**Latency budget (target <100 ms):** client debounce 50 + network 20 + gateway 3 + **trie lookup 1** + serialize 2 = **~76 ms**. The trie lookup itself is ~1 ms (walk the prefix, return the cached list).

> 💡 **The estimation drives three design decisions:** (1) it's read-heavy + cacheable → put a cache/CDN in front and precompute; (2) 20 GB of trie won't fit hot on one box → **shard by prefix**; (3) the 100 ms budget is mostly network+debounce, so the *server* work must be ~1 ms → **precomputed top-K, no ranking on read**.

---

## 3. API design — the contract at every layer

This is the heart of "what params flow where." Three hops on the read path plus one async write path.

### 3.1 Client → API Gateway (the public suggest endpoint)

```
GET /v1/suggest?q=har&limit=10&lang=en&region=IN&session=abc123
Headers:
  Accept-Language: en-IN
  Authorization: Bearer <token>        # or an anonymous device token
  X-Request-Id: <uuid>                 # tracing/idempotency of the request
  User-Agent: <app/version/device>
```
| Param | Meaning | Notes |
|:--|:--|:--|
| `q` | the prefix typed so far | trimmed, URL-encoded; server normalizes (lowercase, collapse spaces) |
| `limit` | how many suggestions (K) | default 10, capped server-side (e.g., ≤ 20) |
| `lang` | language | drives which locale's trie is queried |
| `region` | country/market | suggestions differ by market |
| `session` | session id | for logging/personalization; not required to serve |

**Response**
```json
{
  "query": "har",
  "suggestions": [
    { "text": "harry potter",        "type": "popular",  "score": 0.98,
      "highlight": [[0,3]] },
    { "text": "hard drive",          "type": "popular",  "score": 0.71, "highlight": [[0,3]] },
    { "text": "harvard university",  "type": "trending", "score": 0.65, "highlight": [[0,3]] }
  ],
  "source": "trie-shard-h",
  "latencyMs": 4
}
```
`highlight` = the char ranges matching the prefix (for bolding in the UI); `type` distinguishes popular/trending/personal; `score` is optional (debug/UX).

### 3.2 API Gateway → Suggestion (Trie Serving) Service (internal RPC)

The gateway **normalizes** and enriches, then routes to the right shard:
```json
// internal request
{
  "prefix": "har",                 // normalized: trimmed, lowercased
  "limit": 10,
  "locale": "en-IN",               // resolved from lang+region+Accept-Language
  "userId": "u_9931",              // resolved from the token (null if anonymous)
  "experiment": "typeahead_v3",    // A/B bucket
  "requestId": "…"                 // propagated for tracing
}
```
The gateway picks the **shard** by prefix (§11) and calls that suggestion node. Internal response mirrors the public one (list of `{text, type, score}`), pre-highlight.

### 3.3 Suggestion Service (internal, per shard)

No external params beyond the internal request. It: normalizes again defensively → walks the trie → returns the node's cached top-K → (optionally) merges personalized/trending lists. All in-memory, ~1 ms.

### 3.4 Client → Ingest API (the async logging path — a *separate* call)

When the user **submits** a search (or on sampled keystrokes), the client fires a fire-and-forget event (does **not** block typing):
```
POST /v1/events/search
{
  "eventId": "e_...",              // dedupe
  "query": "harry potter",         // the final submitted query (or the prefix, if logging keystrokes)
  "prefixTyped": "har",            // what was typed when a suggestion was chosen
  "selected": "harry potter",      // which suggestion was clicked (null if typed fully)
  "position": 0,                   // rank of the clicked suggestion (for CTR ranking)
  "ts": 1720000000000,
  "userId": "u_9931",              // or anonId
  "sessionId": "abc123",
  "locale": "en-IN",
  "device": "android"
}
```
This event is what feeds ranking. It's decoupled from serving entirely.

> 💡 **Why separate the log call from the suggest call?** Serving must be blazing fast and must not wait on writes. Logging is a fire-and-forget `POST` (often batched client-side and sent async) into the ingest pipeline. The read path and the write path share nothing but the data they eventually produce/consume — they scale and fail independently.

---

## 4. The client (debounce, params, behavior)

The client is part of the design — it controls how much load reaches the backend:
- **Debounce** ~50 ms: don't fire on every keystroke; wait for a typing pause. Cuts request volume massively.
- **Min prefix length**: don't call for 0–1 chars (too broad).
- **Cancel in-flight**: when a newer keystroke arrives, cancel the older request (or ignore its response) so results don't flicker out of order.
- **Client-side cache**: cache recent prefixes locally (typing "har" then backspacing to "ha" should be instant).
- **Send `lang`/`region`** so the server queries the right locale.

> 💡 **Debounce + client cache is the first line of scaling.** The estimation assumed ~5 suggest calls per search *because* of debouncing; without it you'd have one call per keystroke (~2–3×). The cheapest QPS reduction happens on the client.

---

## 5. The serving layer — trie with precomputed top-K

A **trie (prefix tree)**: each path from the root spells a prefix, so walking it character-by-character is exactly what typeahead needs.

```
        (root)
        /    \
       c      h ...
      /        \
     a          a
    /            \
  "cat"(3K)     "har" node ── cached top-K: ["harry potter"(9K), "hard drive"(4K), ...]
```

**The crucial optimization — cache the top-K at *every* node, not just leaves:**
- Each node stores the **precomputed top-K terms** for the prefix it represents.
- A lookup = walk to the prefix node, **return its cached list**. No descendant scan, no ranking at read time.

**Node structure (conceptual):**
```
TrieNode {
  children: Map<char, TrieNode>     // sparse; next characters
  topK: List<Suggestion>            // precomputed [ {term, score} ] for THIS prefix
  isTerm: bool                      // is this prefix itself a complete term?
}
```

Lives in **RAM** for speed. Because top-K is precomputed, serving does **zero ranking work**.

**The trade-off (state it):** caching top-K at every node uses **more memory** and makes **updates expensive** — when a term's frequency changes, every ancestor node's top-K may need recomputing. That's exactly why updates are **batched offline** (§8), never applied per write.

> 💡 **Why a trie and not a sorted list + binary search, or Elasticsearch?** A sorted list gives you the *matching* terms via binary search but you'd still have to rank them per query. Elasticsearch's completion suggester works but is heavier and harder to hit <5 ms at this QPS. The trie-with-cached-top-K reduces a query to "traverse ~3–20 nodes, return a ready list" — the minimum possible read work. It's the purpose-built structure.

---

## 6. The read path, step by step

```
suggest("har", limit=10, locale="en-IN"):
  1. normalize prefix         → "har"   (trim, lowercase, collapse spaces)
  2. route to shard           → shard owning prefixes starting "h" (§11)
  3. walk the trie            → root → h → a → r      O(len(prefix)) ≈ 3 hops
  4. read node.topK           → cached list already ranked
  5. take first `limit`       → slice K
  6. (optional) merge         → personalized + trending lists, re-rank lightly
  7. add highlight ranges     → [[0,3]] and return
```
Total server time ~1 ms. If the prefix node doesn't exist (no such prefix), return empty (or fall back to the closest ancestor's list).

> 💡 **The read does no counting, sorting, or descendant traversal** — that's the whole point. Everything expensive (counting frequencies, computing top-K) happened offline. The online path is "traverse a handful of nodes, return a cached list."

---

## 7. The logging path (collecting queries)

The offline half starts by durably capturing what people search:
```
[client search events] → [Ingest API] → [Kafka]
```
- The client sends **search events** (§3.4) to an ingest endpoint, which publishes to **Kafka** — a durable, high-throughput log decoupling *collection* from *processing*.
- Kafka handles the ~11.6K events/sec write stream and buffers bursts; consumers read at their own pace.

> 💡 **Why Kafka in the middle?** It decouples the write path (fast, always-available ingest) from the ranking computation (heavy, periodic). If the aggregation job is slow or down, events queue durably in Kafka and nothing is lost; serving is entirely unaffected. This is the standard "queue between producer and heavy consumer" stage.

---

## 8. The build & update pipeline (offline)

Turning raw queries into the ranked trie the serving layer uses:
```
[user queries] → [Kafka] → [Aggregation (Flink/Spark)] → [Trie/Index Builder] → [Serving layer]
                 (stream)   (count over rolling 24h window) (compute top-K)      (RAM, swapped in)
```
1. **Stream** every query into Kafka (§7).
2. **Aggregate** with a stream/batch processor (Flink/Spark): count term frequencies over a **rolling window** (e.g., trailing 24 h) so rankings reflect **recent** popularity, not all-time. Output: `term → count` (per locale).
3. **Build** a fresh trie/index: for each prefix, compute its **top-K** by frequency (walk terms, propagate counts up to ancestors, keep the K best per node).
4. **Swap** it into the serving layer atomically (§13) — rebuild popular terms ~hourly, trending more often.

Because rebuilding is periodic and offline, the heavy ranking work **never touches the latency-critical serving path**.

> 💡 **How top-K per node is computed at build time:** aggregate `term→count`; insert each term into the trie; for each node, its top-K = the K highest-count terms among all descendants (computed bottom-up so each node inherits/merges its children's top-K). This is O(total terms × K) offline — expensive, but done once per rebuild, off the hot path.

---

## 9. Ranking (frequency + recency + more)

Serving returns whatever the build pipeline ranked. Ranking signals (computed offline):
- **Frequency** over the rolling window (the base signal).
- **Recency / trending** — weight recent counts higher (time-decay), so surging terms rise.
- **CTR / selection** — which suggestion users actually click at a given prefix (from the `selected`/`position` fields), a strong quality signal.
- **Personalization** (optional) — the user's own history, blended at serve time (§15).
- **Locale** — separate counts/tries per language/market.

> 💡 **Ranking is a pipeline concern, not a serving concern.** You can evolve the ranking formula (add time-decay, CTR, quality filters) entirely in the offline builder without touching the serving code — the serving layer just reads whatever list the builder produced. That separation is what lets ranking get sophisticated without risking read latency.

---

## 10. End-to-end architecture

```
                          ┌──────────────────────────────────────────────┐
[Client keystroke] ─►[CDN/Edge cache]─►[API Gateway]─►[Trie Serving (RAM, sharded)]  │  ONLINE (<100ms)
   (debounced)                              │                    ▲                    │
                                            │                    │ periodic swap       │
                                            ▼                    │                    │
[Client search event] ─► [Ingest API] ─► [Kafka] ─► [Aggregation (Flink/Spark)] ─► [Trie Builder]  OFFLINE
                                                     (count, rolling 24h window)   (compute top-K → new trie)
```

The clean split is the whole point: the **top half serves precomputed suggestions at memory speed**; the **bottom half gathers and ranks data and periodically refreshes** what the top half serves. The two scale and fail independently — the pipeline can lag or restart with zero impact on serving (you just serve slightly staler suggestions).

---

## 11. Sharding the trie

20 GB (and growing) won't stay hot on one box, and 289K peak QPS needs many servers. **Shard by prefix:**
- **By first character** (~40 buckets incl. digits/symbols) → ~0.5 GB/shard; the gateway routes `q="har"` to the "h" shard.
- **By first 1–2 chars** for finer balance (~1,500 buckets) → ~0.01 GB/shard, but more shards to manage.
- **Replicate each shard** (N replicas) for QPS and HA; a load balancer spreads reads across replicas.

**Hot-shard problem:** some first-letters are far more common ("s", "a") than others ("z", "q"). Balance by (a) sharding on the first *two* characters, (b) consistent hashing with virtual nodes, or (c) splitting hot buckets further. State this — uneven prefix popularity is the real sharding challenge.

> 💡 **Why prefix-sharding works so well here:** a query only ever needs *one* shard (the one owning its prefix) — no scatter-gather across shards, unlike many sharded systems. The routing key (the prefix's first chars) is known from the request, so the gateway sends the query straight to the right node. The only wrinkle is balancing hot prefixes.

---

## 12. Caching & the latency budget

- **CDN / edge cache**: cache `(/v1/suggest?q=…&locale=…)` responses at the edge with a short TTL (seconds–minutes). Popular prefixes ("f", "fa", "fac"…) get served from the edge without hitting the trie — a 70% hit rate cuts the trie fleet to ~87K peak QPS (verified).
- **Client cache**: recent prefixes cached locally (backspacing is instant).
- **In-service**: the trie *is* the cache (RAM); no separate layer needed behind it.

**Latency budget (verified, ~76 ms of the 100 ms):** debounce 50 + network 20 + gateway 3 + trie 1 + serialize 2. Most of the budget is network + debounce, which is *why* the server work must be ~1 ms.

> 💡 **A short TTL is fine because staleness is harmless.** Suggestions being a few seconds/minutes stale never hurts — so you can cache aggressively at the edge. This is a case where weak consistency is a feature: it unlocks heavy caching, which is what makes the QPS affordable.

---

## 13. The atomic swap (updating without downtime)

Rebuilds produce a **whole new trie**; you must swap it in without disrupting reads:
- The builder writes the new trie (per shard) to a location the serving nodes can load.
- Each serving node **loads the new trie into memory alongside the old one**, then flips an **atomic pointer** (`current = newTrie`) — in-flight reads finish on the old trie, new reads use the new one. The old trie is GC'd once no reads reference it.
- Swap ~hourly for popular terms; more often for trending.

> 💡 **Why swap the whole structure instead of mutating in place?** Because top-K is cached at every node, an in-place frequency update can invalidate many nodes' top-K, requiring locking and recomputation on the live structure — risky and slow on the hot path. Building a fresh immutable trie offline and flipping a pointer is simpler, lock-free for readers, and keeps serving at memory speed. This is the classic "immutable snapshot + atomic swap" pattern (also why serving needs no write locks).

---

## 14. Trending / real-time terms

The hourly rebuild is too slow for breaking trends (a term surging in minutes). Add a **fast path**:
- A **streaming job** (Flink) maintains short-window counts (e.g., last 5–10 min) and detects spikes.
- Trending terms are pushed as a **small overlay** the serving layer merges into results (marked `type: "trending"`), refreshed every few minutes — without a full trie rebuild.

> 💡 **Two cadences, two structures:** the big trie captures durable popularity (hourly); a small trending overlay captures the last few minutes. The serving layer blends them. This gives both stable quality *and* responsiveness to breaking events without rebuilding 20 GB every minute.

---

## 15. Extensions (personalization, spell-correct, i18n)

- **Personalization** — blend the user's own recent searches (a small per-user list, fetched by `userId`) into/above the global top-K at serve time.
- **Spell correction / fuzzy** — support typos ("recieve" → "receive") via edit-distance on the trie (e.g., a BK-tree or Levenshtein automaton) — heavier; often a separate path.
- **Infix / multi-word** — suggest on non-prefix matches ("phone" → "iphone"); needs additional indexing (e.g., index suffixes or n-grams).
- **i18n / multi-locale** — a separate trie per language/market; route by `locale`. Unicode normalization for non-Latin scripts.
- **Filtering** — remove offensive/blocked terms in the builder (never serve them).

---

## 16. Failure modes & handling

| Failure | Effect | Handling |
|:--|:--|:--|
| **Aggregation/build job down** | Rankings stop refreshing | Serving unaffected; keep serving the last good trie (stale is fine); alert |
| **Kafka backlog** | Ranking data delayed | Durable log absorbs it; process catches up; no serving impact |
| **A trie shard down** | That prefix range unserved | Replicas take over; if all down, return empty for those prefixes (degrade, don't error) |
| **Hot shard (popular letter)** | Uneven load / latency | Shard on 2 chars, consistent hashing, split hot buckets (§11) |
| **Bad rebuild (corrupt trie)** | Bad suggestions | Validate the new trie before swap; keep the previous one; roll back the pointer |
| **Traffic spike** | QPS surge | Edge cache absorbs most; autoscale replicas; the read path is cheap |
| **Client hammering (no debounce)** | Excess QPS | Enforce min-prefix + rate limits at the gateway |

> 💡 **Graceful degradation is the availability strategy:** stale suggestions are fine, *no* suggestions is acceptable, but errors/slowness are not. Every failure path degrades to "serve the last good data" or "return empty," never to an error that blocks the user's typing.

---

## 17. Senior-level Q&A

**Q: Why precompute top-K at every node instead of ranking at query time?**
The 100 ms budget is mostly network+debounce, so server work must be ~1 ms. Ranking descendants per query can't hit that at 289K QPS. Precomputing offline turns the read into "walk the prefix, return a cached list."

**Q: How do you update rankings without hurting read latency?**
Build a fresh immutable trie offline and swap it in with an atomic pointer flip. Never mutate the live trie (top-K caching makes in-place updates invalidate many nodes).

**Q: How do you shard, and what's the catch?**
By prefix (first 1–2 chars) — each query hits exactly one shard (no scatter-gather). The catch is **hot prefixes** ("s","a"); balance with 2-char sharding / consistent hashing / hot-bucket splitting.

**Q: How fresh are suggestions?**
Popular terms ~hourly (full rebuild); trending terms every few minutes (a streaming overlay). Staleness is harmless, which also lets you cache at the edge.

**Q: How do you keep it available?**
Replicate shards; edge-cache heavily; degrade gracefully (serve last-good or empty, never error). Serving is decoupled from the pipeline, so pipeline outages don't affect reads.

**Q: What params does the client send, and what's logged?**
Serve: `q`, `limit`, `lang`, `region`, locale headers, auth. Log (separate async call): the final/selected query, the prefix typed, the clicked position, user/session, locale, ts — the signals ranking needs.

**Q: Why Kafka between client and aggregation?**
Durability + decoupling: the write path stays fast and available; the heavy ranking job consumes at its own pace; backlogs never affect serving.

**Q: How big is the trie and where does it live?**
~20 GB for 10M terms with top-K per node → sharded (~0.5 GB/shard by first char), in RAM, replicated.

---

## 18. TL;DR

**Two halves, decoupled:**
- **Online (serving):** an in-RAM **trie** with **precomputed top-K at every node**, sharded by prefix, replicated, edge-cached. A read = walk the prefix (O(len)) + return the cached list (~1 ms) — **zero ranking on read**.
- **Offline (pipeline):** client search events → **Kafka** → **Flink/Spark** aggregation (rolling-window counts) → **trie builder** (compute top-K) → **atomic swap** into serving (~hourly; trending overlay every few min).

**API contract (the layers):**
- Client → Gateway: `GET /v1/suggest?q&limit&lang&region` (+ locale/auth headers), debounced client-side.
- Gateway → Suggestion service: normalized `{prefix, limit, locale, userId, experiment}`, routed to the prefix's shard.
- Client → Ingest (async, separate): `POST /events/search {query, prefixTyped, selected, position, user, locale, ts}` → Kafka.

**Why it's fast & scalable:** precompute offline, serve from RAM, shard by prefix (single-shard reads), cache at the edge (staleness is harmless), swap immutable tries atomically.

### Verified numbers
```
1B searches/day → 5B suggest req/day → ~58K QPS avg, ~289K peak
Cache 70% hit → trie sees ~87K peak QPS   |   read:write ≈ 5:1
Trie: 10M terms, ~100M nodes, ~20 GB → shard by 1st char (~40 × ~0.5 GB) or 2 chars (~1500 × ~0.01 GB)
Latency budget ≈ 76 ms (debounce 50 + net 20 + gateway 3 + trie 1 + serialize 2); trie lookup ~1 ms
```

### One-line philosophy
> **Do all the ranking offline and cache the top-K at every trie node, so the online read is just "walk the prefix, return a cached list" (~1 ms) — served from a prefix-sharded, replicated, edge-cached in-RAM trie that's rebuilt periodically by a Kafka→Flink pipeline and swapped in atomically. Serving and the pipeline share nothing but the data, so they scale and fail independently, and staleness is harmless enough to cache hard.**
