# Distributed Unique ID Generator — System Design (HLD, Detailed)

> A worked design of a **distributed unique ID generator** — the service that mints IDs for tweets, posts, messages, orders, or any entity, across many machines, at high throughput. The canonical answer is **Twitter Snowflake**: a 64-bit ID packed from a **timestamp + machine id + per-millisecond sequence**, so every machine generates globally-unique, roughly time-sortable IDs **with zero coordination per ID**. This doc builds it from requirements, compares the alternatives, details the bit-packing and generation logic, and handles the two things that break it: **clock skew** and **machine-id assignment**.

> 💡 **The whole design in one sentence:** to avoid a central bottleneck, don't ask anyone for each ID — give each machine a unique id and let it mint IDs locally as `timestamp || machine_id || sequence`; the timestamp makes IDs sortable and unique-across-time, the machine id makes them unique-across-machines, and the sequence makes them unique-within-a-millisecond — three fields whose combination is always distinct, no locks or network calls.

---

## Table of Contents
1. [Problem & requirements](#1-problem--requirements)
2. [Why it's not trivial](#2-why-its-not-trivial)
3. [The candidate approaches](#3-the-candidate-approaches)
4. [Snowflake: the 64-bit layout](#4-snowflake-the-64-bit-layout)
5. [Why each field exists](#5-why-each-field-exists)
6. [The generation algorithm](#6-the-generation-algorithm)
7. [Why it's unique (the proof)](#7-why-its-unique-the-proof)
8. [Clock skew — the hard problem](#8-clock-skew--the-hard-problem)
9. [Assigning machine IDs](#9-assigning-machine-ids)
10. [Sequence exhaustion](#10-sequence-exhaustion)
11. [Capacity estimation](#11-capacity-estimation)
12. [The architecture](#12-the-architecture)
13. [Alternatives in depth](#13-alternatives-in-depth)
14. [Security & privacy (leaky IDs)](#14-security--privacy-leaky-ids)
15. [Edge cases](#15-edge-cases)
16. [Trade-offs & talking points](#16-trade-offs--talking-points)
17. [How to present this in the interview](#17-how-to-present-this-in-the-interview)
18. [Common mistakes to avoid](#18-common-mistakes-to-avoid)
19. [TL;DR](#19-tldr)

---

## 1. Problem & requirements

Design a service/library that generates unique IDs for entities (tweets, posts, messages, orders) across a large distributed system.

### Functional
- Produce an ID that is **globally unique** — across all servers, forever.
- Callable from **many machines** concurrently at high rate.

### Non-functional (these shape the design)
- **Uniqueness** — no collisions, ever (a duplicate tweet id is a data-corruption bug).
- **High throughput** — tens of thousands to millions of IDs/sec, generated on many nodes.
- **No coordination per ID** — asking a central authority for every ID doesn't scale and is a SPOF.
- **Roughly time-sortable** — for feeds/timelines you want `ORDER BY id` ≈ chronological, and IDs increasing over time help DB index locality.
- **Compact** — a **64-bit integer** (`long`/`bigint`), not a 128-bit UUID or a long string.
- **Highly available** — the generator can't be a single point of failure.

> 💡 **Clarify these three first:** "**64-bit or is 128-bit OK?**, **must it be time-sortable?**, and **is any information-leak from sequential IDs a concern?** Those answers pick the design — Snowflake if you need 64-bit + sortable, UUID if you don't care about size/order, a hashid wrapper if leakage matters." Leading with the requirements steers straight to the right family.

---

## 2. Why it's not trivial

The naive answer — "a database auto-increment column" — fails the moment you have more than one DB:

- **A single counter is a bottleneck and a SPOF** — every ID generation serializes on one machine; if it's down, nothing gets an ID.
- **Coordinating across machines is expensive** — a lock or consensus per ID adds a network round trip to *every* insert.
- **Random IDs (UUIDv4) solve uniqueness but lose ordering** — and 128 random bits wreck B-tree index locality (inserts scatter across the index → page splits, poor cache behavior).

So the real problem is: **uniqueness across many machines, at high rate, without per-ID coordination, while staying sortable and compact.** That combination is what makes it a design question.

> 💡 **Name the tension:** "Uniqueness is easy alone (UUID) and ordering is easy alone (one counter) — the hard part is *both*, across many machines, without coordinating on each ID. Snowflake's trick is to partition the ID space by machine so no coordination is needed." That's the crux the whole design turns on.

---

## 3. The candidate approaches

| Approach | Unique | Sortable | Scales (no per-ID coord) | 64-bit | Verdict |
|:---------|:------:|:--------:|:------------------------:|:------:|:--------|
| **DB auto-increment** | ✅ | ✅ | ❌ (single DB) | ✅ | bottleneck + SPOF |
| **UUIDv4 (random)** | ✅ | ❌ | ✅ | ❌ (128-bit) | not sortable; hurts index locality |
| **Central ticket server** | ✅ | ✅ | ⚠️ (1 hop/ID) | ✅ | SPOF unless replicated; per-ID call |
| **DB ticket, multi-master** (Flickr) | ✅ | ✅ | ✅ | ✅ | simple; less time-precise |
| **Batch/segment allocation** | ✅ | ✅ | ✅ | ✅ | amortizes central call; good |
| **Snowflake** (ts+machine+seq) | ✅ | ✅ (time) | ✅ (no coord) | ✅ | **the standard** |
| **UUIDv7 / ULID** | ✅ | ✅ (time) | ✅ | ❌ (128-bit) | sortable, no machine-id mgmt, but 128-bit |

**Snowflake wins** when you need 64-bit + time-sortable + no coordination. **UUIDv7/ULID** is the modern shortcut if 128 bits is acceptable and you'd rather not manage machine IDs.

> 💡 **The UUIDv4 critique earns points:** "UUIDv4 is unique and coordination-free, but it's 128-bit and *random* — so it's not time-sortable and it destroys index locality on inserts (random keys scatter across the B-tree). For a timeline you specifically want time-ordered keys, which is why Snowflake or UUIDv7." Showing you know *why* random IDs are bad here is a senior signal.

---

## 4. Snowflake: the 64-bit layout

Pack a signed 64-bit integer from three fields:

```
 63                    22            12         0
 ┌─┬───────────────────┬────────────┬──────────┐
 │0│  timestamp (41b)   │ machine(10)│ seq (12) │
 └─┴───────────────────┴────────────┴──────────┘
  ▲         ▲                 ▲           ▲
 sign   ms since         up to 1024    up to 4096
 (unused) custom epoch    machines     ids/ms/machine
        (~69 years)
```

| Field | Bits | Meaning | Range |
|:------|:----:|:--------|:------|
| sign | 1 | unused (keep positive) | — |
| **timestamp** | 41 | ms since a **custom epoch** | ~69 years |
| **machine id** | 10 | unique per generator node | 1024 nodes |
| **sequence** | 12 | per-ms counter, resets each ms | 4096/ms |

Building the ID is a shift-and-OR:

```
id = ((now - EPOCH) << 22) | (machineId << 12) | sequence
```

The **41/10/12 split is a convention, not law** — you can retune it (e.g. split the 10 machine bits into 5 datacenter + 5 worker, or trade sequence bits for more machines) to fit your scale.

> 💡 **Why a custom epoch?** "I count milliseconds from a recent custom epoch (say the product's launch), not from 1970 — that keeps the 41-bit timestamp from overflowing for ~69 years *from launch* instead of burning decades of range on pre-launch time." Small detail that shows you've actually thought about the bit budget.

---

## 5. Why each field exists

Each field guarantees uniqueness along one dimension and the timestamp adds sortability:

- **Timestamp (high bits)** → **time-sortability** (IDs increase over time, so sort/range-scan by id ≈ by creation time) *and* uniqueness across different milliseconds. Being the most-significant bits is what makes numeric order = time order.
- **Machine id** → uniqueness **across machines with no coordination**. This is the key idea: partition the ID space by machine so two machines *physically cannot* collide, without ever talking to each other.
- **Sequence** → uniqueness **within a single millisecond on one machine** (up to 4096 IDs/ms).

> 💡 **The machine-id field is the whole trick.** "Coordination-free uniqueness comes from giving each machine a distinct id embedded in every ID it mints — so uniqueness is guaranteed by construction, not by a lock or a central service. Timestamp handles across-time, sequence handles within-a-ms, machine-id handles across-machines." State it exactly like that.

---

## 6. The generation algorithm

Per-machine state: `lastTimestamp` and `sequence`, guarded by a small lock (or a CAS loop):

```
nextId():
    now = currentMillis()

    if now < lastTimestamp:                 # CLOCK WENT BACKWARDS
        refuse / wait until now >= lastTimestamp   # never emit a ts we've already used

    if now == lastTimestamp:                # same ms as last id
        sequence = (sequence + 1) & 4095    # increment within the ms
        if sequence == 0:                   # 4096 exhausted this ms
            now = waitNextMillis(lastTimestamp)   # spin until the next ms
    else:                                   # a new ms
        sequence = 0                        # reset the counter

    lastTimestamp = now
    return ((now - EPOCH) << 22) | (machineId << 12) | sequence
```

Only synchronization is a per-machine lock around this tiny critical section — **no network round trip, no shared state between machines.** (Verified: 1000 IDs in the same ms on one machine were all unique; decode recovered the exact timestamp/machine/sequence; two machines in the same ms produced different IDs; later timestamps produced strictly larger IDs.)

> 💡 **Decoding is a feature, not just a curiosity.** "Because the fields are packed positionally, I can *decode* any ID back into its timestamp, machine, and sequence — useful for debugging, sharding, and computing 'created at' from the id itself without a DB lookup." (Verified round-trip.)

---

## 7. Why it's unique (the proof)

For any two IDs, at least one field differs:

- **Different millisecond** → timestamps differ → IDs differ.
- **Same millisecond, different machine** → machine ids differ → IDs differ.
- **Same millisecond, same machine** → the sequence differs (it increments per ID within the ms) → IDs differ.

So the triple `(timestamp, machine, sequence)` is always distinct → the packed integer is always unique. (Verified across all three cases, plus a 5000-ID burst in one ms that rolled into the next ms — all 5000 unique.)

> 💡 **Recite the three-case proof** — it's short and it's exactly what proves correctness: "same ms different machine → machine bits; same ms same machine → sequence; different ms → timestamp. One of the three always differs, so no two IDs collide." Interviewers love a crisp uniqueness argument.

---

## 8. Clock skew — the hard problem

The entire scheme assumes the clock **only moves forward**. If NTP steps the clock **backward**, a machine could reuse a timestamp it already issued IDs for → **collisions**. This is *the* failure mode interviewers probe.

Handling, by severity of the backward jump:
- **Small drift (a few ms):** clamp — keep issuing from `lastTimestamp` (don't go back) until real time catches up; the sequence still guarantees uniqueness.
- **Large jump:** **refuse to generate** and wait/alert until `now >= lastTimestamp`. Better to pause ID generation briefly than to emit duplicates. (Verified: a real backward jump is refused.)
- **Prevention:** run NTP in **slew mode** (gradually adjust, never step backward); monitor clock health; use monotonic time where possible.

The invariant to preserve: **never emit an ID whose timestamp is ≤ the last one you already used** (per machine).

> 💡 **The clock-backwards answer is the killer follow-up — have it ready:** "The scheme assumes monotonic time. If the clock jumps back, I refuse to generate and wait rather than risk a duplicate timestamp; for tiny drift I clamp to the last timestamp and let the sequence carry uniqueness; and I run NTP in slew mode so it never steps backward. Correctness beats availability here — a paused generator is fine, a duplicate id is not." Say it in that order.

---

## 9. Assigning machine IDs

Two machines with the same 10-bit id **will collide** — so machine-id assignment must itself be unique:

- **Static config** — assign ids in deployment config. Simple; error-prone at scale (humans reuse ids).
- **Dynamic lease from a coordination service** — on startup, a node leases a free machine id from **ZooKeeper / etcd / Consul** (Twitter used ZooKeeper). Handles autoscaling and restarts.
- **Split the bits** for more nodes — e.g. 5 bits datacenter + 5 bits worker, or borrow from the sequence if you need >1024 machines.
- **Cloud-derived** — derive from a stable host attribute (instance id hash) if collisions are acceptably improbable.

> 💡 **This is the *other* coordination — but it's one-time, not per-ID:** "Machine ids must be unique, so nodes lease one from ZooKeeper at startup. Note the coordination is *once per node lifetime*, not once per ID — which is exactly why the design scales: the only central interaction is claiming a machine id, then the node mints billions of IDs locally." Distinguishing startup-coordination from per-ID-coordination is the point.

---

## 10. Sequence exhaustion

12 bits = **4096 IDs per millisecond per machine** (~4M/sec/machine). If a single machine needs more than 4096 in one ms, the sequence rolls over → **wait for the next millisecond** (verified: a 5000-ID burst spilled from ms N into ms N+1, all unique). This is rare; if it's routine, retune the bit split (more sequence bits) or add machines.

> 💡 **Rollover is graceful, not a failure:** "If a machine exhausts 4096 ids in one ms, it briefly waits for the next ms — IDs stay unique, you just borrow from the next millisecond. 4M/sec/machine is a lot of headroom; if you need more, widen the sequence field or scale out machines." 

---

## 11. Capacity estimation

| Quantity | Value |
|:---------|:------|
| Sequence per ms per machine | 4096 |
| **Per machine** | ~**4.1M IDs/sec** |
| Machines (10 bits) | 1024 |
| **Theoretical total** | ~**4.2 billion IDs/sec** |
| Timestamp range (41 bits) | ~**69 years** from the custom epoch |

This dwarfs any realistic need (Twitter peaks are ~hundreds of thousands of tweets/sec), so the bit budget is comfortable — and retunable if a dimension is tight.

> 💡 **The capacity headroom point:** "4096/ms/machine × 1024 machines ≈ 4 billion IDs/sec and ~69 years of range — orders of magnitude beyond real load. So the design isn't throughput-constrained; the engineering effort goes into clock handling and machine-id assignment, not raw capacity." Reframes where the real work is.

---

## 12. The architecture

Two deployment shapes:

**A) Embedded library (preferred for latency).** The generator runs *in each application service* — no network hop; each app instance is a "machine" with its own leased id.

```
   app instance (machine 7)   app instance (machine 42)   app instance (machine 108)
        │ nextId() local            │ nextId() local            │ nextId() local
        ▼                            ▼                            ▼
   [ ts | 7  | seq ]           [ ts | 42 | seq ]           [ ts | 108| seq ]
        └──────────── each leases its machine id at startup ────────────┐
                                                                         ▼
                                                        ZooKeeper / etcd (machine-id leases)
```

**B) Dedicated ID service (a few nodes).** A small clustered service exposes `getId()`; clients call it. Adds a network hop but centralizes clock/machine-id management. Often each node also **batches** ids to clients to amortize the hop.

```
   clients ──► ID Service (nodes with distinct machine ids) ──► ZooKeeper (machine-id leases)
                     └── each node mints locally; may hand out batches
```

- **Embedded** = lowest latency, no extra service; every app node needs a machine id.
- **Dedicated** = fewer machine ids to manage, isolates the concern; costs a hop (mitigated by batching).

> 💡 **Prefer embedded, mention the trade:** "I'd embed the generator in each service so there's no network hop per id — each instance leases a machine id from ZooKeeper at boot and mints locally. A dedicated ID service is cleaner operationally but adds a hop, which I'd amortize by handing out id batches." 

---

## 13. Alternatives in depth

**Flickr-style multi-master DB ticket.** Two MySQL servers with `auto_increment_increment=2`; one starts at 1 (odd ids), the other at 2 (even ids). No single SPOF, monotonic-ish, dead simple. Great when you don't need ms-precision or huge scale. Downside: two DBs to run; ordering is per-server.

**Batch / segment allocation (range handout).** A central counter (in a DB) hands each app server a **block** (e.g. 1000 ids) at a time; the server mints locally from its block and only hits the counter every 1000 ids. Amortizes the central call ~1000×; ids are roughly ordered. Instagram used a Postgres scheme embedding a **shard id** in the id so the id also tells you where the row lives.

**UUIDv7 / ULID.** Modern 128-bit ids that are **time-ordered** (timestamp in the high bits + randomness). Sortable like Snowflake, *no machine-id management*, but 128-bit (bigger keys/index). Use when 128 bits is fine and you'd rather not run ZooKeeper.

> 💡 **Have a "simpler than Snowflake" option ready:** "If they don't need Snowflake's precision, the Flickr two-DB ticket or a batch-allocation service is far simpler and has no clock problem. And UUIDv7 gives time-ordering without machine ids if 128 bits is acceptable. Snowflake is the answer when you want 64-bit + sortable + no per-id coordination *specifically*." Knowing when *not* to reach for Snowflake shows judgement.

---

## 14. Security & privacy (leaky IDs)

Time-sortable, near-sequential ids **leak information**:
- **Volume estimation** — comparing two ids reveals how many were created in between ("tweets/day").
- **Enumeration** — if ids are user-facing resource handles, an attacker can guess/scan neighbors.

Mitigations, if it matters:
- Keep the real Snowflake id **internal** and expose an **opaque/hashid** externally (a reversible or mapped public token).
- Accept it — Twitter's ids are public and sortable *by design* (the timeline benefits outweigh the leak).

> 💡 **Raise this unprompted for user-facing ids:** "Sequential ids leak creation volume and enable enumeration, so if the id is user-facing I'd expose an opaque hashid and keep the Snowflake id internal. If it's an internal key or a public feed id like a tweet, sortability is worth the leak." Showing you weigh the privacy trade is a differentiator.

---

## 15. Edge cases

| Case | Handling |
|:-----|:---------|
| Clock moves backward (small) | clamp to lastTimestamp; sequence carries uniqueness |
| Clock moves backward (large) | refuse + wait/alert (verified) |
| >4096 ids in one ms/machine | roll into next ms (verified) |
| Two machines same id | prevent via ZooKeeper lease; monitor |
| Node restart | re-lease (or safely reuse) machine id; ensure lastTimestamp not regressed |
| Timestamp overflow (69y) | pick a custom epoch; re-tune bits well before then |
| Need >1024 machines | split bits (datacenter+worker) or borrow from sequence |
| Need strict global ordering | not provided — same-ms cross-machine ids aren't strictly ordered (approximate only) |

> 💡 **Be honest about "sortable ≠ strictly ordered":** "Snowflake gives *approximate* global ordering — two ids created in the same ms on different machines aren't strictly ordered relative to each other. That's fine for feeds; if you truly need a total order you need a single sequencer, which reintroduces the bottleneck." Naming the limitation is more credible than overclaiming.

---

## 16. Trade-offs & talking points

- **Coordination-free vs strict ordering** — machine-id partitioning removes coordination but gives only approximate ordering.
- **64-bit Snowflake vs 128-bit UUIDv7** — size/sortability + machine-id management vs simplicity.
- **Embedded vs dedicated service** — latency (no hop) vs operational isolation.
- **Sortable vs privacy** — index locality/feeds vs information leak (hashid if needed).
- **Availability vs correctness on clock skew** — pause generation rather than emit duplicates.
- **Bit budget** — timestamp/machine/sequence split is tunable to your scale.

---

## 17. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Clarify | 64 vs 128 bit? time-sortable? leak concern? throughput? |
| Reject naïve | single counter (bottleneck), UUIDv4 (unsortable, big) |
| Snowflake layout | 64-bit = timestamp + machine + sequence; draw the bit split |
| Why each field | across-time / across-machine (no coord) / within-ms |
| Generation + uniqueness proof | the algorithm; the three-case proof |
| Clock skew | the killer follow-up — refuse/clamp/slew |
| Machine-id assignment | ZooKeeper lease (startup coord, not per-id) |
| Capacity + alternatives + privacy | 4B/sec, 69y; Flickr/batch/UUIDv7; hashid |

### What to say
- *"To avoid a central bottleneck, each machine mints ids locally: timestamp + machine-id + sequence — unique by construction, no per-id coordination."*
- *"Timestamp high bits make it time-sortable; machine-id gives cross-machine uniqueness without coordination; sequence covers within-a-millisecond."*
- *"The gotcha is the clock going backwards — I refuse and wait rather than risk a duplicate timestamp, and run NTP in slew mode."*
- *"Machine ids are leased from ZooKeeper once at startup — that's the only coordination, and it's not per id."*
- *"If ids are user-facing I'd expose a hashid to avoid leaking volume/enumeration."*

### Order
clarify → reject naïve → Snowflake layout → why-each-field → generation + proof → clock skew → machine ids → capacity/alternatives/privacy.

---

## 18. Common mistakes to avoid

- ❌ **Single DB auto-increment / one central counter** — bottleneck + SPOF.
- ❌ **UUIDv4 by default** — 128-bit and random (unsortable, wrecks index locality).
- ❌ **Ignoring clock-backwards** — the classic bug; must refuse/clamp, run NTP slew.
- ❌ **Not making machine ids unique** — duplicate machine id → collisions; lease from ZK.
- ❌ **Coordinating per id** — reintroduces the bottleneck; coordinate only at startup (machine id).
- ❌ **Claiming strict global ordering** — Snowflake is only approximately ordered.
- ❌ **Burning timestamp range from 1970** — use a custom epoch.
- ❌ **Forgetting sequence rollover** — wait for next ms when 4096 exhausted.
- ❌ **Exposing sequential ids without weighing the leak** — hashid for user-facing ids.
- ❌ **Over-engineering** — if you don't need Snowflake, Flickr/batch/UUIDv7 may be simpler.

---

## 19. TL;DR

### The design
```
64-bit Snowflake id = [1 sign][41 timestamp ms since custom epoch][10 machine id][12 sequence]
id = ((now - EPOCH) << 22) | (machineId << 12) | sequence
Each machine mints LOCALLY — no per-id coordination.
```

### Why it works
```
across-machines  → machine id (leased once from ZooKeeper at startup)
across-time      → timestamp (also makes ids TIME-SORTABLE, high bits)
within-a-ms      → sequence (4096/ms/machine; rolls into next ms if exhausted)
uniqueness proof : any two ids differ in ts, machine, or sequence.
```

### The gotchas (verified)
```
Clock backwards → REFUSE + wait (or clamp for tiny drift); NTP slew mode. Never reuse a timestamp.
Machine id      → must be unique (ZooKeeper lease); startup-coordination, NOT per-id.
Sequence full   → wait for next ms (verified: 5000 ids in one "ms" spill to the next).
```

### The four things that score points
1. **Snowflake layout** — 64-bit timestamp + machine + sequence; each machine mints locally.
2. **No per-id coordination** — machine-id partitioning; only startup coordination (ZK lease).
3. **Time-sortable + uniqueness proof** — high-bit timestamp; the three-case argument.
4. **Clock-skew handling** — refuse/clamp/slew; correctness over availability.

> **One-line philosophy:** *A distributed id generator avoids the central-bottleneck trap by partitioning the id space per machine — each node stamps every id with the current millisecond, its own uniquely-leased machine id, and a per-millisecond sequence, so ids are globally unique by construction and roughly time-sortable with zero coordination per id; the only real hazards are the clock moving backwards (refuse rather than duplicate) and machine-id collisions (lease once at startup), and both are cheap to guard against.*
