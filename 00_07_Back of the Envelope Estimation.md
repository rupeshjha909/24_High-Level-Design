# Back-of-the-Envelope Estimation: An Interview Guide

> A practical reference for sizing systems in your head during interviews — the latency numbers worth memorizing (and the intuition behind them), the powers-of-ten tricks that make the arithmetic easy, a repeatable estimation method, fully worked examples, and the mistakes that cost candidates points.

---

## Table of Contents

1. [Why This Skill Matters](#1-why-this-skill-matters)
2. [The Mindset: Orders of Magnitude, Not Precision](#2-the-mindset-orders-of-magnitude-not-precision)
3. [Latency Numbers Every Engineer Should Know](#3-latency-numbers-every-engineer-should-know)
4. [The Intuition Behind the Numbers](#4-the-intuition-behind-the-numbers)
5. [Units & Powers-of-Ten Cheatsheet](#5-units--powers-of-ten-cheatsheet)
6. [Data Sizes](#6-data-sizes)
7. [The Magic Numbers (Time & QPS)](#7-the-magic-numbers-time--qps)
8. [A Repeatable Estimation Method](#8-a-repeatable-estimation-method)
9. [Worked Examples](#9-worked-examples)
10. [Common Mistakes](#10-common-mistakes)
11. [Interview-Ready Insights](#11-interview-ready-insights)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. Why This Skill Matters

In a system design interview, you'll be asked to size something — "how much storage?", "how many servers?", "what's the QPS?" — **without a calculator.** The interviewer isn't checking your arithmetic; they're checking whether you can **reason quantitatively** about a system: whether your design is plausible, where the bottleneck will be, and whether you'd order one server or a thousand.

The real value of estimation is that it **drives design decisions**. If your math says 40 writes/sec, you don't need a fancy write pipeline. If it says 3 TB, storage isn't your problem. If it says 4K reads/sec, caching becomes central. Good estimation tells you *what to spend your design time on* — that's why it comes early in every interview.

A candidate who can confidently say *"that's about 10K QPS at peak, so a single DB won't cut it, we'll need replicas and a cache"* signals seniority far more than one who designs in a vacuum.

---

## 2. The Mindset: Orders of Magnitude, Not Precision

The single most important rule: **you are estimating, not calculating.** The goal is the right **power of ten**, not the right digit.

- `86,400 seconds/day` → just use **`~100,000` (10⁵)**. Nobody cares about the 13.6% error; it makes every later step trivial.
- `512 bytes` → call it **`~500`** or even `~1KB`.
- `27.3 TB` and `30 TB` are the *same answer* for design purposes.

Rounding aggressively to clean powers of ten is the entire trick that lets you do this in your head. Precision is a *trap* — chasing it burns time and signals that you've missed the point. State your rounding out loud ("call it 10⁵ seconds a day") so the interviewer sees it's deliberate.

---

## 3. Latency Numbers Every Engineer Should Know

These are the classic "Jeff Dean numbers." Memorize the **relative ratios** more than the exact figures — the ratios are what let you reason about where time goes.

| Operation                          | Latency      | Rounded            |
|------------------------------------|--------------|--------------------|
| L1 cache reference                 | 0.5 ns       | ~1 ns              |
| L2 cache reference                 | 7 ns         | ~10 ns             |
| Main memory reference (RAM)        | 100 ns       | ~100 ns            |
| Send 1 KB over 1 Gbps network      | 10 μs        | ~10 μs             |
| Read 1 MB sequentially from memory | 250 μs       | ~250 μs            |
| Round trip within same datacenter  | 500 μs       | ~0.5 ms            |
| Read 1 MB sequentially from SSD    | 1 ms         | ~1 ms              |
| Read 1 MB sequentially from HDD    | 20 ms        | ~20 ms             |
| Round trip across continents       | 150 ms       | ~150 ms            |

> **Modern caveat (worth saying in an interview):** these are the traditional figures. Modern NVMe SSDs are considerably faster (often ~100 μs for 1 MB), and the absolute numbers drift over time. But the **relative orders of magnitude — RAM ≫ network ≫ SSD ≫ disk ≫ cross-continent — are stable and are what you reason with.**

---

## 4. The Intuition Behind the Numbers

Rote memorization fades; intuition sticks. Here's the mental model that makes the table memorable.

**Each "level" of the hierarchy is roughly 10–100× slower than the one above it:**

```
L1 cache        ~1 ns      │
L2 cache        ~10 ns     │  on-CPU: nanoseconds
RAM             ~100 ns    │
─────────────────────────  │
DC round trip   ~0.5 ms    │  within a datacenter: microsecond-to-millisecond
SSD 1MB read    ~1 ms      │
─────────────────────────  │
HDD 1MB read    ~20 ms     │  spinning disk: tens of milliseconds
─────────────────────────  │
Cross-continent ~150 ms    │  speed of light across the planet: ~hundred+ ms
```

The killer takeaways for design reasoning:

- **Memory is ~100,000× faster than a cross-continent round trip.** This is *why* caching exists and why you keep hot data in RAM.
- **A same-datacenter round trip (~0.5 ms) is ~300× faster than a cross-continent one (~150 ms).** This is *why* you co-locate services and put users in nearby regions.
- **Disk seeks are catastrophically slow vs. memory** — *why* random disk I/O is the thing to avoid, and *why* SSDs displaced HDDs.
- **Cross-continent latency is bounded by the speed of light** — you literally cannot beat ~150 ms round-trip around the planet, no matter your hardware. *That's why CDNs and multi-region exist* — move the data closer, since you can't make light faster.

If you remember *"each layer is 10–100× the one above, and the network/disk/distance layers are where the milliseconds hide,"* you can reconstruct the whole table.

---

## 5. Units & Powers-of-Ten Cheatsheet

Time:
```
1 second = 10³ ms = 10⁶ μs = 10⁹ ns
1 ms = 1,000 μs        1 μs = 1,000 ns
```

Doing arithmetic in **exponents** is the speed hack — multiply by adding exponents, divide by subtracting them:
```
10⁸ items / 10⁵ seconds = 10³ per second      (just subtract: 8 − 5 = 3)
4 × 10⁷  ×  100  =  4 × 10⁹                    (add exponents: 7 + 2 = 9)
```

Round every messy number to a single significant digit × a power of ten *before* you start, and the whole calculation becomes adding and subtracting small integers.

---

## 6. Data Sizes

**Per-field sizes** (for estimating record/row sizes):

| Type                 | Size      |
|----------------------|-----------|
| char / ASCII byte    | 1 byte    |
| int                  | 4 bytes   |
| long / timestamp     | 8 bytes   |
| UUID                 | 16 bytes  |

**Scale prefixes** (use the clean powers-of-ten version for estimation):

| Unit | Bytes  |
|------|--------|
| KB   | 10³ B  |
| MB   | 10⁶ B  |
| GB   | 10⁹ B  |
| TB   | 10¹² B |
| PB   | 10¹⁵ B |

> The pedantic point that base-2 (KiB = 1024) differs from base-10 (KB = 1000) is **irrelevant for estimation** — the ~2.4% difference vanishes against your other rounding. Use 10³.

A quick way to size a record: add up its fields, then round. A "user metadata" row with a few ints, a couple of strings, and timestamps is *on the order of* 1 KB — don't agonize over the exact byte count.

---

## 7. The Magic Numbers (Time & QPS)

A handful of derived constants do most of the work:

- **Seconds per day ≈ 86,400 ≈ 10⁵.** This is the single most useful number in estimation.
- **Seconds per month ≈ 2.5 × 10⁶** (≈ 30 × 86,400).
- **Seconds per year ≈ 3 × 10⁷.**

**The QPS ↔ daily-volume shortcut:**
```
daily volume  ≈  QPS × 10⁵
QPS           ≈  daily volume / 10⁵
```
So 1,000 QPS ≈ 10⁸ events/day (100 million/day); 100M events/day ≈ ~1K QPS. Memorize this one conversion and most QPS questions become instant.

**Peak vs. average:** real traffic isn't flat. A standard assumption is **peak ≈ 2–3× average** — always size for peak, and state the multiplier you're using.

---

## 8. A Repeatable Estimation Method

A reliable five-step recipe you can run for almost any sizing question:

1. **State assumptions out loud.** Total users, % active daily, actions per active user, average payload size, retention period, read:write ratio. Make them explicit ("assume 50% of users post once/day") — the interviewer will correct you if needed, and stated assumptions are themselves a senior signal.
2. **Compute the write rate (QPS).** Daily actions ÷ 10⁵ s/day → writes/sec. Then apply the peak multiplier.
3. **Derive reads from the read:write ratio.** Convert this early — it's usually the number that dominates the design.
4. **Compute storage.** Actions/day × bytes/action × retention. Round at each step.
5. **Interpret the result for the design.** This is the part that earns points: *"4K reads/sec means a single DB won't keep up → add replicas and a cache. 3 TB is small → storage isn't the constraint."* Numbers without interpretation are half an answer.

---

## 9. Worked Examples

### Example A — Twitter write & storage sizing

**Assumptions:** 500M users; 50% post once/day; 1 tweet ≈ 300 bytes; peak = 3× average.

```
Tweets/day  = 500M × 50%          = 250M = 2.5 × 10⁸
Write QPS   = 2.5×10⁸ / 10⁵       ≈ 2,500  ≈ ~3K writes/sec  (average)
Peak QPS    = 3 × 3K              ≈ ~9K writes/sec
Storage/day = 2.5×10⁸ × 300 B     = 7.5 × 10¹⁰ B  ≈ 75 GB/day
Storage/yr  = 75 GB × 365         ≈ 27 TB/year
```

**Interpretation:** ~9K peak writes/sec is real but manageable with sharding/queuing; 27 TB/year means you're firmly in distributed-storage territory and should plan multi-year retention and partitioning.

### Example B — User metadata storage over 5 years

**Assumptions:** 1B users; ~1 KB metadata each.

```
Storage = 1 × 10⁹ users × 1 × 10³ B = 1 × 10¹² B = 1 TB
```

**Interpretation:** 1 TB of user metadata is trivial — fits on a single beefy node or a small cluster. So *user metadata isn't the storage problem*; the content (tweets, media) is. That's the kind of conclusion the estimate is *for*.

### Example C — Reads from a ratio (the URL-shortener pattern)

**Assumptions:** 100M new URLs/month; 100:1 read:write.

```
Write QPS = 10⁸ / (2.5 × 10⁶ s/month) ≈ 40 writes/sec
Read QPS  = 40 × 100                    = ~4,000 reads/sec  (peak ~10K)
```

**Interpretation:** writes are tiny (simple write path); reads dominate at ~4K (peak ~10K) → **caching is the central design concern.** The estimate told you where to spend your design budget.

---

## 10. Common Mistakes

- **Chasing precision.** Computing 86,400 × 365 by hand wastes time and misses the point. Round to 10⁵ and 3×10⁷.
- **Forgetting peak load.** Designing for average traffic and getting crushed at peak. Always apply the 2–3× multiplier and say so.
- **Not stating assumptions.** Pulling numbers from nowhere. Say "assuming X" — it's correctable and shows rigor.
- **Mixing units mid-calculation.** ms vs. μs, GB vs. TB. Keep everything in powers of ten and track the exponent.
- **Stopping at the number.** The number is worthless without the *so-what*. Always end with what the estimate implies for the design.
- **Confusing bits and bytes** on network math. 1 Gbps = 10⁹ *bits*/sec; divide by 8 for bytes. (1 KB ≈ 8 Kbits → ~8–10 μs on a 1 Gbps link.)
- **Over-memorizing, under-understanding.** If you only memorize the latency table, you'll blank under pressure. Internalize the *ratios* (Section 4) so you can rebuild it.

---

## 11. Interview-Ready Insights

**Q: How precise should a back-of-the-envelope estimate be?**
Order-of-magnitude only. The goal is the right power of ten to drive a design decision, not the exact figure. Round aggressively to clean powers of ten and state your rounding.

**Q: What's the most useful number to memorize?**
**~10⁵ seconds per day.** Combined with the shortcut `daily volume ≈ QPS × 10⁵`, it converts almost any "X per day" into QPS instantly.

**Q: Why memorize the latency numbers?**
To reason about *where time goes* and justify design choices — caching (RAM ≫ network), co-location (same-DC ≪ cross-continent), avoiding random disk I/O, and CDNs/multi-region (you can't beat the speed of light). The ratios matter more than the exact nanoseconds.

**Q: How do you handle peak vs. average traffic?**
Compute the average from assumptions, then multiply by ~2–3× for peak and **design for peak.** Always state the multiplier you assumed.

**Q: Walk me through sizing a system end-to-end.**
State assumptions → write QPS (daily/10⁵, then peak) → reads from the read:write ratio → storage (volume × size × retention) → **interpret**: what each number means for the architecture. The interpretation is the part that earns the points.

**Q: Why does cross-continent latency matter so much?**
A round trip across the planet is ~150 ms and bounded by the speed of light — you can't engineer it away. That hard floor is *why* CDNs and multi-region deployments exist: you move data closer to users rather than trying to make the network faster.

---

## 12. Quick Glossary

- **Back-of-the-envelope estimate** — a quick, order-of-magnitude calculation to sanity-check a design.
- **Order of magnitude** — the power of ten of a quantity; the precision target for estimation.
- **QPS** — queries (or requests) per second; the core throughput metric.
- **Read:write ratio** — relative volume of reads vs. writes; usually drives the design.
- **Peak vs. average** — sustained mean load vs. the higher short-term spike (typically ~2–3× average).
- **Latency** — time for a single operation to complete (the Jeff Dean numbers).
- **Throughput** — operations completed per unit time (QPS, MB/s).
- **Jeff Dean numbers** — the canonical latency table (L1 → cross-continent) used for reasoning.
- **Powers-of-ten arithmetic** — doing estimation by adding/subtracting exponents.

---

*Reference document. Contributions and corrections welcome.*
