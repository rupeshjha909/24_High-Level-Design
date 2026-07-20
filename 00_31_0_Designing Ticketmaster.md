# Designing Ticketmaster (Event Ticket Booking): A Senior Interview Guide

> A practical, interview-focused reference for a ticket-booking system — a **consistency-critical, contention-heavy** problem that inverts the usual read-heavy-feed priorities. This guide builds the architecture up piece by piece, traces the full life of a booking, goes deep on the two genuinely hard mechanisms (how *exactly one* user wins a contested seat, and how the virtual waiting room physically admits and orders a million-user stampede), nails down the data contracts, and covers hold-and-pay, the payment Saga, bot mitigation, sharding, consistency, and failure modes — with verified contention math and a senior follow-up bank.

> 💡 **The one idea (see §6):** booking a seat is a **CP operation** — the *opposite* CAP choice from Twitter/feed systems. You must **never double-book**, so **correctness beats availability**. The whole design is distributed **concurrency control + admission control + fairness** over a scarce resource — *not* scaling reads. ~333 bookings/sec is trivial; **1M users contending for a fixed seat set** (≈200 users fighting over a single front-row seat) is the hard part.

---

## Table of Contents
1. [How to Approach This (and Why It's Different)](#1-how-to-approach-this-and-why-its-different)
2. [Requirements](#2-requirements)
3. [Capacity: Contention, Not Throughput (verified)](#3-capacity-contention-not-throughput-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a booking](#44-the-end-to-end-life-of-a-booking)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Seat State Machine](#5-the-seat-state-machine)
6. [Hard Problem 1: How Exactly One User Wins a Contested Seat (deep dive)](#6-hard-problem-1-how-exactly-one-user-wins-a-contested-seat-deep-dive)
7. [Hard Problem 2: The Virtual Waiting Room (deep dive)](#7-hard-problem-2-the-virtual-waiting-room-deep-dive)
8. [The Hold-and-Pay Workflow](#8-the-hold-and-pay-workflow)
9. [Payment Failure & the Saga Problem](#9-payment-failure--the-saga-problem)
10. [Bot Mitigation](#10-bot-mitigation)
11. [Data Contracts: Request Fields, Payloads & DB Schemas](#11-data-contracts-request-fields-payloads--db-schemas)
12. [Scaling: Isolate Contention by Event](#12-scaling-isolate-contention-by-event)
13. [Consistency Model](#13-consistency-model)
14. [Failure Modes & Handling](#14-failure-modes--handling)
15. [Senior Follow-Up Questions (with Answers)](#15-senior-follow-up-questions-with-answers)
16. [Quick Glossary](#16-quick-glossary)

---

## 1. How to Approach This (and Why It's Different)

Most large "design X" systems — Twitter, Instagram, YouTube — are **read-heavy and AP** (favor availability + eventual consistency; a slightly stale feed is fine). **Ticketmaster inverts this.** Its defining path — booking a seat — is **contention-heavy and consistency-critical (CP-leaning):** you must **never double-book a seat**, so for booking, **correctness beats availability.** It's far better to make a user wait, or reject a booking, than to sell one seat twice.

That single fact reframes everything: this is a **distributed concurrency-control problem**, not a scale-out-reads problem. Structure: requirements → recognize the challenge is *contention*, not throughput → go deep on the **hard mechanisms** (winning a contested seat, the waiting room) → hold-and-pay → payment-failure Saga.

> 💡 **Senior signal:** explicitly frame booking as a **CP operation** (the opposite CAP choice from the feed systems) and treat "no double-booking" as the requirement everything else serves. **Browsing** is a normal read-heavy cached path; the design effort goes almost entirely into the **booking** path. Say this up front.

---

## 2. Requirements

### Functional
- **Browse** events by city, date, artist. **View** seat map + pricing.
- **Reserve (hold)** a seat + **checkout**. **Payment**. **E-ticket delivery**. **Resale** (optional).

### Non-Functional
- **Concurrency hotspot** — popular events draw millions of users in the same minute.
- **No double-booking** — the critical correctness requirement.
- **High availability** for browse; **fair queueing / anti-bot** for booking.

> Note the split personality: **browsing** is read-heavy (cache it, like a normal site); **booking** is consistency-critical under extreme contention. The design effort goes into the booking path.

---

## 3. Capacity: Contention, Not Throughput (verified)

```
Hot sale: ~1,000,000 concurrent users
Bookings: 100,000 in 5 min → ~333 bookings/sec               (MODEST throughput)
Seat-map/browse: ~8M reqs / 5 min → ~26,700/s                (read-heavy → cache/CDN)
Seats / big show: ~60,000
  Avg contention:  1M / 60k        = ~17 users per seat
  HOT seats (~5%): draw ~60% demand = ~200 users per hot seat  ← thousands of SETNX on one seat
Peak SETNX attempts at launch burst: ~67,000/s                (Redis atomic ops — easy)
Queue drain: admit 1,000/s → 16.7 min for 1M; 2,000/s → 8.3 min; 5,000/s → 3.3 min
Redis holds (one show): ~6 MB (trivial)
Storage: seats ~200 GB (2B rows); bookings ~0.25 TB/yr        (modest)
Hold TTL 10 min → abandoned checkout auto-frees the seat in ≤10 min
```

**The crucial observation:** **~333 bookings/sec is nothing.** The hard part is that **1M users contend for a fixed, scarce seat set at the same instant** — and demand is *skewed*, so a single front-row seat can face **~200+ simultaneous grab attempts**, of which **exactly one must win.** The challenges:
- **Contention** — thousands targeting the *same* seat (the double-booking problem, §6).
- **Thundering herd** — 1M users hitting booking at launch would crash it without admission control (the waiting room, §7).
- **Fairness** — without ordering, bots and the fastest connections win; real fans lose.

> 💡 The design centers on **concurrency control, admission, and fairness over a scarce resource**, not raw QPS. Say: *"100K bookings in 5 min is ~333/s — trivial. The difficulty is a million people reaching for the same seats at once."*

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

A ticket-booking system is fundamentally different from a normal web app because of one requirement: **a fixed, scarce set of seats must be sold to a stampede of users without ever selling the same seat twice, and fairly.** A normal "just take the write" design fails two ways at once: letting a million users hit the booking flow simultaneously would **collapse** it (thundering herd), and letting them race for seats without mutual exclusion would **double-sell** contested seats. So the architecture is built around two gates and one arbiter. The **first gate** is a **virtual waiting room** that converts the stampede into a bounded, ordered (FIFO) trickle — admitting only N users/sec, each carrying a signed **access token**. The **second gate**, for each seat, is a **distributed lock**: a fast **Redis `SETNX` hold** gives instant "this seat is yours for 10 minutes" UX, while the **arbiter** — a durable **DB `SELECT … FOR UPDATE`** at payment time — makes the final, authoritative decision so that even if a cache race let two users both *think* they hold a seat, exactly one commit wins. Everything else (Payment as a Saga, Notification for e-tickets, Search for browse) hangs off that spine, and the whole thing is **sharded by event** so one mega-sale's contention can't touch another show.

### 4.2 The diagram

```
[Client] ─► [DNS] ─► [Global LB] ─► [WAF / Bot Filter]
                                          │
                                          ▼
                                [Virtual Queue Service]   ← FIFO; admit N/sec; MINT signed access tokens
                                          │ (admitted, token-gated)
                                          ▼
                                   [Booking API]
                                     ├─► [Seat Inventory Service] ─► [Redis (holds, TTL'd)]  ← fast front line
                                     │                              └► [Booking DB (SQL, FOR UPDATE)]  ← ARBITER
                                     ├─► [Payment Service] ─► [PSP + ledger]   (idempotent, Saga)
                                     ├─► [Notification Service]   (e-tickets/email/push)
                                     └─► [Search Service] ─► [Elasticsearch + CDN/Redis]   (browse, read-heavy)
```

The key visual idea: traffic is **gated** — WAF/bot filter → virtual queue → booking API — so only **admitted, vetted, ordered** users reach the contention-sensitive inventory. For each seat, **Redis is the fast hold; the DB is the arbiter.** Browse hangs off to the side (cached) so it never touches the booking path.

### 4.3 Each component, in detail

**① Client (the app/web).** Browses the seat map (cached reads), enters the **waiting room**, and — once admitted with a token — sends **hold** and **checkout** requests. It shows the user their queue position/ETA and the 10-minute hold countdown. It holds no authority; the server decides everything.

**② WAF / Bot Filter.** The outermost gate. Blocks obvious bot traffic, enforces CAPTCHA, and rate-limits per IP before anything reaches the queue. Its job is to make sure the *humans* are the ones who get into the waiting room (§10).

**③ Virtual Queue Service (the admission gate).** Holds arriving users in a **FIFO queue** (a Redis sorted set / stream, score = arrival time) and **admits N/sec** at the rate the booking path can safely serve, minting a short-lived **signed access token** for each admitted user. This does load-leveling *and* fairness in one (§7). Without it, a 1M-user launch crashes the system and becomes a bot free-for-all.

**④ Booking API (the orchestrator).** Accepts only **token-gated** requests. Orchestrates the hold (via Seat Inventory), the payment (via Payment), and the final confirm; runs the Saga on failure. Stateless — scales horizontally.

**⑤ Seat Inventory Service (the concurrency core).** Owns seat state and the **two-layer lock**: the fast **Redis `SETNX` hold** (instant UX, TTL auto-release) and the authoritative **DB `SELECT … FOR UPDATE`** confirm (§6). This is where "never double-book" is enforced.

**⑥ Redis (holds).** In-memory, TTL'd holds — the fast front line. ~6 MB for a 60k-seat show (trivial). Atomic `SETNX` gives instant mutual exclusion; the TTL auto-releases abandoned holds. A cache, so a failover *can* lose a hold — acceptable because the DB is the arbiter.

**⑦ Booking DB (the durable arbiter).** SQL, ACID, **the source of truth** for confirmed bookings and true seat state. `SELECT … FOR UPDATE` at payment time breaks any tie a Redis race might have allowed. Sharded by `event_id`.

**⑧ Payment Service.** Charges idempotently, participates in the **Saga** (release/refund on failure), keeps a ledger. Money — correctness-critical.

**⑨ Notification Service.** Sends e-tickets/confirmations (email/push) on successful booking. Off the critical path.

**⑩ Search Service.** Browse by city/date/artist — read-heavy, cached (Elasticsearch + Redis/CDN). Scales independently of the booking path.

### 4.4 The end-to-end life of a booking

Here is exactly what happens when **a fan buys a seat during a hot on-sale**, step by step:

```
BEFORE THE STAMPEDE REACHES INVENTORY:
1.  Fan hits the on-sale → WAF/bot filter (CAPTCHA, rate limit) → allowed through.
2.  Fan enters the VIRTUAL QUEUE (Redis FIFO). Sees "position 842,176, ~9 min."
3.  The queue admits N/sec; when the fan reaches the front, it MINTS a signed
        ACCESS TOKEN (JWT: show + expiry) and admits them to the Booking API.

THE HOLD (fast path):
4.  Fan picks seat S5 → Booking API (token-checked) → Seat Inventory:
        SETNX seat:{show}:S5 = user  EX 600    (atomic 10-min hold in Redis)
        success → "S5 is yours for 10:00"      (thousands may target S5; only ONE SETNX wins — §6)
        failure → "seat just taken," show fresh availability.

PAYMENT + THE AUTHORITATIVE CONFIRM (arbiter):
5.  Fan pays within 10 min → Payment Service charges idempotently.
6.  On payment success → Seat Inventory opens a DB transaction:
        BEGIN; SELECT status FROM seats WHERE id=S5 FOR UPDATE;   -- durable row lock
        verify AVAILABLE/HELD-by-this-user
        UPDATE seats SET status='BOOKED', user_id=fan;
        INSERT booking; COMMIT;                                  -- exactly one commit wins
        (If a Redis race let two users hold S5, only ONE FOR UPDATE commit succeeds; the other → "just taken," refund.)
7.  Booking confirmed durably → Notification sends the e-ticket.

IF THINGS GO WRONG:
8a. Payment fails → release the hold (Redis DEL / let TTL expire) → seat returns. (compensation)
8b. Fan abandons → after 10 min the Redis TTL expires → seat auto-returns to inventory.
8c. Payment OK but booking write fails → Saga: retry the write idempotently, else REFUND + release.
```

The single most important thing to notice: **the Redis hold (step 4) and the DB confirm (step 6) are two different layers with two different jobs.** Redis gives *speed and auto-release* for the hold UX; the DB gives *correctness* at the money moment and **breaks any tie**. That separation — fast front line + durable arbiter — is the whole design.

### 4.5 Why this split? (the design rationale)

Each separation exists for a concrete reason — be ready to justify them:

- **Virtual queue in front of the Booking API** — the booking path can only safely serve a bounded rate; the queue converts a 1M-user stampede into that bounded, *ordered* stream (load-leveling + fairness). Without it the system collapses and bots win.
- **Redis hold vs DB confirm (two-layer lock)** — you want *both* a snappy hold UX *and* absolute correctness. Redis alone can lose a lock on failover (can't be the arbiter); the DB alone is too slow/contended for the instant "seat is yours" interaction. Use Redis for the hold, the DB to decide ties. **This is the CP choice in action.**
- **Booking path separate from browse** — browse is read-heavy/cacheable/AP; booking is contention-heavy/CP. Isolating browse (Elasticsearch + CDN) keeps the on-sale browse load from touching the contention-sensitive inventory.
- **Payment as its own Saga-coordinated service** — you can't atomically commit a seat and a charge across services, so payment is a separate step with compensations (§9).
- **Shard by event** — contention is per-event; isolating each event (and giving mega-events dedicated clusters) means one stampede can't degrade another show (§12).

### 4.6 Where the load actually goes

A senior is expected to know *which* part is hard. The math (verified):

- **Booking throughput:** ~**333/s** — **trivial**; not the bottleneck.
- **Contention:** ~**17 users per seat on average, ~200+ per hot seat**, with **~67K SETNX attempts/s** at the launch burst. This is the real pressure, and it's exactly what Redis's single-threaded atomic `SETNX` is good at — every attempt on one seat is serialized, one wins. The DB `FOR UPDATE` sees far less (only the ~333/s that actually pay).
- **Thundering herd:** ~**1M concurrent** at launch — handled entirely by the **virtual queue** (admit N/s), not by scaling the booking tier.
- **Browse:** ~**26,700/s**, read-heavy → **cache/CDN** absorb it.
- **Holds memory:** ~**6 MB** Redis per show — trivial. **Storage:** seats ~200 GB, bookings ~0.25 TB/yr — modest.

> 💡 **The senior framing:** *"Throughput is trivial (~333/s). The load is contention — hundreds of users per hot seat, tens of thousands of lock attempts per second — plus a 1M-user herd at launch. I handle the herd with a FIFO virtual queue, and the per-seat contention with an atomic Redis hold backed by a DB `FOR UPDATE` arbiter. Everything else (browse) is a cached read path."*

---

## 5. The Seat State Machine

Each seat for a given show is a tiny state machine — the core correctness object:

```
AVAILABLE ──hold (SETNX ok)──► HELD ──confirm (payment ok, FOR UPDATE)──► BOOKED  (terminal)
   ▲                             │
   │      TTL expiry /           │
   └──── release / payment-fail ─┘
```

- **AVAILABLE → HELD**: atomic acquire (Redis `SETNX`; DB check at confirm). Only one user wins.
- **HELD → BOOKED**: authoritative DB commit at payment success (`FOR UPDATE`).
- **HELD → AVAILABLE**: TTL expiry, explicit release, or payment failure (compensation).
- **BOOKED** is terminal (until resale/refund).

> 💡 Every transition is **atomic and guarded by the current state** — you can't book a seat that isn't HELD by you, and you can't hold one that isn't AVAILABLE. That's what makes "never double-book" enforceable under concurrency.

---

## 6. Hard Problem 1: How Exactly One User Wins a Contested Seat (deep dive)

This is the heart of the system and the best interview drill — the analog of "how does a message physically get from server A to server B." A single front-row seat may face **~200+ users clicking it in the same second** (thousands of lock attempts across retries). **Exactly one must win; the seat must never be sold twice.** *How does that physically happen?*

### The naive (wrong) approach
"Read the seat's status; if AVAILABLE, mark it BOOKED." This is a **classic race**: two requests both read AVAILABLE, both write BOOKED — **double-sold.** The check and the write aren't atomic, so under contention it fails exactly when it matters most. You need **mutual exclusion per seat** — a distributed lock — not a read-then-write.

### The key realization: mutual exclusion must be atomic, and you need two layers
There are two independent needs, and conflating them is the confusion:
1. **A fast, snappy hold** so the UX is "this seat is yours for 10 minutes" the instant you click — and it must **auto-release** if you wander off.
2. **An absolutely correct final decision** at the money moment that can **never** double-sell, even if the fast layer glitches.

No single store does both well. Redis is fast and has TTLs but is a cache (can lose a lock on failover). The DB is durable and authoritative but too slow/contended to be the instant hold. So you use **both, in layers.**

### Layer 1 — the fast hold: Redis `SETNX` (atomic)
```
SET seat:{show}:S5  user_X  NX  EX 600      # set-if-not-exists, 10-min TTL
```
`NX` makes this **atomic**: of the ~200 users hammering S5, **exactly one** `SET` succeeds; every other returns "not set" → "seat just taken." Because Redis executes commands **single-threaded**, all those concurrent attempts are **serialized** — there's no read-then-write gap to race through. The **TTL** auto-releases the hold if the winner abandons checkout, so no seat gets stuck (~67K such attempts/s across the show is trivial for Redis).

### Layer 2 — the arbiter: DB `SELECT … FOR UPDATE` (durable, authoritative)
```
BEGIN;
SELECT status FROM seats WHERE show_id=X AND seat_id=S5 FOR UPDATE;  -- ACID row lock
-- verify AVAILABLE / HELD-by-this-user
UPDATE seats SET status='BOOKED', user_id=Y WHERE seat_id=S5;
INSERT booking …;
COMMIT;                                                              -- exactly one commit wins
```
`FOR UPDATE` takes an **ACID row lock** on that seat: concurrent transactions **queue**, and only the first to commit turns it BOOKED; the rest see it's no longer AVAILABLE and abort. This is the **true, durable state**.

### The race that the two layers resolve
Here's the scenario people miss: Redis fails over mid-sale and, briefly, **two users both think they hold S5** (the cache lost the lock). Both proceed to pay. At confirm, both hit the DB:
```
User A: BEGIN; SELECT S5 FOR UPDATE → AVAILABLE → UPDATE BOOKED; COMMIT;   ✓ wins
User B: BEGIN; SELECT S5 FOR UPDATE → (waits for A) → sees BOOKED → ABORT;  ✗ "just taken" → refund
```
**The DB is the final arbiter.** Even if the fast layer glitched, the `FOR UPDATE` serialization guarantees exactly one booking. The Redis hold is an *optimization* for UX and to reduce wasted payments; the DB is what *guarantees correctness*.

### Two layers, two jobs (the crux)
| Layer | Where | Job |
|:--|:--|:--|
| **Redis `SETNX` + TTL** | in-memory cache | **fast hold** + auto-release; serialize the ~200 grabs per seat instantly |
| **DB `SELECT … FOR UPDATE`** | durable SQL | **authoritative confirm**; break any tie the cache allowed; never double-sell |

> 💡 **The senior one-liner:** *"A single seat can face hundreds of simultaneous grabs, so a read-then-write races and double-sells. I use mutual exclusion in two layers: an atomic Redis `SETNX` with a TTL for a fast, auto-releasing hold — Redis is single-threaded so the grabs serialize and exactly one wins — and a durable DB `SELECT … FOR UPDATE` at payment time as the arbiter, so even if a Redis failover let two users both think they hold the seat, only one transaction commits. Redis gives speed and auto-release; the database guarantees we never double-book. When in doubt, the system refuses rather than double-sells — that's the CP choice."*

---

## 7. Hard Problem 2: The Virtual Waiting Room (deep dive)

The second hard mechanism: **1M users arrive in the same minute.** If they all hit the booking flow at once, the system **collapses** and it becomes a bot free-for-all. *How do you physically absorb and order the stampede?*

### The mechanism
```
1. Arriving users don't touch the booking flow — they enter a FIFO QUEUE.
   Implementation: a Redis sorted set / stream per show, score = arrival timestamp
   (or a monotonically increasing ticket number).
2. Each user gets a queue token + polls their POSITION and ETA
   ("you are 842,176 in line, ~9 minutes").
3. An ADMISSION WORKER pops the head of the queue at a fixed rate N/sec
   — the rate the booking + inventory path can safely serve — and for each admitted user
   MINTS a short-lived SIGNED ACCESS TOKEN (JWT: show_id + user + expiry).
4. The Booking API REJECTS any hold/checkout request without a valid, unexpired access token.
5. The access token EXPIRES (e.g., 10–15 min) → an admitted user must finish in that window
   or re-queue → bounds how long each admitted user occupies scarce capacity.
```

### Why it does two jobs at once
- **Load leveling / admission control** — the queue is a **front-door buffer** (the queue-as-buffer principle applied to admission). At admit=1,000/s it drains 1M in ~16.7 min; at 2,000/s in ~8.3 min. You tune N to what inventory can serve, converting a destructive spike into a steady stream. The system stays *up*.
- **Fairness (FIFO)** — first-come-first-served ordering, so it's *not* "fastest connection / most bots win." This is the fair counterpart to rate limiting, applied to a scarce-resource launch.

### Why tokens gate everything downstream
The signed token is the proof of admission. Because the Booking API only accepts token-bearing requests, a bot that skips the queue and hits the booking endpoint directly is **rejected** — the queue isn't just a UI nicety, it's an **enforced gate**. Token expiry also prevents an admitted user from sitting on capacity forever.

> 💡 **The senior one-liner:** *"A million users can't hit booking at once, so I put a FIFO virtual queue in front — a Redis sorted set that admits N/sec at the rate inventory can serve, minting a short-lived signed access token per admitted user. The booking API rejects anything without a valid token, so the queue is an enforced admission gate, not just a screen. That gives me load-leveling (the system stays up) and fairness (first-come, not fastest-bot) in one mechanism, and token expiry bounds how long each user holds capacity."*

---

## 8. The Hold-and-Pay Workflow

Booking is inherently **two-phase**: hold, then pay. The TTL makes it safe.
```
1. Pick a seat → SETNX seat:X with a 10-min TTL          (AVAILABLE → HELD)
2. Pay within 10 min → Payment Service (idempotent)
3. Success → DB FOR UPDATE commit (HELD → BOOKED) → e-ticket   (durable)
4. Timeout/abandon → Redis TTL expiry → seat auto-returns       (HELD → AVAILABLE)
```
The elegance is **TTL-based auto-release**: an abandoned checkout doesn't strand a seat — the hold simply expires (a background sweeper reconciles the DB, but the TTL does the heavy lifting). The hold window balances giving users time to pay vs not locking scarce inventory too long. Worst case a seat is unavailable for ≤10 min before auto-release; an explicit "cancel" frees it instantly.

---

## 9. Payment Failure & the Saga Problem

Booking spans **seat inventory + payment + confirmation** across services with no single ACID wrapper — the **Saga problem**:
- **Payment fails** → **release the hold** (compensating action) so the seat returns. Clean.
- **Payment succeeds but the booking DB write fails** → the **critical inconsistency** (money taken, no seat). You must not lose either. Fix: a **Saga with compensations** — either **retry the booking write idempotently** until it commits, or **refund + release**. **Idempotency keys** on the payment prevent double-charging during retries.

```
hold ──ok──► charge payment ──ok──► confirm booking (DB FOR UPDATE) ──ok──► e-ticket
              │                          │
        fail → release hold        fail → RETRY idempotently, else REFUND + release
```

> 💡 The money-taken-but-no-seat case is the one you design carefully around: **retry-idempotently or compensate, never silently lose.** You can't atomically commit a seat and a charge across services, so Saga-style coordination with idempotent steps + a ledger is the answer.

---

## 10. Bot Mitigation

Scalper bots grab inventory faster than humans and resell at markup — defeating fairness. No single measure suffices (bots adapt), so **layer** them:
- **CAPTCHA** at queue/booking entry.
- **Rate limiting** per IP/user.
- **Behavioral analysis** — mouse/interaction patterns to spot scripts.
- **Token-based throttling** — signed tokens gating and pacing access.

Combined with the **FIFO virtual queue**, these ensure vetted, ordered humans get fair access. Bot mitigation is a **fairness** requirement as much as a security one — it's what makes the queue's FIFO fairness actually benefit real people.

---

## 11. Data Contracts: Request Fields, Payloads & DB Schemas

The sections above describe the *working*; this one nails down the concrete **data contracts**.

### Part A — Key client↔server requests
**Join queue** (client → Virtual Queue): `POST /queue/join {show_id}` + bot/CAPTCHA token →
```json
{ "queue_token":"q_abc", "position":842176, "eta_seconds":540, "status":"WAITING" }
```
**Poll queue** → `{ "status":"ADMITTED", "access_token":"acc_XYZ", "access_expires_at":"…" }`

**Hold a seat** (client → Booking API), gated by the access token + idempotency:
```json
// headers: X-Access-Token: acc_XYZ, Idempotency-Key: 4f1c-…
// request
{ "show_id":"s_101", "seat_ids":["A12-5","A12-6"] }
// response 201 (won) / 409 SEAT_TAKEN (lost, with which seats failed)
{ "hold_id":"h_99", "seat_ids":["A12-5","A12-6"], "status":"HELD",
  "expires_at":"…", "total":10000 }
```
**Checkout** (client → Booking API):
```json
// headers: X-Access-Token, Idempotency-Key
{ "hold_id":"h_99", "payment_token":"tok_psp_…", "quoted_total":10000 }
→ { "booking_id":"b_77", "status":"CONFIRMED", "seat_ids":["A12-5","A12-6"],
    "tickets":[{"seat_id":"A12-5","ticket_url":"…","qr":"…"}] }
```

### Part B — Inter-service payloads
- **Booking API → Seat Inventory (hold):** `{ show_id, seat_ids[], user_id, ttl_s:600 }` → `{ acquired[], failed[] }`.
- **Booking API → Payment:** `{ booking_ref, amount, currency, idempotency_key, payment_token }` → `{ status, payment_id }`.
- **Seat Inventory → Booking DB (confirm):** the `FOR UPDATE` transaction (§6) — the only strongly-consistent write.
- **Booking API → Notification:** `{ user_id, booking_id, seat_ids[], e_ticket_urls[] }` (off critical path).

### Part C — DB schema per store
**Redis (holds & queue) — ephemeral, TTL'd:**
```
SET  seat:{show}:{seat}  = user_id   NX EX 600        -- the atomic hold (§6)
ZSET queue:{show}         member=user  score=arrival_ts -- the FIFO waiting room (§7)
```
**Seats — SQL, sharded by event_id (the arbiter):**
```
seats( seat_id, show_id, section, row, col, type, price,
       status,       -- AVAILABLE | HELD | BOOKED
       user_id, version,
       PRIMARY KEY (show_id, seat_id) )
-- confirm = SELECT … FOR UPDATE then UPDATE status='BOOKED'
```
**Bookings & payments — SQL (ACID):**
```
bookings( booking_id PK, user_id, show_id, seat_ids[], payment_id, status, created_at )
payments( payment_id PK, booking_id, amount, status, idempotency_key UNIQUE )   -- ledger
holds(    hold_id PK, show_id, seat_ids[], user_id, expires_at )                -- mirrors Redis
```
**Events/shows + search:**
```
events( id PK, name, venue_id, city, date )
shows(  id PK, event_id, time, on_sale_at )
-- browse served from Elasticsearch + CDN/Redis (read-heavy)
```

> 💡 **The contract in one line:** *"The client joins a FIFO queue and gets a signed access token; every booking request carries that token plus an idempotency key. A hold is an atomic Redis `SETNX` with a TTL; the confirm is a `SELECT … FOR UPDATE` on the seat row (the arbiter). Seats/bookings/payments live in ACID SQL sharded by event; holds and the queue live in TTL'd Redis. Every field exists for admission, mutual exclusion, ordering, or idempotency."*

---

## 12. Scaling: Isolate Contention by Event

The contention is **per-event**, so partition by it:
- **Shard by `event_id`** — a Taylor Swift stampede is isolated; it can't degrade a small local show. Seats, holds, queue, and booking DB all partition by event/show.
- **Hot events get a dedicated cluster** (DB + Redis + queue) so the mega-event's load is contained.
- **Geo-route** users to the nearest region; **browse** served from CDN/edge, scaling independently of booking.

> 💡 **Isolation is the key scaling move:** because contention doesn't span events, sharding by event turns one terrifying mega-sale into an isolated problem with its own capacity, while the rest of the platform stays healthy. Say: *"Shard by event_id; give the giants dedicated clusters."*

---

## 13. Consistency Model

- **Strong (CP)** for the booking path: seat state (`FOR UPDATE`), bookings, payments (ACID + idempotency + ledger). Never double-book; never double-charge.
- **Eventual (AP)** for browse: event lists and the **seat map are cached and may be slightly stale** — the authoritative check is at hold/commit, so a stale map never causes a wrong sale.

> 💡 The split *is* the design: **booking = CP** (correctness over availability), **browse = AP** (availability over freshness). The stale seat map is safe *because* the hold/commit re-validates against the source of truth.

---

## 14. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Thousands grab one seat | Risk of double-sell | Redis `SETNX` (one wins) + DB `FOR UPDATE` arbiter breaks ties → `409` for losers (§6) |
| Redis failover loses a hold | Two "think" they hold | DB `FOR UPDATE` at payment resolves it — one commits, other `409` + refund |
| User abandons checkout | Seat stuck HELD | TTL auto-release (≤10 min); explicit cancel frees instantly |
| Payment fails | Charged? no | Release hold; no capture |
| **Payment ok, booking write fails** | Money, no seat | **Saga:** retry idempotently, else refund + release; idempotency key prevents double charge |
| 1M-user stampede | System crash | Virtual queue admits N/sec (admission control) |
| Bots grab inventory | Unfair, scalping | CAPTCHA + rate limit + behavioral + token throttling + FIFO queue |
| Hot event overload | Degrades others | Shard by event_id; dedicated cluster |
| Duplicate hold/checkout on retry | Double booking/charge | `Idempotency-Key` returns the original result |

---

## 15. Senior Follow-Up Questions (with Answers)

**Q1. How is this different from a Twitter/feed design?** Feeds are read-heavy + AP; Ticketmaster's booking path is **CP** — never double-book, correctness over availability. The hard problem is distributed concurrency control over scarce seats, not scaling reads.

**Q2. How do you prevent double-booking?** Mutual exclusion per seat in two layers: atomic Redis `SETNX` + TTL for a fast auto-releasing hold, and DB `SELECT … FOR UPDATE` for the authoritative confirm. The DB is the arbiter — if a Redis race let two users hold, exactly one commit wins (§6).

**Q3. Redis lock vs DB row lock — trade-offs?** Redis: fast, atomic, TTL auto-release, but a cache that can lose locks on failover. DB: durable, ACID, authoritative, but slower/contention-prone. Use Redis for the hold, DB for the durable confirm; DB decides ties.

**Q4. Why a virtual waiting room, and how does it work?** A 1M-user stampede would crash booking and be unfair. A FIFO queue (Redis sorted set) admits N/sec (load-leveling) and mints signed access tokens; the booking API rejects tokenless requests (enforced gate). FIFO gives fairness; token expiry bounds capacity use (§7).

**Q5. Throughput vs contention — the key insight?** ~100K bookings in 5 min = ~333/s, trivial. The difficulty is ~1M users fighting over a fixed seat set (~200 per hot seat, ~67K lock attempts/s) → design for concurrency, admission, and fairness, not QPS.

**Q6. How does hold-and-pay avoid stuck seats?** TTL'd hold (10 min): abandon/timeout → seat auto-returns; explicit cancel frees instantly. Payment success confirms durably in the DB + issues the e-ticket.

**Q7. Payment ok but booking write fails?** The critical inconsistency. Saga: retry the booking write idempotently, or refund + release — never money without a seat. Idempotency keys prevent double charges on retry.

**Q8. Why a Saga?** Booking spans inventory + payment + confirmation with no single ACID transaction → local steps with compensating actions (release hold / refund) on failure.

**Q9. How do you scale and isolate hot events?** Shard by `event_id`; dedicated clusters for mega-events; geo-route; browse from CDN. Contention is per-event, so partition by event.

**Q10. How do you keep it fair against bots?** Layered: CAPTCHA, per-IP/user rate limits, behavioral analysis, token throttling — plus the FIFO queue so vetted, ordered users win. No single measure suffices.

**Q11. Is the seat map the source of truth?** No — it's a cached, possibly-stale read. The authoritative check is the atomic hold + the DB `FOR UPDATE` at commit, so a stale map never causes a wrong sale.

**Q12. Why is Redis a good fit for the hold despite being a cache?** It's single-threaded (atomic `SETNX` serializes the ~67K/s grabs with no read-then-write gap) and has native TTLs (auto-release). Its one weakness — losing a lock on failover — is covered by the DB arbiter, so using it for the *temporary* hold is safe.

---

## 16. Quick Glossary
- **CP operation** — chooses consistency over availability (booking); opposite of AP feeds.
- **Double-booking** — selling one seat twice; the critical failure to prevent.
- **Distributed lock** — mutual exclusion across servers (Redis `SETNX`) to hold a seat.
- **SETNX / NX** — Redis set-if-not-exists; atomic lock acquisition; serialized by Redis's single thread.
- **TTL auto-release** — a hold expiring automatically so abandoned seats return.
- **SELECT … FOR UPDATE** — ACID DB row lock; authoritative, durable seat state (the arbiter).
- **Two-layer lock** — fast Redis hold + durable DB confirm; the DB breaks ties.
- **Virtual waiting room / queue** — FIFO admission control pacing/ordering users.
- **Access token** — short-lived signed pass issued on admission, required by booking endpoints (enforced gate).
- **Hold** — a temporary TTL'd reservation during checkout.
- **Seat state machine** — AVAILABLE → HELD → BOOKED with TTL/compensation back to AVAILABLE.
- **Saga** — cross-service transaction with compensating actions (release hold / refund).
- **Compensating action** — an undo for a failed multi-step booking.
- **Idempotency key** — token preventing duplicate charges/bookings on retry.
- **Shard by event_id** — isolating each event's contention onto its own partition/cluster.
- **Contention factor** — how many users target one seat (~17 avg, ~200+ per hot seat).

---

*Reference document. Contributions and corrections welcome.*
