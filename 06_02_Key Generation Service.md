# Key Generation Service (KGS) — Detailed Design (URL Shortener)

> A deep-dive on the **Key Generation Service** approach for a URL shortener: how we **generate** random unique short keys, how we **store** them in the DB, and how we **maintain / operate** the pool in production. KGS pre-bakes random, unique keys offline so that creating a short URL at request time is just "pop a pre-made key" — giving you the counter's collision-free + fast properties *and* unguessable keys at once. Every decision is explained with the trade-off you'd state aloud, plus capacity math (verified), schemas, the atomic hand-off, and failure modes.

> 💡 **Why KGS exists (the one-paragraph recap).** *Hash + truncate* (MD5 → base62 → first 7 chars) reintroduces **collisions** the moment you truncate, so every create must check-and-retry. *Counter + base62* is collision-free but produces **sequential, guessable** keys. KGS gets the best of both: keys are **pre-validated unique** (no request-time collision check) **and random** (not enumerable), at the cost of running an extra service with careful atomic hand-off and supply monitoring.

---

## Table of Contents
1. [Where KGS fits](#1-where-kgs-fits)
2. [The short key: base62 & why 7 chars](#2-the-short-key-base62--why-7-chars)
3. [Capacity & storage estimation (verified)](#3-capacity--storage-estimation-verified)
4. [How we GENERATE keys](#4-how-we-generate-keys)
5. [How we STORE keys in the DB](#5-how-we-store-keys-in-the-db)
6. [The atomic hand-off (pulling a batch)](#6-the-atomic-hand-off-pulling-a-batch)
7. [How the app server uses a batch (request time)](#7-how-the-app-server-uses-a-batch-request-time)
8. [How we MAINTAIN the pool (replenish + monitor)](#8-how-we-maintain-the-pool-replenish--monitor)
9. [Architecture & sequence diagrams](#9-architecture--sequence-diagrams)
10. [Scaling KGS](#10-scaling-kgs)
11. [High availability & the SPOF question](#11-high-availability--the-spof-question)
12. [Failure modes & how we handle them](#12-failure-modes--how-we-handle-them)
13. [KGS vs hash-truncate vs counter+base62](#13-kgs-vs-hash-truncate-vs-counterbase62)
14. [Senior-level Q&A](#14-senior-level-qa)
15. [TL;DR](#15-tldr)

---

## 1. Where KGS fits

KGS is a **separate service** from the URL-shortener app. Its only job: hand out unique, random short keys on demand. The app service owns the actual mapping.

- **URL service** — receives "shorten this long URL", asks its in-memory batch for a key, writes `(short_key → long_url)` to the URL-mapping store, returns the short URL. On redirect, looks up `short_key → long_url` and 301/302s.
- **KGS** — pre-generates keys into a **pool DB**, and hands app servers **batches** of unused keys atomically.

Separation of concerns: the URL service never worries about *how* keys are made or whether they're unique — it just pops one.

---

## 2. The short key: base62 & why 7 chars

A key uses **base62** = `[a-z A-Z 0-9]` = 62 symbols per position (URL-safe, case-sensitive, no special chars). The number of distinct keys of length L is **62^L**.

**Why 7?** It's a *capacity choice*, not derived from any hash width. 7 chars gives ~**3.52 trillion** keys — short enough to type/share, huge enough to never run out at realistic scale (see §3). 6 chars (~56.8B) is tighter; 8 (~218T) is overkill. The key length is a design knob you set from your demand estimate.

> 💡 **7 base62 chars ≈ 41.7 bits of entropy.** So a key can equivalently be seen as "a random ~42-bit number, base62-encoded to 7 characters." That framing matters for generation (§4): pick a random integer in `[0, 62^7)` and encode it.

---

## 3. Capacity & storage estimation (verified)

**Key space by length**
| Length | 62^L | ~ |
|:--|:--|:--|
| 6 | 56,800,235,584 | 56.8 B |
| **7** | **3,521,614,606,208** | **3.52 T** |
| 8 | 218,340,105,584,896 | 218 T |

**Demand assumption:** 500 M new URLs/month → **~193 new URLs/sec** average, peak ~5× ≈ **965/sec**.

**Will 7 chars last?**
- Years to exhaust 62^7 at 500M/month: **~587 years**. 7 chars is plenty.
- Keys consumed in 5 years: **30 billion** = only **0.85%** of 62^7.

**Storage** (pool DB row ≈ **50 bytes**: 7-char key + status + timestamps + index/overhead)
- **Unused pool buffer of 10 M keys** → **~0.50 GB** (tiny).
- If you **keep** every used key for 5 years (30 B rows) → **~1.5 TB**.
- If you **delete** used keys (the URL-mapping table already prevents reuse) → the pool DB stays **~0.5 GB flat**. *(Design choice — see §5.)*

**Crash loss:** with a batch of 1,000, a server crash strands ≤ 1,000 keys. Even 1,000 crashes/day = 1 M keys/day = **0.0001%/yr** of the space — negligible.

**Runway:** at 193 keys/sec, a **2 M-key low-watermark** gives **~2.9 hours** to top up before running dry — comfortable for a background generator + alert.

> 💡 **The estimation is the design justification.** It proves three things you should say aloud: 7 chars won't run out (587 yrs), the *unused pool* is tiny (~0.5 GB) so keeping a big buffer is cheap, and stranded keys from crashes are irrelevant.

---

## 4. How we GENERATE keys

A **background generator** (a KGS component or cron/worker) tops the pool up to a **high watermark**. Two equivalent ways to make one key:

**(a) Random integer → base62 (recommended).** Draw a random integer `n` in `[0, 62^7)` and base62-encode it to 7 chars.
```
n = secureRandom(0, 62^7)          # ~42-bit random number
key = base62_encode(n)             # -> 7 chars, e.g. "9Xk2mPq"
```

**(b) Random 7-char string.** Pick 7 random symbols from the 62-char alphabet directly. Same result.

**Dedup during generation** — the pool must be unique, so rely on a **unique constraint** on the key column and insert-or-skip:
```sql
INSERT INTO key_pool (key, status) VALUES (:key, 'unused')
ON CONFLICT (key) DO NOTHING;      -- collision at generation time -> just skip & regenerate
```
At <1% space utilization the birthday-collision rate is low, and a rejected insert simply means "try another random key." **All collision handling happens offline, at generation time — never on the request path.** That's the whole point.

```
generate_until_high_watermark():
    while unused_count() < HIGH_WATERMARK:            # e.g. 10,000,000
        batch = [base62(randInt(0, 62^7)) for _ in range(N)]
        INSERT ... ON CONFLICT DO NOTHING             # dedup via unique index
```

> 💡 **Why generate randomly and dedup with a unique index, instead of "generate then check"?** The unique constraint *is* the check — the DB enforces it atomically, so two generators can't both insert the same key. You never write application code to "look up if it exists"; you just insert and let a conflict mean "skip." Simple and race-free.

> 💡 **Why not base62-encode a counter offline (like counter+base62)?** That would make keys sequential/guessable — the exact problem KGS avoids. Random integers keep keys **non-enumerable**. (You *could* encode a *permuted/encrypted* counter to guarantee uniqueness without a dedup check — Feistel/`ID → shuffled ID` — a nice variant to mention, but random+unique-index is simpler.)

---

## 5. How we STORE keys in the DB

Two common schemas — pick based on whether you keep used keys.

### Option A — single table with a status flag
```sql
CREATE TABLE key_pool (
    key       CHAR(7)   PRIMARY KEY,     -- the short key (unique by PK)
    status    SMALLINT  NOT NULL,        -- 0 = unused, 1 = used
    created_at TIMESTAMP DEFAULT now(),
    used_at    TIMESTAMP                 -- set when handed out
);
CREATE INDEX idx_pool_unused ON key_pool (status) WHERE status = 0;  -- partial index on unused
```
- Pull = flip `unused → used` on a batch (§6). Keeps history of used keys.
- Cost: table grows to ~1.5 TB over 5 years (still fine, but grows).

### Option B — two tables (classic KGS): `unused_keys` + `used_keys`
```sql
CREATE TABLE unused_keys (key CHAR(7) PRIMARY KEY);
CREATE TABLE used_keys   (key CHAR(7) PRIMARY KEY, used_at TIMESTAMP DEFAULT now());
```
- Pull = **move** N rows from `unused_keys` to `used_keys` in a transaction (delete + insert).
- Keeping `used_keys` lets a future generator avoid ever re-minting an assigned key.

### Option C — delete-on-use (leanest)
- Only an `unused_keys` table; pulling a batch **deletes** those rows. The **URL-mapping table** (`short_key → long_url`) is the source of truth for "what's assigned," so you don't need `used_keys` at all.
- Keeps the pool DB **flat at ~0.5 GB** (only unused keys ever live there). The generator must draw from the full 62^7 space and dedup against *both* the pool and the URL-map — or, simpler, just accept the astronomically-low chance of re-minting an already-assigned key and let the URL-map's unique constraint reject it.

> 💡 **Which to pick?** For an interview, describe **Option B (unused + used)** as the canonical KGS design — it's the one in most references and clearly separates "available" from "spent." Mention **Option C** as the storage-lean variant when you don't want the used table to grow, noting the URL-map table already guarantees no key is reused. The **partial index on `status = unused`** (Option A) is a nice detail to call out: the pull query only scans unused rows.

> 💡 **A separate concern: the URL-mapping store.** KGS's DB holds *keys*; the URL service's DB holds the *mapping* `short_key (PK) → long_url, created_at, expiry, owner`. That's typically a key-value / wide-column store (e.g., Cassandra/DynamoDB) partitioned by `short_key` for O(1) redirect lookups. KGS and the mapping store are independent.

---

## 6. The atomic hand-off (pulling a batch)

The **one correctness-critical operation**: two app servers (or two KGS threads) must **never** receive the same key, or you reintroduce collisions. The mark-as-used must be **atomic**.

### Postgres — `FOR UPDATE SKIP LOCKED` (modern, low-contention)
```sql
-- Option A (status flag): atomically claim 1,000 unused keys and mark them used.
WITH claimed AS (
    SELECT key FROM key_pool
    WHERE status = 0
    ORDER BY key                    -- or random; any deterministic pick
    LIMIT 1000
    FOR UPDATE SKIP LOCKED          -- concurrent pulls skip each other's locked rows
)
UPDATE key_pool p
SET status = 1, used_at = now()
FROM claimed
WHERE p.key = claimed.key
RETURNING p.key;                    -- returns the 1,000 keys to this server
```
`SKIP LOCKED` is the key: two servers pulling simultaneously grab **disjoint** rows instead of blocking on each other.

### Two-table variant (Option B)
```sql
BEGIN;
  -- select + lock 1,000 unused
  SELECT key FROM unused_keys LIMIT 1000 FOR UPDATE SKIP LOCKED;   -- -> :keys
  INSERT INTO used_keys (key) SELECT unnest(:keys);
  DELETE FROM unused_keys WHERE key = ANY(:keys);
COMMIT;
```

### Redis fast-path (optional)
Front the DB pool with a Redis set/list; `SPOP key_pool 1000` (or `RPOP`) pops N atomically in one round-trip. A background job refills Redis from the DB. Cuts hand-off latency to sub-millisecond.

> 💡 **Why `SKIP LOCKED` and not a plain `SELECT … FOR UPDATE`?** Plain `FOR UPDATE` makes the second concurrent puller **block** until the first commits — serializing all pulls. `SKIP LOCKED` lets each puller grab a *different* set of unlocked rows, so N servers pull N disjoint batches in parallel. This is the exact "lock the rows you take, don't serialize everyone" principle. If your DB lacks it, an alternative is a single-row **range cursor** ("next unused offset") incremented atomically — but that reintroduces sequential-ish selection unless the pool rows themselves are randomly ordered.

---

## 7. How the app server uses a batch (request time)

Once a server holds a batch **in memory**, creating a short URL is instant — no per-request KGS call.

```
class KeyBuffer {                       // per app-server, in memory
    Deque<String> keys;                 // e.g. 1,000 pre-fetched
    void refillIfLow() {                // background, when size < LOW
        keys.addAll(kgs.pullBatch(1000));   // atomic hand-off (§6)
    }
    String next() {                     // O(1) pop
        if (keys.size() < LOW) asyncRefill();
        return keys.poll();             // hand this key to the new URL
    }
}

createShortUrl(longUrl):
    key = keyBuffer.next();                     // instant, no collision check
    urlStore.put(key, longUrl);                 // write the mapping
    return BASE_DOMAIN + "/" + key;
```

- **No collision check** — keys were pre-validated unique.
- **No blocking** — refill happens asynchronously before the buffer empties.
- **A create = one in-memory pop + one mapping write.**

> 💡 **Why buffer a batch on the server instead of calling KGS per request?** A per-request KGS round-trip would add latency and make KGS a hot dependency. Pulling 1,000 at once amortizes that to one KGS call per 1,000 creates, and the create path becomes a local pop. Refill *before* running dry (low-watermark on the in-memory buffer) so the create path never waits.

---

## 8. How we MAINTAIN the pool (replenish + monitor)

The pool is a consumable resource; it must be **topped up before it runs dry**, or creates stop.

**Replenishment loop (background generator):**
```
every T seconds:
    unused = SELECT count(*) FROM unused_keys        -- or WHERE status=0
    if unused < LOW_WATERMARK (e.g. 2,000,000):
        generate_until(HIGH_WATERMARK, e.g. 10,000,000)   # §4: random + ON CONFLICT DO NOTHING
```

**Monitoring & alerting (name these unprompted):**
- **`unused_key_count`** gauge — alert if it drops below the low watermark.
- **Consumption rate** (keys/sec) and **generation rate** — dashboards.
- **Runway = unused_count / consumption_rate** — the "hours until dry" number; page if < a few hours. (At 193/sec, a 2M watermark = ~2.9 h of runway — verified.)
- **Generation conflict rate** — if `ON CONFLICT DO NOTHING` skips spike, the space is filling up (time to widen to 8 chars).

**Watermarks:** LOW triggers top-up; HIGH is the refill target. Set LOW so `LOW / peak_consumption` > (generation time + alert response time). With peak 965/sec, 2M gives ~35 min even at peak — still safe; bump to 5–10M if you want more cushion (still < 0.5 GB).

> 💡 **Running out is the one truly bad outcome** (creates fail). So the maintenance story is really about **never hitting zero**: a background generator that stays ahead, watermarks with generous margin, and a runway alert with hours of lead time. Storage is cheap (§3), so err toward a *large* unused buffer.

---

## 9. Architecture & sequence diagrams

```
                      ┌─────────────────────────────────────────┐
   create/redirect →  │             URL Service (N app servers)    │
                      │   each holds an in-memory KeyBuffer(1000)  │
                      └───────┬──────────────────────┬────────────┘
                              │ pullBatch(1000)        │ put(key→longUrl) / get(key)
                              ▼ (atomic hand-off)       ▼
                      ┌───────────────┐        ┌────────────────────┐
                      │      KGS        │        │  URL-Mapping Store  │
                      │  ┌───────────┐  │        │  short_key → longUrl│
                      │  │ pool DB    │  │        │  (Cassandra/Dynamo) │
                      │  │ unused/used│  │        └────────────────────┘
                      │  └───────────┘  │
                      │  ▲ tops up       │
                      │  │ (background    │
                      │  │  generator)    │
                      └──┴──────────────┘
                         ▲ monitor: unused_count, runway, alert
```

**Sequence — pull a batch (atomic):**
```
AppServer            KGS / pool DB
   │  pullBatch(1000)   │
   │───────────────────►│  UPDATE ... WHERE status=unused
   │                    │  LIMIT 1000 FOR UPDATE SKIP LOCKED
   │                    │  SET status=used RETURNING key
   │◄───────────────────│  [1000 unique keys]
   │  (store in buffer) │
```

**Sequence — create a short URL (request time, no KGS call):**
```
Client            AppServer                 URL store
  │  POST /shorten    │                          │
  │──────────────────►│ key = buffer.next()      │  (in-memory pop, O(1))
  │                   │ put(key → longUrl)        │
  │                   │─────────────────────────►│
  │◄──────────────────│  https://sho.rt/9Xk2mPq  │
```

---

## 10. Scaling KGS

- **Multiple app servers** — each pulls its own batches; the atomic hand-off (§6) guarantees disjoint keys. This scales linearly.
- **Multiple KGS/generator instances** — they must not mint the *same* key. Two safe options: (a) the **unique index dedups** inserts regardless of how many generators run (simplest); (b) **partition the key space** — each generator owns a disjoint range or prefix so they can't collide by construction.
- **Shard the pool DB** — if one DB can't hold/serve the pool, shard `unused_keys` by key prefix; a puller hits any shard. Rarely needed (pool is ~0.5 GB).
- **Redis front (§6)** — absorbs pull QPS so the DB only sees periodic refills.

> 💡 **The pull throughput matters more than storage.** The pool is tiny, but if every create needed a KGS round-trip you'd need serious pull QPS. Batching (§7) collapses that to ~1 pull per 1,000 creates, so a single KGS DB (optionally Redis-fronted) handles very high create rates.

---

## 11. High availability & the SPOF question

KGS is a dependency of every create, so treat availability seriously:

- **App-server buffers are a shock absorber.** Because each server holds ~1,000 keys in memory, KGS can be briefly unavailable and creates keep flowing until buffers drain. Size buffers so KGS can restart/failover within that window.
- **Replicate the pool DB** (primary + standby / synchronous replica); failover promotes the standby.
- **Run multiple KGS instances** behind the atomic hand-off; any instance can serve a pull.
- **Redis with persistence/replication** if used as the fast-path.

> 💡 **"Isn't KGS a single point of failure?"** — the honest answer: it's a dependency, but the **in-memory batches decouple creates from KGS availability** (creates survive a short KGS outage), and KGS itself is made HA by DB replication + multiple instances. That decoupling is the reason batching isn't just a latency optimization — it's also a resilience one.

---

## 12. Failure modes & how we handle them

| Failure | Effect | Handling |
|:--|:--|:--|
| **App server crashes with a pulled batch** | Those ≤1,000 keys are marked used but never assigned → **stranded/leaked** | Accept it (negligible — §3). Small batches bound the loss. Optionally a **lease** (mark used with a TTL; a reaper reclaims keys not confirmed-assigned in time) — more complex, usually not worth it. |
| **Two servers pull simultaneously** | Risk of the same key twice | Prevented by the **atomic hand-off** (`FOR UPDATE SKIP LOCKED` / transactional move). |
| **Pool runs dry** | Creates fail | **Watermarks + background top-up + runway alert** (§8); keep a generous buffer. |
| **Generator mints a duplicate** | Would corrupt uniqueness | **Unique index + `ON CONFLICT DO NOTHING`** — duplicate insert is silently skipped. |
| **KGS DB down** | Pulls fail | App buffers keep serving; DB replica failover; multiple KGS instances (§11). |
| **Same long URL shortened twice** | Two keys for one URL (usually fine) | If you want idempotency, index `long_url` (or its hash) and return the existing key on a hit — a separate concern from KGS. |

> 💡 **The stranded-key "leak" is the accepted cost, and you should name it unprompted.** It's exactly like the gaps a counter leaves when IDs are reserved-then-unused. At <1% space utilization and negligible leak rate, it never matters — but saying "I accept a small, bounded key leak on crashes, bounded by batch size" signals you've thought it through.

---

## 13. KGS vs hash-truncate vs counter+base62

| Property | Hash + truncate (MD5→base62→7) | Counter + base62 | **KGS** |
|:--|:--|:--|:--|
| **Collisions** | **Yes** — truncation shrinks space; check + retry per create | No (unique by construction) | **No** (pre-validated unique) |
| **Request-time cost** | hash + collision check (+ retries) | encode a counter | **pop from memory (fastest)** |
| **Guessable / enumerable** | No (random-ish) | **Yes** (sequential) | **No** (random) |
| **Extra infra** | none | an ID generator | **a whole service + pool DB + monitoring** |
| **Main risk** | collision storms as space fills | guessability; hot counter | pool running dry; stranded keys |

> 💡 **The pitch for KGS in one line:** it takes the counter's *collision-free + fast* and adds *unguessable*, by paying for an offline generation service. Choose it when you need **both** speed and non-enumerable keys; choose counter+base62 when guessability is acceptable and you want zero extra infra; avoid hash-truncate unless you're fine handling collisions.

---

## 14. Senior-level Q&A

**Q: How do you guarantee two servers never get the same key?**
The mark-as-used is a single atomic DB operation — `UPDATE … WHERE status=unused … FOR UPDATE SKIP LOCKED … SET status=used RETURNING key` (or a transactional move between `unused_keys`/`used_keys`). Concurrent pullers grab disjoint rows; no key is handed out twice.

**Q: What happens if a server pulls 1,000 keys and dies?**
Those keys are marked used but never assigned — leaked. Bounded by batch size, negligible at scale (0.0001%/yr even at 1,000 crashes/day). If leakage ever mattered, add a lease/TTL + reaper to reclaim unconfirmed keys.

**Q: How do you stop the pool running dry?**
Background generator tops up to a HIGH watermark when unused drops below LOW; monitor `unused_count` and **runway = unused/consumption_rate**, alert hours ahead. Storage is cheap, so keep a large buffer.

**Q: Why 7 characters?**
62^7 ≈ 3.5 T keys — ~587 years of runway at 500M/month; 5 years uses <1%. Short to share, huge headroom. When conflict-rate on generation climbs (space filling), widen to 8 chars.

**Q: How is KGS not a single point of failure?**
In-memory batches on app servers decouple creates from KGS uptime (creates continue during a short outage); KGS itself is HA via DB replication + multiple instances (+ optional Redis).

**Q: Can two generators create duplicate keys?**
The unique index makes duplicate inserts a no-op (`ON CONFLICT DO NOTHING`), so any number of generators is safe; optionally partition the key space so they can't collide at all.

**Q: Where does the short_key → long_url mapping live?**
In a separate URL-mapping store (KV/wide-column, partitioned by `short_key`) for O(1) redirects — independent of KGS, which only supplies keys.

**Q: How is this different from just base62-encoding a counter?**
Counter keys are sequential/guessable. KGS keys are random, so they're non-enumerable — while still being unique and pop-fast.

---

## 15. TL;DR

**Generate:** a background generator makes random keys (random int in `[0,62^7)` → base62 → 7 chars), deduped by a **unique index** (`INSERT … ON CONFLICT DO NOTHING`) — all collision handling is **offline**, never on the request path.

**Store:** a pool DB, either single-table with a `status` flag (+ partial index on unused) or two tables `unused_keys`/`used_keys`; the `short_key → long_url` mapping lives in a **separate** KV store. Unused pool is tiny (~0.5 GB for 10M keys).

**Hand off:** app servers pull **batches** (e.g., 1,000) via an **atomic** claim (`FOR UPDATE SKIP LOCKED` or a transactional move; optional Redis `SPOP`), so no key is ever handed out twice.

**Use:** each server keeps its batch in memory; a create is an **O(1) in-memory pop + one mapping write** — no collision check, no blocking.

**Maintain:** watermark-driven top-up, and monitor `unused_count` + **runway** with an alert hours before dry.

**Costs to name unprompted:** an extra service to operate; the hand-off must be atomic; crashes strand ≤ batch-size keys (accepted, bounded); the pool must never run dry (supply monitoring).

### Capacity cheat sheet (verified)
```
62^7 = 3.52 trillion keys   → ~587 years at 500M/month (5 yrs = 0.85% used)
7 base62 chars ≈ 41.7 bits  → key = random 42-bit int, base62-encoded
Unused pool 10M keys ≈ 0.5 GB   |   keep-all-used 30B ≈ 1.5 TB   |   delete-on-use ≈ flat 0.5 GB
Batch 1000 → crash strands ≤ 1000 keys (0.0001%/yr — negligible)
Runway: 2M watermark @ 193/sec ≈ 2.9 h to top up
```

### One-line philosophy
> **KGS pre-bakes random, unique 7-char base62 keys offline (deduped by a unique index), stores them as an unused/used pool, and hands app servers batches through one atomic claim — so creating a short URL becomes an in-memory pop that's collision-free *and* unguessable. The price is an extra service whose only real risks are running dry (fix with watermarks + runway alerts) and a negligible, bounded key leak on crashes.**
