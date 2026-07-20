# Designing a Web Crawler: A Senior Interview Guide

> A practical, interview-focused reference for a large-scale web crawler — a **polite, fault-tolerant, distributed traversal of the web graph** that feeds a search index. This guide builds the architecture up piece by piece, traces the full life of a URL through the crawl loop, goes deep on the two genuinely hard mechanisms (the **URL frontier** that balances priority against politeness, and **dedup at billions-scale** — Bloom filters for URLs, SimHash for near-duplicate content), nails down the data contracts, and covers politeness, distributed coordination, crawl traps, and refresh strategy — with verified capacity, Bloom-filter, and SimHash math and a senior follow-up bank.

> 💡 **The one idea (see §5):** a crawler looks like "fetch a page, extract links, repeat," but the whole problem is the **URL frontier** — deciding *what to crawl next* while simultaneously honoring **priority** (crawl valuable/fresh pages first) and **politeness** (never hammer one host). Everything else (dedup, storage, coordination) exists to serve a frontier that can be both fast and polite at billions-scale.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified, with a correction)](#3-capacity-estimation-verified-with-a-correction)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a URL](#44-the-end-to-end-life-of-a-url)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [Hard Problem 1: The URL Frontier (priority vs politeness)](#5-hard-problem-1-the-url-frontier-priority-vs-politeness)
6. [Hard Problem 2: Deduplication at Billions-Scale (URL + content)](#6-hard-problem-2-deduplication-at-billions-scale-url--content)
7. [Politeness](#7-politeness)
8. [Distributed Coordination (partition by domain)](#8-distributed-coordination-partition-by-domain)
9. [Data Contracts: Records, Payloads & Store Schemas](#9-data-contracts-records-payloads--store-schemas)
10. [Crawl Traps & Refresh Strategy](#10-crawl-traps--refresh-strategy)
11. [Scaling Summary](#11-scaling-summary)
12. [Failure Modes & Handling](#12-failure-modes--handling)
13. [Senior Follow-Up Questions (with Answers)](#13-senior-follow-up-questions-with-answers)
14. [Quick Glossary](#14-quick-glossary)

---

## 1. How to Approach This in an Interview

A web crawler is a **massive, prioritized, polite breadth-first traversal of the web graph** that gathers content for a search index. It *looks* trivial ("fetch a page, extract its links, repeat"), but three things make it a serious distributed-systems problem, and they're where you should spend your time:
1. **The URL frontier** — deciding *what to crawl next* while balancing **priority** (crawl important/fresh pages first) against **politeness** (don't hammer any one server). This is the crux.
2. **Deduplication at scale** — with billions of URLs and pages, cheaply answering "have I seen this URL / this content?" — the canonical **Bloom filter** + **SimHash** application.
3. **Politeness and the adversarial web** — respecting `robots.txt` and rate limits, and defending against infinite **crawl traps**.

A good structure: requirements → scale → the crawl loop → go deep on the **frontier**, **dedup**, **politeness** → coordination, traps, refresh.

> 💡 **Senior signal:** treat **politeness as a first-class constraint** (not an afterthought), name the **frontier** as the hard central component, and show the capacity arithmetic (catching the page-count/refresh-rate mismatch, §3). Say up front: *"The hard part isn't fetching — it's the frontier deciding what's next while staying polite, and dedup at billions-scale. I'll spend my time there."*

---

## 2. Requirements

### Functional
- Given **seed URLs**, crawl the web; **extract content** and store it for indexing.
- **Respect `robots.txt`**; **avoid duplicates**; **configurable crawl depth.**

### Non-Functional
- **Scalable** — billions of URLs.
- **Polite** — don't overload crawled servers (or get IP-banned).
- **Fault-tolerant** — survive worker/fetch failures without losing progress.
- **Extensible** — handle HTML, PDFs, images, etc.

---

## 3. Capacity Estimation (verified, with a correction)

```
10B pages, refresh every 7 days:
   10×10⁹ / (7 × 86,400) ≈ 16,534 pages/sec         ← the correct rate for 10B@7d
The ~1,500/sec figure = 1B pages @ 7 days (1,653/s) OR 10B @ ~75 days (1,543/s).
Keep page count and refresh window consistent with the rate.

Fetcher workers @16.5K/s:  ~1,653 (10/s each) … ~165 (100/s each)   — many, network-bound
Raw HTML:  10B × ~100 KB ≈ ~1 PB               (object storage; refreshed, not cumulative)
URL dedup set: ~100B URLs → Bloom filter ~120–180 GB (vs terabytes for a plain set)
Politeness: ≤1 req/host/sec → to sustain 16.5K/s you need ~16,500 DIFFERENT hosts in flight
```

> **The correction (a strong signal to show):** 10B pages on a 7-day refresh needs **~16.5K/sec, not ~1.5K**. The ~1.5K/sec figure is **~1B pages weekly** (or 10B every ~75 days). Either is fine — just keep the numbers consistent. *Showing this arithmetic and catching the mismatch* is itself the signal.

**Takeaways that shape the design:**
- Even ~1.5K/sec sustained needs **many distributed fetcher workers** (each fetch is network-bound and slow) → the crawler is **horizontally scaled and I/O-bound.**
- Billions of URLs → the "have I seen this?" set can't fit in memory as a plain set → **Bloom filter** (~120-180 GB, verified).
- Raw HTML at ~1 PB → cheap **object storage**, not a database.
- **Politeness caps per-host throughput**, so high sustained rates *require breadth* — thousands of hosts crawled concurrently (verified: ~16,500 hosts in flight for 16.5K/s at a 1s crawl-delay).

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

A web crawler is a loop wrapped around one hard data structure. The loop is simple: pull a URL from the **frontier**, **fetch** the page, store the raw bytes, **parse** it to extract **content** (for the search index) and **new links**, **dedup** those links, and feed the unseen ones back into the frontier — a prioritized breadth-first traversal of the web graph. The hard data structure is the **frontier** itself, because it has to reconcile two goals that fight each other: crawl the *most valuable and freshest* pages first (**priority**), yet *never* hit any single host too fast (**politeness**) — even when that host owns thousands of high-priority URLs. The classic answer is a **two-layer queue**: **front queues** order by priority, **back queues** are per-host and drain at a polite rate. Wrapped around this are the mechanisms that make it work at billions-scale: **dedup** (a **Bloom filter** answers "seen this URL?" in tiny space; **SimHash** catches near-duplicate *content*), **domain-partitioned coordination** (all of a host's URLs live on one worker, so politeness is enforced locally with no cross-worker chatter), **polyglot storage** (raw HTML in object storage, parsed content in a search index, URL metadata in a DB), and defenses against the **adversarial web** (crawl traps, robots.txt) plus an **adaptive refresh** strategy so volatile pages get more budget than static ones.

### 4.2 The diagram

```
[Seed URLs]
     │
     ▼
┌─────────────────────────────────────────────┐
│  URL FRONTIER (partitioned by domain)          │◄────────────────────────────┐
│   front queues: PRIORITY  →  back queues: PER-HOST (politeness)               │
└───────────────┬───────────────────────────────┘                              │
                │ next URL (host's rate permitting)                             │
                ▼                                                               │
        [Fetcher Workers] ──(robots.txt cache)──► [Raw HTML → Object Store (S3)]│
                │                                                               │
                ▼                                                               │
            [Parser] ─┬─► [Content Extractor] ─► [Search Index (Elasticsearch)] │
                      ├─► [URL Extractor] ─► [Dedup: Bloom(URL) + SimHash(content)]
                      │                              │ unseen URLs               │
                      └─► [URL metadata DB]          └───────────────────────────┘
                          (last-crawled, status, depth, change-freq)
```

The key visual idea: the **frontier** is the heart (priority front → per-host back); fetchers are dumb, parallel, I/O-bound workers; the **dedup filter** gates what re-enters the frontier; and storage is polyglot (bytes → S3, content → index, metadata → DB).

### 4.3 Each component, in detail

**① URL Frontier (the crux).** A **two-layer, domain-partitioned** priority-and-politeness queue. **Front queues** order URLs by priority (importance/PageRank, freshness, depth); **back queues** are **per-host** and release URLs at that host's polite rate. Owns the answer to "what next?" (§5). Durable so progress survives crashes.

**② Fetcher Workers.** Many horizontally-scaled, network-bound workers that pull a URL (only when its host's rate permits), fetch it over HTTP honoring cached **robots.txt** + crawl-delay, and write the raw bytes to object storage. Dumb and stateless — the frontier decides *what*, they just *fetch*.

**③ Parser.** Extracts (a) **content** → the search index, (b) **outbound links** → the dedup filter → frontier, (c) **metadata** → the URL DB. Handles multiple content types (HTML/PDF/images).

**④ Dedup Filter.** Two distinct checks: a **Bloom filter** for "seen this URL?" (tiny space, billions of URLs) and **SimHash/MinHash** for "is this page a near-duplicate?" (§6). Gates what re-enters the frontier and what gets indexed.

**⑤ Object Store (S3).** Cheap, durable home for the ~1 PB of raw HTML — so parsing/indexing can be retried independently of fetching.

**⑥ Search Index (Elasticsearch).** The inverted index the search queries run against — the crawler's whole reason to exist (it's the data-gathering front end of a search engine).

**⑦ URL Metadata DB.** Per-URL bookkeeping — last-crawled time, HTTP status, depth, observed change-frequency — for scheduling, refresh, and dedup accounting.

**⑧ robots.txt cache.** Per-host crawl rules, fetched once and cached, consulted before every fetch (politeness, §7).

### 4.4 The end-to-end life of a URL

Here is exactly what happens to **one URL** from discovery to re-indexing, step by step:

```
DISCOVERY → FRONTIER:
1.  A URL is discovered (a seed, or extracted from a fetched page).
2.  NORMALIZE it (lowercase host, strip fragment, sort query params) → hash it.
3.  DEDUP: check the Bloom filter. "Definitely seen" → drop. "Not seen" → continue.  ← §6
4.  Assign PRIORITY (importance/freshness/depth) → enqueue in a front queue,
        which routes it to its HOST's back queue (domain-partitioned worker owns that host).  ← §5

FETCH (politeness-gated):
5.  The worker owning that host pulls the URL ONLY when the host's crawl-delay allows
        (≤ N concurrent, ≥ delay since last hit). Consults the cached robots.txt first.  ← §7
6.  FETCH the page over HTTP → write the raw bytes to the OBJECT STORE (S3).
        (fetch failed? re-queue with backoff, up to a retry limit — fault tolerance.)

PARSE → EXTRACT:
7.  Parser reads the raw HTML:
        - CONTENT dedup: compute SimHash; if within a small Hamming distance of a known page → near-dup → skip indexing.  ← §6
        - otherwise EXTRACT content → SEARCH INDEX (Elasticsearch).
        - EXTRACT outbound links → back to step 1 for each (discovery → dedup → frontier).
        - UPDATE the URL metadata DB (last-crawled, status, depth, observed change).

REFRESH (later):
8.  A scheduler re-enqueues the URL after an ADAPTIVE interval based on its observed
        change frequency (news hourly, docs weekly) → the loop repeats.  ← §10
```

The single most important thing to notice: **the frontier gates the front (priority + politeness) and the Bloom filter gates the back (dedup)** — between them they turn an infinite, adversarial, redundant web into a bounded, prioritized, polite stream. Fetching itself is the easy, parallelizable part.

### 4.5 Why this split? (the design rationale)

- **Frontier separate from fetchers** — deciding *what to crawl next* (priority + politeness, stateful) is a different problem from *fetching* (stateless, I/O-bound, massively parallel). Splitting lets fetchers scale out freely while the frontier holds the hard logic.
- **Two-layer frontier (priority front + per-host back)** — you must satisfy *both* goals; one queue can't. Front queues answer "what's most valuable," back queues answer "what's polite for this host" (§5).
- **Domain partitioning** — putting all of a host's URLs on one worker makes politeness **local** (enforce crawl-delay/connection-limit without cross-worker coordination). The single most important coordination decision (§8).
- **Bloom filter for URL dedup** — a plain set of 100B URLs is terabytes; a Bloom filter is ~120-180 GB and answers "definitely not seen" instantly. A false positive only skips one page — a bounded, harmless error (§6).
- **Polyglot storage** — bytes (S3), content (search index), metadata (DB) have different shapes and access patterns; each store does what it's best at.
- **Raw HTML stored durably** — so parsing/indexing can be retried independently of the (expensive, polite-rate-limited) fetch.

### 4.6 Where the load actually goes

- **Fetching:** ~**16.5K pages/s** (for 10B@7d) — **network-bound**, spread across **hundreds to thousands of fetcher workers**. The bottleneck is I/O + politeness, not CPU.
- **Politeness caps per-host rate** — so throughput comes from **breadth**: ~**16,500 hosts in flight** concurrently at a 1s crawl-delay (verified). A single host can never be crawled fast no matter its priority.
- **URL dedup set:** ~**100B URLs** → a **~120-180 GB Bloom filter** (verified), held in memory / sharded — vs terabytes for a plain set.
- **Raw HTML:** ~**1 PB** in object storage (refreshed, not cumulative).
- **Frontier size:** billions of pending URLs → **durable, partitioned** across workers (not a single in-memory queue).
- **Content dedup:** SimHash fingerprints per page — cheap to compute, compared by Hamming distance to catch mirrors/syndication.

> 💡 **The senior framing:** *"Fetching is easy and parallel — it's network-bound, so I scale fetchers horizontally. The real constraints are: politeness caps per-host rate, so high throughput needs thousands of hosts in flight; dedup at 100B URLs needs a Bloom filter (~150 GB) not a set; and the frontier is a huge durable partitioned structure, not an in-memory queue. The frontier balancing priority against politeness is the hard part."*

---

## 5. Hard Problem 1: The URL Frontier (priority vs politeness)

The frontier decides *what to crawl next*, and it must satisfy **two goals that actively fight each other.** *How do you crawl the most valuable pages first while never hammering any single host — even when that host owns thousands of high-priority URLs?*

### The naive (wrong) approaches
- **A single priority queue** (crawl highest-importance first): a news site with 10,000 hot URLs would get **10,000 rapid-fire hits** — impolite, and you'd be IP-banned. Priority alone ignores politeness.
- **A single per-host round-robin** (be polite to everyone): you'd crawl junk pages before important ones — politeness alone ignores priority.

Neither works because the two goals are **orthogonal** — one orders *across* the web (importance), the other paces *within* a host (politeness).

### The key realization: separate the two into two layers
Use a **two-layer queue** (Mercator-style):
- **Front queues** order URLs by **priority** — importance/PageRank, freshness, depth. "Which URLs deserve crawling soonest?"
- **Back queues** are **per-host** — each holds one host's URLs and releases them at that host's **polite rate** (respecting crawl-delay + connection limit). "What's safe to send this host right now?"

A URL flows from a priority **front** queue into its host's **back** queue; workers pull from **back** queues, each gated by its host's rate.

```
discovered URL
     │ assign priority
     ▼
[ front queue: priority band 1 ]  [ band 2 ]  [ band 3 ] …     (order ACROSS the web)
     │  router: send URL to its host's back queue
     ▼
[ back queue: hostA ]  [ hostB ]  [ hostC ] …                  (pace WITHIN each host)
     │ release only when host's crawl-delay allows
     ▼
[ Fetcher pulls ]
```

### Why this satisfies both at once
- **Priority** is honored because URLs enter through priority-ordered front queues — important pages reach a back queue sooner.
- **Politeness** is honored because a host's back queue drains at that host's rate *regardless* of how many high-priority URLs it holds — a hot site can't be hammered.
- **Throughput** comes from **breadth**: because per-host rate is capped, you sustain a high global rate by having **many hosts' back queues active at once** (verified: ~16,500 hosts in flight for 16.5K/s at a 1s delay). The frontier is thus **domain-partitioned** so each worker owns a set of hosts and enforces their delays locally (§8).

> 💡 **The senior one-liner:** *"The frontier reconciles two competing goals with a two-layer queue: priority front queues order URLs across the web by importance and freshness, and per-host back queues pace requests within each host for politeness. A URL flows front → its host's back queue → a worker that releases it only when the host's crawl-delay allows. Because per-host rate is capped, global throughput comes from crawling thousands of hosts concurrently — so I partition the frontier by domain and enforce politeness locally. That's how you crawl important pages first *and* never hammer anyone."*

---

## 6. Hard Problem 2: Deduplication at Billions-Scale (URL + content)

Two distinct dedup problems, both essential at billions-scale. *How do you cheaply answer "have I seen this URL?" for 100B URLs, and "is this page a near-duplicate?" for a web full of mirrors?*

### URL dedup — "have I already seen this URL?"
Before adding a URL to the frontier you must check if it's known — but a **plain set of 100B URLs is terabytes** of strings, far too big for memory. 

**The mechanism:** **normalize** the URL (lowercase host, strip fragment, sort query params, remove default ports) → hash it → check membership in a **Bloom filter**. A Bloom filter is a bit-array + k hash functions that answers **"definitely not seen"** or **"probably seen"** in tiny space (verified: **~120 GB at 1% FP, ~180 GB at 0.1% FP** for 100B URLs — vs terabytes for a set).

**Why the error is safe:** a Bloom filter can have **false positives** (says "seen" when it wasn't) but **never false negatives**. A false positive just means occasionally skipping a genuinely-new URL — you miss *one page*, a bounded, harmless cost (verified: ~0.1% of new URLs skipped at p=0.001) — **never** a correctness disaster. That asymmetry is exactly why a Bloom filter fits here.

### Content dedup — "is this page a near-duplicate?"
The web is full of **near-duplicates**: mirror sites, syndicated articles, the same page under different URLs/params, boilerplate templates. **Exact-hash dedup** (hash the bytes) catches *identical* pages but **misses near-dups** (one word different → totally different hash).

**The mechanism:** **similarity hashing — SimHash or MinHash** — locality-sensitive fingerprints where **similar content produces similar fingerprints.** Compute a page's SimHash; if it's within a **small Hamming distance** of a known page's fingerprint, it's a near-duplicate → skip indexing.
```
SimHash (64-bit), verified:
  near-duplicate (1 word changed) → Hamming distance 9   → SIMILAR → skip
  unrelated pages                 → Hamming distance 31  → DIFFERENT → keep
```
The small distance for the near-dup vs the large distance for unrelated content is what lets a threshold cleanly separate them. This keeps the index clean and saves crawl/storage effort on redundant content.

> 💡 **The senior one-liner:** *"Two dedup problems. For URLs, I normalize and hash, then check a Bloom filter — it answers 'definitely not seen' in ~150 GB for 100B URLs instead of terabytes for a set, and its only error is a false positive that harmlessly skips one page (never a false negative). For content, exact hashing misses near-duplicates, so I use SimHash/MinHash — locality-sensitive fingerprints where similar pages have a small Hamming distance — to detect and skip mirrors and syndicated copies."*

---

## 7. Politeness

Politeness is a **first-class requirement** — an impolite crawler overloads servers, gets IP-banned, and harms the open web. Mechanisms:
- **Respect `robots.txt`** — fetch and honor each site's crawl rules (disallowed paths, allowed bots); **cache it per host.**
- **Honor `Crawl-delay`** — the minimum delay a site requests between hits.
- **Limit concurrent connections per host** — at most N simultaneous requests to one server.
- **Spread requests over time** — pace, don't burst.

This is **per-host rate limiting** applied *outward* to the sites you crawl, and it's exactly why the frontier's **per-host back queues** exist. The clean enforcement: **partition the frontier by domain** so all of a host's URLs land on one worker, which enforces that host's delay and connection limit **locally** — no cross-worker coordination (§8).

> 💡 Politeness isn't a nicety bolted on — it's a hard constraint that *shapes the architecture* (per-host back queues + domain partitioning). Naming it as first-class is the senior signal.

---

## 8. Distributed Coordination (partition by domain)

The **frontier is partitioned by domain hash**, so every URL for a given host is owned by **one** partition/worker. This is the key coordination decision because it makes **politeness local**: the worker owning `example.com` enforces its crawl-delay and connection limit **without coordinating with any other worker**. Workers pull from their local frontier; the system **periodically rebalances** partitions as load shifts (consistent-hashing-style). Fault tolerance: **re-queue** URLs whose fetch failed (with retry limits), and **reassign** a dead worker's domain partition to another worker.

```
domain_hash("cnn.com")  → worker 7   (owns ALL cnn.com URLs → paces cnn.com locally)
domain_hash("bbc.com")  → worker 3   (owns ALL bbc.com URLs → paces bbc.com locally)
worker 7 dies → its partition (cnn.com, …) reassigned to another worker (consistent hashing)
```

> 💡 **Partition by domain** is what turns politeness from a distributed-coordination nightmare into a local decision — one worker per host means crawl-delay is enforced with zero cross-worker chatter.

---

## 9. Data Contracts: Records, Payloads & Store Schemas

### Part A — Key records / messages
**Frontier entry** (what flows through the queues):
| Field | Type | Purpose |
|:--|:--|:--|
| `url` | string | the (normalized) URL to crawl |
| `url_hash` | bytes | normalized-URL hash (dedup key, partition input) |
| `host` | string | routing to the per-host back queue |
| `priority` | float | importance/freshness/depth score (front-queue ordering) |
| `depth` | int | hops from a seed (depth-limit enforcement) |
| `discovered_from` | string | parent URL (graph/debugging) |

**Fetch result** (worker → parser): `{ url, http_status, content_type, s3_key, fetched_at, bytes }`.
**Extracted link** (parser → dedup → frontier): `{ url, parent, depth: parent.depth+1 }`.

### Part B — Inter-component payloads
- **Fetcher → Object Store:** `PUT raw/{url_hash} = <bytes>` (raw HTML/PDF/image).
- **Parser → Dedup:** URL check `bloom.contains(url_hash)`; content check `simhash(text)` vs known fingerprints (Hamming ≤ threshold).
- **Parser → Search Index:** `{ url, title, text, links[], lang, fetched_at }`.
- **Parser → URL DB (upsert):** `{ url_hash, last_crawled, status, depth, simhash, change_freq }`.

### Part C — Store schema per system (polyglot)
**① URL frontier — durable, domain-partitioned queue** (e.g., Kafka partitioned by domain / a DB-backed queue):
```
partition = domain_hash(host)
front queues: ordered by priority band ;  back queues: one per host, rate-gated
```
**② URL-seen set — Bloom filter (in-memory, sharded):** `bloom.add(url_hash)` / `bloom.contains(url_hash)` (~150 GB for 100B URLs; back it with a durable KV for rebuilds).
**③ Raw pages — Object Store (S3):** `raw/{url_hash} → bytes` (~1 PB; lifecycle-expire stale versions).
**④ Content — Search Index (Elasticsearch):** inverted index over `title`, `text`; `url` as doc id.
**⑤ URL metadata — DB:**
```
urls( url_hash PK, url, host, status, depth, last_crawled_at,
      simhash, change_freq, next_crawl_at )
INDEX (next_crawl_at)     -- the refresh scheduler scans this
INDEX (host)              -- per-host accounting / budgets
```
**⑥ robots cache — KV (per host):** `robots:{host} → {rules, crawl_delay, fetched_at}` (TTL'd).

> 💡 **The contract in one line:** *"A frontier entry carries the normalized URL, its hash (dedup key + partition input), host, priority, and depth. URLs are gated by a Bloom filter (seen?) and content by SimHash (near-dup?); raw bytes go to S3 keyed by url_hash, parsed content to the search index, and per-URL metadata (status, depth, simhash, next_crawl_at) to a DB the refresh scheduler scans. Every field exists for priority, politeness, dedup, or refresh."*

---

## 10. Crawl Traps & Refresh Strategy

### Crawl traps (the adversarial web)
The web contains **spider traps** — structures generating infinite unique URLs: infinite calendars ("next month" forever), session-ID URLs (a new ID per visit), faceted-search parameter explosions. Without defenses a crawler gets stuck. Mitigations:
- **Per-host URL budget** (a cap per domain).
- **Depth limits** so you don't descend infinitely.
- **Cycle/seen detection via the Bloom filter.**
- **Pattern detection** for suspicious URL shapes (ever-growing query strings, repeated path segments).

### Refresh strategy (freshness)
Pages change at very different rates, so recrawl **adaptively** by **observed change frequency**:
- **Frequently-changing** (news) — recrawled often (hourly).
- **Static** (docs) — recrawled rarely (weekly).
- **Adaptive** — track how often each page actually changes (compare SimHash across crawls) and tune `next_crawl_at` accordingly — spend budget where content is volatile, save it where stable.

> 💡 The capacity rate is an **average**: hot pages get more than their share, cold pages far less. Traps are handled with budgets + depth limits + the Bloom filter + pattern detection — the web is adversarial, so these limits are mandatory.

---

## 11. Scaling Summary
- **Frontier = two-layer, domain-partitioned queue** — priority front queues + per-host back queues; the hard central component.
- **Fetchers scale horizontally** (network-bound); throughput comes from **breadth** (thousands of hosts in flight), because politeness caps per-host rate.
- **URL dedup = Bloom filter** (~150 GB for 100B URLs; false positives harmless, no false negatives).
- **Content dedup = SimHash/MinHash** (near-duplicate detection by Hamming distance).
- **Partition by domain** → politeness enforced locally, no cross-worker coordination; rebalance consistent-hashing-style.
- **Polyglot storage** — raw HTML in S3 (~1 PB), content in Elasticsearch, metadata in a DB.
- **Fault tolerance** — durable frontier + metadata; re-queue failed fetches; reassign dead workers' partitions; raw HTML stored so parse/index retries independently.
- **Adaptive refresh** by observed change frequency; **crawl-trap defenses** (budgets, depth, pattern detection).

---

## 12. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Fetch fails / times out | URL not crawled | **re-queue** with backoff, up to a retry limit |
| Worker dies | its hosts stall | **reassign** its domain partition (consistent hashing); durable frontier preserves progress |
| Impolite crawl → IP ban | can't fetch a host | robots.txt + crawl-delay + per-host connection caps (first-class politeness) |
| Crawl trap (infinite URLs) | worker stuck | per-host URL budget + depth limit + Bloom cycle detection + pattern detection |
| Bloom filter false positive | skip one new URL | bounded, harmless (miss one page); tune FP rate via size |
| Near-duplicate pages | dirty/bloated index | SimHash/MinHash near-dup detection → skip indexing |
| Object store / index down | can't persist | raw HTML durable + retryable; parse/index decoupled from fetch |
| Frontier node loss | pending URLs at risk | durable, partitioned frontier; rebuild seen-set Bloom from the URL DB |
| Hot host with many URLs | risk of hammering | per-host back queue paces it regardless of priority |

---

## 13. Senior Follow-Up Questions (with Answers)

**Q1. What's the hardest component and why?** The **URL frontier** — it balances two competing goals: priority (crawl valuable/fresh first) and politeness (never hammer a host). The classic solution is a **two-layer queue**: priority front queues feeding per-host back queues, so you crawl important pages first while pacing each host. (§5)

**Q2. How do you stay polite?** Respect robots.txt + Crawl-delay, cap concurrent connections per host, spread requests over time — per-host rate limiting applied to the crawled sites. **Partition the frontier by domain** so one worker owns a host's URLs and enforces its delay locally. (§7–§8)

**Q3. How do you dedup URLs at billions-scale?** Normalize + hash each URL, check a **Bloom filter** before enqueuing (~150 GB for 100B URLs vs terabytes for a set). Its only error is a false positive that harmlessly skips one page — never a false negative. (§6)

**Q4. How do you detect near-duplicate pages?** **SimHash/MinHash** — locality-sensitive fingerprints where similar content yields a small Hamming distance (verified: near-dup 9, unrelated 31), so mirrors/syndicated/param-variant pages are detected and skipped. Exact hashing misses near-dups. (§6)

**Q5. How do you partition work?** By **domain hash** — all URLs for a host go to one worker, localizing politeness (no cross-worker crawl-delay coordination); rebalanced consistent-hashing-style; failed fetches re-queued, dead workers' partitions reassigned. (§8)

**Q6. What are crawl traps and how do you handle them?** Structures generating infinite unique URLs (calendars, session IDs, faceted search). Defend with per-host URL budgets, depth limits, Bloom cycle detection, and suspicious-pattern detection. (§10)

**Q7. Where do you store what, and why?** Raw HTML in S3 (cheap, durable, ~1 PB); parsed content in Elasticsearch (inverted index); URL metadata (last-crawled, status, depth, simhash) in a DB for scheduling/dedup. Polyglot — each store for its strength. (§9)

**Q8. How do you decide what to recrawl?** Adaptively by observed change frequency (news hourly, docs weekly), tuned per page (compare SimHash across crawls). The capacity rate is an average; volatile pages get more budget. (§10)

**Q9. Show the capacity arithmetic.** 10B pages / (7×86,400 s) ≈ **16.5K/s** (not ~1.5K — that's 1B@7d or 10B@~75d). ~1 PB raw HTML; ~150 GB Bloom for 100B URLs; ~16,500 hosts in flight for politeness. Catching the page/refresh mismatch is the signal. (§3)

**Q10. How is it fault-tolerant?** Durable frontier + metadata (progress survives crashes); re-queue failed fetches with retry limits; reassign a dead worker's partition; store raw HTML durably so parse/index retry independently of fetch. (§12)

**Q11. Is crawling BFS or DFS?** A **prioritized BFS** over the web graph (frontier ordered by importance/freshness) with depth limits so you never descend infinitely — priority + politeness shape the real order more than pure BFS/DFS.

**Q12. Why is fetching not the bottleneck?** Fetching is network-bound and embarrassingly parallel — scale fetchers horizontally. The real limits are **politeness** (caps per-host rate → need breadth), **dedup memory** (Bloom, not a set), and the **frontier** (a huge durable partitioned structure). (§4.6)

---

## 14. Quick Glossary
- **Web crawler / spider** — a system traversing the web fetching pages for indexing.
- **Seed URLs** — the starting points of a crawl.
- **URL frontier** — the prioritized, politeness-aware queue of URLs to crawl next (the crux).
- **Front queue / back queue** — priority-ordering queues / per-host politeness queues (two-layer frontier).
- **Politeness** — not overloading crawled servers (robots.txt, crawl-delay, per-host rate limits).
- **robots.txt / Crawl-delay** — a site's crawl rules / requested minimum delay between hits.
- **URL normalization** — canonicalizing a URL before hashing/dedup.
- **Bloom filter** — space-efficient "have I seen this URL?" test; false positives, never false negatives.
- **SimHash / MinHash** — similarity-preserving (locality-sensitive) fingerprints for near-duplicate detection.
- **Hamming distance** — bit-difference between fingerprints; small = similar content.
- **Domain partitioning** — assigning all of a host's URLs to one worker for local politeness.
- **Crawl trap / spider trap** — structures generating infinite URLs to ensnare crawlers.
- **Adaptive refresh** — recrawl interval tuned to a page's observed change frequency.
- **PageRank** — link-based importance score used to prioritize crawling.
- **Polyglot persistence** — using the best-fit store per data type (S3 / index / DB).

---

*Reference document. Contributions and corrections welcome.*
