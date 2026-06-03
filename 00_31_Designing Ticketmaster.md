# Designing Ticketmaster (Event Ticket Booking): A Senior Interview Guide

> A practical, interview-focused reference for designing a ticket-booking system — a consistency-critical, contention-heavy problem that inverts the usual read-heavy-feed priorities. Covers preventing double-booking (distributed locks), the virtual waiting room, bot mitigation, the hold-and-pay workflow, payment-failure handling via Saga, and contention-isolating sharding. With trade-offs and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This (and Why It's Different)](#1-how-to-approach-this-and-why-its-different)
2. [Requirements](#2-requirements)
3. [Capacity: It's About Contention, Not Throughput](#3-capacity-its-about-contention-not-throughput)
4. [Critical Challenge 1: Preventing Double-Booking](#4-critical-challenge-1-preventing-double-booking)
5. [Critical Challenge 2: The Virtual Waiting Room](#5-critical-challenge-2-the-virtual-waiting-room)
6. [Critical Challenge 3: Bot Mitigation](#6-critical-challenge-3-bot-mitigation)
7. [The Hold-and-Pay Workflow](#7-the-hold-and-pay-workflow)
8. [Payment Failure & the Saga Problem](#8-payment-failure--the-saga-problem)
9. [Architecture, Data Model & Scaling](#9-architecture-data-model--scaling)
10. [Senior Follow-Up Questions (with Answers)](#10-senior-follow-up-questions-with-answers)
11. [Quick Glossary](#11-quick-glossary)

---

## 1. How to Approach This (and Why It's Different)

Most large "design X" systems in this series — Twitter, Instagram, Facebook, YouTube — are **read-heavy and AP** (favor availability and eventual consistency; a slightly stale feed is fine). **Ticketmaster inverts this.** Its defining path — booking a seat — is **contention-heavy and consistency-critical (CP-leaning):** you must **never double-book a seat**, so for the booking operation, **correctness beats availability.** It is far better to make a user wait, or to reject a booking, than to sell the same seat twice.

That single fact reframes the whole problem: this is a **distributed concurrency-control problem**, not a scale-out-reads problem. The structure: requirements → recognize that the challenge is *contention*, not throughput → go deep on the **three hard problems** (double-booking, the waiting room, bots) → the hold-and-pay workflow → payment-failure handling. Senior signal: explicitly framing booking as a CP operation (the opposite CAP choice from the feed systems), and treating "no double-booking" as the requirement everything else serves.

---

## 2. Requirements

### Functional

- **Browse** events by city, date, artist.
- **View** seat map and pricing.
- **Reserve (hold)** a seat + **checkout.**
- **Payment** processing.
- **E-ticket delivery.**
- **Resale marketplace** (optional).

### Non-Functional

- **Concurrency hotspot** — popular events draw millions of users in the same minute.
- **No double-booking** — the critical correctness requirement.
- **High availability.**
- **Fair queueing / anti-bot.**

> Note the split personality: **browsing** is read-heavy (cache it, like a normal site), but **booking** is consistency-critical under extreme contention. The design effort goes almost entirely into the booking path.

---

## 3. Capacity: It's About Contention, Not Throughput

```
Hot sale: ~1M concurrent users
          ~100K bookings in the first 5 minutes  → 100K / 300s ≈ ~333 bookings/sec
```

The crucial observation: **~333 bookings/sec is modest throughput** — that's not the hard part. The hard part is that **1M users are contending for a limited, fixed set of seats at the same instant.** The challenges are therefore:

- **Contention** — thousands of users trying to grab the *same* seat simultaneously (the double-booking problem).
- **Thundering herd** — 1M users hitting the system at sale launch would crash it without admission control (the waiting room).
- **Fairness** — without ordering, bots and the fastest connections win; real fans lose.

So unlike the feed systems where the challenge was *scale of reads*, here it's **concurrency, correctness, and fairness** over a scarce resource. Frame it that way and the rest follows.

---

## 4. Critical Challenge 1: Preventing Double-Booking

The single most important problem. When thousands of users target the same seat, exactly one must win and the seat must never be sold twice. This needs **mutual exclusion** on each seat — a distributed lock.

### Approach A — Redis distributed lock (fast)

```
SETNX seat:show:S5  user_X  EX 600       # set-if-not-exists, 10-min TTL
```

`SETNX` is **atomic** — only one user can acquire the lock; everyone else's `SETNX` fails and they're told the seat's taken. The **TTL auto-releases** the lock if the user abandons checkout, so no seat gets stuck held forever.

- *Pros:* extremely fast, atomic, automatic cleanup via TTL — great for the snappy "this seat is yours for 10 minutes" UX.
- *Cons:* Redis is a cache, so on failover a lock can be lost (the well-known Redlock caveat). Acceptable for a *temporary hold* because the database is the final arbiter (below).

### Approach B — Database row lock (durable, authoritative)

```
BEGIN;
SELECT * FROM seats WHERE show_id=X AND seat_id=S5 FOR UPDATE;  -- lock the row
-- verify it's available
UPDATE seats SET status='HELD', user_id=Y WHERE seat_id=S5;
COMMIT;
```

`SELECT ... FOR UPDATE` takes an ACID row lock — the seat's **true, durable state**. Slower and prone to lock contention/deadlocks under extreme load, but it's the **source of truth.**

### The practical hybrid

Use the **Redis lock for the fast hold** (instant UX) and the **DB transaction as the authoritative confirm** at payment time. Even if a Redis race let two users both *think* they hold a seat, the **`FOR UPDATE` at commit resolves it**: one transaction wins, the other is told "sorry, just taken." The DB is the final arbiter; Redis is the fast front line. This is the CP choice in action — when in doubt, the system refuses rather than risks a double-sell.

---

## 5. Critical Challenge 2: The Virtual Waiting Room

When a million users arrive at sale launch, letting them all hit the booking flow at once would **crash the system** and produce a chaotic free-for-all. The fix is a **virtual queue (waiting room):**

1. Arriving users enter a **virtual queue** (not the booking flow).
2. The server **admits N users/sec** into the actual booking flow at a rate the system can handle.
3. Users see their **position and estimated wait.**
4. Everyone else waits their turn.

This does two things at once:
- **Load leveling / admission control** — converts a destructive stampede into a steady, bounded stream the booking system can serve (the queue-as-buffer principle from the message-queue and scaling guides, applied as front-door admission control). It protects the system from collapse.
- **Fairness** — FIFO ordering means first-come-first-served rather than "fastest connection / most bots win." It's the fair counterpart to rate limiting (rate-limiter guide), applied to a scarce-resource launch.

The waiting room is what makes a 1M-user launch survivable *and* fair.

---

## 6. Critical Challenge 3: Bot Mitigation

Scalper bots try to grab inventory faster than humans and resell at markup, which both harms fans and can defeat fairness. Defenses (layered):

- **CAPTCHA** at entry (to the queue/booking flow).
- **Rate limiting** per IP/user (rate-limiter guide).
- **Behavioral analysis** — mouse/interaction patterns to distinguish humans from scripts.
- **Token-based throttling** — signed tokens that gate and pace access.

No single measure suffices (bots adapt), so you stack them. Bot mitigation is a fairness requirement as much as a security one — it ensures the waiting room's FIFO fairness actually benefits real people.

---

## 7. The Hold-and-Pay Workflow

Booking is inherently **two-phase**: you hold the seat, then complete payment. The TTL is what makes this safe.

```
1. User picks a seat → API → SETNX seat:X with a 10-min TTL   (HOLD)
2. User completes payment within the 10 minutes
3. Success → confirm the booking durably in the DB → notify (e-ticket)
4. Timeout (no payment in 10 min) → seat auto-releases (TTL expiry)
```

The elegance is the **TTL-based auto-release**: an abandoned checkout doesn't strand a seat — the hold simply expires and the seat returns to inventory automatically, with no manual cleanup required (a background sweeper can reconcile the DB, but the TTL does the heavy lifting). The hold window balances giving users time to pay against not locking scarce inventory too long.

---

## 8. Payment Failure & the Saga Problem

Booking spans multiple services (seat inventory + payment + confirmation), so it's a **distributed transaction** with no single ACID wrapper — exactly the **Saga problem** from the microservices guide. The cases:

- **Payment fails** → **release the hold** (compensating action) so the seat returns to inventory. Clean.
- **Payment succeeds but the booking DB write fails** → the **critical inconsistency**: you've taken the user's money but not given them the seat. You **must not** lose either. The fix is a **Saga with compensating actions** — either retry the booking write **idempotently** until it succeeds, or **refund** the payment and release the seat. **Idempotency keys** on the payment prevent double-charging during retries (notification/idempotency principle from earlier guides).

The senior point: a two-phase hold-then-pay across services needs Saga-style coordination because you can't atomically commit a seat and a charge together — and the money-taken-but-no-seat case is the one you design carefully around (retry-idempotently or compensate, never silently lose).

---

## 9. Architecture, Data Model & Scaling

### Architecture

```
[Client] ─► [DNS] ─► [GLB] ─► [WAF / Bot Filter]
                                    │
                                    ▼
                          [Virtual Queue Service]   ← admit N/sec
                                    │ (admitted)
                                    ▼
                          [Booking API]
                            ├─► [Seat Inventory Service] ─► [Redis (holds) + DB (truth)]
                            ├─► [Payment Service]
                            ├─► [Notification Service]   (e-tickets/email)
                            └─► [Search Service]         ─► [Elasticsearch]
```

The flow gates traffic through **WAF/bot filter → virtual queue → booking API**, so only admitted, vetted users reach the contention-sensitive inventory.

### Seat inventory

- **Redis cluster** holds hot seat availability and the temporary holds (fast, TTL'd).
- **Database** is the **durable source of truth** for confirmed bookings.
- **Background sync** reconciles Redis and the DB. Redis is the fast front line; the DB is the arbiter.

### Data model

```
events(id, name, venue_id, date, ...)
shows(id, event_id, time, ...)
seats(id, show_id, row, col, type, price, status)
holds(id, seat_id, user_id, expires_at)
bookings(id, user_id, seat_ids[], payment_id, status)
```

### Scaling — isolate contention by event

- **Shard by `event_id`** so each event's traffic is **isolated** — a Taylor Swift sale's stampede doesn't degrade a small local show.
- **Hot events get a dedicated cluster**, so the mega-event's load is contained and the rest of the platform stays healthy.
- **Geo-route** users to the nearest region.

This isolation is the key scaling move: the contention is per-event, so partition by event and give the giants their own capacity.

---

## 10. Senior Follow-Up Questions (with Answers)

**Q1. How is this different from a Twitter/feed design?**
The feed systems are read-heavy and AP (favor availability, eventual consistency). Ticketmaster's booking path is consistency-critical and CP-leaning: never double-book, so correctness beats availability. The hard problem is distributed concurrency control over scarce seats, not scaling reads.

**Q2. How do you prevent double-booking?**
Mutual exclusion per seat. A Redis `SETNX` lock with a TTL gives a fast, atomic hold with auto-release on abandonment; the database `SELECT ... FOR UPDATE` transaction is the authoritative confirm at payment time. The DB is the final arbiter — if a Redis race ever let two users both hold, the DB commit lets exactly one win.

**Q3. Redis lock vs DB row lock — trade-offs?**
Redis: fast, atomic, TTL auto-release — great UX for holds, but a cache that can lose locks on failover. DB row lock: durable, ACID, authoritative — but slower and contention/deadlock-prone under load. Use Redis for the fast hold and the DB transaction for the durable confirm; the DB decides ties.

**Q4. Why a virtual waiting room?**
A million simultaneous users would crash the booking system and create an unfair free-for-all. The queue admits N users/sec (load leveling / admission control) so the system stays up, and imposes FIFO ordering (fairness) so it's first-come rather than fastest/most-bots-win.

**Q5. How is throughput vs contention the key insight?**
~100K bookings in 5 min is only ~333/sec — trivial throughput. The difficulty is contention: ~1M users fighting over a fixed, scarce set of seats at the same instant. So the design centers on concurrency control, admission, and fairness, not raw QPS.

**Q6. How does the hold-and-pay flow avoid stuck seats?**
The hold is a TTL'd lock (e.g., 10 min). If the user abandons or doesn't pay in time, the TTL expires and the seat auto-returns to inventory — no manual cleanup needed. Payment success confirms the booking durably in the DB and triggers the e-ticket.

**Q7. What happens if payment succeeds but the booking write fails?**
That's the critical inconsistency. Handle it as a Saga: either retry the booking write idempotently until it commits, or refund and release the seat — never take money without delivering a seat. Idempotency keys on the payment prevent double-charges during retries.

**Q8. Why is this a Saga problem?**
Booking spans seat inventory, payment, and confirmation across services with no single ACID transaction. So you use a Saga: local steps with compensating actions (release hold / refund) on failure — the cross-service transaction pattern from microservices.

**Q9. How do you scale and isolate hot events?**
Shard by event_id so each event's contention is isolated; give mega-events (Taylor Swift) dedicated clusters so their stampede doesn't degrade other events; geo-route to nearest region. Contention is per-event, so partition by event.

**Q10. How do you keep it fair against bots?**
Layer defenses: CAPTCHA at entry, per-IP/user rate limits, behavioral analysis, and token-based throttling — combined with the FIFO virtual queue so vetted, ordered users get fair access. No single measure is enough; bots adapt, so you stack them.

---

## 11. Quick Glossary

- **Double-booking** — selling the same seat to two users; the critical failure to prevent.
- **Distributed lock** — mutual exclusion across servers (e.g., Redis `SETNX`) to hold a seat.
- **SETNX** — Redis set-if-not-exists; atomic lock acquisition.
- **TTL auto-release** — a hold expiring automatically so abandoned seats return to inventory.
- **SELECT ... FOR UPDATE** — a DB row lock giving authoritative, durable seat state.
- **Virtual waiting room / queue** — admission control that paces and orders incoming users.
- **Admission control** — limiting how many users enter the booking flow per second.
- **Fairness (FIFO)** — first-come ordering, defeating fastest-connection/bot advantage.
- **Bot mitigation** — CAPTCHA, rate limits, behavioral analysis, token throttling.
- **Hold** — a temporary, TTL'd reservation of a seat during checkout.
- **Saga** — coordinating a cross-service transaction with compensating actions on failure.
- **Compensating action** — an undo (release hold / refund) for a failed multi-step booking.
- **Idempotency key** — token preventing duplicate charges/bookings on retry.
- **Shard by event_id** — isolating each event's contention onto its own partition/cluster.

---

*Reference document. Contributions and corrections welcome.*
