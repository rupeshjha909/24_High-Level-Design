# Content Delivery Networks (CDNs): A Ground-Up Guide

> A practical reference for how CDNs work and why they exist — edge caching, anycast routing, pull vs push, the cache-invalidation problem and its elegant fix (versioned URLs), static vs dynamic content and edge computing, and DDoS absorption. With trade-offs and interview prep.

---

## Table of Contents

1. [What a CDN Is, and the Physics Behind It](#1-what-a-cdn-is-and-the-physics-behind-it)
2. [Why Use One](#2-why-use-one)
3. [How a Request Flows](#3-how-a-request-flows)
4. [Anycast Routing](#4-anycast-routing)
5. [Pull vs Push](#5-pull-vs-push)
6. [The Hard Part: Cache Invalidation](#6-the-hard-part-cache-invalidation)
7. [Cache Hit Ratio & Cache-Control](#7-cache-hit-ratio--cache-control)
8. [Static vs Dynamic Content & Edge Computing](#8-static-vs-dynamic-content--edge-computing)
9. [CDNs as a Security Shield](#9-cdns-as-a-security-shield)
10. [Major Providers](#10-major-providers)
11. [Interview-Ready Insights](#11-interview-ready-insights)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. What a CDN Is, and the Physics Behind It

A **Content Delivery Network** is a geographically distributed network of **edge servers** (in locations called **PoPs** — Points of Presence) that cache and serve content **close to users.** Instead of every request traveling to your single origin server, it's answered by a nearby edge node.

The deep reason CDNs exist is **physics, not engineering cleverness.** As the estimation guide notes, a round trip across the planet is ~150 ms and is bounded by the **speed of light** — you cannot make the network faster than that. So the only way to cut that latency is to **move the content closer to the user**, which is exactly what a CDN does. A user in Mumbai fetching from a Mumbai edge node experiences a few milliseconds instead of a ~150 ms transcontinental round trip.

The mental model: a CDN is a globally-distributed cache layer sitting in front of your origin, turning "far away and slow" into "nearby and fast" for the content it can cache.

---

## 2. Why Use One

- **Reduce latency** — content is served from a PoP near the user, eliminating long-distance round trips.
- **Reduce origin load** — a well-tuned CDN serves the large majority of requests from the edge (often 95%+), so your origin handles only cache misses and dynamic work. This is enormous offload.
- **Absorb DDoS attacks** — the CDN's massive aggregate edge bandwidth and many PoPs soak up volumetric attacks that would flatten a single origin (more in Section 9).
- **Better availability** — many distributed edges mean no single point of failure for content delivery; the CDN can keep serving cached content even if the origin is briefly down.

These compound: lower latency *and* less origin load *and* better resilience *and* attack protection, which is why CDNs are near-universal for any content-serving system at scale.

---

## 3. How a Request Flows

```
1. User in Mumbai requests  image.com/logo.png
2. DNS (or anycast) routes them to the nearest CDN PoP — the Mumbai edge
3. Edge has the file cached  →  serves it directly         (cache HIT — fast)
4. Edge does NOT have it     →  fetches from origin,
                                caches it, then serves      (cache MISS — slower, once)
```

The first request for an object in a region may be a **miss** (the edge fetches from origin and caches it); subsequent requests in that region are **hits** served straight from the edge. So a CDN "warms up" per region as content is requested. The art is maximizing the **hit ratio** so misses (origin trips) are rare.

---

## 4. Anycast Routing

How does a user automatically reach the *nearest* PoP? The common mechanism is **anycast**: the **same IP address is advertised from many PoPs simultaneously**, and the internet's routing protocol (**BGP**) naturally routes each user to the **topologically nearest** PoP advertising that IP.

```
   PoP-Mumbai  ┐
   PoP-London  ├─ all advertise the SAME IP (e.g., 1.2.3.4) via BGP
   PoP-NewYork ┘
   → each user's packets are routed to whichever PoP is closest on the network
```

Anycast does two valuable things at once:
- **Automatic nearest-PoP routing** without the client doing anything special.
- **DDoS dispersion** — attack traffic aimed at that single IP gets spread across *all* PoPs by the routing fabric, rather than concentrating on one location, so each PoP only absorbs a fraction.

(Some CDNs also use DNS-based geo-routing — resolving the hostname to a region-specific IP — as an alternative or complement to anycast.)

---

## 5. Pull vs Push

Two strategies for getting content onto the edge:

- **Pull (lazy)** — the CDN fetches content from the origin **on the first miss**, then caches it. This is the default and the simplest: you change nothing about how you publish content; the CDN populates itself on demand. The cost is a **cold-cache penalty** — the first user in each region for each object pays the origin-fetch latency.
- **Push (eager)** — the origin **proactively uploads** content to the CDN *before* it's requested. Used when you can predict demand and want to avoid the cold-start miss — e.g., a big software release, a game patch, or a video premiere where a flood of requests will arrive at once and you want the edge pre-warmed.

**Rule of thumb:** pull for the general case (simple, self-managing); push to pre-warm large, predictable, time-sensitive launches.

---

## 6. The Hard Part: Cache Invalidation

Caching's classic hard problem: once content is cached at hundreds of edges, **how do you update or remove it?** Three approaches, increasingly powerful:

- **TTL-based expiry** — each cached object has a **time-to-live**; after it expires, the edge re-fetches from origin. Simple, but it's a blunt trade-off: a **long TTL** maximizes hit ratio but serves stale content longer; a **short TTL** keeps content fresh but increases origin load (more re-fetches).
- **Purge / invalidation API** — explicitly tell the CDN to drop a specific file (or path) across all edges. Precise, but propagating a purge to every PoP takes time and, at scale, frequent purges are operationally heavy.
- **Versioned URLs (the elegant fix)** — change the **URL** whenever the content changes: `logo.png` → `logo.v2.png` (or `logo.png?hash=abc123`). Because the new content has a *new URL*, it's simply a different cache object — the old one never needs invalidating. This lets you set **effectively infinite TTLs** on immutable assets (maximizing hit ratio) while still "updating" instantly by referencing the new URL in your HTML.

**The senior insight:** versioned URLs (a.k.a. cache-busting / content-hashing) sidestep invalidation entirely by making cached objects **immutable**. This is why modern build tools fingerprint filenames with content hashes — you get both maximal caching *and* instant updates, the best of both ends of the TTL trade-off.

---

## 7. Cache Hit Ratio & Cache-Control

The **cache hit ratio** — the fraction of requests served from the edge without touching origin — is the CDN's defining metric. Every percentage point higher means fewer origin trips, lower latency, and less origin cost. Maximizing it is the central tuning goal, and it's driven by:

- **Appropriate TTLs** (long for immutable assets, short for volatile ones).
- **Versioned URLs** for static assets so they can be cached ~forever.
- **Avoiding cache fragmentation** — needlessly varying URLs (e.g., tracking query params) splits one cacheable object into many, hurting the hit ratio.

The origin controls caching behavior via HTTP headers the CDN (and browsers) honor:
- **`Cache-Control`** — directives like `max-age` (TTL), `public`/`private`, `no-store`, `immutable`.
- **`ETag` / `Last-Modified`** — let the edge revalidate cheaply ("has this changed since version X?") and get a tiny `304 Not Modified` instead of refetching the whole object.

So caching policy is a collaboration: the origin declares intent via headers; the CDN enforces it at the edge.

---

## 8. Static vs Dynamic Content & Edge Computing

- **Static content** (images, CSS, JS, fonts, videos) is the **perfect** CDN fit — it's the same for everyone and changes rarely, so it caches beautifully and serves from the edge with a high hit ratio.
- **Dynamic content** (e.g., a personalized HTML page for a logged-in user) is **per-user and changing**, so it can't be cached the traditional way — the naive answer is "it must come from origin."

But the edge can still help with dynamic content via **edge computing**: running code *at the PoP* close to the user (**Cloudflare Workers**, **AWS Lambda@Edge**). This lets you:
- Personalize or assemble responses at the edge instead of round-tripping to a distant origin.
- Do auth, A/B testing, redirects, header rewriting, and lightweight API logic near the user.
- Cache fragments and stitch in dynamic bits at the edge.

Edge computing extends the CDN's latency benefit to *some* dynamic workloads by **moving computation, not just content, closer to the user** — the same "beat distance by relocating, not by going faster" principle.

---

## 9. CDNs as a Security Shield

A CDN naturally sits **in front of your origin**, which makes it a powerful defensive layer:

- **DDoS absorption** — its enormous aggregate bandwidth across many PoPs, combined with anycast dispersion (Section 4), soaks up volumetric floods that would overwhelm a single origin. The attack hits the edge, not you.
- **Origin hiding** — clients only ever talk to the CDN, so your origin's real IP can be concealed, making it harder to attack directly (you lock the origin's firewall to accept traffic only from the CDN).
- **WAF integration** — most CDNs bundle a **Web Application Firewall** to filter malicious requests (SQL injection, XSS) and apply rate limiting at the edge (see the rate-limiter and proxy guides) before traffic reaches you.

So beyond performance, a CDN is often the first line of defense — a shield as much as an accelerator.

---

## 10. Major Providers

**Cloudflare, Akamai, Fastly, AWS CloudFront, Google Cloud CDN.** They differ in PoP footprint, edge-compute capabilities, pricing, and bundled security features, but all provide the same core: distributed edge caching with anycast routing, configurable invalidation, and (increasingly) edge computing.

---

## 11. Interview-Ready Insights

**Q: What problem does a CDN fundamentally solve?**
Latency bounded by the speed of light. You can't make a transcontinental round trip faster than ~150 ms, so a CDN moves content to edge servers near users, turning a long trip into a short one — while also offloading the origin, improving availability, and absorbing attacks.

**Q: How does a user reach the nearest edge?**
Usually **anycast**: the same IP is advertised from many PoPs, and BGP routes each user to the topologically nearest one. This also disperses DDoS traffic across all PoPs. Some CDNs use DNS geo-routing instead/as well.

**Q: Pull vs push?**
Pull (lazy) fetches from origin on first miss — simple and self-managing, with a cold-cache penalty for the first request per region. Push (eager) pre-loads content before it's requested — used to pre-warm large, predictable launches (releases, premieres).

**Q: How do you handle cache invalidation?**
TTL expiry (simple but trades freshness vs. hit ratio), purge APIs (precise but propagation-heavy), and — the elegant answer — **versioned URLs**: change the URL when content changes so cached objects are immutable. That gives near-infinite TTLs *and* instant updates, sidestepping invalidation entirely.

**Q: What's the key CDN metric and how do you maximize it?**
Cache hit ratio. Maximize with appropriate TTLs, versioned/immutable URLs for static assets, correct `Cache-Control` headers, and avoiding URL fragmentation (e.g., needless query params) that splits cacheable objects.

**Q: Can a CDN help with dynamic, personalized content?**
Traditional caching can't cache per-user content, but **edge computing** (Cloudflare Workers, Lambda@Edge) runs logic at the PoP to personalize, authenticate, A/B test, or assemble responses near the user — extending the latency benefit by moving computation closer, not just content.

**Q: How does a CDN protect against DDoS?**
Anycast spreads attack traffic across all PoPs, the edge's huge aggregate bandwidth absorbs volumetric floods, the origin's real IP is hidden behind the CDN, and a bundled WAF filters malicious requests — all before traffic reaches the origin.

---

## 12. Quick Glossary

- **CDN** — a distributed network of edge servers caching content near users.
- **Edge server / PoP (Point of Presence)** — a CDN location serving nearby users.
- **Origin** — the source server holding the authoritative content.
- **Cache hit / miss** — request served from the edge vs. requiring an origin fetch.
- **Cache hit ratio** — fraction of requests served from the edge; the key CDN metric.
- **Anycast** — advertising one IP from many PoPs so BGP routes users to the nearest.
- **BGP** — the internet routing protocol that directs traffic to the nearest anycast PoP.
- **Pull (lazy) caching** — CDN fetches from origin on first miss.
- **Push (eager) caching** — origin pre-loads content to the edge before it's requested.
- **TTL (time-to-live)** — how long a cached object stays valid before re-fetch.
- **Purge / invalidation** — explicitly removing cached content from edges.
- **Versioned URL / cache-busting** — changing the URL on content change so objects stay immutable.
- **Cache-Control / ETag** — HTTP headers governing caching and cheap revalidation.
- **Edge computing** — running code at the PoP (Cloudflare Workers, Lambda@Edge) near users.
- **WAF** — Web Application Firewall, often bundled with CDNs for request filtering.
- **DDoS absorption** — soaking up attack traffic via edge bandwidth and anycast dispersion.

---

*Reference document. Contributions and corrections welcome.*
