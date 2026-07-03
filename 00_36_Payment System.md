# Payment System / Digital Wallet — System Design (HLD, Detailed)

> A worked design of a **digital wallet & payment system** (Paytm/PhonePe/PayPal shape): users hold balances, add money, pay merchants and each other, and withdraw — all while the system **never loses, duplicates, or invents money**. The foundation is a **double-entry ledger** (every movement is two balanced entries, so money is conserved and auditable) plus **exactly-once** transactions (idempotency + atomic balance updates + reconciliation). This doc builds the ledger, the money-movement flows (add/pay/withdraw), reserve/capture holds, concurrency, external-rail integration via saga, and reconciliation.

> 💡 **The whole system in one sentence:** model money as a **double-entry ledger** where every transaction writes two entries that sum to zero (so total money is always conserved and every balance is provable from history), make each transaction **idempotent** (an id so retries can't double-charge) and **atomic per account** (so concurrent debits can't overdraw), keep balances **never negative**, integrate external rails (banks/cards/UPI) via a **saga with compensating refunds**, and **reconcile** against those rails because they're the ultimate source of money truth. Correctness is everything; a payment system that's occasionally wrong is worthless.

---

## Table of Contents
1. [Problem & requirements](#1-problem--requirements)
2. [Why it's hard (money correctness)](#2-why-its-hard-money-correctness)
3. [The double-entry ledger (the foundation)](#3-the-double-entry-ledger-the-foundation)
4. [Accounts & the balance invariant](#4-accounts--the-balance-invariant)
5. [Idempotency & exactly-once](#5-idempotency--exactly-once)
6. [Concurrency (atomic balance updates)](#6-concurrency-atomic-balance-updates)
7. [The money-movement flows](#7-the-money-movement-flows)
8. [Reserve → capture (holds / auth)](#8-reserve--capture-holds--auth)
9. [External rails & the saga](#9-external-rails--the-saga)
10. [Reconciliation & settlement](#10-reconciliation--settlement)
11. [The transaction lifecycle (state machine)](#11-the-transaction-lifecycle-state-machine)
12. [The architecture](#12-the-architecture)
13. [Data model](#13-data-model)
14. [Scale estimation](#14-scale-estimation)
15. [Consistency, availability & storage choice](#15-consistency-availability--storage-choice)
16. [Security, compliance & fraud](#16-security-compliance--fraud)
17. [Failure modes](#17-failure-modes)
18. [Edge cases](#18-edge-cases)
19. [Trade-offs & talking points](#19-trade-offs--talking-points)
20. [How to present this in the interview](#20-how-to-present-this-in-the-interview)
21. [Common mistakes to avoid](#21-common-mistakes-to-avoid)
22. [TL;DR](#22-tldr)

---

## 1. Problem & requirements

Design a digital wallet + payment system where users hold balances and move money.

### Functional
- **Add money** (top-up from bank/card/UPI into the wallet).
- **Pay** — user → merchant, and user → user (P2P transfer).
- **Withdraw** to a bank account.
- **Balance & transaction history** (statement).
- **Refunds / reversals**.
- (Often) holds/auth (reserve then capture), scheduled/recurring payments.

### Non-functional (these dominate — it's money)
- **Correctness above all** — never double-charge, lose, or create money; balances always accurate.
- **Exactly-once** — retries, double-clicks, network re-drives must not move money twice.
- **Consistency** — a balance read must reflect completed transactions; **no overdraft**.
- **Auditability** — every paisa traceable; immutable history; regulatory reporting.
- **Availability** — payments should work; but **correctness beats availability** when they conflict.
- **Durability** — a committed transaction is never lost.

> 💡 **Lead by inverting the usual priorities:** "For most systems I'd optimize latency/throughput; for a wallet, **correctness and auditability come first, and I'll trade availability for consistency when they conflict** — a payment system that's fast but occasionally wrong about money is worthless." Stating that reframes the whole design around a ledger, not a CRUD balance column.

---

## 2. Why it's hard (money correctness)

| Hard thing | Why |
|:-----------|:----|
| **No lost/duplicate/invented money** | a bug here is fraud/loss, not a glitch; must be provably correct |
| **Exactly-once over unreliable networks** | retries and timeouts are constant; naive retry double-pays |
| **Concurrency on a balance** | two debits on one account can overdraw if not serialized |
| **External rails are async & fallible** | bank/card/UPI can time out, settle T+1, or partially fail |
| **Auditability & compliance** | must reconstruct any balance from history; regulators require it |
| **Ambiguous failures** | "did the debit go through?" — you can't just retry a charge |

> 💡 **The crux:** "It's an exercise in *moving money exactly once across systems that fail*, while keeping a provably-correct, auditable record. The two tools are a double-entry ledger (correctness + audit) and idempotency + reconciliation (exactly-once)." That's the senior framing.

---

## 3. The double-entry ledger (the foundation)

Don't store a wallet as a single mutable `balance` number you increment/decrement — that's untraceable and easy to corrupt. Instead use the **accountant's model, 500 years old and correct: double-entry.**

Every money movement is a **transaction of two (or more) entries that sum to zero** — a **debit** from one account and a **credit** to another:

```
Alice pays merchant ₹300:
   entry 1:  Alice     −300   (debit)
   entry 2:  Merchant  +300   (credit)
   ─────────────────────────
   sum        0                ← ALWAYS balances
```

Properties this buys you (all verified):
- **Money is conserved** — every transaction nets to zero, so the sum of all balances never changes except when money genuinely enters (top-up) or leaves (withdrawal) the system through designated external accounts.
- **Balances are derivable** — a balance is the sum of an account's entries; you can *reconstruct* it from history, which is what makes the system auditable and self-checking.
- **The ledger is append-only & immutable** — you never edit an entry; a correction is a *new* reversing transaction. History is truth.

(Verified: each transaction's two legs net to zero; system-wide Σbalances stayed constant at 1000; a paid balance is provable from entries.)

> 💡 **"Every movement is two balanced entries" is the headline.** "I model money as a double-entry ledger — each transaction is a debit and a credit that sum to zero, appended immutably. That guarantees money is conserved, every balance is provable from history, and corrections are reversing entries, not edits. It's why the system is auditable and can't silently lose money." Say this first; it's what separates a real payment design from a `balance += amount` toy.

---

## 4. Accounts & the balance invariant

- **Accounts** aren't only users — there are also **system accounts**: a bank/settlement account (money entering via top-up), a merchant payout account, a fees/revenue account, an escrow/holding account. Money entering the ecosystem is a transfer from an external/bank account *into* a user's wallet; money leaving is the reverse. This keeps *everything* balanced, even top-ups and withdrawals.
- **Balance = current available funds** (materialized for fast reads, but reconcilable to the entry sum).
- **The invariant: a wallet balance never goes negative** (no overdraft) — enforced atomically at debit time (verified: overdraft blocked).

> 💡 **Model external money as accounts too:** "Top-up isn't 'balance += X' out of nowhere — it's a transfer from the bank/settlement account into the wallet, so even money entering the system is a balanced double-entry. That's what keeps Σbalances a meaningful, conserved quantity." A subtle point that shows you actually understand the ledger.

---

## 5. Idempotency & exactly-once

Money movement **must be exactly-once**, but networks give you at-least-once (retries, timeouts, double-clicks). Bridge the gap with an **idempotency key**:

```
transfer(idempotencyKey, src, dst, amount):
   if key already applied:  return the ORIGINAL result   # replay → no-op, no double-move
   else: apply the transaction (atomically), record the key, return result
```

- The client generates a stable key per intended payment (a request id); a retry carries the **same** key.
- The ledger records applied keys; a duplicate is a **no-op returning the first result** (verified: replay of the same txn id didn't double-charge).
- This turns "did my payment go through? let me retry" into a **safe** operation.

> 💡 **Idempotency is what makes retries safe:** "Every money operation carries an idempotency key; if I see it again I return the original outcome instead of re-applying it. So a client can safely retry an ambiguous payment — the key guarantees the money moves at most once." This is *the* exactly-once mechanism; pair it with reconciliation (§10) for external rails.

---

## 6. Concurrency (atomic balance updates)

Two debits racing on one account can both read the same balance, both "succeed," and **overdraw** (verified: non-atomic → both withdrawals of 100 from a balance of 100 allowed). The debit is a **read-modify-write**, so it must be atomic:

- **Per-account locking / serialization** — take a lock on the account (or use a DB row lock / `SELECT … FOR UPDATE` / conditional update) so concurrent debits serialize; the second sees the reduced balance and fails (verified: atomic → exactly one wins).
- **Conditional update** — `UPDATE … SET balance = balance - amt WHERE balance >= amt` in one atomic statement is a clean, lock-light way to enforce no-overdraft.
- **Lock the account, not the whole ledger** — different accounts proceed in parallel; only same-account operations serialize.
- **A transaction spanning two accounts** (transfer) must update both atomically — a DB transaction, or a saga if they're in different systems (§9).

> 💡 **The debit is a read-modify-write → must be atomic:** "Concurrent debits on one balance can overdraw, so I serialize per account — a row lock or a conditional `WHERE balance >= amount` update — never a read-then-write in app code. Per-account, not global, so unrelated payments stay parallel." Same atomicity discipline as your rate-limiter and payroll docs.

---

## 7. The money-movement flows

All three flows are just double-entry transactions (with the right accounts):

**Add money (top-up):** external card/bank/UPI → wallet.
```
1. user initiates top-up (idempotency key)
2. charge the external rail (bank/card/UPI)  ← async, may be PENDING
3. on success: ledger transfer  Bank/Settlement −X , Wallet +X
4. on failure: no ledger entry (money never entered)
```

**Pay (user → merchant, or P2P):** wallet → wallet.
```
1. user pays (idempotency key), amount ≤ balance
2. atomic double-entry:  Payer −X , Payee +X   (+ optional  Payer −fee , Fees +fee)
3. balances updated in one transaction; history appended
```

**Withdraw:** wallet → external bank.
```
1. user requests withdrawal (idempotency key), amount ≤ balance
2. reserve/debit wallet:  Wallet −X , PayoutEscrow +X
3. initiate bank payout (external, async)  ← saga
4. on payout success: PayoutEscrow −X , Bank/Settlement −? (settle); mark COMPLETED
   on payout failure:  refund  PayoutEscrow −X , Wallet +X  (compensate)
```

Each is idempotent and atomic; external legs (top-up charge, payout) are async and go through the saga (§9) + reconciliation (§10).

> 💡 **All flows reduce to double-entry + rails:** "Pay is a pure internal double-entry; top-up and withdraw add an *external* leg (bank/card) that's async and fallible, so those wrap the ledger move in a saga with compensation. Fees are just extra entries. One ledger model covers everything." 

---

## 8. Reserve → capture (holds / auth)

Some flows need a **hold** before the final charge (card auth, withdrawal in flight, order not yet fulfilled): move funds from **available** to **held**, then **capture** (finalize) or **release** (cancel/refund) later.

```
reserve(X):  available −X , held +X     # funds locked, not yet spent
capture(X):  held −X                     # finalize the spend (to payee)
release(X):  held −X , available +X      # cancel → funds return
```

(Verified: reserve 200 → available 300 / held 200; capture → held 0.) This is the **reserve/commit/release** pattern from your OMS/leave LLDs, applied to money — it prevents double-spend of the same funds while a payment is in flight.

> 💡 **Reserve/capture prevents in-flight double-spend:** "For auth-then-settle flows I move funds to a *held* bucket on reserve so they can't be spent twice while the payment is pending, then capture or release. Same reserve/commit/release discipline as inventory/leave, applied to a balance." 

---

## 9. External rails & the saga

Top-ups and withdrawals touch **external systems** (banks, card networks, UPI) that are async, slow, and fallible — and you often must **debit your ledger AND move external money**, which can't be one ACID transaction across systems. Use a **saga** (a sequence of local steps, each with a compensating action):

```
Withdrawal saga:
   1. debit wallet (Wallet −X → Escrow +X)          [local, atomic]
   2. call bank payout (external)                    [async]
      success → capture (Escrow −X); COMPLETED
      failure → COMPENSATE: refund (Escrow −X → Wallet +X)   ← keep money consistent
```

(Verified: pay-fails → saga produces DEBIT then REFUND, leaving the user whole.) The **ambiguous case** (bank call times out — did it go through?) is the dangerous one: **never blindly retry the payout** — status-check by the idempotency key/reference, and let **reconciliation** (§10) be the final arbiter before refunding.

> 💡 **Saga + compensation + don't-blind-retry:** "External money moves via a saga: local ledger debit, then the external call; if the external leg fails I compensate with a refund so the ledger stays consistent. On an ambiguous timeout I status-check rather than retry the charge, and reconciliation settles anything still unclear." This is the exactly-once-over-external-rails answer (same as your bill-pay/payroll docs).

---

## 10. Reconciliation & settlement

You **cannot fully trust your own record** of external money — the bank/card network is the ultimate truth. **Reconcile** continuously:

- **Ingest settlement/statement files** from banks/card networks/UPI (often daily / T+1).
- **Match** each of your transactions (by reference/idempotency key) against the external record: did it settle? correct amount?
- **Resolve mismatches** — you marked success but it's not in the file (investigate/reverse); a PENDING now settled (mark success); money left but no matching ledger entry (a **suspense/exceptions** account holds it until resolved).
- **The ledger is your book; reconciliation aligns it with the bank's book.** Discrepancies drive reversing entries — never silent edits.

> 💡 **Reconciliation is the money-truth step:** "I don't trust my own 'SUCCESS' for external money — I reconcile against the bank/network settlement files, and unmatched items sit in a suspense account until resolved with reversing entries. The ledger plus reconciliation is what makes the system provably correct end-to-end." 

---

## 11. The transaction lifecycle (state machine)

A transaction (especially one with an external leg) has explicit, persisted states:

```
INITIATED ─► PENDING ─► COMPLETED
     │          │
     └──────────┴──► FAILED  ─► (compensation: REVERSED/REFUNDED)
```

| State | Meaning |
|:------|:--------|
| INITIATED | created (idempotency key assigned) |
| PENDING | external leg in flight (top-up charge / payout) |
| COMPLETED | ledger committed + external settled/reconciled |
| FAILED → REVERSED | failed; compensating entries applied |

Purely-internal transfers (wallet→wallet) can go INITIATED→COMPLETED atomically; external ones sit in PENDING until the rail confirms (callback/poll) and reconciliation clears them. Terminal states are sealed; corrections are new reversing transactions.

> 💡 **PENDING is first-class for external money:** "Internal transfers commit atomically; anything touching a bank sits in PENDING until confirmed by callback/poll and cleared by reconciliation. The lifecycle is a guarded state machine so a late/duplicate callback can't corrupt a settled transaction." 

---

## 12. The architecture

```
   client/app ──► API Gateway (auth, rate-limit, idempotency-key) ──► Payment/Wallet Service
                                                                          │
        ┌──────────────────────────────────────────────────────────────┼──────────────────┐
        │ INTERNAL (wallet↔wallet): atomic double-entry                   │ EXTERNAL leg      │
        ▼                                                                 ▼                   │
   ┌──────────────────────┐   append + balance update (one txn)    ┌──────────────────────┐  │
   │  LEDGER (double-entry) │◄──────────────────────────────────────│ Transaction Orchestr. │  │
   │  entries (append-only) │   per-account atomic / conditional     │ (saga, status FSM,    │  │
   │  balances (materialized)│  update; idempotency keys             │  compensation)        │  │
   └──────────────────────┘                                        └──────────┬───────────┘  │
        ▲ derivable / auditable                                                │ call rail     │
        │                                                                       ▼               │
   ┌──────────────────────┐   T+1 settlement files    ┌──────────────────────────────────┐    │
   │ Reconciliation Service│◄──────────────────────────│ Rail Connectors (bank/card/UPI)   │    │
   │ match, suspense, revers│                           │ idempotent, retries, callbacks    │    │
   └──────────────────────┘                           └──────────────────────────────────┘    │
   + Fraud/Risk (pre-txn checks) + Notification (receipts) + Statement/History service ◄───────┘
```

Components: **Payment/Wallet Service** (entry point, idempotency), **Ledger** (double-entry, the source of truth for internal money), **Transaction Orchestrator** (saga + status FSM for external legs), **Rail Connectors** (bank/card/UPI), **Reconciliation**, **Fraud/Risk**, **Notification**, **Statement** service.

---

## 13. Data model

| Entity | Store | Notes |
|:-------|:------|:------|
| **Account** | strongly-consistent DB | user & system accounts (bank, fees, escrow, suspense) |
| **LedgerEntry** | append-only, immutable | (txn_id, account, delta, timestamp); the source of truth |
| **Transaction** | durable DB | idempotency key, type, status FSM, external ref |
| **Balance** | DB (materialized) | current available (+held); reconcilable to entry sum |
| **IdempotencyKey** | fast store / DB | applied keys → original result (dedup) |
| **SettlementRecord** | DB | ingested external records for reconciliation |

The ledger is **append-only and immutable**; balances are a materialized projection you can always re-derive; transactions carry the state machine + idempotency key. Money records are never edited — corrections are reversing entries.

> 💡 **Append-only ledger + materialized balance:** "The ledger entries are immutable truth; the balance is a fast materialized view I can rebuild by summing entries. That gives me both fast reads and provable correctness — and reconciliation/audit works because history is never mutated." (Same immutable-artifact discipline as payroll payslips.)

---

## 14. Scale estimation

| Quantity | Value | Note |
|:---------|:------|:-----|
| Users | ~100M+ | wallets |
| Payments/day | ~tens–hundreds of millions | avg hundreds–thousands/sec |
| Peak | ×5–10 (festivals, paydays, sale events) | spiky |
| Writes | 2+ ledger entries per txn | append-heavy |
| Consistency | strong per account | not eventually-consistent for balances |

Throughput is real but **modest vs consumer-scale reads** — the constraint is **strongly-consistent, atomic, correct writes**, not raw QPS. Shard by account/user; hot merchant accounts (a huge merchant receiving many payments) may need special handling (sharded sub-accounts / batched credits).

> 💡 **The estimation tell:** "Payment QPS is modest compared to a feed; the hard requirement is *strongly-consistent atomic writes* to balances, so I shard by account and design for correctness under contention — a hot merchant account is the scaling edge, which I'd handle with sub-account sharding or batched credits." 

---

## 15. Consistency, availability & storage choice

- **Strong consistency for balances** — a read must reflect committed transactions; no overdraft. This pushes toward an **ACID / strongly-consistent store** (a relational DB, or a NewSQL/distributed-SQL system like Spanner/CockroachDB) for the ledger — *not* an eventually-consistent store for the money-of-record.
- **CAP stance: choose consistency** — on a partition, a wallet should refuse/queue rather than risk double-spend. Correctness beats availability for money.
- **Append-only ledger** fits well and is easy to replicate/audit; balances materialized for reads.
- **Idempotency store** fast (Redis/DB) for dedup.

> 💡 **Pick a consistent store, and say why:** "Balances need ACID/strong consistency — I'd use a relational or distributed-SQL store for the ledger, not an eventually-consistent NoSQL, because a stale balance means overdraft or double-spend. This is the one system where I deliberately choose C over A." (Contrast with WhatsApp/presence where you chose AP.)

---

## 16. Security, compliance & fraud

- **Fraud/risk checks** *before* committing (velocity limits, anomaly detection, limits per user/KYC tier).
- **KYC/AML** — identity verification, limits, suspicious-activity reporting (regulatory).
- **Encryption** of sensitive data; **PCI-DSS** for card data (tokenize; don't store raw PANs).
- **Idempotency + audit log** double as security controls (traceability).
- **Authorization** — MFA/PIN for payments; per-transaction limits.
- **Immutable audit** for regulators and dispute resolution.

> 💡 **Fraud check is a pre-commit gate:** "Risk/fraud evaluation runs before the ledger commit — velocity limits, anomaly scoring, KYC-tier limits — because once money moves it's hard to claw back. Compliance (KYC/AML, PCI for cards) and the immutable audit log are first-class, not afterthoughts, in a regulated payment system." 

---

## 17. Failure modes

| Failure | Effect | Handling |
|:--------|:-------|:---------|
| Retry / double-click | double-charge risk | idempotency key → no-op (verified) |
| Concurrent debits | overdraft | per-account atomic/conditional update (verified) |
| External rail timeout (ambiguous) | did it move? | status-check by key/ref; **don't blind-retry**; reconcile |
| Debited but external leg failed | user out money | saga compensation → refund (verified) |
| Crash mid-transaction | partial state | transaction/state machine resumes; idempotency makes replay safe |
| Reconciliation mismatch | book ≠ bank | suspense account + reversing entries |
| Duplicate external callback | double-settle | idempotent status update; FSM guards terminal states |
| Hot merchant account contention | lock contention | sub-account sharding / batched credits |

> 💡 **The ambiguous-external-timeout is the killer follow-up:** "If a bank/card call times out I never blindly retry the charge — I status-check by the idempotency key, and reconciliation is the final arbiter before I refund. Retrying a *read* (balance) is free; retrying a *charge* can double-move money." Have this ready.

---

## 18. Edge cases

| Case | Handling |
|:-----|:---------|
| Insufficient balance | atomic check; reject (no overdraft) |
| Partial refund | reversing entries for the refunded portion |
| Refund after settlement | refund flow (new transaction), T+1 aware |
| Currency / rounding | fixed decimal (BigDecimal), defined rounding; store minor units (paise) |
| Fees & taxes | extra ledger entries (Payer −fee, Fees +fee) |
| Reversal of a reversal | new reversing txn; history preserved |
| Frozen/blocked account | reject debits/credits per compliance hold |
| Multi-currency | per-currency accounts + FX transaction (two-legged with an FX account) |

> 💡 **Store money in minor units, fixed decimal:** "I store amounts as integer minor units (paise/cents) or fixed-precision decimals with a defined rounding mode — never floating point, which introduces rounding errors that break the balanced-ledger invariant." 

---

## 19. Trade-offs & talking points

- **Consistency over availability** — the defining CAP choice for money.
- **Double-entry ledger vs mutable balance** — auditability/correctness vs simplicity (always pick the ledger).
- **Idempotency + reconciliation** — the exactly-once combo over unreliable rails.
- **Saga vs distributed transaction** — you can't 2PC across a bank; saga + compensation.
- **Materialized balance vs derive-on-read** — fast reads vs pure auditability (do both: materialized + reconcilable).
- **Strong-consistent SQL/NewSQL vs NoSQL** — money-of-record needs the former.

---

## 20. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Reframe | correctness/auditability first; consistency over availability |
| Double-entry ledger | every txn = balanced debit+credit; conserved, auditable, append-only |
| Balance invariant | no overdraft; system accounts for external money |
| Exactly-once | idempotency key; atomic per-account updates |
| Flows | add / pay / withdraw as double-entry (+ external leg) |
| External rails | saga + compensation; ambiguous-timeout → status-check |
| Reconciliation + storage | settle against bank; strong-consistent store; fraud/compliance |

### What to say
- *"I model money as a double-entry ledger — every transaction is a balanced debit and credit, appended immutably, so money is conserved and every balance is provable."*
- *"Money moves exactly once: an idempotency key makes retries no-ops, and balance debits are atomic per account so concurrency can't overdraw."*
- *"External legs (bank/card) go through a saga with compensating refunds, and I reconcile against the bank's settlement — the ultimate source of truth."*
- *"On an ambiguous external timeout I status-check, never blind-retry a charge."*
- *"Balances need strong consistency, so I'd use an ACID/NewSQL store and choose C over A on a partition."*

### Order
reframe → ledger → invariant/accounts → idempotency+atomicity → flows → saga → reconciliation → storage/fraud → failures.

---

## 21. Common mistakes to avoid

- ❌ **A single mutable `balance` column** — untraceable; use a double-entry ledger.
- ❌ **No idempotency key** — retries/double-clicks double-charge.
- ❌ **Non-atomic debit (read-then-write)** — concurrent debits overdraw; lock per account / conditional update.
- ❌ **Trusting your own "success" for external money** — reconcile against the bank.
- ❌ **Blindly retrying an ambiguous charge** — status-check first; can double-move money.
- ❌ **No compensation/saga** — a failed external leg leaves the user out of pocket.
- ❌ **Floating-point money** — use integer minor units / fixed decimal.
- ❌ **Editing ledger entries for corrections** — append reversing entries; history is immutable.
- ❌ **Eventually-consistent store for balances** — stale balance → overdraft; use strong consistency.
- ❌ **Ignoring fraud/KYC/PCI** — they're first-class in a regulated payment system.

---

## 22. TL;DR

### The design
```
Money = DOUBLE-ENTRY LEDGER: every transaction = balanced debit + credit (sums to 0),
        append-only & immutable → money conserved, balances provable, corrections = reversing entries.
Exactly-once = IDEMPOTENCY KEY (retry = no-op) + ATOMIC per-account debit (no overdraft).
Flows: add(bank→wallet) · pay(wallet→wallet) · withdraw(wallet→bank) — all double-entry;
       external legs via SAGA + compensating refund; RECONCILE vs bank settlement (money truth).
Store: strong-consistent (ACID/NewSQL); choose C over A. Money in minor units / fixed decimal.
```

### The mechanics (verified)
```
Double-entry : every txn nets to 0; Σbalances conserved
Idempotency  : replay of a txn id → no double-charge
Atomicity    : per-account lock/conditional update → concurrent debits can't overdraw
Reserve/capture: available→held→capture/release (in-flight funds locked)
Saga         : debit → external pay → on-failure refund (compensation)
```

### The four things that score points
1. **Double-entry ledger** — balanced, append-only, auditable; money conserved & provable (not a mutable balance).
2. **Exactly-once** — idempotency key (safe retries) + atomic per-account debit (no overdraft/double-spend).
3. **External rails via saga + reconciliation** — compensate on failure; reconcile against the bank; never blind-retry a charge.
4. **Consistency over availability** — strong-consistent store for balances; fixed-decimal money; fraud/KYC/PCI first-class.

> **One-line philosophy:** *A payment system is a double-entry ledger with exactly-once money movement: model every transaction as a balanced debit and credit appended immutably so money is conserved and every balance is provable, make each movement idempotent and atomic-per-account so retries and concurrency can never double-charge or overdraw, move external money through sagas that compensate on failure and reconcile against the bank as the ultimate truth, and choose consistency over availability — because a payment system that is fast but occasionally wrong about money is worse than useless.*
