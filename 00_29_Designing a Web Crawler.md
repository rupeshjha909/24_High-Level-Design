# Designing a Web Crawler: A Senior Interview Guide

> A practical, interview-focused reference for designing a large-scale web crawler — a polite, fault-tolerant, distributed traversal of the web graph that feeds a search index. Covers the URL frontier (the crux), politeness, URL and content deduplication, crawl traps, distributed coordination, and refresh strategy. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (with a Correction)](#3-capacity-estimation-with-a-correction)
4. [The Crawl Loop](#4-the-crawl-loop)
5. [The URL Frontier (the Crux)](#5-the-url-frontier-the-crux)
6. [Politeness](#6-politeness)
7. [Deduplication: URL and Content](#7-deduplication-url-and-content)
8. [Storage & Distributed Coordination](#8-storage--distributed-coordination)
9. [Crawl Traps & Refresh Strategy](#9-crawl-traps--refresh-strategy)
10. [Senior Follow-Up Questions (with Answers)](#10-senior-follow-up-questions-with-answers)
11. [Quick Glossary](#11-quick-glossary)

---

## 1. How to Approach This in an Interview

A web crawler is essentially a **massive, prioritized, polite breadth-first traversal of the web graph** that gathers content for a search index. It looks simple ("fetch a page, extract its links, repeat") but three things make it a serious distributed-systems problem, and they're where you should spend your time:

1. **The URL frontier** — deciding *what to crawl next* while balancing **priority** (crawl important/fresh pages first) against **politeness** (don't hammer any one server). This is the crux.
2. **Deduplication at scale** — with billions of URLs and pages, you must cheaply answer "have I seen this URL/content?" — the canonical **Bloom filter** application (Bloom-filter guide).
3. **Politeness and the adversarial web** — respecting `robots.txt` and rate limits, and defending against infinite "crawl traps."

The structure: requirements → scale → the crawl loop → go deep on the **frontier**, **politeness**, and **dedup** → coordination, traps, refresh. Senior signal: treating politeness as a first-class constraint (not an afterthought) and recognizing the frontier as the hard, central component.

---

## 2. Requirements

### Functional

- Given **seed URLs**, crawl the web.
- **Extract content** and store it for indexing.
- **Respect `robots.txt`.**
- **Avoid duplicates.**
- **Configurable crawl depth.**

### Non-Functional

- **Scalable** — billions of URLs.
- **Polite** — don't overload the servers being crawled.
- **Fault-tolerant** — survive worker/fetch failures.
- **Extensible** — handle HTML, PDFs, images, etc.

---

## 3. Capacity Estimation (with a Correction)

The target: index a large corpus and refresh it on a cadence. Doing the arithmetic:

```
10B pages, refresh every 7 days:
   10 × 10⁹ / (7 × 86,400 s)  ≈  10 × 10⁹ / 604,800  ≈  ~16,500 pages/sec
```

> **Correction worth noting:** 10B pages on a 7-day refresh needs **~16.5K pages/sec**, not ~1.5K. The ~1.5K/sec figure corresponds to **~1B pages** refreshed weekly (1 × 10⁹ / 604,800 ≈ 1,653/sec), or to 10B pages refreshed every ~75 days. Either is fine — just keep the page count and refresh window consistent with the rate. In an interview, *showing this arithmetic* (and catching the mismatch) is itself a strong signal.

**Takeaways either way:**
- Even ~1.5K/sec sustained requires **many distributed fetcher workers** (each fetch is network-bound and slow), so the crawler is **horizontally scaled and I/O-bound**.
- Billions of URLs mean the "have I seen this?" set can't fit in memory as a plain set → **Bloom filter**.
- Raw HTML at this volume → cheap **object storage**, not a database.

---

## 4. The Crawl Loop

```
[Seed URLs]
     │
     ▼
[URL Frontier (priority queue)] ◄───────────────┐
     │                                           │
     ▼                                           │
[Fetcher Workers] ──► [Raw HTML → S3]            │
     │                                           │
     ▼                                           │
[Parser] ─┬─► [Content Extractor] ─► [Search Index (ES)]
          ├─► [URL Extractor] ─► [Dedup Filter] ─► back to Frontier ─┘
          └─► [Indexer]
```

The cycle: pull a URL from the **frontier** → **fetch** the page → store raw HTML → **parse** it → extract **content** (to the search index) and extract **new URLs** → **dedup** those URLs → feed unseen ones back into the frontier. It's a **prioritized BFS over the web graph**, with depth limits to avoid descending forever, and it feeds a search index (the data-gathering front end of a search engine — conceptually the build side, like the autocomplete pipeline).

---

## 5. The URL Frontier (the Crux)

The frontier decides *what to crawl next*, and it must satisfy **two competing goals simultaneously:**

- **Priority** — crawl the most valuable/fresh pages first (ranked by **PageRank/importance, freshness, and depth**).
- **Politeness** — never hit a single host too fast, regardless of how many high-priority URLs that host has.

The classic design (Mercator-style) is a **two-layer queue:**
- **Front queues** order URLs by **priority** (which URLs deserve crawling soonest).
- **Back queues** are **per-host**, enforcing **politeness** (each host's URLs drain at a controlled rate, with a delay between requests).

A URL flows from a priority front queue into its host's back queue; workers pull from back queues respecting each host's rate. This separates the two concerns cleanly — you can crawl important pages first *and* be polite to every host. Designing this balance is the heart of the crawler and the thing to emphasize.

---

## 6. Politeness

Politeness is a **first-class requirement**, not an afterthought — an impolite crawler overloads servers, gets IP-banned, and harms the open web. The mechanisms:

- **Respect `robots.txt`** — fetch and honor each site's crawl rules (disallowed paths, allowed bots). Cache it per host.
- **Honor `Crawl-delay`** — the delay a site requests between hits.
- **Limit concurrent connections per host** — at most N simultaneous requests to any one server.
- **Spread requests over time** — pace requests to a host rather than bursting.

This is **per-host rate limiting** (the rate-limiter guide, applied outward to the sites you crawl), and it's why the frontier's per-host back queues exist. A clean way to enforce it: **partition the frontier by domain** so all of a host's URLs land on one worker/partition, which can then enforce that host's delay and connection limits *locally* without cross-worker coordination (next section).

---

## 7. Deduplication: URL and Content

Two distinct dedup problems, both essential at billions-scale:

### URL deduplication — "have I already seen this URL?"

Normalize the URL (lowercase host, strip fragments, sort query params, etc.) and hash it; check membership in a **Bloom filter** before adding it to the frontier. This is the **canonical Bloom-filter use case** (the Bloom-filter guide literally lists web crawlers): at billions of URLs you can't hold them all in a set, but a Bloom filter answers "definitely not seen" in tiny space. A **false positive** just means occasionally skipping a genuinely-new URL — a minor, acceptable miss (you don't crawl one page), never a correctness disaster, because the cost of the error is bounded.

### Content deduplication — "is this page a near-duplicate?"

The web is full of **near-duplicates**: mirror sites, syndicated articles, the same page under different URLs/params. Exact-hash dedup catches identical bytes but misses near-dups. The fix is **similarity hashing — SimHash or MinHash** — fingerprints designed so that *similar content produces similar hashes* (locality-sensitive hashing), letting you detect and skip near-duplicate pages. This keeps the index clean and saves crawl/storage effort on redundant content.

---

## 8. Storage & Distributed Coordination

### Storage (polyglot, tiered)

- **Raw HTML → S3** — cheap, durable object storage for the bulky fetched pages (storage-types guide).
- **Parsed content → search index (Elasticsearch)** — the inverted index queries run against (search/autocomplete guides).
- **URL metadata → a database** — last-crawled time, status, depth, etc., for scheduling and dedup bookkeeping.

Each store does what it's best at — textbook polyglot persistence.

### Distributed coordination

The **URL frontier is partitioned by domain hash**, so every URL for a given host is owned by one partition/worker. This is the key coordination decision because it makes **politeness local**: the worker owning a domain enforces that host's crawl-delay and connection limit without coordinating with others. Workers pull from their local frontier and the system **periodically rebalances** partitions as load shifts (consistent-hashing-style partitioning, from the consistent-hashing and database-scaling guides). Fault tolerance comes from re-queuing URLs whose fetch failed and reassigning a dead worker's partition.

---

## 9. Crawl Traps & Refresh Strategy

### Crawl traps (the adversarial web)

The web contains **spider traps** — structures that generate infinite unique URLs: infinite calendars ("next month" forever), session-ID URLs (a new ID per visit), faceted-search parameter explosions. Without defenses a crawler gets stuck. Mitigations:

- **Limit URLs crawled per host** (a budget per domain).
- **Depth limits** so you don't descend infinitely.
- **Cycle/seen detection via the Bloom filter** to avoid revisiting.
- **Pattern detection** for suspicious URL shapes (e.g., ever-growing query strings).

### Refresh strategy (freshness)

Pages change at very different rates, so recrawl **adaptively** based on **observed change frequency:**
- **Frequently-changing sites** (news) — recrawled often (e.g., hourly).
- **Static sites** (docs) — recrawled rarely (e.g., weekly).
- **Adaptive** — track how often each page actually changes and tune its recrawl interval accordingly, spending crawl budget where content is volatile and saving it where content is stable.

This makes the refresh rate from the capacity estimate an *average* — hot pages get more than their share, cold pages far less.

---

## 10. Senior Follow-Up Questions (with Answers)

**Q1. What's the hardest component and why?**
The URL frontier. It must balance two competing goals: priority (crawl valuable/fresh pages first) and politeness (never hammer a host). The classic solution is a two-layer queue — priority front queues feeding per-host back queues — so you crawl important pages first while pacing each host.

**Q2. How do you stay polite?**
Respect robots.txt and Crawl-delay, cap concurrent connections per host, and spread requests over time — per-host rate limiting applied to the sites you crawl. Partitioning the frontier by domain makes this enforceable locally, since one worker owns all of a host's URLs.

**Q3. How do you deduplicate URLs at billions-scale?**
Normalize and hash each URL and check a Bloom filter before enqueuing — the canonical Bloom-filter use. It answers "definitely not seen" in tiny space; a false positive only skips one new URL occasionally, which is harmless. A plain set of billions of URLs wouldn't fit in memory.

**Q4. How do you detect near-duplicate pages?**
Similarity hashing — SimHash/MinHash — produces fingerprints where similar content yields similar hashes (locality-sensitive hashing), so mirrors, syndicated copies, and param-variant pages are detected and skipped. Exact content hashing alone misses near-dups.

**Q5. How do you partition work across machines?**
Partition the frontier by domain hash so all URLs for a host go to one worker/partition. This localizes politeness enforcement (no cross-worker coordination for crawl-delay) and is rebalanced consistent-hashing-style as load shifts. Failed fetches are re-queued; a dead worker's partition is reassigned.

**Q6. What are crawl traps and how do you handle them?**
Structures generating infinite unique URLs — infinite calendars, session-ID URLs, faceted-search explosions. Defend with per-host URL budgets, depth limits, Bloom-filter cycle detection, and suspicious-URL-pattern detection. The web is adversarial, so these limits are mandatory.

**Q7. Where do you store what, and why?**
Raw HTML in S3 (cheap, durable, bulky); parsed content in Elasticsearch (inverted index for search); URL metadata (last-crawled, status, depth) in a database for scheduling/dedup. Polyglot persistence — each store for its strength.

**Q8. How do you decide what to recrawl and how often?**
Adaptively, by observed change frequency: news hourly, docs weekly, tuned per page based on how often it actually changes. The capacity rate is an average; volatile pages get more budget, stable pages far less.

**Q9. How does the crawler stay fault-tolerant?**
Durable frontier and metadata so progress survives crashes; re-queue URLs whose fetch fails (with retry limits); reassign a failed worker's domain partition; store raw HTML durably so parsing/indexing can be retried independently of fetching.

**Q10. Is crawling BFS or DFS?**
Effectively a prioritized BFS over the web graph (frontier ordered by importance/freshness), with depth limits to avoid going infinitely deep down any path. Priority + politeness shape the actual order more than pure BFS/DFS.

---

## 11. Quick Glossary

- **Web crawler / spider** — a system that traverses the web fetching pages for indexing.
- **Seed URLs** — the starting points of a crawl.
- **URL frontier** — the prioritized, politeness-aware queue of URLs to crawl next.
- **Politeness** — not overloading crawled servers (robots.txt, crawl-delay, rate limits).
- **robots.txt** — a site's file declaring crawl rules for bots.
- **Crawl-delay** — requested minimum delay between requests to a host.
- **Per-host queue (back queue)** — frontier queue enforcing one host's crawl rate.
- **Priority queue (front queue)** — frontier queue ordering URLs by importance/freshness.
- **URL normalization** — canonicalizing a URL before hashing/dedup.
- **Bloom filter** — space-efficient "have I seen this URL?" membership test.
- **SimHash / MinHash** — similarity-preserving fingerprints for near-duplicate detection.
- **Crawl trap / spider trap** — structures generating infinite URLs to ensnare crawlers.
- **Domain partitioning** — assigning all of a host's URLs to one worker for local politeness.
- **Adaptive refresh** — recrawl interval tuned to a page's observed change frequency.
- **PageRank** — a link-based importance score used to prioritize crawling.

---

*Reference document. Contributions and corrections welcome.*
