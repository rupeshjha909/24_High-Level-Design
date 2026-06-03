# Designing Facebook News Feed: A Senior Interview Guide

> A practical, interview-focused reference for designing a personalized social feed — sharing the fan-out feed-generation core of Twitter and Instagram, but distinguished by a rich social-graph model (objects + edges), the TAO graph-caching layer, ML ranking as the central differentiator, and a nuanced privacy/audience model. With trade-offs and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This (and What's Genuinely New)](#1-how-to-approach-this-and-whats-genuinely-new)
2. [Requirements](#2-requirements)
3. [Feed Generation: Same Core as Twitter/Instagram](#3-feed-generation-same-core-as-twitterinstagram)
4. [Feed Ranking — The Secret Sauce](#4-feed-ranking--the-secret-sauce)
5. [Architecture](#5-architecture)
6. [The Social Graph & TAO](#6-the-social-graph--tao)
7. [Data Models: Objects & Edges](#7-data-models-objects--edges)
8. [Privacy, Edge Cases & Caching](#8-privacy-edge-cases--caching)
9. [Senior Follow-Up Questions (with Answers)](#9-senior-follow-up-questions-with-answers)
10. [Quick Glossary](#10-quick-glossary)

---

## 1. How to Approach This (and What's Genuinely New)

Facebook's News Feed shares its **feed-generation core with Twitter and Instagram** — the same hybrid push/pull fan-out and the same read-heavy, eventually-consistent profile. So, as with those designs, state that parallel fast and don't re-derive fan-out. Spend your time on the **three things that distinguish Facebook:**

1. **The social graph and TAO** — Facebook models *everything* as **objects and edges** (users, posts, likes, friendships) and serves the colossal graph read volume through **TAO**, a custom graph-aware caching layer over sharded MySQL. This is the signature infrastructure.
2. **ML ranking as the central product** — even more than Instagram, the feed's value *is* the ranking ("the secret sauce"): hundreds of signals into a deep model.
3. **A rich privacy/audience model** — posts can be Public, Friends-only, or custom audiences, and the feed must enforce that — more nuanced than Twitter's mostly-public model.

Senior signal: recognizing that the feed *plumbing* is the solved Twitter/Instagram problem, while the **graph serving layer (TAO)**, the **ranking model**, and **privacy enforcement** are where Facebook's real engineering and differentiation live.

---

## 2. Requirements

### Functional

- Show a **feed of friends' posts.**
- **Personalized ranking** (not strictly chronological).
- Post types: **text, photo, video, link, share.**
- **Comments, reactions.**
- **Stories** (24h ephemeral).

### Non-Functional

- **Read-heavy** (~50:1, and graph reads far more).
- **Low latency** — under ~200 ms.
- **High availability.**
- **Massive scale** — 3B+ users.

---

## 3. Feed Generation: Same Core as Twitter/Instagram

The fundamental problem — *assemble a personalized feed at request time from many sources* — is identical to Twitter and Instagram, and so is the solution: **hybrid fan-out.**

- **Fan-out on write (push)** — when a user posts, write it into friends' precomputed feed lists. Read = a single cache lookup. Heavy on writes for highly-connected users.
- **Fan-out on read (pull)** — on a feed request, query each friend's recent posts. Light writes, heavy reads.
- **Hybrid (Facebook's actual approach)** — push for users with manageable connection counts (most people); pull for highly-connected entities (celebrities, public pages with millions of followers), merged at read time.

A Facebook-specific nuance: the **friend graph is bidirectional and bounded** (a cap of ~5,000 friends), so fan-out-on-write is tractable for ordinary users — there's no 100M-follower explosion for *friends*. The pull treatment is reserved for **public figures and Pages** with unbounded followers. See the Twitter guide for the full fan-out derivation; here, just note the bounded-friend-graph difference.

---

## 4. Feed Ranking — The Secret Sauce

This is where Facebook's News Feed earns its name and its value — the feed is **ranked by a machine-learning model, not chronological**, and the ranking is arguably *the* product:

- For each `(user, candidate post)` pair, compute **hundreds of features**: recency, **affinity** (how much this user interacts with the author), post type, content quality, and **predicted engagement / click-through rate**.
- These feed a **deep learning model** that scores each candidate; the feed shows the highest-scoring posts.

The pipeline (as in Instagram, but with more signals and weight): **candidate generation** (gather posts from friends and followed pages) → **ranking** (score the candidates with the ML model) → **aggregation** (assemble the final ordered feed, interleaving ads). To hit the <200 ms budget, the candidate set is **pre-narrowed** and expensive features (like **affinity scores**) are **precomputed offline** (e.g., daily) and looked up at request time. Results are **cached per user and refreshed every few minutes.** The shift from chronological to ML-ranked is the defining modern-feed change, and Facebook pushes it furthest.

---

## 5. Architecture

```
[Client] ─► [Edge / Load Balancer]
             ├─► [Feed Service]
             │     ├─► [Candidate Generation]
             │     │     ├─► [Friends' recent posts (cache)]
             │     │     └─► [Followed Pages' posts]
             │     ├─► [Ranking Service (ML)]
             │     └─► [Feed Aggregator]
             ├─► [Post Service]     ─► [TAO / Cassandra]
             ├─► [Reaction Service]
             ├─► [Comment Service]
             └─► [Ad Service]       (interleave ads into the feed)
```

The Feed Service orchestrates **candidate generation → ML ranking → aggregation**; the **Ad Service** interleaves ads (a Facebook-specific monetization step); and everything reads the social graph through **TAO**.

---

## 6. The Social Graph & TAO

At its heart, Facebook is a **social graph**: a vast set of entities and the relationships between them. Almost every feed operation is a graph read — "who are my friends," "what did they post," "who reacted to this," "what's the comment thread." This is conceptually a **graph database problem** (SQL-vs-NoSQL guide), but the *read volume* is so enormous (far beyond the 50:1 headline, since each feed load fans out into many graph reads) that it demands aggressive caching.

**TAO (The Associations and Objects)** is Facebook's answer: a **custom, graph-aware caching layer over sharded MySQL.**

- It stores **objects** (posts, comments, users — the nodes) and **associations/edges** (friendships, likes, comments — the relationships).
- It's **read-optimized and eventually consistent**, tuned for the social graph's access pattern (lots of "fetch this object" and "fetch the edges of type X from this node" queries).
- MySQL is the durable backing store; TAO is the high-throughput cache in front of it, serving the overwhelming read load close to the request path.

TAO is the distinctive Facebook infrastructure piece — a graph abstraction served at massive scale by treating the problem as **cache the objects and edges**, rather than running a pure graph database at 3B-user volume.

---

## 7. Data Models: Objects & Edges

Everything reduces to two primitives:

### Object (a node)

```
Post: { post_id, user_id, content_type, text, media_url, ts, audience }
```

(Users, comments, photos are also objects.) Note the **`audience`** field — central to privacy (next section). Media is referenced by URL (stored in object storage + CDN, as in the Instagram/Twitter designs).

### Edge / Association (a relationship)

```
(from_id, edge_type, to_id)
   e.g.  (alice, friend, bob)
         (alice, likes, post123)
         (alice, comment, post456)
```

This object+edge model *is* the social graph, and TAO is built precisely to serve it. A feed query becomes a series of graph traversals — fetch a user's `friend` edges, fetch those friends' recent post objects, fetch reaction/comment edges — all served from TAO's cache.

---

## 8. Privacy, Edge Cases & Caching

### Privacy / audience (richer than Twitter)

Every post carries an **audience**: Public, Friends-only, or a **custom audience**. Feed generation must **enforce this** — a Friends-only post must never surface to a non-friend. So audience is checked during candidate generation/ranking (access control woven into the feed, akin to the ACL enforcement in the Drive/Pastebin designs, but applied per-post at feed-build time). This is more complex than Twitter's largely-public model and is a correctness-and-trust requirement, not an afterthought.

### Edge cases

- **New post** — must appear in active friends' feeds **within seconds** (timely fan-out / cache update).
- **Deleted post** — must **disappear from all caches** promptly (cache invalidation, from the caching guide — a deletion must propagate so stale feeds don't keep showing it).
- **Privacy changes** — audience edits must be respected on subsequent reads.

### Caching strategy

- **Hot (viral) posts** are heavily cached at every layer.
- **Affinity scores precomputed daily** — the expensive ranking features are computed offline and looked up at request time.
- **Fan-out to *active* users only** — when a user posts, push to the feeds of friends who are *active*; **inactive users' feeds are computed lazily on their next visit.** This is a key optimization (the active-user fan-out / pre-warm idea from the caching and Twitter guides): you don't waste fan-out work pushing to feeds nobody will look at soon, which dramatically cuts write amplification at 3B-user scale.

---

## 9. Senior Follow-Up Questions (with Answers)

**Q1. How is this different from Twitter/Instagram?**
The feed-generation core (hybrid fan-out) and ML ranking are the same. The differences: a bidirectional, bounded *friend* graph (so push works for most users; pull only for public figures/Pages), the object+edge social-graph model served by TAO (a graph-aware cache over MySQL), an even more central ML ranking, and a richer privacy/audience model.

**Q2. What is TAO and why does Facebook need it?**
TAO is a custom graph-aware caching layer over sharded MySQL that stores objects (posts, users) and edges (friendships, likes). Feed operations are overwhelmingly graph reads at enormous volume, so TAO caches the graph close to the request path (eventually consistent, read-optimized) rather than hammering MySQL or running a pure graph DB at 3B-user scale.

**Q3. Why is the friend graph easier for fan-out than Twitter's follow graph?**
Friends are bidirectional and capped (~5,000), so fan-out-on-write is tractable for ordinary users — no 100M-follower explosion. Pull is reserved for public figures and Pages with unbounded followers. The hybrid threshold is about connection count.

**Q4. How does ranking work and how does it fit the latency budget?**
Generate candidate posts → score each with a deep model over hundreds of features (recency, affinity, post type, predicted engagement) → aggregate (and interleave ads). To stay under ~200 ms, narrow the candidate set and precompute expensive features (affinity) offline daily; cache the ranked feed per user, refreshing every few minutes.

**Q5. How is everything modeled?**
As objects (nodes: posts, users, comments) and edges/associations (relationships: friend, likes, comment) — the social graph. Feed queries are graph traversals (a user's friend edges → friends' post objects → reaction/comment edges), all served from TAO's cache.

**Q6. How do you enforce privacy in the feed?**
Each post has an audience (Public/Friends/custom). Feed generation checks audience during candidate selection so restricted posts never reach unauthorized users — access control woven into feed-building. It's a correctness/trust requirement, more nuanced than Twitter's public-by-default.

**Q7. How do new and deleted posts propagate?**
New posts fan out to active friends' feeds within seconds; deleted posts must be invalidated across caches promptly so stale feeds stop showing them; audience changes are honored on subsequent reads. This is cache invalidation applied to feeds.

**Q8. What's the key caching optimization at this scale?**
Fan-out to *active* users only — push new posts to active friends' feeds and compute inactive users' feeds lazily on their next visit. This avoids wasting enormous fan-out work on feeds nobody will read soon, cutting write amplification dramatically. Plus heavy caching of viral posts and offline-precomputed affinity scores.

**Q9. What's the consistency model?**
Read-heavy and eventually consistent (TAO is read-optimized/eventually-consistent; feeds, counts, and rankings can lag slightly). Acceptable because users prefer fast, always-available feeds over instant consistency — and it enables async fan-out and heavy caching, like Twitter/Instagram.

**Q10. Where do ads fit?**
The Ad Service interleaves ads into the ranked feed during aggregation — a monetization step folded into feed assembly, ranked/targeted with its own signals alongside organic content.

---

## 10. Quick Glossary

- **News Feed** — the personalized, ranked stream of friends'/pages' posts.
- **Social graph** — the network of users (objects) and relationships (edges).
- **Object** — a node in the graph (post, user, comment).
- **Edge / association** — a relationship `(from_id, edge_type, to_id)` (friend, like, comment).
- **TAO** — Facebook's graph-aware caching layer (objects + edges) over sharded MySQL.
- **Hybrid fan-out** — push for ordinary users, pull for highly-connected entities.
- **Friend graph** — bidirectional, bounded (~5,000) relationships (vs. unbounded follows).
- **Candidate generation** — gathering the posts eligible to appear in a feed.
- **Affinity** — how strongly a user engages with an author; a precomputed ranking feature.
- **ML feed ranking** — scoring candidates by predicted engagement instead of chronologically.
- **Audience** — a post's visibility scope (Public, Friends, custom).
- **Active-user fan-out** — pushing only to active users' feeds; lazy compute for inactive ones.
- **Ad interleaving** — inserting ranked ads into the organic feed during aggregation.

---

*Reference document. Contributions and corrections welcome.*
