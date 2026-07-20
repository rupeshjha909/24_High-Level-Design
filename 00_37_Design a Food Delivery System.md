# Design a Food Delivery System — Detailed HLD (presentation-ready, full request/response)

> A complete, interview-ready high-level design for a food-delivery platform (Swiggy / Zomato / DoorDash / Uber Eats). Covers all four actors (customer, restaurant, delivery partner, platform), the full service map, **request/response for every major API**, the **order state machine**, the two hard problems — **geospatial matching (dispatch)** and **real-time location tracking at scale** — data models, capacity math (verified), scaling, consistency, failure modes, and a senior Q&A. Nothing left out.

> 💡 **The two things that define this problem:** (1) it's a **geospatial, real-time system** — the scale isn't order volume (modest) but the **rider-location firehose (~83K updates/sec)** and **hundreds of thousands of live tracking connections**; (2) it's a **multi-party workflow** — a single order coordinates customer, restaurant, and rider through a **state machine**. Get those two right and the rest is standard CRUD + caching.

---

## Table of Contents
1. [Requirements](#1-requirements)
2. [Capacity estimation (verified)](#2-capacity-estimation-verified)
3. [The actors & the service map](#3-the-actors--the-service-map)
4. [Architecture](#4-architecture)
5. [The order state machine](#5-the-order-state-machine)
6. [API design — full contract for every endpoint](#6-api-design--full-contract-for-every-endpoint)
7. [Data models & schemas (polyglot)](#7-data-models--schemas-polyglot)
8. [Geospatial search — finding nearby restaurants & riders](#8-geospatial-search--finding-nearby-restaurants--riders)
9. [The order-placement flow (end to end)](#9-the-order-placement-flow-end-to-end)
10. [Delivery-partner matching / dispatch (hard part #1)](#10-delivery-partner-matching--dispatch-hard-part-1)
11. [Real-time location tracking (hard part #2)](#11-real-time-location-tracking-hard-part-2)
12. [Payments & the order saga](#12-payments--the-order-saga)
13. [Notifications](#13-notifications)
14. [Caching & CDN](#14-caching--cdn)
15. [Scaling & sharding (geo-partitioning)](#15-scaling--sharding-geo-partitioning)
16. [Consistency model](#16-consistency-model)
17. [Failure modes & handling](#17-failure-modes--handling)
18. [Senior follow-up Q&A](#18-senior-follow-up-qa)
19. [TL;DR](#19-tldr)
20. [Glossary](#20-glossary)

---

## 1. Requirements

### Functional
- **Customer:** discover restaurants near a location; browse menu; build a cart; place & pay for an order; track it live (rider location + ETA); rate the order.
- **Restaurant:** receive orders; accept/reject; mark preparing/ready; manage menu & availability.
- **Delivery partner (rider):** go online/offline; receive assignment offers; accept/reject; report live location; mark picked-up/delivered.
- **Platform:** match orders to riders (dispatch); compute pricing (item total, delivery fee, taxes, surge); send notifications.

### Non-functional
- **Low latency** on discovery/browse (feels instant) and on live tracking (near-real-time).
- **High availability** — ordering must stay up at meal-time peaks; degrade gracefully.
- **Consistency where it matters** — order state and payment must be correct (no double-charge, no lost order); location can be eventually consistent.
- **Geospatial + real-time** at scale — the location firehose and live tracking are the scaling story.
- **Peaky load** — traffic concentrates at lunch/dinner.

---

## 2. Capacity estimation (verified)

**Assumptions:** 30M DAU, 5M orders/day, ~15 browse/menu views per order, ~40-min delivery, rider pings every 4 s.

```
Orders:      5M/day → ~58/s avg, ~140/s at meal-time peak  (order QPS is MODEST)
Browse/search: 75M/day → ~2,100/s peak  (read-heavy → cache/CDN)
Active riders at peak: ~333,000 concurrent
LOCATION UPDATES: ~83,000/s at peak  ← the write firehose (don't hit the primary DB)
WebSocket conns at peak: ~667,000  (customers tracking + riders) → scalable fanout tier
Orders storage: 10.2 GB/day → 3.74 TB/yr (1.8B orders/yr)
Restaurants ~500k; ~50M menu items (small, mostly static → cache/CDN)
```

**Takeaways:**
- **Order QPS is small; the firehose is location updates (~83K/s)** — so the architecture centers on ingesting/serving live positions cheaply (Redis/in-memory + Kafka), *not* on scaling order writes.
- **~667K live connections** → a dedicated WebSocket/fanout tier.
- **Browse is read-heavy & cacheable** → CDN + Redis for restaurant/menu data.
- **Orders/riders are geo-local** → **shard by city/region** (a Bangalore rider never serves Delhi), which makes each shard independent and small.

> 💡 The estimation flips the naive assumption: *"food delivery = lots of orders."* No — orders are ~140/s. The real scale is **real-time geospatial data** (location ingestion + live tracking connections). Say this; it reframes the whole design.

---

## 3. The actors & the service map

| Service | Responsibility |
|:--|:--|
| **API Gateway** | Auth, routing, rate limiting, request fan-in |
| **Restaurant/Discovery** | Search nearby restaurants (geo), details, availability |
| **Menu** | Menu items, prices, customizations (read-heavy, cached) |
| **Cart** | Ephemeral cart (Redis) |
| **Order** | Create order, order state machine, order history |
| **Payment** | Charge, refunds, idempotency, payment saga |
| **Pricing** | Item total, delivery fee, taxes, surge |
| **Dispatch/Matching** | Assign orders to riders (geospatial + ranking) |
| **Location** | Ingest rider pings, serve live positions, ETA |
| **Tracking/WebSocket** | Push live order+rider updates to customers |
| **Notification** | Push/SMS/email to all parties |
| **Rating** | Order/restaurant/rider ratings |
| **Search** | Restaurant/dish full-text search (Elasticsearch) |

---

## 4. Architecture

```
 Customer app ─┐                                   ┌─ Restaurant app
 Rider app  ───┼──► [API Gateway / LB] ──► services │─ (accept/ready)
 (websockets)  │         │                          └─ Rider app (accept/location)
               │         ├─► [Discovery/Search] ──► [Restaurant DB (SQL)] + [ES] + [Geo index]
               │         ├─► [Menu] ──► [Menu store + Redis + CDN]
               │         ├─► [Cart] ──► [Redis]
               │         ├─► [Order] ──► [Order DB (SQL, geo-sharded)] ──► [Kafka: order-events]
               │         ├─► [Payment] ──► [PSP] + [ledger]
               │         ├─► [Pricing]
               │         ├─► [Dispatch/Matching] ◄── [Location svc (nearby riders)]
               │         └─► [Rating]
               │
   (live) ─────┴─► [WebSocket/Tracking tier] ◄── [Kafka: location + order-status] ◄── [Location svc]
                                                     ▲
 Rider pings ──────────────────────────────────────►│  [Location svc: Redis GEO / in-mem quadtree]
                                                     └─ (latest position only; sampled to lake for analytics)
```

- **Stateless services** behind the gateway; **Kafka** carries order events + the location stream; **Redis** holds carts + live rider positions; the **WebSocket tier** fans live updates to customers.

---

## 5. The order state machine

An order is a **state machine** coordinating three parties. Model transitions explicitly (guard illegal ones):

```
CREATED
  → PAYMENT_PENDING → (paid)  PLACED        (payment failed → PAYMENT_FAILED → CANCELLED)
  → PLACED → ACCEPTED           (restaurant accepts; or REJECTED → refund → CANCELLED)
  → ACCEPTED → PREPARING
  → PREPARING → READY_FOR_PICKUP
  → (dispatch)  RIDER_ASSIGNED  (may happen in parallel from ACCEPTED onward)
  → RIDER_ASSIGNED → PICKED_UP
  → PICKED_UP → OUT_FOR_DELIVERY
  → OUT_FOR_DELIVERY → DELIVERED   (terminal, success)
  Any pre-pickup state → CANCELLED (by user/restaurant/system) → refund if paid
```

> 💡 **Why an explicit state machine?** Three actors mutate one order concurrently (restaurant marks ready while dispatch assigns a rider). A guarded state machine (only legal transitions, each transition atomic and idempotent) prevents illegal states ("delivered before picked up") and makes retries safe. Each transition emits a Kafka **order-event** that drives notifications and tracking.

---

## 6. API design — full contract for every endpoint

### 6.0 Conventions
- **Base:** `https://api.food.example/v1`; **Auth:** `Authorization: Bearer <token>` (role: customer/restaurant/rider).
- **Idempotency:** order create & payment accept `Idempotency-Key`.
- **Errors:** `{ "error": { "code": "...", "message": "...", "requestId": "..." } }`.
- **Status:** 200/201/202/400/401/403/404/409(conflict, e.g. illegal transition)/402(payment)/429/500.

---

### 6.1 Discover restaurants — `GET /v1/restaurants`
**Request**
```
GET /v1/restaurants?lat=12.9352&lng=77.6245&cuisine=pizza&sort=eta&veg=true&page=1&limit=20
Authorization: Bearer <token>
```
| Param | Meaning |
|:--|:--|
| `lat,lng` | customer location (drives the geo query) |
| `cuisine`, `veg`, `priceRange` | filters |
| `sort` | `eta` \| `rating` \| `distance` \| `popularity` |
| `page,limit` | pagination |

**Response `200`**
```json
{
  "restaurants": [
    { "id": "r_918", "name": "Napoli Pizza", "cuisines": ["pizza","italian"],
      "rating": 4.4, "etaMinutes": 32, "distanceKm": 1.8, "priceForTwo": 500,
      "isOpen": true, "deliveryFee": 30, "imageUrl": "https://cdn/…" }
  ],
  "page": 1, "nextPage": 2
}
```

---

### 6.2 Restaurant details + menu — `GET /v1/restaurants/{id}` and `GET /v1/restaurants/{id}/menu`
**Menu response `200`**
```json
{
  "restaurantId": "r_918",
  "categories": [
    { "name": "Pizzas", "items": [
      { "itemId": "i_101", "name": "Margherita", "price": 299, "veg": true,
        "customizations": [ { "group": "Size", "required": true,
          "options": [ {"id":"sz_m","name":"Medium","price":0},
                       {"id":"sz_l","name":"Large","price":120} ] } ],
        "available": true } ] }
  ]
}
```

---

### 6.3 Cart — `POST /v1/cart/items` (Redis-backed, ephemeral)
```json
// request
{ "restaurantId": "r_918", "itemId": "i_101", "qty": 2,
  "customizations": ["sz_l"], "notes": "extra basil" }
// response 200 → the recomputed cart (subtotal, item lines)
{ "cartId": "c_77", "restaurantId": "r_918",
  "items": [ { "itemId":"i_101","name":"Margherita (Large)","qty":2,"lineTotal":838 } ],
  "subtotal": 838 }
```
> 💡 A cart is tied to **one restaurant**; adding an item from another restaurant prompts "start a new cart." Cart lives in **Redis** (ephemeral, no need to persist forever).

---

### 6.4 Quote / pricing — `POST /v1/orders/quote`
```json
// request
{ "cartId": "c_77", "addressId": "a_5", "couponCode": "FIRST50" }
// response 200
{ "subtotal": 838, "deliveryFee": 30, "surge": 10, "taxes": 44,
  "discount": 50, "total": 872, "etaMinutes": 35 }
```
Pricing is a **separate call** so the customer sees fees/ETA before committing.

---

### 6.5 Place order — `POST /v1/orders`
**Request**
```
POST /v1/orders
Idempotency-Key: 9f2c-...
```
```json
{ "cartId": "c_77", "addressId": "a_5", "paymentMethod": "upi",
  "couponCode": "FIRST50", "quotedTotal": 872 }
```
**Response `201`**
```json
{ "orderId": "o_5521", "status": "PAYMENT_PENDING", "total": 872,
  "paymentIntent": { "provider": "razorpay", "clientToken": "..." },
  "etaMinutes": 35 }
```
> 💡 `quotedTotal` is echoed so the server can reject if the price changed since the quote (`409`). `Idempotency-Key` prevents a double order on retry.

---

### 6.6 Order status & tracking
- `GET /v1/orders/{id}` → current state + timeline.
```json
{ "orderId":"o_5521","status":"OUT_FOR_DELIVERY",
  "timeline":[{"state":"PLACED","at":"…"},{"state":"ACCEPTED","at":"…"},
              {"state":"PICKED_UP","at":"…"}],
  "rider":{"id":"d_44","name":"Ravi","phoneMasked":"+91-xxxxx-12","vehicle":"bike"},
  "etaMinutes": 8 }
```
- `GET /v1/orders/{id}/track` → latest rider location + ETA (poll fallback).
```json
{ "orderId":"o_5521","rider":{"lat":12.9401,"lng":77.6190,"bearing":72},
  "etaMinutes":8,"polyline":"…encoded route…" }
```
- **WebSocket** `wss://api.food.example/ws/orders/{id}` → server pushes `{type:"location", lat,lng,eta}` and `{type:"status", state}` events live (preferred over polling).

---

### 6.7 Restaurant app
```
PATCH /v1/orders/{id}/accept        { "prepTimeMinutes": 20 }   → 200, status ACCEPTED
PATCH /v1/orders/{id}/reject        { "reason": "out of stock" } → 200, status REJECTED (triggers refund)
PATCH /v1/orders/{id}/ready                                     → 200, status READY_FOR_PICKUP
PATCH /v1/restaurants/{id}/menu/items/{itemId}  { "available": false }
```

### 6.8 Delivery partner (rider) app
```
POST  /v1/partners/{id}/online      { "lat":…, "lng":… }        → go online (enters dispatch pool)
POST  /v1/partners/{id}/location    { "lat":…, "lng":…, "ts":…, "bearing":… }  → 202 (firehose; see §11)
PATCH /v1/orders/{id}/offer/respond { "accept": true }          → accept/reject an assignment offer
PATCH /v1/orders/{id}/pickup                                    → status PICKED_UP
PATCH /v1/orders/{id}/deliver       { "proof": "otp:1234" }     → status DELIVERED
```

### 6.9 Rating — `POST /v1/orders/{id}/rating`
```json
{ "restaurantRating": 5, "riderRating": 4, "comment": "hot & fast" }  → 201
```

---

## 7. Data models & schemas (polyglot)

| Data | Store | Why |
|:--|:--|:--|
| Users, addresses | **PostgreSQL** | relational, transactional |
| Restaurants, menu | **SQL/Document + Redis + CDN** | read-heavy, mostly static → cache hard |
| Restaurant **geo index** | **geohash/S2 cells** (in DB or a geo service) | "restaurants near (lat,lng)" |
| Orders | **SQL, geo-sharded by city** | transactional; state machine; strong consistency |
| Cart | **Redis** | ephemeral |
| Live rider location | **Redis GEO / in-memory quadtree** | firehose; latest-only; sub-ms geo queries |
| Search (restaurants/dishes) | **Elasticsearch** | full-text + faceted filters |
| Events (order/location) | **Kafka** | async fan-out to dispatch, tracking, notifications, analytics |
| Payments/ledger | **SQL (ACID)** | money — correctness critical |
| Analytics | **data lake** | sampled location history, order analytics |

**Orders table (sharded by city_id):**
```
orders( order_id PK, city_id, customer_id, restaurant_id, rider_id,
        status, items(json), pricing(json), address(json),
        created_at, accepted_at, picked_at, delivered_at, version )
INDEX (customer_id, created_at DESC)   -- order history
INDEX (rider_id, status)               -- rider's active orders
```

**Restaurant geo record:**
```
restaurant( id PK, name, geohash, lat, lng, cuisines[], rating,
            avg_prep_min, is_open, city_id )
INDEX (geohash)  -- prefix match for nearby lookup
```

---

## 8. Geospatial search — finding nearby restaurants & riders

Two geo problems: **static** (restaurants — rarely move) and **dynamic** (riders — move constantly).

- **Restaurants (static):** index by **geohash** (or **S2 cell** / quadtree). A geohash encodes lat/lng into a string where shared prefixes = spatial proximity. "Nearby" = restaurants in the customer's cell + the 8 neighboring cells, then filter by exact distance. Precomputed and cached — restaurants don't move.
- **Riders (dynamic):** their location changes every few seconds, so use an **in-memory quadtree** or **Redis GEO** (`GEOADD`/`GEOSEARCH`) keyed by city, updated from the location firehose. Dispatch queries "available riders within R km of the restaurant."

```
nearby(lat,lng, radius):
  cell = geohash(lat,lng, precision)
  candidates = union(cell, neighbors(cell))      # coarse, index-driven
  return [c for c in candidates if haversine(c, (lat,lng)) <= radius]  # exact filter
```

> 💡 **Static vs dynamic geo needs different structures.** Restaurants → a persisted geohash index (build once, cache). Riders → a live, in-memory/Redis geo index rebuilt continuously from pings. Conflating them (persisting every rider position to a DB geo index) is the classic mistake — it can't keep up with 83K writes/sec.

---

## 9. The order-placement flow (end to end)

```
Customer      API/Order      Payment    Restaurant   Dispatch    Rider    Tracking(WS)
  │ POST /orders  │             │            │           │          │          │
  │──────────────►│ create order (PAYMENT_PENDING), idempotency check          │
  │◄ paymentIntent│             │            │           │          │          │
  │  pay ─────────────────────► │ charge (idempotent)     │          │          │
  │               │◄ paid ──────│            │           │          │          │
  │               │ status=PLACED → Kafka order-event ───►│ (notify) │          │
  │               │             │  restaurant ACCEPTS ────│          │          │
  │               │ status=ACCEPTED/PREPARING → event ─────────────► dispatch starts
  │               │             │            │  READY    │  find nearby riders  │
  │               │             │            │───────────► offer → accept ─────►│
  │               │ RIDER_ASSIGNED → event ──────────────────────────────────► push to customer
  │               │             │  rider PICKED_UP → OUT_FOR_DELIVERY → DELIVERED
  │◄═══ live location + status via WebSocket throughout ════════════════════════│
```

Each transition is **atomic + idempotent**, emits a **Kafka order-event**, and drives notifications + the customer's live tracking.

> 💡 **Dispatch can start before READY** (as soon as ACCEPTED + a prep-time estimate) so the rider arrives just as the food is ready — reducing rider idle time and total delivery time. That timing optimization is a good senior detail.

---

## 10. Delivery-partner matching / dispatch (hard part #1)

When an order needs a rider (at/near READY), the **Dispatch service**:
1. **Find candidates** — query the live rider geo-index for **available riders within R km** of the restaurant (§8).
2. **Rank** them — by ETA to restaurant (distance + traffic), current load (batching), rating, acceptance rate, fairness.
3. **Offer** — send an assignment **offer** to the best rider(s) with a **timeout**; on accept → assign; on reject/timeout → offer the next. (Or auto-assign in dense markets.)
4. **Handle race** — only one rider can win an order (atomic assign — `compareAndSet(order.rider, null → riderId)`); a rider is offered one order at a time.
5. **Batching** — assign **multiple nearby orders** to one rider along an efficient route (big efficiency lever in dense areas); the ranking accounts for detour cost.

> 💡 **Dispatch is a real-time matching + optimization problem, not a simple assignment.** State the pieces: geo candidate search → multi-factor ranking (ETA, load, fairness) → offer/accept with timeout and atomic claim → optional batching. It's structurally like Uber's driver matching. The atomic claim is what guarantees two riders never get the same order under concurrency.

---

## 11. Real-time location tracking (hard part #2)

The firehose: ~**83K rider pings/sec** at peak, and ~**667K customers/riders on live connections**. Design so this never touches the primary DB.

**Ingestion (write path):**
```
Rider app --(POST /partners/{id}/location every 4s)--> Location Service
   → update latest position in Redis GEO / in-memory quadtree (overwrite, not append)
   → publish to Kafka topic `rider-location` (for tracking fanout + sampled analytics)
   → 202 Accepted (fire-and-forget; app doesn't wait)
```
- **Latest-only in Redis/memory** — you only need the *current* position; overwrite it. Do **not** insert a DB row per ping.
- **Sample to the data lake** (e.g., 1 in N pings) for route analytics/history — not every ping.

**Serving (read path / live tracking):**
```
Customer --(WebSocket wss://…/ws/orders/{id})--> Tracking tier
   Tracking tier subscribes to that rider's location stream (Kafka / Redis pub-sub)
   → pushes {lat,lng,eta} to the customer every few seconds
ETA computed from rider position + route (Maps/road-network service)
```
- A **dedicated WebSocket/fanout tier** holds the ~667K connections (sticky, horizontally scaled); it maps `orderId → riderId` and forwards that rider's positions to the watching customer.
- **Fallback:** clients that can't hold a socket poll `GET /orders/{id}/track` every few seconds.

> 💡 **The core trick: separate "latest position" (Redis, overwrite) from "position history" (sampled to a lake), and never persist the firehose to your OLTP DB.** Tracking is served from the live in-memory/Redis layer + a pub-sub fanout, so 83K writes/sec and 667K readers cost the primary database nothing. This decoupling is the whole scaling answer for tracking.

---

## 12. Payments & the order saga

Order placement spans multiple steps (reserve order → charge → confirm) across services — a classic **saga**:
- **Idempotent charge** — `Idempotency-Key` + a unique constraint so a retry never double-charges.
- **On payment success** → order `PLACED`; **on failure/timeout** → `PAYMENT_FAILED` → `CANCELLED` (compensating action).
- **On restaurant reject or cancellation** → issue a **refund** (compensating transaction) and move to `CANCELLED`.
- **Ledger** — append-only record of charges/refunds for auditability.

> 💡 Model order+payment as a **saga with compensations** (refund on reject/cancel), and make every step **idempotent** so retries after a crash are safe. "No double charge, no lost order" is the correctness bar — enforce it with idempotency keys + a unique constraint + a ledger.

---

## 13. Notifications

Order-events on Kafka fan out to a **Notification service** that pushes to the right party at each transition: customer ("order accepted", "rider on the way", "delivered"), restaurant ("new order"), rider ("new offer", "pickup ready"). Channels: push (FCM/APNs), SMS, in-app. Decoupled via Kafka so notification latency/failures never block the order path.

---

## 14. Caching & CDN

- **Restaurant lists & menus** → Redis + **CDN** (mostly static, read-heavy; ~2,100 browse/s peak). Cache per (geo-cell, filters); invalidate on menu/availability change.
- **Search** → Elasticsearch with its own caching.
- **Live location** → the Redis/in-memory layer *is* the cache (§11).
- **Menus behind a CDN** with short TTL; availability toggles (item sold out) pushed as cache invalidations or checked at order time.

---

## 15. Scaling & sharding (geo-partitioning)

- **Shard by city/region.** Orders, riders, and dispatch are **geo-local** — a Bangalore order only ever involves Bangalore restaurants/riders. So partition orders and the live-rider index **by city**, giving independent shards, each small (peak order load per city ≪ 140/s). This is the single biggest scaling lever.
- **Stateless services** autoscale behind the gateway; **read replicas** for restaurant/menu reads; **CDN** absorbs browse.
- **WebSocket tier** scales horizontally with sticky routing (by orderId/riderId); a pub-sub layer (Kafka/Redis) fans location to the right connection.
- **Kafka** partitioned by city/rider for the event + location streams.
- **Hot restaurants** (a viral place) — cache aggressively; the order write is still just one row.

> 💡 **Geo-sharding is natural here because the domain is geo-local.** Unlike systems where you invent a shard key, food delivery hands you one: the city. Each city is an almost-independent mini-system. Say this — it makes the "how do you scale?" answer clean.

---

## 16. Consistency model

- **Strong** where money/state lives: **orders** (guarded state machine, atomic transitions) and **payments** (ACID, idempotent, ledger). Two riders can't claim one order (atomic assign); a customer can't be double-charged (idempotency key).
- **Eventual** where it's fine: **rider location** (a position a few seconds stale is acceptable), **restaurant lists/ETAs** (cached), **notifications** (async).

> 💡 Split consistency by data criticality: orders/payments = strong; location/discovery/notifications = eventual. Trying to make live location strongly consistent would be both impossible at 83K/s and pointless (nobody needs a rider's centimeter-exact position).

---

## 17. Failure modes & handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Payment succeeds, order write fails | Charged, no order | Saga: reconcile → auto-refund or complete; idempotency keys make retry safe |
| Restaurant never accepts | Order stuck | Timeout → auto-cancel + refund, or reassign restaurant/notify |
| No riders available | Can't dispatch | Widen radius, surge incentive, queue + retry, or cancel+refund with apology |
| Rider goes offline mid-delivery | Delivery stalls | Detect via missing pings → reassign to nearby rider |
| Location firehose overload | Ingestion lag | Latest-only overwrite + Kafka buffer; drop stale pings; degrade ping frequency |
| WebSocket tier node dies | Customers lose live view | Reconnect to another node (state re-derived from Kafka/Redis); poll fallback |
| Two riders claim one order | Double assignment | Atomic `compareAndSet` on `order.rider` — only one wins |
| Meal-time spike | Load surge | Geo-sharding + autoscale + cached browse; order QPS is modest anyway |
| Duplicate order on retry | Two orders | `Idempotency-Key` returns the original |

---

## 18. Senior follow-up Q&A

**Q1. What's the real scaling challenge?** Not order QPS (~140/s peak) — it's the **rider-location firehose (~83K/s)** and **~667K live tracking connections**. Design ingestion (latest-only in Redis + Kafka) and a WebSocket fanout tier around that.

**Q2. How do you find restaurants near me?** Geohash/S2 index on restaurants (static): query the customer's cell + neighbors, filter by exact distance, cache results.

**Q3. How do you match an order to a rider?** Query the live rider geo-index for available riders near the restaurant → rank by ETA/load/fairness → offer with timeout → atomic claim on accept → optional batching. Dispatch can start before food is ready to cut idle time.

**Q4. How do you avoid two riders getting the same order?** Atomic `compareAndSet(order.rider, null → riderId)` — only one assignment wins under concurrency.

**Q5. How do you track the rider live without crushing the DB?** Overwrite latest position in Redis/in-memory (never a DB row per ping), stream via Kafka to a WebSocket fanout tier, sample to a lake for history. The OLTP DB never sees the firehose.

**Q6. How do you handle payments correctly?** Order+payment as a **saga** with idempotent steps and compensations (refund on reject/cancel), a ledger, and an idempotency key + unique constraint to prevent double charges.

**Q7. How do you shard?** By **city/region** — the domain is geo-local, so each city is an independent shard (orders, riders, dispatch). Biggest scaling lever.

**Q8. Consistency?** Strong for orders/payments (state machine + ACID + idempotency); eventual for location/discovery/notifications.

**Q9. How does the order state machine prevent bad states?** Only legal transitions allowed, each atomic + idempotent, guarded by the current state (+ a version for optimistic concurrency) — so concurrent actors (restaurant/dispatch/rider) can't produce "delivered before picked up."

**Q10. What if no rider is available / restaurant rejects?** Widen radius/surge/queue for riders; timeout+auto-cancel+refund for restaurant no-accept — always with a compensating refund if already paid.

**Q11. How do you compute ETA?** Rider position + route via a maps/road-network service (distance + live traffic), recomputed as the rider moves; shown in tracking.

**Q12. Batching?** Assign multiple nearby orders to one rider along an efficient route; ranking includes detour cost — a big efficiency lever in dense markets.

---

## 19. TL;DR

**Design:** stateless services behind a gateway — Discovery (geo), Menu (cached), Cart (Redis), **Order (state machine, geo-sharded SQL)**, Payment (saga), **Dispatch (geo matching)**, **Location (firehose → Redis + Kafka)**, **WebSocket tracking tier**, Notifications, Search (ES). Kafka carries order + location events.

**The two hard parts:** (1) **Dispatch** — find nearby available riders (live geo index) → rank by ETA/load/fairness → offer w/ timeout → atomic claim → batch. (2) **Tracking** — ingest ~83K pings/s as *latest-only* in Redis/in-memory, stream via Kafka to a WebSocket fanout tier serving ~667K live connections; never persist the firehose to OLTP.

**APIs:** discover (`GET /restaurants?lat&lng&…`), menu, cart, quote, **place order** (`POST /orders`, idempotent, saga), status + `track` + **WebSocket**, restaurant accept/ready, rider online/location/pickup/deliver, rating — full request/response above.

**Correctness & scale:** guarded order state machine + idempotent payment saga (strong consistency); location/discovery eventual; **shard by city** (geo-local domain); cache/CDN the read-heavy browse.

### Verified numbers
```
5M orders/day → ~58/s avg, ~140/s peak (order QPS is MODEST)
Browse 75M/day → ~2,100/s peak (cache/CDN)
Location firehose: ~83,000/s at peak  ← the real scale
Live WebSocket conns: ~667,000 at peak
Orders storage: 3.74 TB/yr | Restaurants ~500k, ~50M menu items (cacheable)
Shard by city (~500) → each shard independent & small
```

### One-line philosophy
> **A food-delivery system is a geospatial, multi-party real-time workflow: model the order as a guarded state machine coordinating customer/restaurant/rider, dispatch by querying a live rider geo-index and atomically claiming the order, and — the actual scale challenge — serve live tracking by ingesting the ~83K/s location firehose as latest-only in Redis and fanning it over a WebSocket tier, never through your OLTP DB. Shard by city because the domain is geo-local, keep orders/payments strongly consistent via a saga, and cache the read-heavy browse hard.**

---

## 20. Glossary
- **Dispatch / matching** — assigning an order to the best available rider (geo + ranking).
- **Location firehose** — the high-rate stream of rider GPS pings (~83K/s).
- **Geohash / S2 cell / quadtree** — spatial indexes for "near me" queries.
- **Order state machine** — the guarded lifecycle PLACED→ACCEPTED→…→DELIVERED.
- **Saga** — a multi-step transaction with compensating actions (e.g., refund on cancel).
- **Batching** — one rider carrying multiple nearby orders along a route.
- **WebSocket fanout tier** — the service holding live connections and pushing updates.
- **Geo-sharding** — partitioning by city/region because the domain is geo-local.
- **Latest-only** — storing just the current rider position (overwrite), not a history of pings.
- **ETA** — estimated arrival, from rider position + route + traffic.

---

*Reference document — presentation-ready. Contributions and corrections welcome.*
