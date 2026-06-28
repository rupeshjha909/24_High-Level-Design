# Designing a Rate Limiter (High-Level Design): A Ground-Up Guide

> A practical reference for the high-level design of a distributed rate limiter — why you need one, where to place it, the four core algorithms and their trade-offs, the distributed-state problem and its Redis-based solution, response strategy, and the failure modes that matter. With trade-offs and interview prep.

---

## Table of Contents

1. [Why Rate Limiting Exists](#1-why-rate-limiting-exists)
2. [What to Limit By](#2-what-to-limit-by)
3. [Where to Put the Rate Limiter](#3-where-to-put-the-rate-limiter)
4. [The Algorithms](#4-the-algorithms)
5. [The Core HLD Challenge: Distributed State](#5-the-core-hld-challenge-distributed-state)
6. [Architecture](#6-architecture)
7. [Response Strategy](#7-response-strategy)
8. [Failure Modes: Fail-Open vs Fail-Closed](#8-failure-modes-fail-open-vs-fail-closed)
9. [Performance: The Latency Trade-off](#9-performance-the-latency-trade-off)
10. [Interview-Ready Insights](#10-interview-ready-insights)
11. [Quick Glossary](#11-quick-glossary)

---

## 1. Why Rate Limiting Exists

A rate limiter caps how many requests a client can make in a given time window. It's a piece of infrastructure that protects a system from being overwhelmed — by abuse, by bugs, or just by success. The motivations:

- **Protect resources** — stop any one client (or a buggy retry loop) from exhausting capacity and degrading service for everyone.
- **Prevent abuse / DoS** — throttle malicious floods and brute-force attacks at the edge before they reach your services.
- **Ensure fairness** — share finite capacity equitably across many users so a few heavy users don't starve the rest.
- **Control cost** — cap usage of expensive downstream resources (third-party APIs, compute) and enforce billing tiers.

The mental model: a rate limiter is a **gatekeeper that counts requests per client and rejects (or delays) those over the limit.** The HLD question is *where that gate sits and how the counting stays correct across many machines* — which is harder than it sounds once you're distributed.

---

## 2. What to Limit By

Before placement and algorithms, decide the **key** you count against — it defines "a client":

- **By API key / user ID** — the most common for APIs; enables per-customer tiers (free vs. paid limits).
- **By IP address** — useful for anonymous traffic and basic abuse defense, but imperfect (NAT/shared IPs lump many users together; attackers rotate IPs).
- **By a combination** — e.g., per-endpoint *and* per-user, since an expensive endpoint may warrant a tighter limit than a cheap one.

The limit itself is usually expressed as **N requests per time window** (e.g., 100 req/min), often with **different tiers** per user class. This keying decision shapes what state you must track.

---

## 3. Where to Put the Rate Limiter

Placement is a central HLD decision, with four options trading off centralization, bypass-resistance, and load.

### 1. Client-side

The client throttles itself.

- **Pros:** zero server cost; good for UX hints ("slow down").
- **Cons:** **trivially bypassed** — you control nothing on the client. Useful only as a courtesy, never as enforcement.

### 2. Server-side middleware (per service)

Each service enforces limits in its own request-handling middleware.

- **Pros:** consistent enforcement close to the logic; each service can tune its own limits.
- **Cons:** logic duplicated across every service; harder to manage centrally; the request already reached the service before being rejected (wasting some capacity).

### 3. API Gateway (the common choice)

A central gateway in front of all services enforces limits.

- **Pros:** **central enforcement** in one place; **rejects excess traffic before it reaches your services**, protecting them; natural home alongside auth and routing (see the microservices guide's API Gateway pattern).
- **Cons:** the gateway must be highly available and fast (it's on every request path).

### 4. Sidecar proxy (service mesh)

A proxy deployed alongside each service instance (e.g., Istio/Envoy) enforces limits per-pod.

- **Pros:** central *policy* with distributed *enforcement*; consistent across services without touching app code (see the sidecar/service-mesh pattern).
- **Cons:** more infrastructure; per-pod state still needs coordination.

**The usual answer:** enforce at the **API Gateway** (and/or sidecar in a mesh) so excess traffic dies at the edge and your services stay protected. Client-side is only ever a UX nicety on top.

---

## 4. The Algorithms

Four algorithms, each with a distributed (Redis-backed) implementation. The trade-off axis is **accuracy vs. memory vs. burst behavior.**

### Fixed Window Counter

Count requests per fixed window (e.g., per calendar minute); reset the count when the window rolls over.

- **Distributed impl:** Redis `INCR` on a key like `user:123:minute:42`, with `EXPIRE` to auto-clean.
- **Pros:** simplest, low memory, cheap.
- **Cons:** the **boundary-burst problem** — a client can send a full window's worth of requests at the *end* of one window and another full window's worth at the *start* of the next, pushing **up to 2× the limit** across the boundary.

```
limit = 100/min
  0:59 → 100 requests  ┐
  1:00 → 100 requests  ┘  200 requests in ~2 seconds — limit defeated
```

### Sliding Window Log

Store the **timestamp of every request** in a sorted set; to check, count timestamps within the trailing window.

- **Distributed impl:** Redis **sorted set** (ZADD timestamp, then count/trim entries older than the window).
- **Pros:** **perfectly accurate** — no boundary problem.
- **Cons:** **memory-heavy** — stores every request timestamp; expensive at high volume.

### Sliding Window Counter (the practical favorite)

A hybrid: keep per-window counts and compute a **weighted estimate** blending the current and previous window based on how far into the current window you are. Approximates the sliding log with a fraction of the memory.

- **Pros:** smooths the boundary burst; far cheaper than the log.
- **Cons:** slightly approximate (good enough for almost all uses).

### Token Bucket (the most popular)

A bucket holds up to *B* tokens and **refills at a steady rate**; each request consumes one token; if the bucket is empty, reject. **Accumulated tokens allow short bursts** up to the bucket size, while the long-run rate is capped by the refill rate.

- **Distributed impl:** a Redis **Lua script** that atomically computes refill-since-last-check and consumes a token (atomicity is essential — see below).
- **Pros:** allows controlled bursts (often desirable), simple state (token count + last-refill time), widely used.
- **Cons:** must reason about burst size vs. average rate.

### Leaky Bucket

Requests enter a queue and are **processed at a fixed, constant rate**; overflow is dropped. Unlike token bucket, it **smooths output to a steady stream with no bursts.**

- **Distributed impl:** a Redis list (queue) drained by a worker at a fixed rate.
- **Pros:** enforces a perfectly smooth outflow — good when downstream needs steady load.
- **Cons:** no burst allowance; added queueing latency.

| Algorithm | Distributed impl | Burst? | Accuracy | Memory |
|-----------|------------------|--------|----------|--------|
| Fixed Window | `INCR` + `EXPIRE` | boundary burst | low | very low |
| Sliding Window Log | sorted set of timestamps | no | exact | high |
| Sliding Window Counter | weighted window counts | smoothed | near-exact | low |
| Token Bucket | Lua (refill + consume) | yes (bounded) | good | low |
| Leaky Bucket | list + draining worker | no (smoothed) | good | medium |

**Default pick:** **Token Bucket** when bounded bursts are fine (most APIs), or **Sliding Window Counter** when you want smooth, accurate limiting cheaply.

---

## 5. The Core HLD Challenge: Distributed State

Here's what makes this a *distributed-systems* problem rather than a one-liner. You don't have one rate-limiter instance — you have **many gateway/proxy nodes behind a load balancer**, and a given client's requests can hit *any* of them. If each node keeps its **own local counter**, a client limited to 100/min could send 100 to *each* of 10 nodes and get 1,000 through. The counters must be **shared and coordinated.**

### The solution: a centralized store with atomic operations

Use a **shared Redis** (cluster) that all rate-limiter nodes read/write, so there's one authoritative counter per client.

The subtlety — and a classic interview trap — is the **race condition**: "read count → check limit → increment" done as three separate steps lets two concurrent requests both read 99, both think they're under 100, and both proceed. The fix is **atomicity**:

- **`INCR`** is atomic — increment-and-return in one operation (great for fixed window).
- **Lua scripts** run atomically on Redis, so multi-step logic (token bucket's refill + consume, or check-then-set) executes as a single indivisible operation, eliminating the race.

This atomic-shared-counter design is the heart of a distributed rate limiter. Redis is chosen because it's in-memory (fast — sub-millisecond), supports the right atomic primitives, and scales.

---

## 6. Architecture

```
[Client] ──► [API Gateway + Rate Limiter] ──► [Service]
                      │  (allow / reject)
                      ▼
                [Redis Cluster]   ← authoritative, atomic counters
```

- The **API Gateway** intercepts every request, derives the client key, and asks the rate-limiter logic whether to allow it.
- The rate-limiter logic performs an **atomic** check-and-update against the **Redis cluster**.
- **Allowed** → forward to the service; **over limit** → reject immediately (never touches the service).
- Redis is **clustered/replicated** for availability — it's now on the critical path of every request, so it must not be a single point of failure.

---

## 7. Response Strategy

How you communicate limits matters for good API citizenship.

- **Informational headers (soft signal)** on every response so clients can self-regulate:
  - `X-RateLimit-Limit` — the cap.
  - `X-RateLimit-Remaining` — requests left in the window.
  - `X-RateLimit-Reset` — when the window resets.
- **Hard rejection** when over the limit:
  - HTTP **`429 Too Many Requests`**.
  - **`Retry-After`** header telling the client how long to wait before retrying — crucial so well-behaved clients back off instead of hammering.

Returning `429` with `Retry-After` (rather than silently dropping or returning a generic error) lets clients implement correct backoff and is the expected, well-mannered behavior.

---

## 8. Failure Modes: Fail-Open vs Fail-Closed

A senior-level concern your notes hint at but is worth making explicit: **what happens when Redis (the counter store) is unavailable?** Two philosophies:

- **Fail-open** — if the limiter can't check the count, **allow the request.** Prioritizes availability; the risk is that a Redis outage temporarily removes protection.
- **Fail-closed** — if it can't check, **reject the request.** Prioritizes protection; the risk is that a Redis outage takes down your whole API even though the services are healthy.

**The usual choice is fail-open** for rate limiting: a brief lapse in throttling is far less bad than rejecting all legitimate traffic because an auxiliary system blipped. (Security-critical limits — e.g., login brute-force protection — may justify fail-closed.) Either way, **state the choice and its rationale** — this is exactly the kind of trade-off interviewers probe.

---

## 9. Performance: The Latency Trade-off

Every request now makes a round trip to Redis to check the counter. Within a datacenter that's ~0.5 ms (see the estimation guide), usually acceptable — but at very high volume it adds up and makes Redis a hotspot.

Common optimizations:

- **Local counters + periodic sync** — each node tracks an approximate local count and reconciles with Redis periodically, trading a little accuracy for far fewer Redis round trips.
- **Pipelining / batching** Redis operations.
- **Sharding the Redis keyspace** (consistent hashing) so counter load spreads across the cluster and no single key/node is a bottleneck.

The trade-off is the recurring one: **perfectly-accurate centralized counting vs. lower latency and load.** For most systems a small inaccuracy is an acceptable price for performance.

---

## 10. Interview-Ready Insights

**Q: Where should a rate limiter live?**
At the **API Gateway** (and/or a sidecar in a service mesh) so excess traffic is rejected at the edge before reaching services. Client-side is only a UX hint — trivially bypassed. Per-service middleware works but duplicates logic and lets traffic reach the service first.

**Q: What's the hardest part of a *distributed* rate limiter?**
Shared state. With many limiter nodes behind a load balancer, per-node local counters let a client multiply their limit by the node count. You need a **centralized store (Redis)** with **atomic operations** (`INCR`, Lua scripts) to keep one authoritative count and avoid the read-check-increment race condition.

**Q: Compare the algorithms.**
Fixed window is cheapest but has a 2×-at-the-boundary burst flaw. Sliding window log is exact but memory-heavy. Sliding window counter approximates it cheaply. Token bucket allows bounded bursts (most popular). Leaky bucket smooths output to a constant rate with no bursts. Pick token bucket or sliding-window-counter for most APIs.

**Q: Token bucket vs leaky bucket?**
Token bucket accumulates tokens, so it **permits short bursts** up to the bucket size while capping the average rate. Leaky bucket processes at a **fixed constant rate** with no bursts, smoothing output. Choose based on whether bursts are acceptable or downstream needs steady load.

**Q: Why must the counter update be atomic?**
Because "read → check → increment" as separate steps lets two concurrent requests both see 99 and both proceed, exceeding the limit. Atomic `INCR` or a Lua script makes check-and-update a single indivisible operation, eliminating the race.

**Q: What happens if Redis goes down — fail-open or fail-closed?**
Usually **fail-open**: allow requests when you can't check, since briefly losing throttling is better than rejecting all traffic. Security-sensitive limits may fail-closed. The key is to decide deliberately and justify it.

**Q: How do you respond when a client is over the limit?**
HTTP `429 Too Many Requests` with a `Retry-After` header, plus `X-RateLimit-*` headers on normal responses so clients can self-regulate and back off correctly.

---

## 11. Quick Glossary

- **Rate limiter** — infrastructure that caps requests per client per time window.
- **Throttling** — rejecting or delaying requests that exceed the limit.
- **Limit key** — what you count against (API key, user ID, IP, or a combination).
- **Fixed window** — count per fixed time bucket; simple but allows boundary bursts.
- **Sliding window log** — exact limiting by storing every request timestamp; memory-heavy.
- **Sliding window counter** — cheap approximation of sliding window via weighted window counts.
- **Token bucket** — tokens refill at a rate; requests consume them; allows bounded bursts.
- **Leaky bucket** — requests drained at a constant rate; smooths output, no bursts.
- **Atomic operation** — an indivisible read-modify-write (`INCR`, Lua) that prevents races.
- **Fail-open / fail-closed** — allow vs. reject requests when the counter store is unavailable.
- **429 Too Many Requests** — the HTTP status for exceeding a rate limit.
- **Retry-After** — header telling the client when it may retry.
- **X-RateLimit-* headers** — inform clients of their limit, remaining quota, and reset time.

---

*Reference document. Contributions and corrections welcome.*
