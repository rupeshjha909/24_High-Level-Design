# Designing a Cab Booking System (Uber / Ola): A Senior Interview Guide

> A practical, interview-focused reference for a ride-hailing platform — a **geospatial, real-time matching engine over a location firehose**. The distinctive challenge isn't ride volume (modest) but **ingesting millions of driver-location writes per second**, **finding the nearest available drivers in milliseconds**, and **dispatching each ride to exactly one driver** under contention. This guide builds the architecture up piece by piece, traces the full life of a ride, goes deep on the two genuinely hard mechanisms (how a driver's location physically reaches the rider's screen, and how a ride is claimed by exactly one driver), nails down the data contracts, and covers the geospatial index + boundary problem, surge, payments, sharding, and failure modes — with verified capacity math and a senior follow-up bank.

> 💡 **The whole system in one sentence:** drivers continuously stream location into an in-memory **geospatial index** (geohash / quadtree / S2 / Redis GEO); when a rider requests, you query that index for **nearby drivers (searching neighbor cells, not just one)**, rank by road **ETA**, **atomically dispatch** to one so no driver is double-booked, then drive the ride through a durable **state machine** while pushing live updates over a **WebSocket gateway** — with **surge** from local supply/demand and **exactly-once** payment at the end.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified)](#3-capacity-estimation-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a ride](#44-the-end-to-end-life-of-a-ride)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Trip State Machine](#5-the-trip-state-machine)
6. [Hard Problem 1: How a Driver's Location Reaches the Rider's Screen](#6-hard-problem-1-how-a-drivers-location-reaches-the-riders-screen)
7. [Hard Problem 2: How a Ride Is Matched to Exactly One Driver](#7-hard-problem-2-how-a-ride-is-matched-to-exactly-one-driver)
8. [Geospatial Search: How "Nearest Drivers" Actually Works](#8-geospatial-search-how-nearest-drivers-actually-works)
9. [Data Contracts: Request Fields, Payloads & DB Schemas](#9-data-contracts-request-fields-payloads--db-schemas)
10. [Surge / Dynamic Pricing](#10-surge--dynamic-pricing)
11. [Payments (exactly-once)](#11-payments-exactly-once)
12. [Scaling Summary (geo-sharding)](#12-scaling-summary-geo-sharding)
13. [Failure Modes & Handling](#13-failure-modes--handling)
14. [Senior Follow-Up Questions (with Answers)](#14-senior-follow-up-questions-with-answers)
15. [Quick Glossary](#15-quick-glossary)

---

## 1. How to Approach This in an Interview

The thing that makes ride-hailing distinctive — and where you should spend your time — is that it's a **real-time geospatial system**: the platform must continuously know **where every driver is**, **find the nearest available ones in milliseconds**, and **match a ride to exactly one driver** out of thousands, under contention, in seconds — then stream the driver's live position to the rider. Those mechanisms, not ride throughput, are the heart of the design.

A good structure:
1. **Clarify requirements** — request/match, live tracking, trip lifecycle, surge, payment.
2. **Estimate scale** — and *frame the two workloads*: a **firehose of driver-location writes** (~millions/sec, mostly discardable) and **low-latency proximity reads** at request time. Naming that split shapes everything.
3. **Establish the trip state machine** — the durable source of truth for a ride.
4. **Go deep on the two hard mechanisms** — real-time tracking fan-out and driver matching (dispatch), plus the geospatial index and its boundary problem.
5. **Cover surge, payments (exactly-once), sharding, failure modes.**

> 💡 **Senior signal:** say up front — *"Two very different loads: a location firehose I keep in an in-memory geo index (last-write-wins), and low-latency proximity reads at request time. The signature challenge is geospatial — 'nearest available drivers to a point,' which a B-tree can't do; you need a spatial index (geohash/quadtree/S2) so proximity is a cell lookup, not a full scan."* That framing is what the interview is really testing.

---

## 2. Requirements

### Functional
- **Rider** requests a ride (pickup, drop, ride type); sees nearby cars, gets a **quote**; tracks the driver **live** (position + ETA); pays; rates.
- **Driver** goes online/offline; streams **location**; receives ride **offers**; accepts/declines; marks arrived/started/completed.
- **Platform** finds **nearby available drivers**, **matches** exactly one, prices with **surge**, charges **exactly once**.

### Non-Functional
- **Low-latency matching** — a driver within seconds.
- **High write throughput** — millions of drivers pinging every few seconds.
- **Real-time** — bidirectional push (driver moving, ride accepted), not polling.
- **Consistent dispatch** — one driver ↔ exactly one ride (no double-booking).
- **Geo-scalable** — sharded by city; a New York surge shouldn't affect Mumbai.
- **Exactly-once payments**; **available** — a matching hiccup shouldn't strand riders.

> 💡 **Frame the two workloads.** *"There are two very different loads: a firehose of driver-location writes (millions/sec, mostly discardable) and low-latency proximity reads at request time. The geo index is optimized for both — cheap frequent upserts and fast nearest-neighbor queries."*

---

## 3. Capacity Estimation (verified)

**Assumptions:** ~5M active drivers at peak; ping every 4 s; 30M rides/day; ~25-min trips; rush-hour concentration.

```
LOCATION FIREHOSE  = 5M / 4s     ≈ 1,250,000 writes/sec   ← the real write load (ephemeral upserts)
Ride requests      = 30M/day → ~347/s avg, ~833/s peak    (MODEST — the low-latency read path)
   → the location firehose is ~1,500× the ride-request load
Concurrent trips (peak)          ≈ 1,250,000
WebSocket conns (peak)           ≈ 6,250,000  (all online drivers + watching riders)
   → gateway fleet ≈ 13–63 nodes (at 500k–100k conns/node)
Geo index memory  = 5M × ~100 B  ≈ 500 MB  (shard by city → per-city slice is tiny)
Proximity query   = rider cell + 8 neighbors = 9 cell lookups → small candidate set
```

**Takeaways that shape the design:**
- **Ride/dispatch QPS is modest (~833/s peak)** — *not* the hard part.
- **The location firehose (~1.25M writes/s, ~1,500× the ride load) is the real write pressure** → keep latest positions **in an in-memory geo index (last-write-wins)**, never a durable DB per ping.
- **~6.25M live connections** → a dedicated **WebSocket gateway fleet**.
- **Drivers/riders/surge are geo-local** → **shard by city**; each market is an almost-independent mini-system.

> 💡 The estimation reframes the naive assumption ("Uber = lots of rides"). No — rides are ~833/s. The scale is the **location firehose + live connections**. Say that; it directs the whole design.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

A ride-hailing system is fundamentally three loops running at once. **Loop 1 (location)** is a firehose: millions of drivers each ping their GPS every few seconds, and the platform must absorb those writes cheaply — the only thing that matters is each driver's *latest* position, so it's an **upsert into an in-memory geospatial index**, never a durable write per ping. **Loop 2 (matching/dispatch)** is a real-time geospatial problem: when a rider requests, you query that index for **nearby available drivers** (searching the rider's cell *and its neighbors*, because a close driver can sit just across a cell boundary), rank by road **ETA**, offer the ride, and let **exactly one** driver claim it via an **atomic** operation so a driver is never double-booked. **Loop 3 (tracking)** is a real-time streaming problem: once matched, the driver's ongoing pings must be pushed to the **one rider** watching that trip, over a **WebSocket gateway**. Around these sit a durable **Trip state machine** (the auditable source of truth for money and disputes), **surge pricing** derived from per-cell supply/demand, and **exactly-once** payment. And because a Mumbai driver never serves a Delhi rider, **everything is sharded by city** — the firehose, the index, dispatch, and surge all stay local.

### 4.2 The diagram

```
   rider app ── WebSocket ──┐            ┌── WebSocket ── driver app
                            ▼            ▼   (location pings every ~4s)
                     ┌───────────────────────────┐
                     │   Gateway (WebSocket fleet) │  registry(user→node) + pub/sub backplane
                     └───┬───────────────┬────────┘
          ride request   │               │ location ping
                         ▼               ▼
              ┌────────────────┐   ┌────────────────────┐  upsert latest (LWW)
              │ Trip Service    │   │ Location Service    │──────────────┐
              │ (state machine, │   │ (ingest firehose)   │              ▼
              │  durable DB)    │   └────────────────────┘     ┌────────────────────┐
              └───┬────────────┘                                │ Geospatial Index    │
                  │ match request                               │ (Redis GEO/geohash/ │
                  ▼                                             │  quadtree; in-mem)  │
          ┌────────────────────┐  query nearby (cell+neighbors) └─────────┬──────────┘
          │ Matching Service    │◄────────────────────────────────────────┘
          │ (proximity+ETA rank)│
          └───┬────────────────┘
              │ ranked candidates
              ▼
          ┌────────────────────┐  atomic claim (SETNX/CAS)   ┌──────────────────┐
          │ Dispatch Service    │───────────────────────────►│ Driver state      │
          │ (offer/accept)      │                            │ (FREE/BUSY)       │
          └───┬────────────────┘                            └──────────────────┘
              │ on complete
              ▼   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
                  │ Pricing/Surge   │  │ ETA/Routing     │  │ Payment (idem   │
                  │ (per-cell x)    │  │ (map graph)     │  │ key + reconcile)│
                  └────────────────┘  └────────────────┘  └────────────────┘
```

The key visual idea: driver pings flow **up** into the Location Service and the in-memory **geo index**; a ride request triggers a **proximity query** (cell + neighbors) → **rank by ETA** → **atomic dispatch**; and the matched driver's pings flow **out** over the gateway to the watching rider. The durable **Trip** is the system of record; location is ephemeral.

### 4.3 Each component, in detail

**① Client apps (rider, driver).** The **driver** app holds a WebSocket while online, streams a GPS **ping every ~4 s**, and receives ride **offers**. The **rider** app requests rides and, once matched, holds a WebSocket to receive the driver's live position + ETA. Neither holds authority; the server decides matching, pricing, and state.

**② Gateway (WebSocket fleet).** Terminates the ~6.25M live connections and routes events to the right socket via a **`user→node` registry + pub/sub backplane** (same pattern as any real-time system). Dumb on purpose — no business logic — so it scales and restarts freely; heartbeats + reconnect-resume handle mobile churn.

**③ Location Service (intake of Loop 1).** Ingests the ping firehose. For each ping it **upserts the driver's latest position** into the geo index (last-write-wins) and optionally samples a trail for the active trip/analytics. It **never** writes every ping to a durable DB — that would be a firehose of throwaway data.

**④ Geospatial Index (the heart).** An in-memory spatial structure — **Redis GEO / geohash / quadtree / S2** — that buckets the map into cells so "nearest drivers" is a **cell lookup**, not a full scan. Keyed by cell; sharded by city. ~500 MB total, tiny per city.

**⑤ Matching Service (brain of Loop 2, read side).** On a request, queries the index for candidates in the rider's **cell + neighbors** (§8), ranks them by **road ETA** (not straight-line), and returns the top few to Dispatch.

**⑥ Dispatch Service (brain of Loop 2, write side).** Offers the ride to the best candidate(s) with a timeout, and on accept performs the **atomic claim** (mark driver BUSY via `SETNX`/CAS) so exactly one ride wins the driver (§7). On decline/timeout, offers the next.

**⑦ Trip Service (the durable source of truth).** Owns the **trip state machine** (REQUESTED → … → PAID) in a durable, auditable store — every money/dispute-relevant fact (timestamps, fare, who cancelled) lives here. Comparatively low volume; strongly consistent.

**⑧ Pricing/Surge Service.** Computes a **per-cell** surge multiplier from live supply (drivers in the index) and demand (recent requests), quoted up front and locked for the request (§10).

**⑨ ETA/Routing Service.** Road-network routing (traffic-aware) for ranking candidates and estimating trip time/fare — a dedicated service or third party (Google Maps/OSRM).

**⑩ Payment Service.** Charges the fare **exactly once** via an idempotency key (trip id) + reconciliation; driver payout is a separate settlement (§11).

### 4.4 The end-to-end life of a ride

Here is exactly what happens when **a rider books a cab**, step by step:

```
CONTINUOUSLY (Loop 1):
0.  Every online driver streams a GPS ping every ~4s → Gateway → Location Service
        → UPSERT latest position into the in-memory geo index (last-write-wins). No DB write per ping.

REQUEST + MATCH (Loop 2):
1.  Rider requests (pickup, drop, ride type) → Trip Service creates trip (REQUESTED)
        → Pricing returns a SURGE-adjusted QUOTE, locked for this request.
2.  Matching Service queries the geo index: drivers in the rider's CELL + 8 NEIGHBORS
        → ranks the union by road ETA → top-K candidates.   ← §8 (boundary problem)
3.  Dispatch OFFERS the ride to the best candidate (push over the gateway; ~N-second timeout).
4.  Driver accepts → Dispatch performs the ATOMIC CLAIM:
        SETNX/CAS mark driver BUSY  → exactly ONE ride wins the driver   ← §7
        → trip MATCHED → ACCEPTED.   (decline/timeout → offer next candidate)

PICKUP → DROP (Loops 2+3):
5.  Driver's live pings now stream to the RIDER watching this trip:
        ping → Location Service → published to the driver's channel
        → Gateway node holding the rider's socket pushes {lat,lng,eta}   ← §6
        → the rider sees the car approach with a live ETA.
6.  Driver ARRIVES → trip ARRIVED; rider boards → IN_PROGRESS; route/ETA updates stream.
7.  Drop-off → trip COMPLETED → finalize fare (actual distance/time × locked surge).

PAY:
8.  Charge the fare EXACTLY ONCE (idempotency key = trip id; reconcile) → trip PAID
        → driver marked FREE (re-enters the dispatch pool) → ratings.
```

The single most important thing to notice: **the ephemeral location firehose (steps 0, 5) and the durable trip state machine (steps 1–8) are completely separate systems.** Location is huge, throwaway, in-memory, eventually consistent; the trip is comparatively small, durable, auditable, strongly consistent. Putting the firehose through your system-of-record is the classic mistake.

### 4.5 Why this split? (the design rationale)

- **Location Service separate from Trip Service** — location is a huge *ephemeral* firehose (last-write-wins, in-memory); trips are *durable* and few. Mixing them puts a firehose through your system-of-record. Different volume, durability, and consistency → different systems.
- **Matching (read) separate from Dispatch (write)** — proximity search is a read-heavy geospatial query; the atomic claim is a contended write. Splitting lets each scale on its own axis.
- **Trip as the durable source of truth** — money and disputes need an auditable state machine; location pings are not that.
- **Gateway separate and dumb** — holding ~6M connections is a different engineering problem from any business logic; isolating it lets it scale and restart freely.
- **Everything geo-local sharded by city** — a driver never serves another city, so city shards are independent: the firehose, index, dispatch, and surge all stay local, and one city's spike can't degrade another.

### 4.6 Where the load actually goes

A senior is expected to know *which* part is hard. The math (verified):

- **Ride/dispatch writes:** ~**833/s peak** — **trivial**; a modest cluster handles it. The atomic claim happens ~833×/s.
- **Location firehose:** ~**1.25M writes/s** — ~**1,500× the ride load**. This is the real write pressure, absorbed by **in-memory upserts** into the geo index (last-write-wins), **never** a durable DB per ping. The geo index is only ~**500 MB** total (tiny per city).
- **Live connections:** ~**6.25M** WebSockets → a **gateway fleet of ~13–63 nodes**, routed by a `user→node` registry + pub/sub backplane.
- **Proximity reads:** one 9-cell lookup + ETA rank per request at ~833/s — **cheap**.
- **Trips (durable):** millions/day, but low QPS and small rows — the auditable source of truth, not a bottleneck.

> 💡 **The senior framing:** *"Ride QPS is modest (~833/s). The load is the location firehose — ~1.25M writes/s, ~1,500× the ride load — plus ~6M live connections. So the design is: absorb the firehose with in-memory last-write-wins upserts into a spatial index, fan driver positions out over a WebSocket gateway, and keep the durable trip state machine entirely off the firehose. Shard by city because the domain is geo-local."*

---

## 5. The Trip State Machine

A trip is the durable **source of truth**, driven by an explicit, guarded state machine:

```
REQUESTED ─► MATCHED ─► ACCEPTED ─► ARRIVED ─► IN_PROGRESS ─► COMPLETED ─► PAID
     │           │          │
     └───────────┴──────────┴──────────► CANCELLED (by rider/driver, with policy)
                                          NO_DRIVERS_FOUND
```

| State | Meaning |
|:--|:--|
| REQUESTED | rider asked; matching in progress |
| MATCHED → ACCEPTED | driver offered → accepted |
| ARRIVED | driver at pickup |
| IN_PROGRESS | trip underway |
| COMPLETED → PAID | dropped off → fare charged |
| CANCELLED / NO_DRIVERS_FOUND | terminal branches |

Transitions are **guarded, atomic, and persisted**, so the Trip Service can recover and each event (arrive/start/complete) is validated. Cancellation policy (free window, cancel fee) hangs off the transitions.

> 💡 **The trip is the system of record; location is not.** Everything money- and dispute-relevant (matched/accepted/started/completed timestamps, fare, who cancelled) lives on the trip state machine. Location pings are ephemeral; the trip is durable and auditable.

---

## 6. Hard Problem 1: How a Driver's Location Reaches the Rider's Screen

This is the step most designs hand-wave — directly analogous to "how does a chat message get from server A to server B." The matched driver emits a GPS ping every few seconds; the rider watching that trip must see the car move. *How do the bytes physically get from the driver's phone to that one specific rider's screen, at ~1.25M pings/sec across the fleet?*

### The naive (wrong) approach
"Write each ping to the trips database; the rider polls for the driver's position." This fails two ways: **1.25M writes/sec into your OLTP database** will crush it (these are throwaway values, ~1,500× the ride load), and **polling** either wastes requests or lags. You must not persist the firehose, and you should push, not poll.

### The key realizations
**(1) You only ever need the *latest* position.** Nobody cares where the driver was 30 seconds ago (for the live view), so the ingestion store is an **upsert/overwrite**, not an append — `driver_id → (lat,lng,ts)` in the in-memory geo index. That turns an unbounded write stream into a tiny fixed-size store (~500 MB for 5M drivers).

**(2) A WebSocket is pinned to the process that holds it.** The rider's tracking socket lives on **one** node of the gateway fleet; the driver's ping arrives at the **Location Service** (a different process). So, exactly like cross-gateway delivery in a chat system, *something must carry the driver's position from the Location Service to the gateway node holding that rider's socket.* That "something" is a **pub/sub backplane**.

**(3) You must map trip → driver → watcher.** The rider is watching a **trip**; the position belongs to a **driver**. Dispatch established `trip → driver`. When the rider opens tracking, the gateway node **subscribes to that driver's location channel**.

### The mechanism, end to end
```
Driver app --(ping every 4s: lat,lng,ts)--> Location Service
   1. UPSERT latest position in the geo index:  driver:{id} → (lat,lng,ts)   (last-write-wins)
   2. PUBLISH the position to a channel:         driver-loc:{driverId}
   3. optionally sample to a durable trail for the active trip / analytics (NOT every ping)

Rider --(opens live tracking on trip T)--> Gateway node N
   4. node N resolves trip T → driver D (from the Trip/Dispatch data)
   5. node N SUBSCRIBES to channel driver-loc:{D}   (Redis Pub/Sub / NATS / Kafka)

On each ping:
   6. the backplane delivers driver-loc:{D} to whichever node(s) are subscribed → node N
   7. node N looks up the rider's socket in its LOCAL in-memory table
   8. node N attaches ETA (routing service) and writes {lat,lng,eta} down the rider's socket
   9. the rider's map animates the car.
```

Two maps, exactly like the chat/food-delivery designs:

| Map | Where | Answers |
|:--|:--|:--|
| **`trip → driver`** (+ driver latest pos in the geo index) | shared/global | "whose position do I need, and where is it?" |
| **Local socket table** | in each gateway node's RAM | "which actual socket on THIS node is this rider's?" |

The **backplane** physically moves a driver's position to the node holding the watcher's socket; the **local table** does the final write. The Location Service never talks to a rider socket directly — it publishes; the node holding that rider is subscribed.

> 💡 **The senior one-liner:** *"I never persist the firehose. Each ping upserts the driver's latest position into the in-memory geo index (last-write-wins) and is published to a per-driver pub/sub channel. When a rider opens tracking, their gateway node subscribes to that driver's channel, so every ping is pushed to exactly the node holding the rider's socket, which writes it down with an ETA. Last-write-wins storage plus a pub/sub backplane is what makes 1.25M pings/sec and millions of watchers affordable — and it stays completely off the trip database."*

---

## 7. Hard Problem 2: How a Ride Is Matched to Exactly One Driver

The second hard mechanism: a ride needs a driver; two nearby riders may target the same driver at the same instant; and **exactly one** ride must claim each driver — never two. *How does that physically happen under concurrency?*

### The steps
```
1. CANDIDATE SEARCH — Matching queries the geo index: "available drivers near the pickup?"
        = rider's cell + 8 NEIGHBOR cells (§8), union the drivers found. Widen radius if too few.
2. RANK — score candidates by road ETA to pickup (not straight-line), plus rating/acceptance.
3. OFFER — Dispatch pushes a ride OFFER to the best candidate with a TIMEOUT (~N seconds).
        Two dispatch styles: OFFER-ONE (sequential, fair) or BROADCAST-MANY (first-accept-wins, fast-fill).
4. RESPOND — driver ACCEPTS or DECLINES/TIMES-OUT:
        accept → go to 5;  decline/timeout → offer the next-best candidate (loop).
5. ATOMIC CLAIM — mark the driver BUSY ONLY IF currently FREE, in one indivisible step:
        SETNX driver:{id}:state = BUSY    (or a CAS / conditional UPDATE)
        success → trip ACCEPTED;  failure → someone else grabbed them → offer this ride elsewhere.
```

### Why the atomic claim is the crux
The dangerous moment is concurrency: two riders' dispatches could both find driver D FREE and both assign D. A naive **"check FREE, then set BUSY"** is a read-modify-write with a gap two requests race through — **double-booking** (verified: the non-atomic version double-books). The guarantee comes from a **single atomic test-and-set**: `SETNX`/CAS (or `UPDATE drivers SET state='BUSY' WHERE id=:d AND state='FREE'`). Only the first writer succeeds; the second sees the driver already BUSY and falls through to the next candidate (verified: 1 winner out of 1000 concurrent grabs). This is the same "atomic claim under contention" pattern as reserving a seat or a parking slot — *the operation itself*, not a lock you remember to take, guarantees single-assignment.

### Two dispatch styles to mention
- **Offer-one (sequential):** offer to the single best driver, wait, then the next — controlled and fair, slightly slower to fill.
- **Broadcast-to-many (first-accept-wins):** offer to several at once; the first to accept wins the atomic claim, the rest are told "taken" — faster fill, more contention on the claim.

> 💡 **The senior one-liner:** *"Matching is candidate geo-search → ETA rank → offer-with-timeout → **atomic claim**. Two riders can never get the same driver because assignment is a single test-and-set (`SETNX`/CAS marking the driver BUSY only if FREE) — the first wins, the rest fall through to the next candidate. A non-atomic check-then-set would double-book. Offer-one is fair; broadcast-many fills faster."*

---

## 8. Geospatial Search: How "Nearest Drivers" Actually Works

To answer "nearest drivers to (lat,lng)" fast you can't scan millions of drivers — you need a **spatial index** that buckets the map so proximity is a **cell lookup**:

| Technique | Idea |
|:--|:--|
| **Geohash** | encode (lat,lng) into a string; shared prefix = proximity; bucket drivers by prefix/cell |
| **Quadtree** | recursively subdivide space into 4; dense areas subdivide more (great for hotspots) |
| **S2 (Google)** | sphere → hierarchical cell ids (handles Earth's curvature) |
| **Redis GEO** | built-in `GEOADD`/`GEOSEARCH` (geohash under the hood) — the pragmatic choice |

Drivers are stored **keyed by cell**; a query looks up the rider's cell (and neighbors) → a small candidate set instead of everyone.

### The boundary problem (the classic follow-up)
The subtle bug: a driver can be **very close to the rider but in an adjacent cell**. Search only the rider's cell and you miss them. **Fix: search the rider's cell + its 8 neighbors** (a 3×3 block, or a ring sized to the radius), union the drivers, then rank.
```
query(rider):
   cells = riderCell + 8 neighbors        # cover boundary-straddling drivers
   candidates = union(drivers in those cells)
   rank by road ETA (haversine as a cheap first filter); keep within radius; return top-K
```
Then **rank by true cost — road ETA, not straight-line** (the nearest driver "as the crow flies" may be across a river). Haversine filters cheaply; ETA ranks accurately.

> 💡 **The senior point:** *"Cells create edges — a driver 50m away can be just across a boundary — so I always search the neighboring cells too, then rank the union by road ETA. Straight-line distance is a first filter; ETA is the real cost. And for hotspots like airports, a quadtree/S2 subdivides dense areas so one cell doesn't overflow with thousands of drivers."* Raising the boundary problem unprompted signals you've actually implemented proximity search.

---

## 9. Data Contracts: Request Fields, Payloads & DB Schemas

The sections above describe the *working*; this one nails down the concrete **data contracts**.

### Part A — Key client↔server requests
**Driver location ping** (driver → Location Service, every ~4s):
| Field | Type | Purpose |
|:--|:--|:--|
| `driver_id` | string | whose position |
| `lat`,`lng` | double | current position |
| `bearing` | int/null | heading (map arrow) |
| `ts` | epoch ms | drop stale/out-of-order pings |
> Fire-and-forget: server returns **202**, doesn't block; only the *latest* ping matters (LWW).

**Request a ride** (rider → Trip Service):
```json
{ "client_request_id":"uuid",   // idempotency — retries don't create two trips
  "pickup":{"lat":..,"lng":..}, "drop":{"lat":..,"lng":..},
  "ride_type":"economy", "quoted_fare":420, "surge":1.5 }
→ { "trip_id":"t_88", "state":"REQUESTED", "eta_pickup_min":4 }
```
**Ride offer** (Dispatch → driver): `{ "type":"OFFER","trip_id":"t_88","pickup":{..},"fare":420,"expires_in_s":15 }`
**Offer response** (driver → Dispatch): `{ "trip_id":"t_88","accept":true }`

### Part B — Inter-service payloads
- **Location Service → pub/sub `driver-loc:{driverId}`:** `{ "driver_id":"d_44","lat":..,"lng":..,"bearing":72,"ts":.. }` (fanned to the watcher's gateway node, §6).
- **Gateway → rider socket:** `{ "type":"LOCATION","trip_id":"t_88","lat":..,"lng":..,"eta_min":3 }`.
- **Matching → Dispatch:** `{ trip_id, candidates:[{driver_id, eta_s, rating}] }` (ranked).
- **Dispatch → Driver-state store (atomic claim):** `SETNX driver:{d}:state BUSY` / conditional UPDATE.
- **Trip Service → Payment (on complete):** `{ trip_id (=idempotency key), amount, currency, surge }`.

### Part C — DB schema per store (polyglot)
**① Driver location — in-memory geo index (Redis GEO), ephemeral, LWW:**
```
GEOADD city:{cityId}:drivers  lng lat  driver_id      -- upsert overwrites (the firehose sink)
driver:{id} → {lat,lng,bearing,ts}  TTL ~30s          -- stale = went offline
```
**② Driver state — fast store (atomic claim target):**
```
driver:{id}:state → FREE | BUSY | OFFLINE             -- SETNX/CAS on dispatch (§7)
```
**③ Trips — durable SQL, sharded by city_id (source of truth):**
```
trips( trip_id PK, city_id, rider_id, driver_id, state,
       pickup(json), drop(json), fare, surge,
       requested_at, matched_at, accepted_at, started_at, completed_at, version )
INDEX (rider_id, requested_at DESC)   -- ride history
INDEX (driver_id, state)              -- a driver's active trip
```
**④ Surge — cache, per geo cell:** `surge:{cityId}:{cell} → multiplier` (refreshed continuously).
**⑤ Payments/ledger — SQL (ACID):** `payments(id PK, trip_id, amount, status, idempotency_key UNIQUE)`.
**⑥ Users/drivers — SQL:** identity, vehicle, ratings. **Trip trail/history — warehouse:** sampled route for analytics.

> 💡 **The contract in one line:** *"Driver pings are fire-and-forget, upserted last-write-wins into an in-memory geo index and republished to a per-driver pub/sub channel the rider's gateway node subscribes to. A ride request carries a `client_request_id` (idempotency) and a locked `quoted_fare`; the trip lives in a city-sharded SQL row driven by a guarded state machine, and a driver is claimed via a single test-and-set on `driver:{id}:state`. Location is ephemeral/eventual; trips and payments are durable/strong. Every field exists for idempotency, ordering, geo, or exactly-once money."*

---

## 10. Surge / Dynamic Pricing

When demand outstrips supply in an area, raise price to balance (ration demand, attract drivers). Surge is computed **per geo cell, continuously**:
```
surge(cell) = clamp( openRequests(cell) / availableDrivers(cell), 1.0, CAP )
fare = base + perKm*distance + perMin*time, then × surge
```
(Verified: 5/50 → ×1.0; 40/20 → ×2.0; 100/10 → ×3.0 capped; zero supply → cap.)
- Computed by the **Pricing/Surge Service** from live supply (drivers in the geo index) and demand (recent requests) per cell.
- **Quote the price up front** and **lock** it for the request — never surprise the rider post-trip.
- **Smoothed/debounced** so it doesn't oscillate wildly.

> 💡 **Surge = local supply/demand ratio, quoted up front.** It's economic load-shedding: raise price to reduce demand and pull in supply where it's scarce — per cell, capped, smoothed, and locked at request time.

---

## 11. Payments (exactly-once)

On COMPLETED, charge the fare **exactly once**:
- Fare finalized from actual distance/time × the **locked** surge.
- Charge via a provider with an **idempotency key** (trip id) so retries/crashes never double-charge; **reconcile** against the provider.
- Handle failures (retry, alternate method, post-trip collection); **driver payout is a separate settlement** flow.

> 💡 **Exactly-once fare = idempotency key + reconciliation.** The charge carries the trip id as an idempotency key so a retry can't double-charge, reconciled against the provider — the same exactly-once-money pattern as any payments system.

---

## 12. Scaling Summary (geo-sharding)

- **Shard everything geo-local by `city_id`** — the location firehose, the geo index, dispatch, and surge. A Mumbai driver and rider only ever touch the Mumbai shard, so a New Year's-eve surge in one city can't affect another. **The single biggest lever.**
- **Keep the location firehose in memory (last-write-wins)** — never a durable DB per ping; sample a trail for analytics.
- **Fan live tracking over a WebSocket gateway fleet** (~13–63 nodes for ~6M conns) with a `user→node` registry + pub/sub backplane.
- **Matching = cell + neighbor lookups**, ranked by ETA; **dispatch = atomic claim**.
- **Trip state machine strongly consistent** in a durable store; **location eventually consistent** in memory.
- **Quadtree/S2 for hotspots** (airports/venues) so dense cells stay balanced.

---

## 13. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Driver offline mid-trip | location stops | grace + last-known; reassign if pre-pickup; support flow if in-trip |
| No drivers found | rider waiting | expand radius; retry; `NO_DRIVERS_FOUND` with notice |
| Double-booking race | one driver, two rides | **atomic claim** (SETNX/CAS) — one wins (verified) |
| Geo-index node loss | stale proximity | in-memory rebuilds from pings in seconds; replicate hot cells |
| Dispatch offer timeout | driver ignores | offer next candidate; cap total wait |
| Payment failure | unpaid trip | retry idempotently; alt method; post-trip collection |
| Gateway node crash | apps disconnect | reconnect-resume; registry TTL cleans stale routes |
| Surge oscillation | flapping prices | smooth/debounce the multiplier |
| GPS jitter / bad pings | wrong position | smoothing/validation; ignore implausible jumps |

> 💡 **The in-memory geo index self-heals:** losing a node isn't catastrophic — drivers ping every few seconds, so positions rebuild almost immediately; replicate hot cells for continuity. The *trip* state, which you can't lose, is in a durable store.

---

## 14. Senior Follow-Up Questions (with Answers)

**Q1. What's the real scaling challenge?** Not ride QPS (~833/s peak). It's the **location firehose (~1.25M writes/s, ~1,500× the ride load)** and **~6M live connections**. Absorb the firehose with in-memory last-write-wins upserts; fan positions out over a WebSocket gateway; keep it all off the durable trip DB.

**Q2. How does a driver's live position reach the rider watching that trip?** Each ping upserts the driver's latest position in the geo index and is published to a per-driver pub/sub channel. The rider's gateway node subscribes to that channel (resolved via `trip → driver`), so every ping is delivered to the node holding the rider's socket, which writes it down with an ETA. (§6)

**Q3. How do you match a ride to exactly one driver?** Candidate geo-search (cell + neighbors) → ETA rank → offer with timeout → **atomic claim** (`SETNX`/CAS marking the driver BUSY only if FREE). The test-and-set guarantees exactly one winner; a non-atomic check-then-set double-books (verified). (§7)

**Q4. Why don't you persist every location ping?** ~1.25M writes/s of throwaway values would crush the OLTP DB, and you only need the *latest* position. So upsert last-write-wins into an in-memory geo index and sample a trail for analytics — never a DB row per ping.

**Q5. How does "nearest drivers" work, and what's the boundary problem?** A spatial index (geohash/Redis GEO/quadtree/S2) buckets the map so proximity is a cell lookup. A driver close by can be just across a cell edge, so you search the rider's cell **plus its 8 neighbors**, union, and rank by **road ETA** (not straight-line). (§8)

**Q6. How do you price with surge?** Per geo cell: `clamp(openRequests/availableDrivers, 1, CAP)`, smoothed, quoted **up front** and locked for the request. Economic load-shedding, computed locally. (§10)

**Q7. How do you charge exactly once?** Idempotency key = trip id on the charge so retries/crashes can't double-charge; reconcile against the provider. Payouts are a separate settlement. (§11)

**Q8. How do you shard?** By **city/region** — the domain is geo-local, so each city is an independent shard for the firehose, index, dispatch, and surge. One city's spike can't affect another.

**Q9. What's the consistency model?** Strong for the trip state machine and payments (guarded transitions, ACID, idempotency); eventual for location and surge (a few-seconds-stale position/price is fine and making them strong would be infeasible at 1.25M/s).

**Q10. Why separate the Location Service from the Trip Service?** Location is a huge ephemeral firehose (in-memory, LWW); trips are durable, auditable, and few. Mixing them puts a firehose through your system-of-record. Different volume/durability/consistency → different systems.

**Q11. Offer-one vs broadcast-many dispatch?** Offer-one (sequential, fair, slightly slower) vs broadcast-to-many (first-accept-wins, faster fill, more claim contention). Both end in the same atomic claim so only one ride wins the driver.

**Q12. How does the gateway route a driver's ping to the right rider?** A `user→node` registry says which gateway node holds the rider's socket; a pub/sub backplane carries the driver's position to that node (the socket is pinned to its process), which does the final write from its local socket table. (§6)

---

## 15. Quick Glossary
- **Location firehose** — the high-rate stream of driver GPS pings (~1.25M/s); absorbed in memory, never persisted per-ping.
- **Last-write-wins (LWW) upsert** — storing only a driver's current position, overwriting the previous.
- **Geospatial index** — spatial structure (geohash/quadtree/S2/Redis GEO) bucketing the map so "nearest" is a cell lookup.
- **Boundary problem** — a nearby driver in an adjacent cell; fixed by searching cell + 8 neighbors.
- **Haversine vs ETA** — straight-line distance (cheap filter) vs road ETA (accurate rank).
- **Dispatch / atomic claim** — assigning a ride to one driver via a single test-and-set (`SETNX`/CAS) marking BUSY.
- **Offer-one / broadcast-many** — sequential fair dispatch vs first-accept-wins fast-fill.
- **Trip state machine** — the guarded, durable lifecycle REQUESTED→…→PAID (source of truth).
- **Pub/sub backplane** — internal bus routing a driver's pings to the gateway node holding the rider's socket.
- **WebSocket gateway** — the fleet holding live driver/rider connections and pushing updates.
- **Surge** — per-cell supply/demand multiplier, quoted up front and locked.
- **Exactly-once fare** — charge with an idempotency key (trip id) + reconciliation.
- **Geo-sharding (by city)** — partitioning by city because the domain is geo-local; independent markets.

---

*Reference document. Contributions and corrections welcome.*
