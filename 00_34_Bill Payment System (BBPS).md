# Bill Payment System (BBPS) — System Design (HLD, Detailed)

> A worked design of a **bill payment platform** that fetches and pays bills through **BBPS** (Bharat Bill Payment System, NPCI's interoperable bill-payment network) — at **50 million bill-fetches/day**. The interesting part is *not* raw QPS (~600/sec avg) — it's being a **resilient orchestration layer over a slow, unreliable external network** (BBPS → biller systems you don't control), while handling **exactly-once money** on the payment side. This doc builds the fetch path (read, cacheable, resilient), the payment path (write, exactly-once, async status), the scheduled bulk-fetch batch (autopay/reminders — the real source of 50M), and reconciliation.

> 💡 **The whole system in one sentence:** you are a **Customer OU (COU)** in BBPS — you take a customer's biller + account, **fetch** the bill via BBPS (cache it, wrap every outbound call in timeouts + retries + a per-biller circuit breaker because billers are flaky), let the customer **pay** (exactly-once with an idempotency key and an async status state machine, because settlement is T+1 and confirmations are often deferred), and **reconcile** against BBPS clearing files. The design is dominated by *resilience to an external dependency* and *money correctness*, not throughput.

---

## Table of Contents
1. [Problem & requirements](#1-problem--requirements)
2. [BBPS domain primer (who's who)](#2-bbps-domain-primer-whos-who)
3. [Why it's hard](#3-why-its-hard)
4. [The two workloads (on-demand vs scheduled)](#4-the-two-workloads-on-demand-vs-scheduled)
5. [Scale estimation](#5-scale-estimation)
6. [The bill-fetch flow (read path)](#6-the-bill-fetch-flow-read-path)
7. [Caching fetched bills](#7-caching-fetched-bills)
8. [Resilience to BBPS/billers (the crux)](#8-resilience-to-bbpsbillers-the-crux)
9. [The payment flow (write path, exactly-once)](#9-the-payment-flow-write-path-exactly-once)
10. [Async payment status (pending/callback)](#10-async-payment-status-pendingcallback)
11. [Scheduled bulk fetch (autopay & reminders)](#11-scheduled-bulk-fetch-autopay--reminders)
12. [Reconciliation & settlement](#12-reconciliation--settlement)
13. [Biller catalog & routing](#13-biller-catalog--routing)
14. [The architecture](#14-the-architecture)
15. [Data model](#15-data-model)
16. [End-to-end flow](#16-end-to-end-flow)
17. [Failure modes](#17-failure-modes)
18. [Edge cases](#18-edge-cases)
19. [Trade-offs & talking points](#19-trade-offs--talking-points)
20. [How to present this in the interview](#20-how-to-present-this-in-the-interview)
21. [Common mistakes to avoid](#21-common-mistakes-to-avoid)
22. [TL;DR](#22-tldr)

---

## 1. Problem & requirements

Design a bill-payment platform (a BBPS Customer OU, like Paytm/PhonePe) that fetches ~50M bills/day and lets users pay them.

### Functional
- User picks a **biller** (electricity, water, gas, DTH, mobile-postpaid, insurance, loan EMI, FASTag, etc.) and enters an **account identifier** (consumer number).
- System **fetches the bill** (amount, due date, customer name, bill period) via BBPS.
- User **pays** the bill; system confirms and issues a receipt.
- **Autopay** and **reminders**: proactively fetch bills near due date; auto-debit for opted-in users.
- Transaction history, receipts, refunds on failure.

### Non-functional
- **50M fetches/day** — avg ~600/sec, peak a few thousand/sec.
- **Resilient to external failure** — BBPS and biller systems are **slow and unreliable**; one bad biller mustn't degrade the platform.
- **Exactly-once payment** — never double-charge; never lose money.
- **Bounded latency** — fetch should return in seconds or fail gracefully (cache / retry-later).
- **Reconcilable** — every txn matched against BBPS settlement (T+1).
- **Idempotent** — retries/double-clicks are safe.

> 💡 **Reframe away from QPS:** "50M/day is only ~600/sec average — trivially servable. The hard part is that every fetch is a call to an **external system I don't control** (BBPS → biller), which is slow and flaky, and every payment moves **real money**. So this is a *resilience + exactly-once* design, not a throughput one." That reframe is the senior move.

---

## 2. BBPS domain primer (who's who)

BBPS is NPCI's **interoperable** bill-payment network. Knowing the roles makes the design concrete:

| Entity | Role |
|:-------|:-----|
| **BBPCU** (Central Unit) | NPCI's central switch — routing, clearing & settlement between OUs |
| **COU** (Customer OU) | **customer-facing** OU — the app/bank (e.g. Paytm) onboarding users; **this is what we're designing** |
| **BOU** (Biller OU) | **biller-facing** OU — onboards billers, integrates with biller systems |
| **Biller** | the actual utility/company whose bill is paid |
| **Agent** | physical/digital touchpoints under a COU |

```
customer → [ COU (us) ] → BBPCU (central switch) → [ BOU ] → biller system
                ▲                                              │ bill / payment status
                └──────────────── response ────────────────────┘
```

A **bill fetch** or **payment** originates at the COU, routes through BBPCU to the biller's BOU, which talks to the biller. We control the COU; **everything past BBPCU is external and untrusted for reliability.**

> 💡 **Naming the BBPS roles signals domain depth:** "We're the COU. We send fetch/pay requests to BBPCU, which routes to the biller's BOU. The boundary at BBPCU is my *external dependency edge* — beyond it I have no control over latency or uptime, which is exactly why the resilience patterns live at that edge." 

---

## 3. Why it's hard

| Hard thing | Why |
|:-----------|:----|
| **External, unreliable dependency** | BBPS/billers are slow, time out, go down; you don't control them |
| **Per-biller variability** | one biller (say a state electricity board) can be down while others are fine — must isolate |
| **Money correctness** | payment must be exactly-once; double-charge or lost payment is catastrophic |
| **Async settlement** | BBPS settlement is T+1; payment status is often **deferred** (pending → later success/fail) |
| **Reconciliation** | must match every txn against BBPS clearing files; handle mismatches |
| **Dual workload** | on-demand user fetches *and* a large scheduled bulk-fetch (autopay/reminders) |
| **Rate limits** | BBPS/billers throttle you → must shape outbound traffic |
| **Idempotency** | network retries, double-clicks, re-drives — all must be safe |

> 💡 **The crux in one line:** "The design is an exercise in *depending on something unreliable without becoming unreliable yourself* — timeouts, retries, circuit breakers, caching, and async status — plus *moving money exactly once* on top of a T+1, deferred-confirmation network." 

---

## 4. The two workloads (on-demand vs scheduled)

The 50M/day is not all user taps — most of it is **proactive fetching** for autopay/reminders. Two distinct patterns:

| Workload | Trigger | Latency need | Shape |
|:---------|:--------|:-------------|:------|
| **On-demand fetch** | user opens app, enters account | seconds (interactive) | spiky (evenings, near due dates) |
| **Scheduled bulk fetch** | system, near due date, for saved billers | not interactive; hours OK | **batch**, spread over time, rate-limited |

The scheduled batch is where most volume comes from — millions of registered billers fetched proactively so users get reminders and autopay can run. It's a **batch job** (like your payroll/attendance work): fan out fetches, spread them to respect biller rate limits, retry failures, populate the cache (which then serves user on-demand fetches cheaply).

> 💡 **Split the workloads explicitly:** "Most of the 50M is a *scheduled batch* — proactively fetching bills for autopay/reminder users near due dates — not interactive taps. I design them separately: on-demand is low-latency and cache-served; scheduled is a rate-shaped batch that also *warms the cache*, so a user opening the app often hits an already-fetched bill." This is the insight that makes 50M manageable.

---

## 5. Scale estimation

| Quantity | Value | Note |
|:---------|:------|:-----|
| Bill fetches/day | 50M | avg **~600/sec** |
| Peak fetch rate | ~2,000–3,000/sec | evenings / near due dates (×3–5) |
| External BBPS calls | **less than fetches, thanks to cache** | e.g. 50% cache hit → ~25M/day (~290/sec) |
| Payments/day | far fewer (a fraction of fetches; not every fetch → pay) | money path |
| Cache | fetched bills, TTL hours; sharded | serves on-demand from warm scheduled fetches |

The headline: **caching + scheduled pre-fetch shrink the load on the external network.** (Verified: 50% cache hit → 25M external calls/day (~290/sec); 70% → 15M.) The scheduled batch pre-fetches and warms the cache, so interactive fetches often avoid the slow external hop entirely.

> 💡 **The estimation tell:** "600/sec average is nothing; the real number I care about is *external calls/sec to BBPS*, which caching and pre-fetch cut substantially — and I shape it to stay under BBPS/biller rate limits. I'm not sizing for QPS, I'm sizing for how gently I can hit a fragile external network." 

---

## 6. The bill-fetch flow (read path)

Fetching a bill is a **read** — cacheable and freely retryable:

```
1. user submits (biller, accountId)
2. check CACHE for (biller, accountId) → if fresh, return it (no external call)
3. else: route (biller → BOU) and call BBPS Bill-Fetch  (with timeout)
4. BBPCU → BOU → biller → bill (amount, dueDate, name, period)
5. store in cache (TTL) + return to user
6. on timeout/error: retry (bounded) / circuit-breaker fast-fail / show cached-or-"try later"
```

The fetch is **idempotent and safe to retry** (it's a read), which is what lets you be aggressive with retries and caching here — very different from the payment path.

> 💡 **Fetch is a read — exploit that.** "Because fetching a bill has no side effects, I cache it and retry freely. That's the opposite of the payment path, where retries are dangerous. Separating the safe read path from the dangerous write path is the first design decision." 

---

## 7. Caching fetched bills

Bills change on a **monthly cycle**, so a freshly-fetched bill is valid for a while — cache it:

- **Key:** `(biller, accountId)`; **value:** the bill; **TTL:** hours (tunable per biller category).
- **Warmed by the scheduled batch** — proactive fetches populate the cache; user taps then hit it.
- **Invalidate on payment** — once paid, the cached bill's status is stale → invalidate/refresh so the user doesn't see "due" after paying.
- **Careful with staleness** — amount can change (partial payment, late fee); short-ish TTL + invalidate-on-pay keeps it honest. Show "fetched at HH:MM" if needed.

(The cache is why external calls < fetches — verified 25–35M external for 50M fetches at 30–50% hit.)

> 💡 **Cache with invalidate-on-pay:** "I cache bills with an hours TTL keyed by (biller, account), warmed by the scheduled pre-fetch — but I invalidate on payment so a paid bill doesn't still show as due. The TTL trades external-call savings against staleness; short enough that amount changes surface." 

---

## 8. Resilience to BBPS/billers (the crux)

Everything past BBPCU is unreliable. The resilience toolkit at the external edge:

- **Timeouts** — every outbound call has a strict timeout; never hang waiting on a slow biller.
- **Bounded retries with backoff + jitter** — for the *idempotent fetch*, retry a couple of times; jitter avoids synchronized retry storms.
- **Circuit breaker per biller/BOU** — if a biller's failure rate crosses a threshold, **open the breaker** and fast-fail (serve cache or "try later") for a cooldown, then **half-open** to probe recovery. This stops one dead biller from consuming all threads/queues and cascading. (Verified: 3 failures → OPEN → fast-fail during cooldown → HALF_OPEN probe → CLOSED on success.)
- **Bulkheads** — isolate resources per biller/category (separate thread pools/queues) so a slow biller can't starve others.
- **Outbound rate limiting** — throttle per biller/BOU to stay under their limits (token bucket, per your rate-limiter doc).
- **Graceful degradation** — on failure, return cached bill or a clear "couldn't fetch, try later," never a hang or a crash.

```
per-biller breaker:  CLOSED ──(failures ≥ threshold)──► OPEN ──(cooldown)──► HALF_OPEN
                        ▲                                                        │
                        └──────────────── probe succeeds ────────────────────────┘
                                          probe fails → OPEN
```

> 💡 **The circuit breaker is the headline resilience answer:** "I wrap each biller/BOU in a circuit breaker — if a biller starts failing, I open the breaker and fast-fail to cache or 'try later' instead of piling requests onto a dead dependency, then half-open to probe recovery. Combined with per-biller bulkheads, one broken biller can't take down bill-pay for everyone." Say breaker + bulkhead together.

---

## 9. The payment flow (write path, exactly-once)

Payment moves money — the **exactly-once** discipline (your payroll/cab pattern):

```
1. user confirms payment for a fetched bill
2. create Transaction (INITIATED) with an IDEMPOTENCY KEY (client-generated req id)
3. debit the customer's source (wallet/UPI/card) — with the idem key
4. send BBPS Pay request (idem key / unique ref) → BBPCU → BOU → biller
5. record biller/BBPS reference (RRN); status may be SUCCESS or PENDING (§10)
6. on SUCCESS: mark PAID, receipt, invalidate cached bill
   on FAILURE: mark FAILED, auto-refund the debit
```

- **Idempotency key** on the whole payment so a retry/double-click doesn't double-pay (verified: retry with same key → DUPLICATE, ledger has one entry).
- **Debit + biller-pay must be consistent** — you debited the customer *and* must pay the biller; handle "debited but biller-pay failed" by **auto-refund** (a saga: debit → pay → on failure compensate).
- **Reference tracking** — store BBPS RRN for reconciliation and customer support.

> 💡 **Exactly-once = idempotency key + saga + refund-on-failure:** "The payment carries an idempotency key so retries can't double-charge; debit and biller-payment form a saga — if the biller-pay fails after I debited, I compensate with an auto-refund. And I persist the BBPS reference for reconciliation. Never double-charge, never leave the customer debited without a paid bill." 

---

## 10. Async payment status (pending/callback)

BBPS payment confirmation is often **not immediate** — the biller may confirm later, and settlement is **T+1**. So payment status is a **state machine that can sit in PENDING**:

```
INITIATED ─► PENDING ─► SUCCESS
     │          │
     └──────────┴──► FAILED  (→ auto-refund)
```

(Verified FSM: INITIATED→PENDING→SUCCESS/FAILED; terminals sealed.)

Resolving PENDING:
- **Callbacks** — BBPS/BOU pushes a status update webhook → update the txn.
- **Polling / status-check** — a background job polls BBPS for pending txns (with backoff) until resolved or a deadline.
- **Deadline + reconciliation** — if still unresolved by a cutoff, resolve via the settlement file (§12); refund if not settled.

> 💡 **PENDING is a first-class state, not an error:** "Payment status is frequently deferred, so PENDING is expected — I resolve it via BBPS callbacks or a polling job, and anything still ambiguous is settled by the T+1 reconciliation file (refund if it didn't go through). The user sees 'processing' until it resolves." Handling deferred confirmation is a key BBPS-specific point.

---

## 11. Scheduled bulk fetch (autopay & reminders)

The bulk of the 50M: proactively fetch bills for users who saved billers (for reminders) or opted into autopay. This is a **batch/scheduled system** (your payroll/attendance patterns):

- A **scheduler** enqueues fetch tasks for saved billers near their due dates.
- **Fan out** fetch tasks to workers; each fetch is idempotent and cache-populating.
- **Rate-shape** per biller/BOU (respect limits; don't hammer one biller) — spread the batch over hours, not a spike.
- **Retry failures** (with breaker awareness — skip billers whose breaker is open, retry later).
- **On fetch:** update reminder state; for **autopay**, if a bill is due and within the user's limit, trigger the payment flow (§9) automatically (with the same exactly-once guarantees).

> 💡 **The batch warms the cache and drives autopay:** "The scheduled bulk fetch is a rate-shaped batch that pre-fetches bills near due dates — it powers reminders, triggers autopay, and warms the cache so interactive fetches are cheap. It's the same idempotent-fan-out + rate-limit pattern as any large batch, spread over time to be gentle on billers." 

---

## 12. Reconciliation & settlement

BBPS settles **T+1** via clearing files; you must **reconcile** every transaction:

- **Ingest the daily settlement/clearing file** from BBPS.
- **Match** each of your txns (by RRN/reference) against the file: settled? amount correct?
- **Resolve mismatches** — a txn you marked SUCCESS but not in the file (investigate/refund); a PENDING now settled (mark SUCCESS); a debit with no successful payment (refund).
- **Immutable ledger** — payments are append-only, auditable (double-entry for money movement); the reconciliation output drives refunds/adjustments.

> 💡 **Reconciliation is the money-truth step:** "I don't trust my own 'SUCCESS' — I reconcile every txn against BBPS's T+1 settlement file, which is the source of truth for what actually settled. Mismatches drive refunds/adjustments. Same 'verify against the external ledger' discipline as payroll disbursement." 

---

## 13. Biller catalog & routing

- **Biller catalog** — a registry of billers (id, name, category, required fields, the BOU that serves them, fetch/pay capabilities). Synced from BBPS; cached (read-heavy, rarely changing).
- **Routing** — map `biller → BOU/category` to route fetch/pay and to attach the right circuit breaker / rate limiter.
- **Per-biller config** — validation rules for the account identifier (formats differ by biller), fetch support (some billers are pay-only), category TTLs.

> 💡 **The catalog drives validation & routing:** "A cached biller catalog tells me each biller's required fields, its BOU, and whether it supports fetch — so I validate the consumer number up front, route to the right BOU, and apply per-biller breakers/limits. It's the control-plane the fetch/pay data-plane reads." 

---

## 14. The architecture

```
   user app ──► API Gateway (auth, rate-limit) ──► Bill-Pay Service
                                                        │
        ┌───────────────────────────────────────────────┼───────────────────────────┐
        │ FETCH (read)                                    │ PAY (write, exactly-once)  │
        ▼                                                 ▼                            │
   ┌──────────────┐  hit   ┌───────────────┐      ┌──────────────────┐                │
   │ Bill Cache    │◄──────│ Fetch Service  │      │ Payment Service   │  idem key      │
   │ (biller,acct) │       │ timeout+retry  │      │ debit→pay saga    │──────┐         │
   └──────────────┘  miss  │ +CIRCUIT BREAKER│      │ status FSM        │      ▼         │
                           │  per biller     │      └────────┬─────────┘  ┌──────────┐  │
                           └───────┬────────┘               │            │ Wallet/UPI│  │
                                   │ route (biller→BOU)      │            │ /Card     │  │
        ┌──────────────┐          ▼                          ▼            └──────────┘  │
        │ Biller Catalog│   ┌──────────────────────────────────────────┐               │
        │ (routing/cfg) │   │        BBPS Connector (outbound)           │               │
        └──────────────┘   │  timeouts, retries, rate-limit per BOU     │               │
                           └───────────────────┬───────────────────────┘               │
                                               ▼                                         │
                                        BBPCU → BOU → biller  (EXTERNAL)                 │
   ┌───────────────────────────────────────────────────────────────────────────────────┘
   ▼
   Scheduler ──► Bulk-Fetch Queue ──► Fetch Workers (rate-shaped) ──► warms cache + autopay/reminders
   Reconciliation Service ◄── daily BBPS settlement file ──► ledger, refunds, mismatch handling
   + Notification Service (reminders, receipts)   + Callback handler (async payment status)
```

Components: **Bill-Pay Service** (orchestrator), **Fetch Service + Cache**, **Payment Service** (saga + FSM), **BBPS Connector** (the single hardened outbound edge with breakers/limits), **Biller Catalog**, **Scheduler + Bulk-Fetch workers**, **Reconciliation**, **Callback handler**, **Notifications**.

---

## 15. Data model

| Entity | Store | Notes |
|:-------|:------|:------|
| **BillerCatalog** | cache/DB | biller id, category, required fields, BOU, capabilities |
| **CachedBill** | cache (Redis) | (biller, accountId) → bill; TTL; warmed by batch; invalidated on pay |
| **Transaction** | durable DB | idem key, biller, account, amount, **status FSM**, BBPS RRN — source of truth |
| **Debit/Refund (ledger)** | durable DB | append-only money movement; double-entry; reconciliation-driven |
| **SavedBiller / Autopay** | DB | user's saved billers, autopay opt-in + limits, reminder prefs |
| **SettlementRecord** | DB | ingested BBPS clearing rows for reconciliation |

Bills are ephemeral/cacheable; transactions & ledger are durable/append-only/auditable — the key split (fetch vs pay).

---

## 16. End-to-end flow

**On-demand fetch + pay:**
1. User picks biller + enters account → validate via catalog.
2. Fetch: cache hit → return; miss → BBPS Connector (timeout/retry/breaker) → BBPCU→BOU→biller → cache + return.
3. User pays → Transaction (INITIATED, idem key) → debit → BBPS pay → record RRN → SUCCESS/PENDING.
4. PENDING → resolved by callback/poll; SUCCESS → receipt, invalidate cached bill; FAILURE → auto-refund.
5. Next day → reconcile against settlement file.

**Scheduled (autopay/reminder):**
1. Scheduler enqueues fetches for saved billers near due date.
2. Rate-shaped workers fetch (breaker-aware) → warm cache → reminders; autopay-eligible & within limit → trigger pay flow.

---

## 17. Failure modes

| Failure | Effect | Handling |
|:--------|:-------|:---------|
| Biller/BOU down | fetch/pay fails for that biller | circuit breaker fast-fails; serve cache / "try later"; bulkhead isolates it (verified) |
| BBPS timeout on fetch | slow/hanging | strict timeout + bounded retry + jitter; degrade to cache |
| BBPS timeout on **pay** (ambiguous) | did it go through? | do **not** blindly retry-charge; status-check by idem key/RRN; reconcile before refund |
| Debited but biller-pay failed | customer out money | saga compensation → **auto-refund** |
| Payment stuck PENDING | user waiting | poll/callback; deadline → reconcile → refund if unsettled |
| Duplicate submit / double-click | double-charge risk | idempotency key (verified) |
| Reconciliation mismatch | our status ≠ BBPS | investigate; refund unsettled; correct status |
| Bulk-fetch spike hammers a biller | biller rate-limits/blocks us | outbound rate-limit + spread the batch |
| Cache serving stale paid bill | shows due after pay | invalidate-on-pay |

> 💡 **The ambiguous-pay-timeout is the killer follow-up:** "If a *payment* times out I never blindly retry the charge — I status-check by the idempotency key / RRN to learn if it actually went through, and let reconciliation be the final arbiter before any refund. Retrying a fetch is free; retrying a charge can double-pay." Have this ready.

---

## 18. Edge cases

| Case | Handling |
|:-----|:---------|
| Biller supports pay but not fetch | catalog flag; skip fetch, take amount from user (with validation) |
| Bill already paid elsewhere | fetch shows nil/paid; block double payment |
| Partial payment / amount changed | short TTL + re-fetch before pay; validate amount at pay time |
| Invalid consumer number | catalog validation rules up front |
| Autopay amount > user limit | skip auto-debit; send reminder instead |
| Late fee added after fetch | re-fetch/confirm amount at payment |
| BBPS reference collision/retry | idempotency key dedups |
| Refund of a settled txn | refund flow post-settlement (T+1 aware) |

> 💡 **Validate the amount at pay-time, not just fetch-time:** "Because bills can change (late fees, partial pay) between fetch and pay, I re-confirm the amount at payment for anything past a short freshness window — so the user pays the current due, not a stale cached figure." 

---

## 19. Trade-offs & talking points

- **Cache freshness vs external load** — longer TTL cuts BBPS calls but risks stale amounts; invalidate-on-pay + short TTL.
- **Retry aggressiveness: fetch vs pay** — retry fetch freely (read); never blindly retry a charge (write).
- **Sync vs async payment** — expose PENDING; resolve via callback/poll/reconcile.
- **On-demand vs scheduled** — interactive low-latency vs rate-shaped batch that warms cache.
- **Consistency** — money path strongly consistent + idempotent; bill display eventually consistent.
- **Isolation** — per-biller breakers/bulkheads so one biller's outage is contained.

---

## 20. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Reframe | not QPS — resilience over an unreliable external network + exactly-once money |
| BBPS roles | COU (us) → BBPCU → BOU → biller; the external edge |
| Two workloads | on-demand fetch vs scheduled bulk fetch (source of 50M) |
| Fetch path | cache + timeout/retry + **circuit breaker per biller** + bulkhead |
| Pay path | idempotency key + debit/pay **saga** + refund-on-failure |
| Async status | PENDING FSM; callback/poll; reconcile |
| Reconciliation | T+1 settlement file = money truth |

### What to say
- *"50M/day is ~600/sec — the challenge is depending on BBPS/billers, which are slow and flaky, without becoming unreliable myself."*
- *"Fetch is a cacheable read I retry freely; payment is a write I make exactly-once with an idempotency key."*
- *"Each biller gets a circuit breaker and a bulkhead, so one dead biller fast-fails to cache instead of cascading."*
- *"Payment status can be PENDING — I resolve it via callback/poll and settle ambiguity with the T+1 reconciliation file; on an ambiguous pay-timeout I status-check, never blind-retry the charge."*
- *"Most of the 50M is a rate-shaped scheduled batch that pre-fetches bills, warms the cache, and drives autopay/reminders."*

### Order
reframe → BBPS roles → two workloads → fetch (cache+breaker) → pay (idempotency+saga) → async status → reconciliation → failures.

---

## 21. Common mistakes to avoid

- ❌ **Treating it as a QPS problem** — it's resilience + exactly-once, not throughput.
- ❌ **No circuit breaker per biller** — one dead biller cascades and takes down bill-pay.
- ❌ **Retrying a payment on timeout** — can double-charge; status-check by idem key, reconcile.
- ❌ **No idempotency key on pay** — double-clicks/retries double-charge.
- ❌ **Ignoring PENDING** — BBPS confirmations are deferred; PENDING is a first-class state.
- ❌ **No reconciliation** — you can't trust your own SUCCESS; match the T+1 settlement file.
- ❌ **Caching without invalidate-on-pay** — shows a paid bill as still due.
- ❌ **Hammering billers with the bulk fetch** — rate-shape and spread; respect biller limits.
- ❌ **No saga/refund for debited-but-not-paid** — leaves the customer out of pocket.
- ❌ **One shared thread pool for all billers** — a slow biller starves others; use bulkheads.

---

## 22. TL;DR

### The design
```
We are a BBPS COU: user → COU → BBPCU → BOU → biller (everything past BBPCU is EXTERNAL/unreliable).
FETCH (read): cache (biller,acct, TTL, warmed by batch, invalidate-on-pay) → else BBPS call with
              TIMEOUT + bounded RETRY + CIRCUIT BREAKER per biller + BULKHEAD + outbound RATE-LIMIT.
PAY (write):  Transaction(idempotency key) → debit→pay SAGA (refund on failure) → status FSM
              INITIATED→PENDING→SUCCESS/FAILED (callbacks/poll) → RECONCILE vs T+1 settlement file.
SCHEDULED:    rate-shaped bulk pre-fetch for autopay/reminders → warms cache, drives autopay.
```

### The mechanics (verified)
```
Capacity   : 50M/day ≈ 600/sec avg; cache 50% → ~25M external/day (~290/sec)
Breaker    : CLOSED →(failures)→ OPEN (fast-fail) →(cooldown)→ HALF_OPEN (probe) → CLOSED
Payment    : idempotency key → retry = DUPLICATE, no double-charge
Status FSM : INITIATED→PENDING→SUCCESS/FAILED; PENDING resolved by callback/poll/reconcile
```

### The four things that score points
1. **Reframe:** resilience over an unreliable external network + exactly-once money, not QPS.
2. **Fetch = cacheable read** with **per-biller circuit breaker + bulkhead**; **pay = exactly-once write** (idempotency + saga + refund).
3. **Async/PENDING status** resolved by callback/poll and the **T+1 reconciliation** (money truth); never blind-retry a charge.
4. **Two workloads** — on-demand fetch vs rate-shaped scheduled bulk-fetch that warms the cache and drives autopay.

> **One-line philosophy:** *A BBPS bill-payment platform is a resilient orchestration layer over a slow, unreliable external network — cache and freely-retry the read (bill fetch) behind per-biller circuit breakers and bulkheads so one broken biller can't cascade, move money exactly once on the write (payment) with an idempotency key, a debit→pay saga, and refund-on-failure, treat deferred PENDING status as first-class (resolved by callbacks, polling, and the T+1 settlement reconciliation that is the real money truth), and serve most of the 50M/day from a rate-shaped scheduled pre-fetch that warms the cache and powers autopay — so you depend on something unreliable without ever becoming unreliable, or wrong about money, yourself.*
