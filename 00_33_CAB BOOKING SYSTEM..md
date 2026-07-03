# Cab Booking System (Uber / Ola) — System Design (HLD, Detailed)

> A worked design of a ride-hailing platform: match riders to nearby drivers in real time, track live locations, run the trip lifecycle, price dynamically (surge), and handle payments — at city/global scale. The standout piece is **geospatial proximity matching** (find the nearest available drivers fast), sitting on top of **high-write location ingestion**, **concurrent dispatch** (two riders must not grab one driver), a **trip state machine**, and **real-time push** to phones. This doc builds each, with the mechanics verified.

> 💡 **The whole system in one sentence:** drivers continuously stream their location into a **geospatial index** (geohash / quadtree / S2 / Redis GEO); when a rider requests, you query that index for **nearby drivers (searching neighbor cells, not just one)**, rank by real distance/ETA, **atomically dispatch** to one so no driver is double-booked, then drive the trip through a **state machine** while pushing updates over a **WebSocket gateway** — with surge pricing derived from local supply/demand and exactly-once payment at the end.

---

## Table of Contents
1. [Problem & requirements](#1-problem--requirements)
2. [Why it's hard](#2-why-its-hard)
3. [The core services](#3-the-core-services)
4. [Location ingestion (high write)](#4-location-ingestion-high-write)
5. [The geospatial index (the heart)](#5-the-geospatial-index-the-heart)
6. [Proximity search & the boundary problem](#6-proximity-search--the-boundary-problem)
7. [Matching & dispatch (concurrency)](#7-matching--dispatch-concurrency)
8. [The trip lifecycle (state machine)](#8-the-trip-lifecycle-state-machine)
9. [Real-time communication (gateway)](#9-real-time-communication-gateway)
10. [Surge / dynamic pricing](#10-surge--dynamic-pricing)
11. [ETA & routing](#11-eta--routing)
12. [Payments](#12-payments)
13. [The architecture](#13-the-architecture)
14. [Data model](#14-data-model)
15. [End-to-end flow](#15-end-to-end-flow)
16. [Scale estimation](#16-scale-estimation)
17. [Failure modes](#17-failure-modes)
18. [Edge cases](#18-edge-cases)
19. [Trade-offs & talking points](#19-trade-offs--talking-points)
20. [How to present this in the interview](#20-how-to-present-this-in-the-interview)
21. [Common mistakes to avoid](#21-common-mistakes-to-avoid)
22. [TL;DR](#22-tldr)

---

## 1. Problem & requirements

Design a ride-hailing system where riders request rides and get matched to nearby drivers.

### Functional
- Rider requests a ride (pickup, drop, ride type).
- System finds **nearby available drivers**, matches one, driver accepts.
- Both parties see **live location / ETA**; trip runs through pickup → drop.
- **Fare** computed (distance/time + surge); **payment** on completion.
- Cancellations, ratings, ride history.

### Non-functional
- **Low-latency matching** — a rider expects a driver within seconds.
- **High write throughput** — millions of drivers pinging location every few seconds.
- **Real-time** — bidirectional push (driver moving, ride accepted) — not polling.
- **Consistent dispatch** — one driver assigned to exactly one ride at a time (no double-booking).
- **Geo-scalable** — sharded by region/city; a New York surge shouldn't affect Mumbai.
- **Accurate payments** — exactly-once charging.
- **Available** — degrade gracefully; a matching hiccup shouldn't strand riders.

> 💡 **Frame the two workloads:** "There are two very different loads here — a *firehose of driver location writes* (millions/sec, mostly discardable) and *low-latency proximity reads* at request time. The design optimizes the geospatial index for both: cheap frequent updates and fast nearest-neighbor queries." Naming that split shapes the whole design.

---

## 2. Why it's hard

| Hard thing | Why |
|:-----------|:----|
| **Proximity search at scale** | "find drivers within 3km of this point" over millions of moving drivers, in ms |
| **High-write location stream** | every driver pings every ~4s → millions of writes/sec |
| **Concurrent dispatch** | two riders near one driver → must assign to exactly one |
| **Real-time bidirectional** | live tracking both ways → persistent connections, not polling |
| **Moving targets** | both driver and rider move; the index is constantly stale |
| **Geo-partitioning** | shard by area; handle city boundaries and hotspots |
| **Surge** | price by *local* supply/demand, updated continuously |
| **Correctness of money** | exactly-once fare charge |

> 💡 **The signature challenge is geospatial:** "The core is 'nearest available drivers to a point,' which a normal B-tree index can't do — you need a spatial index (geohash/quadtree/S2) that buckets the map into cells so proximity is a cell lookup, not a full scan." Lead with that; it's what the interview is really testing.

---

## 3. The core services

Split by concern (each scales independently):

| Service | Responsibility |
|:--------|:---------------|
| **Location Service** | ingest driver location pings; update the geospatial index |
| **Geospatial / Matching Service** | proximity search; find & rank nearby drivers |
| **Dispatch Service** | offer ride to driver(s); atomic assignment; handle accept/decline |
| **Trip Service** | trip lifecycle state machine; source of truth for a ride |
| **Pricing/Surge Service** | fare estimate; surge multiplier per area |
| **ETA/Routing Service** | distance/time/route (map data) |
| **Payment Service** | exactly-once fare charging |
| **Gateway (WebSocket)** | real-time push to rider & driver apps |
| **Notification Service** | push/SMS for out-of-app events |

> 💡 **Separate location from trip.** "I keep the high-churn location firehose in its own service + in-memory geo index, separate from the durable Trip service. Location is ephemeral and huge; trips are durable and comparatively few. Mixing them would put a firehose through your system-of-record." 

---

## 4. Location ingestion (high write)

Every active driver sends a location ping every ~3–5s. At millions of active drivers that's **millions of writes/sec** — and each write is *ephemeral* (only the latest matters).

- Pings arrive over the driver's **WebSocket** to the Gateway → forwarded to the **Location Service**.
- The Location Service **updates the driver's position in the geospatial index** (in-memory / Redis) — an **upsert of the latest**, not an append. Old positions are worthless.
- **Don't persist every ping to a durable DB** — it's a firehose of throwaway data. Keep current positions in memory (Redis GEO), optionally sample a trail for the active trip / analytics.

> 💡 **Location is "last-write-wins, in memory."** "Only the latest position matters, so I upsert into an in-memory geo index rather than writing every ping to disk — millions of writes/sec against a database would be pointless and ruinous. Durable storage is for trips and sampled trails, not the location firehose." 

---

## 5. The geospatial index (the heart)

To answer "nearest drivers to (lat,lng)" fast, you can't scan all drivers — you need a **spatial index** that buckets the map so proximity is a **cell lookup**:

| Technique | Idea |
|:----------|:-----|
| **Geohash** | encode (lat,lng) into a string; shared prefix = spatial proximity; bucket drivers by prefix |
| **Quadtree** | recursively subdivide space into 4 quadrants; dense areas subdivide more |
| **S2 (Google)** | map sphere → cells with hierarchical ids (handles Earth's curvature well) |
| **Redis GEO** | built-in `GEOADD` / `GEOSEARCH` (geohash under the hood) — often the pragmatic choice |

Drivers are stored **keyed by their cell**; a proximity query looks up the rider's cell (and neighbors — §6), yielding a small candidate set instead of everyone. (Verified with a grid stand-in: bucketing by cell then searching returns only the local candidates.)

> 💡 **Name a concrete choice + why:** "I'd use Redis GEO (or geohash) — `GEOADD` to upsert a driver's position, `GEOSEARCH` to find drivers within a radius. Geohash gives O(1)-ish proximity via prefix/cell lookup, handles the high update rate in memory, and shards naturally by cell/region." Concrete tech + the reason beats hand-waving "a spatial index."

---

## 6. Proximity search & the boundary problem

The subtle bug: a driver can be **very close to the rider but in an adjacent cell**. If you search only the rider's cell, you miss them. **Fix: search the rider's cell plus its neighboring cells** (the 3×3 block, or a ring sized to the search radius), then rank.

```
query(rider):
   cells = riderCell + 8 neighbors           # cover boundary-straddling drivers
   candidates = union(drivers in those cells)
   rank candidates by haversine distance (or ETA); keep within radius; return top-K
```

(Verified: a driver 78m from the rider sat in an *adjacent* cell — searching only the rider's cell missed them; searching neighbors found them, and haversine ranked the two nearby drivers nearest-first at 49m and 78m.)

Then **rank by true distance** (haversine) — or better, by **road ETA** (a straight-line-nearest driver may be across a river). Return the top few candidates to dispatch.

> 💡 **The boundary problem is the classic follow-up.** "Cells create edges — a driver 50m away can be just across a cell boundary — so I always search the neighboring cells too, not just the rider's cell, then rank the union by distance/ETA. Straight-line distance is a first filter; road ETA is the real ranking." Raising this unprompted signals you've actually implemented proximity search.

---

## 7. Matching & dispatch (concurrency)

Finding candidates is step one; **assigning** one is where concurrency bites: two nearby riders might target the same driver simultaneously → **double-booking**.

Dispatch flow:
1. Matching returns ranked candidate drivers.
2. **Offer** the ride to the best candidate (push a request; driver has ~N seconds to accept).
3. On accept, **atomically claim** the driver — mark them BUSY in one indivisible step (Redis `SETNX`/CAS, or a per-driver lock).
4. If they decline/timeout, **offer the next** candidate.

The atomic claim is essential: a naive "check free then mark busy" lets two requests both read "free" and both assign (verified: non-atomic double-books; atomic test-and-set lets exactly one win). Same read-modify-write atomicity as your rate-limiter and payroll idempotency.

> 💡 **The dispatch race + fix:** "Two riders can target one driver, so the assignment must be atomic — a compare-and-set / `SETNX` that marks the driver BUSY, so exactly one request wins and the other falls through to the next candidate. Without atomicity you double-book." Also mention *offer-and-accept* (sequential offers with timeouts) vs *broadcast-to-many* (first-accept-wins) as the two dispatch styles.

---

## 8. The trip lifecycle (state machine)

A trip is the durable **source of truth**, driven by an explicit state machine:

```
REQUESTED ─► MATCHED ─► ACCEPTED ─► ARRIVED ─► IN_PROGRESS ─► COMPLETED ─► PAID
     │           │          │
     └───────────┴──────────┴────────► CANCELLED   (by rider/driver, with policy)
                                        NO_DRIVERS_FOUND
```

| State | Meaning |
|:------|:--------|
| REQUESTED | rider asked; matching in progress |
| MATCHED → ACCEPTED | driver offered → accepted |
| ARRIVED | driver at pickup |
| IN_PROGRESS | trip underway |
| COMPLETED → PAID | dropped off → fare charged |
| CANCELLED / NO_DRIVERS_FOUND | terminal branches |

Transitions are guarded and persisted, so the Trip Service can recover and so each event (arrive, start, complete) is validated. Cancellation policy (free window, cancel fee) hangs off the transitions.

> 💡 **The trip is the system of record; location is not.** "Everything money- and dispute-relevant lives on the trip state machine — matched/accepted/started/completed timestamps, fare, who cancelled. Location pings are ephemeral; the trip is durable and auditable." (Same state-machine discipline as your OMS.)

---

## 9. Real-time communication (gateway)

Both apps need **bidirectional real-time push**: driver location → rider's map, ride offers → driver, status changes → both. This is a **long-lived WebSocket to a Gateway** (your gateway doc): drivers and riders each hold one connection; the backend routes events to the right connection via a **registry (user→node) + pub/sub backplane**.

- Driver app: streams location up; receives ride offers down.
- Rider app: receives driver-location/ETA and status down.
- Connections are stateful & sticky; heartbeats + reconnect+resume handle mobile churn.

> 💡 **Reuse the gateway pattern:** "Live tracking and instant offers need push, not polling — so both apps hold a WebSocket to a stateful Gateway fleet, and the backend routes to the right socket via a user→node registry over a pub/sub backplane. Same connection-fleet design as any real-time system." Connecting it to the gateway topic shows systems thinking.

---

## 10. Surge / dynamic pricing

When demand outstrips supply in an area, raise price to balance (ration demand, attract drivers). Surge is computed **per area (geo cell), continuously**:

```
surge(area) = clamp( openRequests(area) / availableDrivers(area), 1.0, CAP )
fare = base + perKm*distance + perMin*time, then × surge
```

(Verified: demand 5 / supply 50 → ×1.0; 40/20 → ×2.0; 100/10 → ×3.0 capped; zero supply → cap.)

- Computed by the **Pricing/Surge Service** from live supply (drivers in the geo index) and demand (recent requests) per cell.
- **Quote the price up front** and lock it for the request (rider agrees before matching); don't surprise them post-trip.
- Smoothed/debounced so it doesn't oscillate wildly.

> 💡 **Surge = local supply/demand ratio, quoted up front.** "I compute a multiplier per geo cell from open-requests over available-drivers, capped and smoothed, and lock the quoted fare when the rider requests. It's economic load-shedding — raise price to reduce demand and pull in supply where it's scarce." 

---

## 11. ETA & routing

- **ETA to pickup** ranks candidate drivers (road ETA, not straight-line) and shows the rider a wait time.
- **Trip route & fare estimate** use a **map/routing service** (road network graph, traffic-aware). Usually a dedicated service or a third party (Google Maps / OSRM / internal).
- Ranking by ETA rather than haversine matters in real geography (rivers, one-ways) — haversine is a cheap first filter; ETA is the real cost.

> 💡 **Haversine filters, ETA ranks:** "Straight-line distance narrows candidates cheaply, but I rank by *road ETA* from a routing service — the closest driver as the crow flies may be a 15-minute drive away across a river." 

---

## 12. Payments

On COMPLETED, charge the fare — **exactly once** (the payroll discipline):
- Fare finalized from actual distance/time × the locked surge.
- Charge via a payment provider with an **idempotency key** (trip id) so retries/crashes don't double-charge; **reconcile** against the provider.
- Handle failures (retry, alternate method, post-trip collection); driver payout is a separate settlement flow.

> 💡 **Exactly-once fare = idempotency key + reconciliation:** "The charge carries the trip id as an idempotency key so a retry can't double-charge, and I reconcile against the payment provider — same exactly-once-money pattern as payroll. Payouts to drivers are a separate settlement." 

---

## 13. The architecture

```
   rider app ── WebSocket ──┐            ┌── WebSocket ── driver app
                            ▼            ▼   (location pings every ~4s)
                     ┌───────────────────────────┐
                     │   Gateway (WebSocket fleet) │  registry(user→node)+backplane
                     └───┬───────────────┬────────┘
          ride request   │               │ location ping
                         ▼               ▼
              ┌────────────────┐   ┌────────────────────┐   upsert latest
              │ Trip Service    │   │ Location Service    │──────────────┐
              │ (state machine, │   │ (ingest pings)      │              ▼
              │  durable DB)    │   └────────────────────┘     ┌────────────────────┐
              └───┬────────────┘                                │ Geospatial Index    │
                  │ match request                               │ (Redis GEO / geohash│
                  ▼                                             │  / quadtree; in-mem)│
          ┌────────────────────┐   query nearby (cell+neighbors)└─────────┬──────────┘
          │ Matching Service    │◄──────────────────────────────────────── ┘
          │ (proximity+rank ETA)│
          └───┬────────────────┘
              │ candidates
              ▼
          ┌────────────────────┐  atomic claim (SETNX/CAS)   ┌──────────────────┐
          │ Dispatch Service    │───────────────────────────►│ Driver state      │
          │ (offer/accept)      │                            │ (FREE/BUSY)       │
          └───┬────────────────┘                            └──────────────────┘
              │ on complete
              ▼
   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
   │ Pricing/Surge   │  │ ETA/Routing     │  │ Payment (idem   │
   │ (per-cell x)    │  │ (map graph)     │  │ key + reconcile)│
   └────────────────┘  └────────────────┘  └────────────────┘
```

---

## 14. Data model

| Entity | Store | Notes |
|:-------|:------|:------|
| **Driver location** | in-memory geo index (Redis GEO) | latest only; upsert; ephemeral; keyed by cell |
| **Driver state** | fast store | FREE / BUSY / OFFLINE; atomic claim on dispatch |
| **Trip** | durable DB | state machine, rider/driver, timestamps, fare, surge — **source of truth** |
| **Surge** | cache, per geo cell | multiplier, refreshed continuously |
| **User/Driver profile** | DB | identity, vehicle, ratings |
| **Trip history / trail** | DB / warehouse | completed trips, sampled route (analytics) |

Location is ephemeral & in-memory; trips are durable & auditable — the key modeling split.

---

## 15. End-to-end flow

1. **Drivers stream location** → Location Service → geo index (upsert latest).
2. **Rider requests** (pickup/drop) → Trip Service creates trip (REQUESTED) → gets a **surge-adjusted quote** (locked).
3. **Match** → Matching Service queries the geo index (rider cell + neighbors), ranks by ETA → candidate drivers.
4. **Dispatch** → offer to best candidate; on accept, **atomically mark BUSY** → trip MATCHED→ACCEPTED (else next candidate).
5. **Pickup** → live driver location pushed to rider via Gateway; ARRIVED → IN_PROGRESS.
6. **Trip** → route/ETA updates; on drop → COMPLETED, finalize fare.
7. **Pay** → charge with idempotency key, reconcile → PAID; driver freed (FREE); ratings.

---

## 16. Scale estimation

| Quantity | Value | Note |
|:---------|:------|:-----|
| Active drivers | ~millions | each pings every ~4s |
| Location writes | **~millions/sec** | ephemeral upserts to in-memory index |
| Ride requests | far fewer (thousands/sec peak) | the low-latency read path |
| Geo index | in-memory, sharded by region/city | reads = cell+neighbor lookups |
| Trips | durable; millions/day | source of record |

The **write load is the location firehose** (handled by in-memory upserts, not disk); the **read load is proximity queries** (handled by the cell-indexed geo store). Shard both **by region/city** so hotspots and load stay local.

> 💡 **Shard by geography:** "Everything partitions by region/city — a driver in Mumbai and a rider in Mumbai only ever touch the Mumbai shard. That localizes the location firehose, the geo index, and surge, and means New Year's-eve surge in one city doesn't affect another." Geo-sharding is the scaling answer here.

---

## 17. Failure modes

| Failure | Effect | Handling |
|:--------|:-------|:---------|
| Driver goes offline mid-trip | location stops | grace + last-known; reassign if pre-pickup; support flow if in-trip |
| No drivers found | rider waiting | expand radius; retry; NO_DRIVERS_FOUND with notice |
| Double-booking race | one driver, two rides | atomic claim (verified) |
| Location index node loss | stale proximity | in-memory rebuilds from pings in seconds; replicate hot cells |
| Dispatch offer timeout | driver ignores | offer next candidate; cap total wait |
| Payment failure | unpaid trip | retry idempotently; alt method; post-trip collection |
| Gateway node crash | apps disconnect | reconnect+resume (gateway doc); registry TTL cleans stale routes |
| Surge oscillation | flapping prices | smooth/debounce the multiplier |

> 💡 **In-memory geo index self-heals:** "Losing a geo-index node isn't catastrophic — drivers ping every few seconds, so positions rebuild almost immediately; I replicate hot cells for continuity. The *trip* state, which I can't lose, is in a durable store." 

---

## 18. Edge cases

| Case | Handling |
|:-----|:---------|
| Driver just across a cell boundary | search neighbor cells (verified) |
| Nearest by line ≠ nearest by road | rank by ETA, not haversine |
| Rider cancels after match | cancellation policy / fee; free the driver |
| Driver cancels after accept | reassign; penalize per policy |
| Surge with zero supply | cap the multiplier (verified) |
| Simultaneous requests, sparse drivers | atomic dispatch; queue/expand radius |
| GPS jitter / bad pings | smoothing/validation; ignore implausible jumps |
| Airport/venue hotspots | finer cells (quadtree subdivides dense areas); zone queues |
| Multi-city / boundaries | geo-shard; handle cross-boundary queries at edges |

> 💡 **Quadtree shines in hotspots:** "Uniform grid cells are too coarse at an airport (thousands of drivers in one cell) and too fine in the countryside. A quadtree/S2 subdivides dense areas more, keeping candidate sets balanced — a good thing to mention for hotspot handling." 

---

## 19. Trade-offs & talking points

- **Consistency vs availability** — dispatch/payment strongly consistent (no double-book/charge); location eventually consistent (stale-by-seconds is fine).
- **In-memory location vs durable** — ephemeral firehose stays in memory; durable only for trips/trails.
- **Grid vs quadtree/S2** — uniform simplicity vs adaptive density (hotspots).
- **Offer-one vs broadcast-many dispatch** — controlled/fair vs fast-fill (first-accept-wins).
- **Haversine vs ETA ranking** — cheap filter vs accurate cost.
- **Geo-sharding** — locality & blast-radius vs cross-boundary complexity.

---

## 20. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Clarify + two workloads | location firehose (write) vs proximity match (read) |
| Services | location / geo-match / dispatch / trip / pricing / gateway |
| Geospatial index | geohash/quadtree/S2/Redis GEO; cell bucketing |
| Proximity + boundary | search cell + neighbors; rank by ETA |
| Dispatch concurrency | offer/accept; **atomic claim** (no double-book) |
| Trip state machine + gateway | durable lifecycle; WebSocket push |
| Surge + payments + sharding | per-cell multiplier; exactly-once fare; geo-shard |

### What to say
- *"Two workloads: a location firehose I keep in an in-memory geo index (last-write-wins), and low-latency proximity reads at request time."*
- *"Nearby-driver search is a spatial index — geohash/Redis GEO — and I search the rider's cell plus neighbors, then rank by road ETA."*
- *"Dispatch must be atomic — a SETNX/CAS marks the driver BUSY so two riders can't grab one driver."*
- *"The trip is a durable state machine and the source of truth; location is ephemeral. Both apps push over a WebSocket gateway."*
- *"Surge is a per-cell supply/demand multiplier quoted up front; the fare charge is exactly-once via an idempotency key. Everything shards by city."*

### Order
clarify → two workloads → geo index → proximity+boundary → dispatch atomicity → trip FSM + gateway → surge/payments/sharding → failures.

---

## 21. Common mistakes to avoid

- ❌ **Scanning all drivers for proximity** — use a spatial index (geohash/quadtree/S2).
- ❌ **Searching only the rider's cell** — misses drivers just across a boundary; search neighbors.
- ❌ **Persisting every location ping to a DB** — it's an ephemeral firehose; keep latest in memory.
- ❌ **Non-atomic dispatch** — double-books a driver; use SETNX/CAS/lock.
- ❌ **Ranking by straight-line only** — rank by road ETA where it matters.
- ❌ **No trip state machine** — you lose the durable, auditable source of truth.
- ❌ **Polling for location** — use WebSocket push (gateway).
- ❌ **Uniform grid everywhere** — hotspots overflow one cell; quadtree/S2 adapts.
- ❌ **Surge computed globally** — it's per-area; and quote it up front.
- ❌ **Double-charging fares** — idempotency key + reconciliation.
- ❌ **Not geo-sharding** — one city's spike shouldn't affect another.

---

## 22. TL;DR

### The design
```
Drivers stream location → in-memory GEOSPATIAL INDEX (geohash / quadtree / S2 / Redis GEO), last-write-wins.
Rider requests → query index for NEARBY drivers (rider cell + NEIGHBORS) → rank by road ETA
             → DISPATCH via atomic claim (SETNX/CAS) so one driver ≠ two rides
             → TRIP STATE MACHINE (durable source of truth) → push updates over WebSocket GATEWAY
             → SURGE = per-cell demand/supply (quoted up front) → EXACTLY-ONCE fare (idem key + reconcile).
Everything GEO-SHARDED by city.
```

### The mechanics (verified)
```
Proximity  : bucket by cell; search cell + 8 neighbors (boundary problem); haversine/ETA rank
Dispatch   : atomic test-and-set marks driver BUSY → exactly one rider wins
Surge      : clamp(openRequests/availableDrivers, 1.0, CAP) per area
```

### The four things that score points
1. **Geospatial index + neighbor search** — the core; search cell *and* neighbors, rank by ETA.
2. **Location = ephemeral in-memory firehose; trip = durable state machine** (the workload split).
3. **Atomic dispatch** — no double-booking (read-modify-write must be atomic).
4. **Real-time via WebSocket gateway; surge per-cell quoted up front; exactly-once fare; geo-sharded.**

> **One-line philosophy:** *A cab booking system is a geospatial matching engine over a location firehose — keep drivers' latest positions in an in-memory spatial index so "nearest drivers" is a cell lookup (remembering to search neighbor cells and rank by road ETA), atomically dispatch so one driver is never double-booked, drive each ride through a durable trip state machine with real-time push over a WebSocket gateway, and price by local supply-and-demand — all sharded by city so each market scales and fails independently.*
