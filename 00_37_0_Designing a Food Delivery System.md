# Designing a Food Delivery System (Swiggy / DoorDash): A Senior Interview Guide

> A practical, interview-focused reference for designing a food-delivery platform — a **geospatial, real-time, multi-party workflow**. The distinctive challenge isn't order volume (modest) but **tracking a live location firehose** and **matching each order to exactly one rider under contention**. This guide builds the architecture up piece by piece, traces the full life of an order, goes deep on the two genuinely hard mechanisms (how a rider's GPS ping physically reaches the customer's screen, and how an order is physically claimed by one rider), and nails down the data contracts — with capacity math and a senior follow-up bank.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of an order](#44-the-end-to-end-life-of-an-order)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Order State Machine](#5-the-order-state-machine)
6. [Hard Problem 1: How a Rider's Location Reaches the Customer's Screen](#6-hard-problem-1-how-a-riders-location-reaches-the-customers-screen)
7. [Hard Problem 2: How an Order Is Matched to Exactly One Rider](#7-hard-problem-2-how-an-order-is-matched-to-exactly-one-rider)
8. [Geospatial Search: How "Restaurants Near Me" Actually Works](#8-geospatial-search-how-restaurants-near-me-actually-works)
9. [Data Contracts: Request Fields, Payloads & DB Schemas](#9-data-contracts-request-fields-payloads--db-schemas)
10. [Payments & the Order Saga](#10-payments--the-order-saga)
11. [Notifications](#11-notifications)
12. [Scaling Summary](#12-scaling-summary)
13. [Senior Follow-Up Questions (with Answers)](#13-senior-follow-up-questions-with-answers)
14. [Quick Glossary](#14-quick-glossary)

---

## 1. How to Approach This in an Interview

The thing that makes food delivery distinctive — and where you should spend your time — is that it's a **real-time geospatial system**: the platform must continuously know **where every rider is** and push that to **the right watching customer**, and it must **match a fresh order to one nearby rider** out of thousands, under contention, in seconds. Those two mechanisms, not order throughput, are the heart of the design.

A good structure:
1. **Clarify requirements** — discovery, cart/order, payment, dispatch, live tracking, ratings.
2. **Estimate scale** — and *notice the surprise*: order QPS is small (~140/s peak), but the **rider-location firehose (~83K pings/s)** and **~667K live tracking connections** are the real load.
3. **Establish the order state machine** — one order coordinates customer, restaurant, and rider through a guarded lifecycle.
4. **Go deep on the two hard mechanisms** — real-time tracking fan-out and rider matching (dispatch).
5. **Cover geospatial search, data contracts, payments (saga), scaling, failure modes.**

> 💡 **Senior signal:** say up front — *"The order path itself is easy CRUD. The genuinely hard parts are (1) ingesting and fanning out a live location firehose, (2) matching each order to exactly one rider under concurrency, and (3) geospatial 'near me' queries. I'd spend my time there."* That immediately shows you know where the difficulty lives.

---

## 2. Requirements

### Functional
- **Customer:** discover restaurants near a location; browse menu; cart; place & pay; **track the order live** (rider position + ETA); rate.
- **Restaurant:** receive orders; accept/reject; mark preparing/ready; manage menu availability.
- **Rider:** go online/offline; receive assignment **offers**; accept/reject; **report live location**; mark picked-up/delivered.
- **Platform:** **match** orders to riders (dispatch); price (fees, taxes, surge); notify all parties.

### Non-Functional
- **Low latency** on browse (feels instant) and on live tracking (near-real-time).
- **High availability** — ordering must survive meal-time peaks; degrade gracefully.
- **Correctness where it matters** — order state and payment must be exact (no double-charge, no lost order, no two riders for one order); location can be eventually consistent.
- **Peaky load** — traffic concentrates hard at lunch/dinner.

> Note the split personality: **discovery/browse** is read-heavy and cacheable (a normal web path); **ordering + dispatch + tracking** is the real-time, correctness-and-geo-sensitive core. The design effort goes into the latter.

---

## 3. Capacity Estimation

**Assumptions:** 30M DAU; 5M orders/day; ~15 browse/menu views per order; ~40-min delivery; rider pings every 4 s.

```
Order QPS      = 5M/day → ~58/s avg, ~140/s at meal-time peak     (MODEST throughput)
Browse/search  = 75M/day → ~2,100/s peak                          (read-heavy → cache/CDN)
Active riders (peak)     ≈ 333,000 concurrent
LOCATION FIREHOSE        ≈ 83,000 pings/sec at peak               ← the real write load
Live WebSocket conns     ≈ 667,000 at peak (customers + riders)   ← needs a fan-out tier
Orders storage           = 10.2 GB/day → 3.74 TB/yr (1.8B orders/yr)
Restaurants ~500k; ~50M menu items                               (small, mostly static → cache)
```

**Takeaways that shape the design:**
- **Order throughput is trivial (~140/s)** — this is *not* a scale-out-writes problem for orders.
- **The location firehose (~83K/s) is ~600× the order-write load** — so the architecture centers on ingesting/serving live positions cheaply (Redis + Kafka), and **never persisting the firehose to the OLTP DB.**
- **~667K live connections** → a dedicated WebSocket/fan-out tier.
- **Orders and riders are geo-local** — a Bangalore order only ever involves Bangalore restaurants/riders — so you **shard by city**, and each city becomes an almost-independent mini-system.

> 💡 The estimation flips the naive assumption ("food delivery = lots of orders"). No — orders are ~140/s. The real scale is **real-time geospatial data**. Saying that reframes the whole problem.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

A food-delivery system is fundamentally three loops running at once. **Loop 1 (ordering)** is a mostly-ordinary transactional workflow: a customer browses (cached reads), places an order, pays, and the order advances through a **state machine** as the restaurant accepts/prepares. **Loop 2 (dispatch)** is a real-time matching problem: when an order is nearly ready, the platform must find nearby available riders, offer the job, and let **exactly one** rider claim it. **Loop 3 (tracking)** is a real-time streaming problem: the assigned rider's phone emits a GPS ping every few seconds, and the platform must push those positions to the **one customer** watching that order — hundreds of thousands of times per second across the fleet. The architecture exists to keep these three loops from interfering: the **Order service** owns the state machine and is strongly consistent; the **Dispatch service** does geospatial matching with an atomic claim; the **Location service + WebSocket tier** absorb the firehose and fan it out; and everything geo-local is **sharded by city** so one city's dinner rush never touches another's.

### 4.2 The diagram

```
 Customer app ─┐                                  ┌─ Restaurant app (accept / ready)
 Rider app  ───┼─► [API Gateway / LB] ─► services │─ Rider app (online / accept offer)
 (websockets)  │        │                         └──────────────────────────────┐
               │        ├─► [Discovery/Search] ─► [Restaurant DB] + [ES] + [Geo index]
               │        ├─► [Menu] ─► [store + Redis + CDN]
               │        ├─► [Cart] ─► [Redis]
               │        ├─► [Order svc] ─► [Order DB (SQL, city-sharded)] ─► [Kafka: order-events]
               │        ├─► [Payment svc] ─► [PSP + ledger]   (idempotent, saga)
               │        ├─► [Pricing]
               │        └─► [Dispatch/Matching] ◄── nearby riders ── [Location svc]
               │
   (live) ─────┴─► [WebSocket / Tracking tier] ◄── [Kafka: location + order-status]
                                                        ▲
 Rider pings ───────────────────────────────────────────┤ [Location svc: Redis GEO / in-mem quadtree]
                                                        └─ latest position only; sampled to lake
```

The key visual idea: an order comes **in** through the gateway to the **Order service** (persist + advance the state machine); **Dispatch** matches it to a rider using **live rider positions** from the **Location service**; and the rider's ongoing GPS pings flow **up** into the Location service and **out** through the **WebSocket tier** to the watching customer. Browse/menu hang off to the side (cached) so they never clog the real-time path.

### 4.3 Each component, in detail

**① Client apps (customer, restaurant, rider).** Three different apps on the same backend. The **customer** app browses over normal HTTPS and opens a **WebSocket** only while watching a live order. The **rider** app holds a WebSocket while online, emits a GPS **ping every ~4 s**, and receives assignment **offers**. The **restaurant** app receives new-order events and sends accept/ready.

**② API Gateway.** Auth, routing, rate limiting. Dumb on purpose — no business logic — so it scales and restarts freely.

**③ Discovery / Search service.** Answers "restaurants near me, filtered/sorted." Backed by a **geohash/S2 geo-index** for "near me" and **Elasticsearch** for text/cuisine search. Almost entirely **read-heavy and cacheable** (restaurants barely change), so it's fronted by Redis + CDN.

**④ Order service (the brain of Loop 1).** Owns the **order state machine** and is the **strongly-consistent source of truth** for orders. Every transition (PLACED → ACCEPTED → … → DELIVERED) is atomic, guarded by the current state, and emits a **Kafka order-event** that drives notifications and tracking. Order data lives in a SQL store **sharded by city**.

**⑤ Dispatch / Matching service (the brain of Loop 2).** When an order is near-ready, it asks the Location service for **nearby available riders**, **ranks** them (ETA, load, fairness), **offers** the job with a timeout, and performs the **atomic claim** so exactly one rider wins (§7). Structurally similar to Uber's driver matching.

**⑥ Location service (the intake of Loop 3).** Receives the rider-ping firehose. For each ping it **overwrites** the rider's latest position in **Redis GEO / an in-memory quadtree** (never appends a DB row) and **publishes** to a Kafka `rider-location` topic. It also answers Dispatch's "who's near (lat,lng)?" query. This is the component that makes 83K pings/s affordable.

**⑦ WebSocket / Tracking tier (the output of Loop 3).** Holds the ~667K live customer/rider connections and **fans out** location + order-status updates to the right connection. It maps `orderId → riderId` and forwards that rider's positions to the watching customer (§6). Kept separate because holding connections is a different engineering problem from any business logic.

**⑧ Payment service.** Charges idempotently, participates in the **order saga** (refund/compensate on failure), keeps a ledger. Money — correctness-critical.

**⑨ Notification service → push (FCM/APNs)/SMS.** Consumes order-events and pushes the right message to the right party at each transition. Decoupled via Kafka so notification latency never blocks the order path.

### 4.4 The end-to-end life of an order

Here is exactly what happens when **a customer orders from a restaurant**, step by step:

```
1.  Customer browses → Discovery service returns nearby restaurants (cached, geo-indexed).
2.  Customer builds a cart (Redis) → requests a quote (fees/taxes/ETA) → places the order.
3.  Order service PERSISTS the order (state = PAYMENT_PENDING) and returns a payment intent.
        (Durability + strong consistency first — the order is the source of truth.)
4.  Payment succeeds (idempotent) → Order service moves state to PLACED
        → emits a Kafka order-event.
5.  Restaurant app (subscribed to its orders) shows the new order → restaurant ACCEPTS
        with a prep-time estimate → state = ACCEPTED → order-event.
6.  DISPATCH begins (can start now, before food is ready, so the rider arrives just in time):
        Dispatch asks Location svc for available riders near the restaurant
        → ranks them → OFFERS to the best rider(s) with a timeout
        → the winning rider's app accepts → ATOMIC CLAIM assigns the order   ← §7
        → state = RIDER_ASSIGNED → order-event.
7.  Restaurant marks READY_FOR_PICKUP; rider arrives, marks PICKED_UP → OUT_FOR_DELIVERY.
8.  Throughout, the rider's app emits a GPS ping every ~4 s:
        ping → Location svc (overwrite latest in Redis + publish to Kafka)
        → WebSocket tier forwards to the customer watching THIS order   ← §6
        → the customer sees the rider move on the map, with a live ETA.
9.  Rider marks DELIVERED (proof/OTP) → state = DELIVERED (terminal) → order-event
        → customer & restaurant notified → customer rates the order.
```

The single most important thing to notice: **the order state machine (steps 3–7, strongly consistent) and the location firehose (step 8, eventually consistent) are completely separate systems.** The order path needs correctness; the tracking path needs throughput. Mixing them — e.g., writing every GPS ping into the order database — is the classic mistake that sinks the design.

### 4.5 Why this split? (the design rationale)

Each separation exists for a concrete reason — be ready to justify them:

- **Order service separate from Location service** — orders need **strong consistency** and are low-volume (~140/s); locations need **raw throughput** (~83K/s) and only eventual consistency. Different correctness and different scale → different systems. Never let the firehose touch the OLTP order DB.
- **Location service separate from the WebSocket tier** — *ingesting* pings (write path, overwrite-latest) and *fanning them out* to watchers (read path, hold connections) are different jobs with different scaling axes. Splitting lets each scale independently.
- **Dispatch separate from Order** — matching is a real-time geospatial + optimization problem with its own state (offers, timeouts, rider pool); keeping it out of the order state machine keeps both clean.
- **Discovery/browse separate and cached** — it's read-heavy and static; isolating it (Redis/CDN) means the dinner-rush browse load never contends with the ordering or tracking paths.
- **Everything geo-local sharded by city** — because a rider never serves another city, city shards are independent, so one city's peak can't degrade another's.

### 4.6 Where the load actually goes

A senior is expected to know *which* part is hard. The math (verified):

- **Order writes:** ~140/s at peak — **trivial**; a single well-indexed SQL shard per city handles it easily.
- **Location firehose:** ~**83,000 pings/sec** — ~**600× the order-write load**. This is the real write pressure. Handled by **overwrite-latest in Redis GEO** (the latest-position store is only ~**33 MB** for 333K riders — trivial) + **publish to Kafka**; ingestion fits in ~**1 node per city-cluster** at ~100K writes/s.
- **Live connections:** ~**667K** concurrent WebSockets → a **tracking fleet of ~3–7 nodes** (at 250K–100K conns/node). Modest compared to a chat system, but still a dedicated tier.
- **Dispatch:** one geo-query + rank of ~50 candidates per order at ~140 orders/s — **cheap**.
- **Browse:** ~2,100/s, **read-heavy and cacheable** → CDN/Redis absorb it.

> 💡 **The senior framing:** *"Order QPS is nothing (~140/s). The load is the location firehose — ~83K pings/s, ~600× the order writes — plus ~667K live tracking connections. So the design is: absorb the firehose with overwrite-latest Redis + Kafka, fan it out over a WebSocket tier, and keep it entirely off the strongly-consistent order database. Shard by city because the domain is geo-local."*

---

## 5. The Order State Machine

An order is a **state machine** coordinating three parties. Model transitions explicitly and guard illegal ones:

```
CREATED
  → PAYMENT_PENDING → (paid)  PLACED        (payment fails → PAYMENT_FAILED → CANCELLED)
  → PLACED → ACCEPTED           (restaurant accepts; or REJECTED → refund → CANCELLED)
  → ACCEPTED → PREPARING → READY_FOR_PICKUP
  → (dispatch, may run in parallel from ACCEPTED)  RIDER_ASSIGNED
  → RIDER_ASSIGNED → PICKED_UP → OUT_FOR_DELIVERY → DELIVERED   (terminal)
  Any pre-pickup state → CANCELLED (user/restaurant/system) → refund if paid
```

**Why an explicit state machine?** Three actors mutate one order concurrently — the restaurant marks it ready while Dispatch assigns a rider while the customer might cancel. A guarded state machine (only legal transitions, each atomic and idempotent, protected by a `version` for optimistic concurrency) prevents illegal states like "delivered before picked up" and makes retries safe. Each transition emits a Kafka order-event that drives notifications and the customer's live tracking. **Ordering is only needed *within one order's* lifecycle** — a far cheaper guarantee than global ordering.

---

## 6. Hard Problem 1: How a Rider's Location Reaches the Customer's Screen

This is the step most designs hand-wave, and the best interview drill for this system — directly analogous to "how does a chat message get from server A to server B." The rider emits a GPS ping; the customer watching that order must see the pin move. *How do the bytes physically get from the rider's phone to that one specific customer's screen, at 83,000 pings/sec across the fleet?*

### The naive (wrong) approach

"Write each ping to the orders database; the customer polls `GET /orders/{id}/track`." This fails two ways: **83K writes/sec into your OLTP database** will crush it (it's ~600× your order-write load, and these are throwaway values), and **polling** either wastes requests or lags. You must not persist the firehose, and you should push, not poll.

### The key realizations

**(1) You only ever need the *latest* position.** Nobody cares where the rider was 30 seconds ago (for the live view). So the ingestion store is an **overwrite**, not an append — `rider_id → (lat,lng,ts)` in Redis GEO / an in-memory quadtree. That single decision turns an unbounded write stream into a tiny fixed-size store (~33 MB for 333K riders).

**(2) A WebSocket is pinned to the process that holds it** — exactly as in the chat design. The customer's tracking socket lives on **one** node of the WebSocket tier. The rider's ping arrives at the **Location service** (a different process). So, just like cross-gateway delivery in a chat system, *something must carry the rider's position from the Location service to the specific WebSocket node holding that customer's socket.* That "something" is a **pub/sub backplane**.

**(3) You must map the order to the rider to the watcher.** The customer is watching an **order**; the position belongs to a **rider**. Dispatch established `order → rider`. When the customer opens tracking, the WebSocket node **subscribes to that rider's location channel**.

### The mechanism, end to end

```
Rider app --(ping every 4s: lat,lng,ts)--> Location Service
   1. OVERWRITE latest position in Redis GEO:  rider:{id} → (lat,lng,ts)   (no append)
   2. PUBLISH the position to a channel:        rider-loc:{riderId}
   3. also stream to Kafka `rider-location` (sampled → data lake for analytics/ETA training)

Customer --(opens live tracking)--> WebSocket tier node N
   4. node N resolves order → rider (from Dispatch/order data)
   5. node N SUBSCRIBES to channel rider-loc:{riderId}   (via Redis Pub/Sub / NATS / Kafka)

On each ping:
   6. the backplane delivers rider-loc:{riderId} to whichever node(s) are subscribed → node N
   7. node N looks up the customer's socket in its LOCAL in-memory table
   8. node N computes/attaches ETA and writes {lat,lng,eta} down the customer's socket
   9. the customer's map animates the pin.
```

Two maps again, exactly like the chat design:

| Map | Where | Answers |
|:--|:--|:--|
| **`order → rider`** (+ rider live pos in Redis GEO) | shared/global | "whose position do I need, and where is it?" |
| **Local socket table** | in each WebSocket node's RAM | "which actual socket on THIS node is this customer's?" |

The **backplane** (pub/sub) is what physically moves a rider's position to the node holding the watcher's socket; the **local table** is what does the final write. The Location service never talks to a customer socket directly — it publishes, and the node holding that customer is subscribed.

> 💡 **The senior one-liner:** *"I never persist the firehose. Each ping overwrites the rider's latest position in Redis GEO and is published to a per-rider pub/sub channel. When a customer opens tracking, their WebSocket node subscribes to that rider's channel, so every ping is pushed to exactly the node holding that customer's socket, which writes it down the socket with an ETA. Latest-only storage plus a pub/sub backplane is what makes 83K pings/sec and hundreds of thousands of watchers affordable — and it stays completely off the order database."*

---

## 7. Hard Problem 2: How an Order Is Matched to Exactly One Rider

The second hard mechanism: an order needs a rider; there are thousands of riders and thousands of orders at peak; and **exactly one** rider must end up assigned to each order — never zero (if avoidable), never two. *How does that physically happen under concurrency?*

### The steps

```
When an order is near-ready (Dispatch triggered by the ACCEPTED/PREPARING event):
  1. CANDIDATE SEARCH — ask the Location service: "available riders within R km of the restaurant?"
        (a geo-query over the live rider index; §8). Widen R if too few.
  2. RANK — score candidates by: ETA to restaurant (distance + live traffic),
        current load (already carrying orders? batching), rating, acceptance rate, fairness.
  3. OFFER — send an assignment OFFER to the best rider (or top few) with a TIMEOUT (e.g., 15s).
        A rider is offered one order at a time.
  4. RESPOND — rider ACCEPTS or REJECTS/TIMES-OUT:
        accept → go to 5;  reject/timeout → offer the next-best candidate (loop).
  5. ATOMIC CLAIM — assign the order to that rider ONLY IF it's still unassigned:
        compareAndSet(order.rider, null → riderId)      (single atomic operation)
        success → RIDER_ASSIGNED;  failure → someone else already got it, offer this rider elsewhere.
```

### Why the atomic claim is the crux

The dangerous moment is concurrency: two Dispatch workers (or two offers) could try to assign the same order, or one rider could be offered two orders and accept both. The guarantee comes from a **single atomic compare-and-set** on the order's rider field (a conditional DB update `UPDATE orders SET rider_id=:r WHERE id=:o AND rider_id IS NULL`, or a Redis atomic op). Only the first writer succeeds; the second sees `rider_id` already set and backs off. This is the same "atomic claim under contention" pattern as reserving a parking slot or a ticket seat — the operation itself, not a lock you remember to take, is what guarantees single-assignment.

### Two efficiency levers to mention

- **Dispatch before ready.** Start matching at ACCEPTED + a prep-time estimate, so the rider arrives just as the food is ready — cutting rider idle time and total delivery time.
- **Batching.** Assign multiple nearby orders (e.g., two pickups from the same plaza going the same direction) to one rider along an efficient route; the ranking includes the detour cost. A big efficiency win in dense markets.

> 💡 **The senior one-liner:** *"Dispatch is candidate geo-search → multi-factor ranking → offer-with-timeout → **atomic claim**. Two riders can never get the same order because assignment is a single conditional update (`set rider if currently null`) — the first wins, the rest see it's taken. I'd start dispatch before the food is ready and batch nearby orders to cut idle time."*

---

## 8. Geospatial Search: How "Restaurants Near Me" Actually Works

Two geo problems, and they need **different** structures because one set moves and the other doesn't.

**Restaurants (static).** Index each restaurant by a **geohash** (or S2 cell / quadtree). A geohash encodes (lat,lng) into a short string where **shared prefixes mean spatial proximity** — nearby places share a cell. "Near me" = look in the customer's cell + its 8 neighbors (to handle edges), then filter by exact haversine distance:
```
cell        = geohash(lat, lng, precision)
candidates  = restaurants in cell ∪ its 8 neighbor cells      # coarse, index-driven
result      = [c for c in candidates if haversine(c, (lat,lng)) <= radius]   # exact filter
```
Because restaurants don't move, this index is **precomputed and cached** — it's effectively static data.

**Riders (dynamic).** A rider's position changes every 4 seconds, so their geo-index must be **live** — an **in-memory quadtree or Redis GEO** keyed by city, updated from the firehose (§6). Dispatch queries it for "available riders within R km of the restaurant."

> 💡 **The senior point:** *"Static geo (restaurants) → a persisted, cached geohash index. Dynamic geo (riders) → a live in-memory/Redis GEO index rebuilt continuously from pings. Conflating them — trying to persist every rider position into a DB geo-index — is the classic mistake; it can't keep up with 83K writes/sec. Different mutation rates demand different structures."*

---

## 9. Data Contracts: Request Fields, Payloads & DB Schemas

The sections above describe the *working*; this one nails down the concrete **data contracts** — the key fields, the inter-service payloads, and the schema of each store. (Formats illustrative; fields are what matter.)

### Part A — Key client→server requests

**Place order** (customer → Order service):
| Field | Type | Purpose |
|:--|:--|:--|
| `client_order_id` | UUID | **idempotency key** — retries on flaky mobile don't create a duplicate order |
| `cart_id` | string | the cart being ordered (resolves to restaurant + items) |
| `address_id` | string | delivery destination |
| `payment_method` | enum | `upi`/`card`/`wallet`/`cod` |
| `quoted_total` | int (minor units) | echoed from the quote; server rejects (409) if price changed |
| `coupon_code` | string/null | discount |

**Rider location ping** (rider → Location service, every ~4s):
| Field | Type | Purpose |
|:--|:--|:--|
| `rider_id` | string | whose position |
| `lat`,`lng` | double | current position |
| `bearing` | int/null | heading (for the map arrow) |
| `ts` | epoch ms | ping time (server drops stale/out-of-order pings) |
> This is fire-and-forget: the server returns **202** and does not block; only the *latest* ping matters.

**Other frames:** restaurant `accept {prep_minutes}` / `ready`; rider `offer_respond {accept}` / `pickup` / `deliver {proof}`; customer `rating {restaurant, rider}`.

### Part B — Inter-service payloads

**① Order service → Kafka `order-events`** (drives notifications, dispatch, tracking):
```json
{ "order_id":"o_5521","city_id":"blr","state":"ACCEPTED",
  "restaurant_id":"r_918","customer_id":"u_9931","seq":3,"ts":1730000000200 }
```
**② Dispatch → rider (assignment offer):**
```json
{ "type":"OFFER","order_id":"o_5521","restaurant":{"name":"Napoli","lat":..,"lng":..},
  "dropoff":{"lat":..,"lng":..},"payout":72,"expires_in_s":15 }
```
**③ Location service → pub/sub `rider-loc:{riderId}`** (fanned to the watcher's WS node, §6):
```json
{ "rider_id":"d_44","lat":12.9401,"lng":77.6190,"bearing":72,"ts":1730000000200 }
```
**④ WebSocket tier → customer socket** (the enriched live update):
```json
{ "type":"LOCATION","order_id":"o_5521","lat":12.9401,"lng":77.6190,"eta_minutes":8 }
```
**⑤ Order/Payment → Notification service (offline push):** generic, no sensitive content —
```json
{ "recipient":"u_9931","title":"Your order is on the way 🛵","collapse_key":"o_5521" }
```

### Part C — DB schema per store (polyglot)

**① Orders — SQL, sharded by `city_id`** (strong consistency, state machine):
```
orders( order_id PK, city_id, customer_id, restaurant_id, rider_id,
        state, items(json), pricing(json), address(json),
        created_at, accepted_at, picked_at, delivered_at, version )
INDEX (customer_id, created_at DESC)   -- order history
INDEX (rider_id, state)                -- a rider's active orders
```
The **atomic claim** (§7) is `UPDATE orders SET rider_id=:r WHERE order_id=:o AND rider_id IS NULL`.

**② Restaurants + geo — SQL/Document + cache** (static, geohash-indexed):
```
restaurant( id PK, name, geohash, lat, lng, city_id, cuisines[], rating, avg_prep_min, is_open )
INDEX (geohash)
```
**③ Live rider location — Redis GEO / in-memory** (overwrite-latest, the firehose sink):
```
GEO key city:{cityId}:riders  →  member rider_id at (lng,lat)      -- GEOADD overwrites
rider:{id} → {lat,lng,bearing,ts,status}  (TTL a few min; stale = went offline)
```
**④ Cart — Redis** (ephemeral): `cart:{id} → {restaurant_id, items[]}`.
**⑤ Payments/ledger — SQL (ACID):** `payments(id PK, order_id, amount, status, idempotency_key UNIQUE)`.
**⑥ Events — Kafka**, partitioned by `city_id`/`rider_id` (order-events + rider-location streams).
**⑦ Search — Elasticsearch** (restaurants/dishes); **Analytics — data lake** (sampled location history).

> 💡 **The contract in one line:** *"The order request carries a `client_order_id` (idempotency) and a `quoted_total` (price-lock); the order lives in a city-sharded SQL row advanced by a guarded state machine, and is claimed by a rider via a single conditional update. Rider pings are fire-and-forget, overwrite-latest in Redis GEO, and republished to a per-rider pub/sub channel that the watcher's WebSocket node subscribes to. Orders/payments are strongly consistent; location is eventual. Every field exists for idempotency, ordering, price-integrity, or geo."*

---

## 10. Payments & the Order Saga

Order placement spans **order + payment + confirmation** across services with no single ACID wrapper — the **saga** pattern:
- **Payment fails** → move to `PAYMENT_FAILED → CANCELLED` (compensating action); nothing charged.
- **Restaurant rejects / order cancelled after charge** → **refund** (compensating transaction) + `CANCELLED`.
- **Payment succeeds but the order write fails** → the critical inconsistency (money, no order): **retry the order write idempotently**, or **refund**. **Idempotency keys** (`client_order_id`, payment idempotency key) + a unique constraint + a **ledger** guarantee no double-charge on retry.

> 💡 The bar is *"no double charge, no lost order."* You can't atomically commit an order and a charge across services, so use a saga with idempotent steps and compensations; design carefully around the money-taken-but-no-order case.

---

## 11. Notifications

Order-events on Kafka fan out to a **Notification service** that pushes the right message to the right party at each transition: customer ("order accepted", "rider on the way", "delivered"), restaurant ("new order"), rider ("new offer", "pickup ready"). Channels: push (FCM/APNs), SMS, in-app. Decoupled via Kafka so notification latency or failures never block the order or tracking paths.

---

## 12. Scaling Summary

- **Shard everything geo-local by `city_id`** — orders, the live-rider index, dispatch, Kafka partitions. A city is an almost-independent mini-system; one city's dinner rush can't degrade another's. **The single biggest lever.**
- **Keep the location firehose off the OLTP DB** — overwrite-latest in Redis GEO + publish to Kafka; sample to a lake for history.
- **Fan out live tracking over a dedicated WebSocket tier** (~3–7 nodes for ~667K conns) with a pub/sub backplane mapping `rider → watchers`.
- **Cache/CDN the read-heavy browse** (restaurants/menus are near-static).
- **Stateless services autoscale**; read replicas for restaurant/menu reads.
- **Dispatch is cheap** (one geo-query + rank per order) — no special scaling needed beyond city-sharding the rider index.
- **Strong consistency only where it matters** (orders/payments); eventual everywhere else (location/discovery/notifications).

---

## 13. Senior Follow-Up Questions (with Answers)

**Q1. What's the real scaling challenge here?**
Not order QPS (~140/s). It's the **rider-location firehose (~83K pings/s, ~600× the order writes)** and **~667K live tracking connections**. The design centers on absorbing the firehose (overwrite-latest Redis + Kafka) and fanning it out over a WebSocket tier — entirely off the order database.

**Q2. How does a rider's live position reach the exact customer watching that order?**
Each ping overwrites the rider's latest position in Redis GEO and is published to a per-rider pub/sub channel. When a customer opens tracking, their WebSocket node subscribes to that rider's channel (resolved via `order → rider`), so every ping is delivered to the node holding that customer's socket, which writes it down with an ETA. Latest-only + pub/sub is what makes it affordable. (§6)

**Q3. How do you match an order to exactly one rider?**
Candidate geo-search (available riders near the restaurant) → rank by ETA/load/fairness → offer with a timeout → **atomic claim** (`UPDATE orders SET rider_id=:r WHERE rider_id IS NULL`). The conditional update guarantees only one rider wins under concurrency. Start dispatch before food is ready; batch nearby orders. (§7)

**Q4. Why don't you persist every GPS ping?**
83K writes/s of throwaway values would crush the OLTP DB, and you only ever need the *latest* position for the live view. So you overwrite the latest in Redis GEO and sample (1-in-N) to a data lake for analytics — never a DB row per ping.

**Q5. How does "restaurants near me" work, and why is it different for riders?**
Restaurants (static) use a precomputed, cached **geohash** index — query the customer's cell + neighbors, filter by exact distance. Riders (dynamic) need a **live** Redis GEO / in-memory index rebuilt from pings. Different mutation rates → different structures. (§8)

**Q6. How do you keep orders and payments correct?**
Order = a guarded **state machine** (atomic, idempotent, version-checked transitions) in a city-sharded SQL store; payment = an idempotent **saga** with compensations (refund on reject/cancel) + a ledger + idempotency keys. No double-charge, no lost order.

**Q7. How do you shard, and why is it natural here?**
By **city/region** — the domain is geo-local (a rider never serves another city), so each city is an independent shard for orders, riders, dispatch, and event streams. The domain hands you the shard key.

**Q8. What's the consistency model?**
Strong (CP-ish) for orders/payments; eventual (AP) for location, discovery, and notifications — a few-seconds-stale rider pin is fine, and making it strongly consistent at 83K/s would be both impossible and pointless.

**Q9. Why separate the Location service from the WebSocket tier?**
Ingesting pings (write path, overwrite-latest) and fanning them to watchers (read path, holding ~667K connections) are different jobs with different scaling axes. Splitting lets each scale independently and keeps the firehose off the connection tier's concerns.

**Q10. What happens if no rider is available, or the restaurant rejects?**
No rider → widen the search radius, add a surge incentive, queue and retry, or cancel+refund with an apology. Restaurant reject/no-accept → timeout → auto-cancel + refund (a saga compensation). Always refund if already charged.

**Q11. How is the customer's WebSocket different from the rider's?**
The rider holds a socket while online to send pings and receive offers; the customer holds one only while watching a live order to receive location/status pushes. Both are pinned to one node of the tracking tier, which is why the pub/sub backplane is needed to route a rider's pings to the customer's node. (§6)

**Q12. How do you compute ETA?**
From the rider's live position + the road-network route (distance + live traffic) via a maps service, recomputed as the rider moves and attached to each pushed location update.

---

## 14. Quick Glossary
- **Location firehose** — the high-rate stream of rider GPS pings (~83K/s); absorbed, never persisted per-ping.
- **Overwrite-latest** — storing only a rider's current position (Redis GEO), not a history of pings.
- **Pub/sub backplane** — internal bus (Redis Pub/Sub / NATS / Kafka) that routes a rider's pings to the WebSocket node holding the watching customer's socket.
- **WebSocket / tracking tier** — the fleet holding live customer/rider connections and pushing updates.
- **Dispatch / matching** — assigning an order to the best available rider (geo-search → rank → offer → atomic claim).
- **Atomic claim** — a single conditional update (`set rider if null`) guaranteeing exactly one rider per order.
- **Order state machine** — the guarded lifecycle PLACED→ACCEPTED→…→DELIVERED across three actors.
- **Geohash / S2 cell / quadtree** — spatial indexes for "near me"; static for restaurants, live for riders.
- **Batching** — one rider carrying multiple nearby orders along a route.
- **Saga** — cross-service transaction with compensating actions (refund on cancel).
- **Idempotency key** (`client_order_id`) — prevents duplicate orders/charges on retry.
- **Geo-sharding (by city)** — partitioning by city because the domain is geo-local; independent shards.
- **ETA** — estimated arrival from rider position + route + live traffic.

---

*Reference document. Contributions and corrections welcome.*
