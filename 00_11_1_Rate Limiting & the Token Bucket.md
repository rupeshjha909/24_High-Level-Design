# Rate Limiting & the Token Bucket — System Design (HLD, Detailed)

> Rate limiting protects a service from being overwhelmed — by abuse, buggy clients, traffic spikes, or cost runaway — and enforces fairness and per-tier quotas. Of the handful of algorithms, **Token Bucket is the most widely used** because it caps the long-run rate *while allowing controlled bursts*, with tiny state and a clean distributed implementation. This doc covers the full landscape briefly, then goes deep on Token Bucket: the model, the **lazy-refill** trick, the single-node code, the **distributed Redis + Lua** implementation, and — critically — **why atomicity is non-negotiable**.

> 💡 **The one idea to anchor on:** a token bucket has two knobs — **capacity = the maximum burst** and **refill rate = the sustained long-run rate**. Tokens accrue while you're quiet and let you spend them in a burst later, but you can never exceed the refill rate over time. "Bursty but bounded" is the whole value proposition, and it's why APIs, gateways, and CDNs reach for it.

---

## Table of Contents
1. [Why rate limit](#1-why-rate-limit)
2. [Requirements](#2-requirements)
3. [Where rate limiting lives](#3-where-rate-limiting-lives)
4. [The algorithm landscape (comparison)](#4-the-algorithm-landscape-comparison)
5. [Token Bucket: the model](#5-token-bucket-the-model)
6. [The lazy-refill trick (no timer)](#6-the-lazy-refill-trick-no-timer)
7. [Burst vs sustained rate (the two knobs)](#7-burst-vs-sustained-rate-the-two-knobs)
8. [Single-node implementation](#8-single-node-implementation)
9. [Distributed implementation: Redis + Lua](#9-distributed-implementation-redis--lua)
10. [Why atomicity is essential](#10-why-atomicity-is-essential)
11. [Token Bucket vs Leaky Bucket](#11-token-bucket-vs-leaky-bucket)
12. [Distributed rate-limiting challenges](#12-distributed-rate-limiting-challenges)
13. [Key design decisions](#13-key-design-decisions)
14. [The response (429, headers)](#14-the-response-429-headers)
15. [Edge cases](#15-edge-cases)
16. [Trade-offs & talking points](#16-trade-offs--talking-points)
17. [How to present this in the interview](#17-how-to-present-this-in-the-interview)
18. [Common mistakes to avoid](#18-common-mistakes-to-avoid)
19. [TL;DR](#19-tldr)

---

## 1. Why rate limit

| Reason | Example |
|:-------|:--------|
| **Protect resources** | stop one client from saturating CPU/DB and degrading everyone |
| **Prevent abuse / DDoS** | throttle credential-stuffing, scraping, brute force |
| **Fairness** | no single tenant starves others (noisy neighbor) |
| **Cost control** | cap calls to expensive downstreams (LLM APIs, third parties) |
| **Quotas / monetization** | free vs paid tiers (100 vs 10,000 req/day) |
| **Stability** | smooth spikes so the system degrades gracefully, not catastrophically |

> 💡 **Frame it as protection + fairness + monetization.** Rate limiting isn't only a security control — it's how you enforce SLA tiers and protect shared resources from noisy neighbors. Naming all three shows you see it as a product feature, not just a firewall rule.

---

## 2. Requirements

**Functional:** given a key (user / API key / IP) and a limit (e.g. 100 req/min), decide allow/reject per request; return remaining quota / retry-after; support multiple limits/tiers.

**Non-functional:**
- **Low latency** — the check is on *every* request's hot path; must add ≪ a few ms.
- **Accurate enough** — exact vs approximate is a real trade (perfect accuracy costs latency/coordination).
- **Distributed** — the limit is global across many app servers, not per-server.
- **Highly available** — the limiter must not become a single point of failure; decide **fail-open vs fail-closed**.
- **Memory-efficient** — millions of keys; small state per key; expire idle keys.

> 💡 **The hot-path constraint dominates.** "This runs on every request, so it must be sub-millisecond and can't be a SPOF — which pushes me toward a fast in-memory store (Redis), an atomic single-round-trip check, and an explicit fail-open/closed policy." State that early; it justifies the whole design.

---

## 3. Where rate limiting lives

```
client (optional self-throttle)
   │
   ▼
API Gateway / reverse proxy / middleware   ◄── most common place (NGINX, Envoy, API GW, Kong)
   │   (check limit BEFORE work is done)
   ▼
service(s)   ◄── can also limit at the service or per-downstream-dependency
```

Best placed **at the edge / gateway**, *before* expensive work — reject early, cheaply. Often layered: coarse IP limit at the edge + fine per-user/per-endpoint limit deeper in.

> 💡 **"Reject early" is the principle.** Check the limit at the gateway before the request touches your services or DB — a rejected request should cost almost nothing. Layered limits (IP at edge, user/endpoint inside) catch different abuse shapes.

---

## 4. The algorithm landscape (comparison)

| Algorithm | Idea | Bursts | Accuracy | State | Notes |
|:----------|:-----|:-------|:---------|:------|:------|
| **Fixed Window Counter** | count per fixed window (e.g. per minute) | spiky at edges | low | 1 counter | simplest; **2× burst at window boundary** |
| **Sliding Window Log** | timestamp of every request, count those in window | none (exact) | exact | O(requests) | accurate but memory-heavy |
| **Sliding Window Counter** | weighted blend of current+previous window | smoothed | good approx | 2 counters | great accuracy/cost balance |
| **Token Bucket** | tokens refill at rate; spend 1/request | **controlled burst** | good | 2 numbers | **most popular**; bursty-but-bounded |
| **Leaky Bucket** | queue drains at constant rate | smoothed output | good | queue | enforces steady *output* rate |

**The boundary problem (fixed window):** with a 100/min limit, a client can send 100 at 0:59 and 100 at 1:00 → **200 requests in ~1 second** across the boundary. Sliding-window and token-bucket avoid this.

> 💡 **Know why Token Bucket wins for APIs:** it allows *desirable* bursts (a user clicking quickly, a batch job) while bounding the long-run rate, with just two numbers of state and an atomic O(1) check. Fixed window is simpler but has the boundary bug; sliding-window-log is exact but expensive. Token bucket is the pragmatic default.

---

## 5. Token Bucket: the model

```
            refill at R tokens/sec (steady drip)
                     │
                     ▼
        ┌───────────────────────┐
        │   ●  ●  ●  ●  ●        │   capacity = B tokens (max)
        │   bucket               │
        └───────────┬───────────┘
                    │ each request takes 1 token
                    ▼
        token available?  ── yes ──► ALLOW (tokens -= 1)
                          ── no  ──► REJECT (429)
```

- A bucket holds up to **B** tokens (capacity).
- Tokens refill at a steady **R** tokens/sec.
- Each request consumes **1** token (or N for weighted cost).
- Empty bucket → reject.
- Idle time lets tokens **accumulate up to B**, so a quiet client can later burst up to B requests — but the **long-run rate can't exceed R**.

> 💡 **The intuition:** tokens are "permission to make a request" that the system mints at rate R. You can save up to B of them while idle and spend them in a burst, but the mint rate caps your sustained throughput. Capacity controls *burstiness*; refill rate controls *sustained throughput*.

---

## 6. The lazy-refill trick (no timer)

You do **not** run a background timer adding tokens — that's millions of timers. Instead, **compute refill lazily on each request** from elapsed time:

```
on request at time `now`:
    elapsed = now - last_refill
    tokens  = min(capacity, tokens + elapsed * rate)   # accrue what we'd have refilled
    last_refill = now
    if tokens >= 1: tokens -= 1; ALLOW
    else:           REJECT
```

State is just **two values: `tokens` and `last_refill`.** No timers, no per-bucket threads — the elapsed-time multiply reconstructs exactly how many tokens *would* have dripped in. (Verified: this reproduces burst, sustained-rate, and the capacity cap correctly.)

> 💡 **Lazy refill is the implementation insight interviewers want.** "I don't add tokens on a schedule — I store the token count and a timestamp, and on each request I top up by `elapsed × rate`, capped at capacity. Two numbers of state, O(1), no timers." That's the difference between a textbook answer and a buildable one.

---

## 7. Burst vs sustained rate (the two knobs)

The two parameters map directly to two behaviors (verified):

| Knob | Controls | Example |
|:-----|:---------|:--------|
| **capacity B** | **maximum burst** — how many requests back-to-back from a full bucket | B=5 → up to 5 instantly |
| **refill rate R** | **sustained long-run rate** | R=1/s → ~1 req/sec over time |

Upper bound over a window of T seconds ≈ `B + R×T` (the initial burst plus what refilled). With B=10, R=2/s, 10 seconds → ≤ `10 + 2×9 = 28` (verified). Choosing them is the design judgement: **big B = tolerant of bursts** (good UX, but a bigger momentary spike downstream); **small B = strict smoothing** (closer to leaky bucket).

> 💡 **The "con" to discuss:** you must reason about burst size vs average rate together. A generous capacity is friendlier to clients but lets a bigger instantaneous spike through to your backend — so size B to what your downstream can absorb, not arbitrarily.

---

## 8. Single-node implementation

```java
class TokenBucket {
    private final double capacity, refillRatePerSec;
    private double tokens;
    private long lastRefillNanos;

    TokenBucket(double capacity, double refillRatePerSec) {
        this.capacity = capacity; this.refillRatePerSec = refillRatePerSec;
        this.tokens = capacity; this.lastRefillNanos = System.nanoTime();
    }
    synchronized boolean allow(int cost) {                 // synchronized = atomic per bucket
        long now = System.nanoTime();
        double elapsedSec = (now - lastRefillNanos) / 1e9;
        tokens = Math.min(capacity, tokens + elapsedSec * refillRatePerSec);  // lazy refill
        lastRefillNanos = now;
        if (tokens >= cost) { tokens -= cost; return true; }
        return false;
    }
}
```

`synchronized` makes the read-refill-decrement atomic **on one machine**. The catch: this is **per-process** state — five app servers each get their own bucket, so the real global limit is 5×. That's why distributed systems need shared state.

> 💡 **Call out the per-process trap.** "In-process token buckets are fine for one node, but behind a load balancer each server has its own bucket, so the effective limit multiplies by the server count. For a global limit you need shared, atomic state — which is the Redis+Lua design." This transition is exactly what the interviewer is fishing for.

---

## 9. Distributed implementation: Redis + Lua

For a **global** limit across all app servers, keep the bucket in a shared fast store — **Redis** — and do the whole read-refill-decrement as **one atomic operation** via a **Lua script** (Redis runs each script atomically, single-threaded). One round trip per request.

```lua
-- KEYS[1] = bucket key, e.g. "rl:user:123"
-- ARGV[1] = capacity (B)
-- ARGV[2] = refill_rate (tokens/sec, R)
-- ARGV[3] = now (seconds, from caller or redis TIME)
-- ARGV[4] = requested tokens (cost, usually 1)
local capacity = tonumber(ARGV[1])
local rate     = tonumber(ARGV[2])
local now      = tonumber(ARGV[3])
local cost     = tonumber(ARGV[4])

local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(bucket[1])
local ts     = tonumber(bucket[2])

if tokens == nil then           -- first time: start full
    tokens = capacity
    ts     = now
end

local elapsed = math.max(0, now - ts)              -- LAZY REFILL
tokens = math.min(capacity, tokens + elapsed * rate)

local allowed = 0
if tokens >= cost then
    tokens = tokens - cost
    allowed = 1
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
-- expire idle buckets so memory doesn't grow unbounded
local ttl = math.ceil(capacity / rate) * 2
redis.call('EXPIRE', KEYS[1], ttl)

return allowed                  -- 1 = allow, 0 = reject (return tokens too for headers)
```

Per request: app calls `EVALSHA <script> 1 rl:user:123 B R now 1` → gets allow/reject. The script's body (read both fields, refill, compare, decrement, write) executes as **one indivisible unit**.

> 💡 **Why Lua specifically?** Because the check is a **read-modify-write** that must be atomic, and a Lua script is the way to run multi-step logic atomically inside Redis in a single round trip. A naive `GET` then `SET` from the app has a race window (next section); `INCR` alone can't do the elapsed-time refill math. Lua gives you atomicity *and* arbitrary logic. (Alternatives: `MULTI/EXEC` transactions, or `WATCH`-based optimistic CAS — but Lua is cleaner here.)

---

## 10. Why atomicity is essential

The check is a classic **read-modify-write**: read tokens → compute refill → compare → decrement → write. If two requests interleave without atomicity, both read the *same* token count, both decide "allowed," both decrement — **over-admitting**:

```
tokens = 1
Request A: read tokens=1 ──┐
Request B: read tokens=1 ──┤  (both read before either writes)
A: 1>=1 → allow, write 0   │
B: 1>=1 → allow, write 0   ┘   ← TWO requests admitted from ONE token  ✗
```

(Verified: the non-atomic path admits 2 from 1 token.) Under high concurrency this leaks far over the limit. Wrapping read+refill+decrement in **one atomic step** (the Lua script) means only one of A/B sees the token and the other is correctly rejected.

> 💡 **This is the single most important correctness point.** "Rate limiting is a read-modify-write, so without atomicity concurrent requests over-admit — they all see the same count and all pass. The Lua script makes the whole check atomic in Redis, which is why a plain GET-then-SET is wrong." If you remember one thing about distributed rate limiting, it's this.

---

## 11. Token Bucket vs Leaky Bucket

They're often confused; the difference is **bursts**:

| | Token Bucket | Leaky Bucket |
|:--|:-------------|:-------------|
| Model | tokens refill; spend per request | requests queue; drain at fixed rate |
| Bursts | **allowed** up to capacity | **smoothed away** — output is steady |
| Output rate | variable (up to burst), bounded long-run | **constant** |
| Use when | bursts are OK/desirable (most APIs) | you need a **steady downstream rate** (e.g. protecting a fragile dependency) |
| Reject vs queue | reject when empty | reject when queue full (or delay) |

Token bucket says "you may burst, but not sustain"; leaky bucket says "I will emit at a constant rate no matter how you arrive." Token bucket is the default for user-facing APIs (bursts are normal); leaky bucket suits feeding a downstream that needs a smooth, constant input.

> 💡 **The crisp distinction:** "Token bucket *allows* bursts up to capacity; leaky bucket *removes* bursts and emits a constant rate. Pick token bucket when bursts are fine and you just want a long-run cap; pick leaky bucket when the thing you're protecting needs a steady, smoothed flow."

---

## 12. Distributed rate-limiting challenges

| Challenge | Handling |
|:----------|:---------|
| **Global state** | centralized Redis holds the bucket per key; all app servers consult it |
| **Atomicity** | Lua script (or MULTI/EXEC / WATCH) — §10 |
| **Latency** (Redis call per request) | co-locate Redis; pipeline; or **local token-bucket with periodic central sync** (approximate) for ultra-hot paths |
| **Redis as SPOF** | Redis HA (replicas, Cluster, Sentinel); **fail-open** (allow on Redis outage) vs **fail-closed** (reject) — a deliberate choice |
| **Hot keys** | a single very hot key bottlenecks one shard → shard the limit, or local pre-check |
| **Clock source** | pass `now` from app (clock skew risk) or use `redis.call('TIME')` (one clock, but historically a replication caveat) |
| **Memory** | TTL on idle buckets so millions of keys don't pile up |

**Approximate distributed designs** trade exactness for latency: each node keeps a local bucket and periodically reconciles with a central counter, or the global allowance is split across nodes. Cheaper and faster, slightly less precise — fine for most limits.

> 💡 **The latency vs accuracy lever:** "A central Redis check per request is exact but adds a network hop; for the hottest paths I'd let each node hold a local bucket and sync to Redis periodically — approximate but sub-microsecond. Rate limits rarely need to be exact to the request, so approximate-and-fast is usually the right call." Naming that trade is a senior signal.

---

## 13. Key design decisions

- **What to key on** — user id, API key, IP, or a composite (`user + endpoint`). IP catches anonymous abuse but NAT groups many users; API key is precise for paid tiers. Often layer several.
- **Multiple limits at once** — e.g. 10/sec **and** 1000/hour **and** per-endpoint — check several buckets; reject if any is empty.
- **Tiered limits** — free vs paid; the limit params come from the user's plan.
- **Cost-weighted requests** — expensive endpoints consume more tokens (`cost = N`), cheap ones 1.
- **Fail-open vs fail-closed** — if the limiter is down: allow (favor availability, risk overload) or reject (favor protection, risk false denials). Usually **fail-open** for user-facing, **fail-closed** for abuse/security limits.

> 💡 **"What's the key?" is the first clarifying question.** Always pin down the dimension (per-user? per-IP? per-endpoint?) and whether multiple limits stack. It shapes the whole design and shows product thinking.

---

## 14. The response (429, headers)

When rejecting, return **HTTP 429 Too Many Requests**, and help the client behave:
- `Retry-After: 5` — when to try again.
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` — quota visibility (the Lua script can return remaining tokens for these).

Good clients then back off; well-designed limits make compliance easy.

> 💡 **Return *why* and *when*, not just a rejection.** A 429 with `Retry-After` and remaining-quota headers turns a hostile-looking block into a cooperative contract — clients can self-throttle, which reduces retries and load.

---

## 15. Edge cases

| Case | Handling |
|:-----|:---------|
| First request for a key | bucket initialized full (start at capacity) |
| Long idle, then burst | tokens capped at capacity — no unlimited banking (verified) |
| Clock skew across app servers | prefer a single clock (`redis TIME`) or NTP-synced hosts |
| Redis down | fail-open/closed per policy; degrade gracefully |
| Very hot single key | shard/partition the limit or local pre-check |
| Cost > capacity | request can never succeed → reject (or validate config) |
| Idle buckets accumulating | TTL expiry reclaims memory |
| Distributed clock for `now` | one source of truth to avoid refill drift |

> 💡 **The capacity-cap invariant:** "Idle time accrues tokens only up to capacity, so a client that's been quiet for an hour still can't dump an hour's worth at once — just one bucketful." That's the property that keeps token bucket safe despite allowing bursts.

---

## 16. Trade-offs & talking points

- **Bursts: feature or risk?** Token bucket allows them (good UX); size capacity to what downstream tolerates.
- **Exact vs approximate** — central atomic (exact, +1 hop) vs local+sync (approximate, faster).
- **Availability vs protection** — fail-open vs fail-closed.
- **Algorithm choice** — token bucket (bursty APIs) vs leaky bucket (steady output) vs sliding window (smooth, no boundary bug) vs fixed window (simplest, has the bug).
- **Granularity** — per-user vs per-IP vs per-endpoint; layered.

---

## 17. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Clarify | what's the key (user/IP/endpoint)? one limit or several? exact or approximate? |
| Why + where | protection/fairness/quotas; check at the gateway, reject early |
| Algorithms | quick compare; pick **token bucket** for bursty-but-bounded, name the boundary bug of fixed window |
| Token bucket | model (capacity=burst, rate=sustained), **lazy refill** (2 numbers, no timer) |
| Single → distributed | per-process trap → shared Redis state |
| Redis + Lua | the script; **atomicity** (the race) |
| Productionize | fail-open/closed, HA, hot keys, 429 + Retry-After, TTL |

### What to say
- *"Token bucket: capacity is the max burst, refill rate is the sustained rate — bursty but bounded."*
- *"I refill lazily — store tokens + a timestamp, top up by elapsed×rate on each request. No timers."*
- *"In-process buckets multiply by server count, so for a global limit I keep the bucket in Redis."*
- *"The check is read-modify-write, so it must be atomic — a Lua script, or concurrent requests over-admit."*
- *"On Redis failure I'd fail-open for user traffic, fail-closed for abuse limits; 429 with Retry-After."*

### Order
clarify the key → pick token bucket (why) → lazy-refill model → single-node → Redis+Lua + atomicity → fail-open/HA/hot-keys/headers.

---

## 18. Common mistakes to avoid

- ❌ **Non-atomic GET-then-SET** — the race over-admits; use an atomic Lua script.
- ❌ **In-process counters behind a load balancer** — effective limit = N× the intended limit.
- ❌ **Background timers refilling tokens** — use lazy elapsed-time refill (2 numbers of state).
- ❌ **Fixed window without knowing the boundary bug** — 2× burst at the edge; mention it.
- ❌ **No TTL on buckets** — millions of idle keys leak memory.
- ❌ **Ignoring Redis as a SPOF** — need HA + an explicit fail-open/closed policy.
- ❌ **Confusing token vs leaky bucket** — token *allows* bursts; leaky *smooths* them.
- ❌ **Clock skew across nodes** — refill math drifts; single clock source.
- ❌ **Hot single key** — bottlenecks one shard; shard or pre-check locally.
- ❌ **Bare 429 with no `Retry-After`** — clients can't self-throttle; you get retry storms.
- ❌ **Capacity chosen arbitrarily** — size burst to what downstream can absorb.

---

## 19. TL;DR

### The model
```
Token bucket: capacity B (max BURST) + refill rate R (sustained RATE).
Each request takes 1 token; empty → reject. Idle banks tokens up to B (no more).
Long-run admitted over T seconds ≈ B + R*T.
```

### The implementation
```
LAZY REFILL (no timer), state = {tokens, last_refill_ts}:
   elapsed = now - ts
   tokens  = min(B, tokens + elapsed*R)
   ts      = now
   if tokens >= 1: tokens -= 1 → ALLOW   else → REJECT (429 + Retry-After)
```

### Distributed
```
Global limit → bucket in REDIS, checked via a LUA SCRIPT (atomic, 1 round trip).
ATOMICITY is mandatory: read-modify-write, or concurrent requests over-admit (2 from 1 token).
Add: TTL on idle buckets, Redis HA, fail-open vs fail-closed, shard hot keys.
```

### The four things that score points
1. **Token bucket = bursty but bounded** — capacity is burst, refill is sustained rate.
2. **Lazy refill** — two numbers + elapsed×rate, no timers (the buildable insight).
3. **Distributed = Redis + atomic Lua** — because the check is read-modify-write and must be atomic, or it over-admits.
4. **Productionize** — fail-open/closed, HA, hot-key sharding, TTL, 429 + Retry-After.

> **One-line philosophy:** *Token bucket caps the long-run rate while permitting controlled bursts by minting tokens at a steady rate up to a fixed capacity; implement it with lazy elapsed-time refill over just a token count and a timestamp, and make it global by running the read-refill-decrement as one atomic Redis Lua script — because the moment that check isn't atomic, concurrent requests slip through and your limit leaks.*
