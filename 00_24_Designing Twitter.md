# Designing Twitter: A Senior Interview Guide

> A practical, interview-focused reference for designing a Twitter-scale social feed — dominated by one problem: generating home timelines for a massively read-heavy, fan-out-shaped workload. Covers push vs pull vs hybrid fan-out, the data model, search, trending, media, and the asymmetric-load reality. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Core Challenge: Home Timeline Generation](#4-the-core-challenge-home-timeline-generation)
5. [The Hybrid Fan-out Model (Twitter's Answer)](#5-the-hybrid-fan-out-model-twitters-answer)
6. [Architecture](#6-architecture)
7. [Data Model](#7-data-model)
8. [Fan-out & Timeline Read Paths](#8-fan-out--timeline-read-paths)
9. [Search, Trending & Media](#9-search-trending--media)
10. [Asymmetric Load & Optimizations](#10-asymmetric-load--optimizations)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This in an Interview

Twitter is the canonical large-scale design question, and almost the entire interview hinges on **one problem: how do you build each user's home timeline?** Everything else (storing tweets, the follow graph, search, media) is comparatively standard. So the winning approach is to set up the requirements and scale quickly, then **go deep on timeline generation** — specifically the **fan-out** trade-off (push vs. pull vs. hybrid) and the **celebrity problem** that breaks the naive answers.

Two facts drive every decision and should anchor your reasoning:
1. **It's extraordinarily read-heavy** (~100:1, and home-timeline views dwarf tweets), so you want to **do expensive work at write time, not read time.**
2. **Eventual consistency is acceptable** (a reply showing up a couple seconds late is fine), which *licenses* async fan-out, caching, and precomputation.

Senior signal: recognizing the fan-out cost asymmetry (one tweet can require millions of writes), arriving at the **hybrid** model, and connecting it to the read-heavy + eventual-consistency facts.

---

## 2. Requirements

### Functional

- **Post tweets** (280 chars, optional media).
- **Follow** users.
- **Home timeline** — tweets from followed users, newest first.
- **User timeline** — a user's own tweets + retweets.
- **Like, retweet, reply, search.**
- **Trending topics, notifications.**

### Non-Functional

- **Read-heavy** (~100:1 read:write).
- **Low latency** — home timeline under ~200 ms.
- **Eventual consistency OK** — slight delays for replies/propagation are acceptable.
- **Scale** — 500M users, 500M tweets/day.

---

## 3. Capacity Estimation

```
Tweets/day        = 500M          → 500M / 10⁵ ≈ ~6,000 writes/sec ; peak ~18,000/sec
Home-timeline views ≈ 30B/day      → 30B / 10⁵  ≈ ~350,000 reads/sec
Text storage      = 280 B × 500M   = ~140 GB/day text ; ~1 PB/year with media
```

**Takeaways:**
- Read QPS (~350K/sec) is **~60× the write QPS** — Twitter is one of the most read-heavy systems you'll design. This is *why* the home timeline must be a cheap cache lookup, not a computation.
- ~1 PB/year (dominated by media) means **tiered storage** and object storage for blobs are mandatory.
- The asymmetry between cheap writes and enormous reads is the entire justification for **precomputing timelines at write time.**

---

## 4. The Core Challenge: Home Timeline Generation

Your home timeline is "the most recent tweets from everyone you follow, merged and sorted by time." There are two fundamental ways to produce it, and the choice defines the system.

### Pull Model — fan-out on read (compute on read)

```
on visit:  SELECT * FROM tweets
           WHERE author IN (followees of X)
           ORDER BY ts DESC LIMIT 50
```

Compute the timeline *when the user asks for it* by querying all their followees' recent tweets and merging.

- **Pros:** no precomputation; writes are trivial (just store the tweet).
- **Cons:** **reads are expensive** — a user following 1,000 accounts triggers a huge multi-source query-and-merge *on every timeline load*. At ~350K reads/sec, this is far too slow. It puts the heavy work on the **hot path** (reads), which is exactly backwards for a read-heavy system.

### Push Model — fan-out on write (precompute on tweet)

```
on tweet by U:
    for each follower F of U:
        redis.zadd("timeline:" + F, score=tweet.ts, value=tweet.id)
```

When someone tweets, immediately **insert it into every follower's precomputed timeline** (a Redis sorted set).

- **Pros:** the home-timeline read becomes a **single fast cache lookup** — perfect for a read-heavy system; the work moved to write time.
- **Cons:** the **celebrity problem** — a user with 100M followers generates **100M writes for one tweet**, a fan-out explosion that can overwhelm the system. It's also wasteful (fanning out to inactive followers who'll never look).

The tension is clear: pull makes reads too slow; push makes celebrity writes explode. Neither pure model works at Twitter scale.

---

## 5. The Hybrid Fan-out Model (Twitter's Answer)

The resolution — and the centerpiece of a strong answer — is to **choose push or pull *per author*, based on follower count:**

- **Regular users → push (fan-out on write).** Their tweets are fanned out into followers' precomputed timelines. Most users have manageable follower counts, so the fan-out is cheap and reads stay fast.
- **Celebrities (e.g., >1M followers) → pull (fan-out on read).** Their tweets are **not** fanned out (avoiding the millions-of-writes explosion). Instead, at read time, a user's precomputed timeline (from the people they follow who *aren't* celebrities) is **merged with a fresh pull of the recent tweets from the celebrities they follow.**

```
home_timeline(user) =
     merge_by_time(
        precomputed timeline (push, regular followees),   ← cheap cache read
        recent tweets pulled from celebrity followees      ← small pull + merge
     )[:50]
```

This neatly defeats both problems: regular reads are mostly a cache lookup (fast), and celebrity tweets avoid the write explosion by being pulled for the (relatively few) celebrities each user follows. **The decision is per-tweet, made on the cost asymmetry of fan-out** — the same push-vs-pull trade-off seen in the WhatsApp group-messaging design, but here it's the heart of the system.

---

## 6. Architecture

```
[Client] ─► [LB] ─► [API] ─┬─► [Tweet Service]    ─► [Cassandra]  (tweets)
                           ├─► [Timeline Service] ─► [Redis]      (precomputed timelines)
                           ├─► [Fan-out Service]  ─► (push to followers' timelines)
                           ├─► [User/Graph Service] ─► [Postgres / graph DB]  (follows)
                           └─► [Search Service]   ─► [Elasticsearch]
   media → [S3] → [CDN]              trending → [Flink stream aggregator]
```

- **Tweet Service** stores tweets (Cassandra).
- **Fan-out Service** pushes new tweets into followers' Redis timelines (for non-celebrities).
- **Timeline Service** reads precomputed timelines and merges in celebrity pulls.
- **User/Graph Service** manages the follow graph.
- **Search** (Elasticsearch) and **Trending** (Flink) run off the tweet stream; media goes to S3 + CDN.

---

## 7. Data Model

### Tweets — Cassandra (wide-column, write-heavy)

```
PRIMARY KEY ((user_id), tweet_id DESC)
columns: text, media_url, like_count, retweet_count, ts
```

Partitioning by `user_id` (sorted by `tweet_id` descending) makes a **user timeline a single-partition scan** — fast (same partition-key reasoning as the WhatsApp and KV-store guides). Cassandra suits the write-heavy, time-ordered tweet stream.

### Follow graph — relational or graph DB

Stored in PostgreSQL or a graph DB, and **indexed both directions**: "who I follow" (followees — needed for pull/read) and "who follows me" (followers — needed for fan-out/write). Both directions are required because fan-out iterates followers while timeline pulls iterate followees.

### Home timeline — Redis sorted sets

```
ZADD timeline:user_X  <score = tweet_ts>  <tweet_id>
```

A **sorted set per user**, scored by timestamp so `ZREVRANGE` returns newest-first in O(log N), with inserts in O(log N). **Trim to the last ~800 tweets** so timelines stay bounded in memory (nobody scrolls back thousands of tweets in the hot path). Storing tweet *IDs* (not full tweets) keeps the set small; hydrate the IDs from Cassandra/cache on read.

---

## 8. Fan-out & Timeline Read Paths

### Fan-out on a new tweet

```
on new tweet T by user U:
    followers = get_followers(U)
    if len(followers) < 1M:                       # regular user → PUSH
        for f in followers:
            redis.zadd("timeline:"+f, score=T.ts, value=T.id)
    else:                                          # celebrity → PULL
        mark T as a celebrity tweet (do NOT fan out)
```

Fan-out is done **asynchronously** (off a queue) so posting a tweet returns instantly and the writes drain in the background — eventual consistency in action, and exactly why a queue (message-queue guide) sits here.

### Reading the home timeline

```
def home_timeline(user):
    cached     = redis.zrevrange("timeline:"+user, 0, 49)         # push timeline
    celeb      = get_recent_celeb_tweets(user.celebrity_followees) # pull
    return merge_sorted_by_ts(cached, celeb)[:50]                  # hybrid merge
```

The read is dominated by the cheap cached lookup, plus a small pull-and-merge for the handful of celebrities the user follows — keeping it within the ~200 ms budget.

---

## 9. Search, Trending & Media

- **Search** — tweets are indexed into **Elasticsearch** on write (an inverted index, term → tweets — see the autocomplete and database-scaling guides). Search queries hit ES, not the primary tweet store.
- **Trending topics** — the tweet stream feeds a **real-time aggregator (Flink)** that counts hashtags/terms over a **rolling window** and surfaces the top ones — the same streaming-aggregation pattern as the autocomplete build pipeline. Trending tolerates slight staleness, so windowed batch counting is fine.
- **Media** — uploaded to **S3** (the URL stored in the tweet record), served via **CDN** (Cloudflare/Akamai) for low-latency global delivery — the media pattern from the WhatsApp, Pastebin, and CDN guides. Media never flows through the tweet path.

---

## 10. Asymmetric Load & Optimizations

The defining real-world property: **load is wildly asymmetric — the top ~1% of users generate ~90% of read traffic** (a steep power law). This shapes several optimizations:

- **Special-case hot users** — heavily-followed/heavily-read accounts get dedicated caching and the pull treatment, so they don't distort the system (the hot-key problem from the caching guide, at user granularity).
- **Tiered storage** — **hot** (Redis precomputed timelines) → **warm** (Cassandra tweets) → **cold** (S3 for media and old tweets) — matching storage cost to access frequency (storage-types guide).
- **Pre-warm caches** — predictively push timelines for active users during quiet hours so peak-time reads are already cached.
- **Bounded timelines** — trimming Redis sets to ~800 entries caps memory; deep scrolls fall back to computing from Cassandra.

These all flow from the same insight: in a read-heavy, power-law system, you invest in making the **common, hot read path** as cheap as possible and special-case the heavy hitters.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. How do you generate the home timeline at scale?**
Don't compute it on read (pull) — that's too slow for a read-heavy system. Precompute it on write (push) into per-user Redis timelines so reads are a cache lookup. But pure push breaks on celebrities (a tweet → millions of writes), so use a **hybrid**: push for regular users, pull for celebrities, merging both at read time.

**Q2. What's the celebrity problem and how does hybrid solve it?**
Fan-out on write means a user with 100M followers triggers 100M writes per tweet — an explosion. Hybrid stops fanning out celebrity tweets; instead, at read time, each user's precomputed timeline is merged with a fresh pull of recent tweets from the (few) celebrities they follow. Decision is per-author based on follower count.

**Q3. Why precompute timelines instead of querying on demand?**
Because reads vastly outnumber writes (~350K reads/sec vs ~6K writes/sec) and are latency-critical (<200 ms). You move expensive work to the rare write path so the frequent read path stays cheap — a single cache lookup. Eventual consistency makes async precomputation acceptable.

**Q4. Why Redis sorted sets for timelines, and why trim them?**
Score = timestamp gives newest-first ordering with O(log N) inserts and range reads; store tweet IDs (not full tweets) to stay small. Trim to ~800 entries because nobody scrolls thousands of tweets on the hot path — this bounds memory; deeper history falls back to Cassandra.

**Q5. Why partition tweets by user_id in Cassandra?**
It co-locates a user's tweets in one partition, making a user timeline a fast single-partition scan, and Cassandra's write-optimized engine suits the high, time-ordered write rate. Same partition-key reasoning as other write-heavy designs.

**Q6. Why index the follow graph in both directions?**
Fan-out (write) needs a user's *followers*; timeline pull (read) needs a user's *followees*. Both queries are hot, so both directions are indexed.

**Q7. How does fan-out stay fast for the user posting a tweet?**
It's asynchronous — the tweet is stored and acknowledged immediately, and a queue/worker performs the fan-out writes in the background. The poster never waits for millions of timeline inserts; eventual consistency covers the brief propagation delay.

**Q8. How do search and trending work?**
Search indexes tweets into Elasticsearch (inverted index) on write; queries hit ES. Trending streams the tweet firehose into a real-time aggregator (Flink) that counts hashtags over a rolling window — the same streaming pattern as autocomplete, and it tolerates slight staleness.

**Q9. What's the consistency model and is it acceptable?**
Eventually consistent (AP-leaning): timelines, counts, and replies may lag a few seconds. That's the right trade — users vastly prefer fast, always-available timelines over perfectly instantaneous consistency, and it's what enables async fan-out and heavy caching.

**Q10. How do you handle the load asymmetry?**
The top ~1% of users drive ~90% of reads, so special-case them (dedicated caching, pull treatment), use tiered storage (Redis → Cassandra → S3), pre-warm active users' caches in quiet hours, and bound timeline sizes. Optimize the hot common path; isolate the heavy hitters.

---

## 12. Quick Glossary

- **Home timeline** — the merged, time-sorted feed of tweets from accounts a user follows.
- **User timeline** — a single user's own tweets and retweets.
- **Fan-out on write (push)** — pushing a new tweet into all followers' precomputed timelines.
- **Fan-out on read (pull)** — assembling the timeline at read time by querying followees.
- **Hybrid fan-out** — push for regular users, pull for celebrities; merge at read time.
- **Celebrity problem** — a high-follower user's tweet causing a fan-out write explosion.
- **Followees / followers** — accounts a user follows / accounts that follow a user.
- **Redis sorted set (ZSET)** — score-ordered set; used for time-ordered timelines.
- **Timeline trimming** — bounding a cached timeline to the most recent N entries.
- **Inverted index** — term → list of tweets; powers search (Elasticsearch).
- **Rolling-window aggregation** — counting over a recent time window (trending via Flink).
- **Tiered storage** — hot (Redis) → warm (Cassandra) → cold (S3) by access frequency.
- **Asymmetric load** — a small fraction of users generating most traffic (power law).
- **Eventual consistency** — accepting brief propagation delays for availability and speed.

---

*Reference document. Contributions and corrections welcome.*
