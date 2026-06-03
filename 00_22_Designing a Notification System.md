# Designing a Notification System (High-Level Design): A Senior Interview Guide

> A practical, interview-focused reference for designing a multi-channel notification system — async delivery via queues, priority handling for transactional vs promotional traffic, idempotency, retries and dead-letter queues, user preferences, templating, and delivery-status tracking. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Why Async (Queue-Based)](#5-why-async-queue-based)
6. [Priority: Transactional vs Promotional](#6-priority-transactional-vs-promotional)
7. [Reliability: Retries & Dead-Letter Queues](#7-reliability-retries--dead-letter-queues)
8. [Idempotency: No Duplicate Sends](#8-idempotency-no-duplicate-sends)
9. [Preferences, Templates & Status Tracking](#9-preferences-templates--status-tracking)
10. [Scaling & Provider Management](#10-scaling--provider-management)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This in an Interview

A notification system looks like plumbing but is a rich design problem because of three tensions a senior candidate should surface early:

1. **Reliability vs. volume** — you must *never* lose a critical message (an OTP, a password reset) while also absorbing massive promotional bursts cheaply.
2. **Different traffic, different guarantees** — a one-time-password needs sub-5-second delivery and aggressive retries; a marketing blast is best-effort and can wait. They can't share one undifferentiated pipe.
3. **You don't actually deliver anything** — email, SMS, and push all go through **third-party providers** (SendGrid, Twilio, FCM/APNs) that are rate-limited and fail. The system is mostly **reliable orchestration around unreliable external services.**

The structure: clarify requirements and channels → estimate scale → establish an **async, queue-based** core → fan out to **per-channel workers** → layer in **priority, retries, idempotency, preferences, and status tracking**. Senior signal: recognizing that idempotency and prioritization are the genuinely hard parts, and that integrating flaky third-party providers (retries, rate limits, failover) is most of the real work.

---

## 2. Requirements

### Functional

- Send via **Email, SMS, Push, In-app**.
- Support **transactional** (OTP, receipts) and **promotional** (marketing) traffic.
- **Per-channel user preferences** (opt-in/opt-out).
- **Template-based** messages with variables and localization.
- **Retry on failure**, and **track delivery status**.
- **Schedule** future notifications.

### Non-Functional

- **High throughput** — millions per day.
- **Reliability** — no lost notifications for critical messages.
- **Low latency for transactional** — OTPs under ~5 seconds.
- **Idempotent** — no duplicate sends.

> The key non-functional tension: **reliability and low latency for transactional** vs. **throughput and cost-efficiency for promotional.** This split drives the priority-queue design.

---

## 3. Capacity Estimation

**Assumptions:** 100M users × ~5 notifications/day.

```
Volume/day  = 100M × 5            = 500M notifications/day
Average QPS = 500M / 10⁵ s/day    ≈ ~6,000 notifications/sec
Peak QPS    ≈ 5× average          ≈ ~30,000/sec
```

**Takeaways:**
- ~6K/sec average is moderate, but traffic is **extremely bursty** — a single promo campaign can fire millions at once, hence the ~30K peak (and the need for a queue to absorb it).
- The bursty, fan-out-heavy, third-party-bound profile means the system is **I/O- and orchestration-bound**, not compute-bound — design for buffering and reliable delivery, not raw CPU.

---

## 4. High-Level Architecture

```
                                       ┌─► [Email Worker]  → SendGrid
[Service] ──► [Notification API] ──► [Kafka] ──► [SMS Worker]    → Twilio
                   │                  (topics)  ─► [Push Worker]   → FCM / APNs
                   │                            └─► [In-App Worker]
                   ▼
         [Template Service]   [User Preferences]   [Rate Limiter]
                                                          │
                                              [Status Store (Cassandra)]
```

- **Notification API** — the single entry point services call; validates, applies preferences, renders templates, and enqueues.
- **Kafka topics** — the durable async buffer between senders and channel workers (the backbone).
- **Per-channel workers** — each consumes its topic and integrates with the relevant **third-party provider** (SendGrid for email, Twilio for SMS, FCM/APNs for push).
- **Supporting services** — Template Service, User Preferences, Rate Limiter, and a Status Store for delivery tracking.

---

## 5. Why Async (Queue-Based)

The core decision is that notification sending is **asynchronous**, decoupled by a queue (Kafka). The reasons (the queue motivations from the message-queue guide, applied here):

- **The sender shouldn't wait for delivery** — a service triggering "order shipped" must not block on SMS/email round trips to slow third parties. It enqueues and returns instantly.
- **Buffer spikes** — promo campaign blasts (millions at once) fill the queue and drain at a sustainable rate, rather than overwhelming providers or your workers.
- **Replay & retry** — durable messages survive worker crashes and can be reprocessed; failed sends can be retried without the original sender's involvement.
- **Decouple senders from channels** — senders emit one logical "notify" event; the system handles which channels, providers, and formatting — senders don't know or care.

This async core is what makes the system both responsive (to senders) and resilient (to provider failures and bursts).

---

## 6. Priority: Transactional vs Promotional

A single undifferentiated queue is a trap: a 10-million-message promo blast would sit *ahead* of an urgent OTP, blowing the <5s latency requirement. The fix is **separate topics/queues by priority**:

- **Transactional** (OTP, password reset, security alerts) — **consumed first / fast lane**, with dedicated workers and aggressive retries. Latency-critical.
- **Promotional** (marketing) — **lower priority**, can be throttled, batched, and **dequeued during quiet hours** to spread load and respect user time zones. Best-effort.

This separation also lets you apply **different reliability guarantees per class** — transactional gets hard retries and never-drop semantics; promotional can be dropped or delayed under load. Matching effort to criticality is the senior move here.

---

## 7. Reliability: Retries & Dead-Letter Queues

Third-party providers fail transiently (timeouts, rate limits, blips), so workers must retry intelligently:

- **Exponential backoff** — retry after 1s, 2s, 4s, 8s… so you don't hammer a struggling provider. **Add jitter** (randomize the delay) to avoid synchronized retry storms (the same jitter idea as the caching guide's thundering-herd fix).
- **Max retries** — cap attempts (e.g., 5); infinite retries waste resources on permanently-failing messages.
- **Dead-Letter Queue (DLQ)** — messages that exhaust retries go to a DLQ for investigation/alerting rather than being silently dropped. The DLQ is your safety net and your debugging surface.

This retry-with-backoff-then-DLQ pattern is standard for any system depending on unreliable downstreams, and it's how you honor "no lost critical notifications" without retrying forever.

---

## 8. Idempotency: No Duplicate Sends

The hardest correctness problem, and a guaranteed interview probe. Because the queue gives **at-least-once delivery** (message-queue guide), a worker might process the same message twice (e.g., it sent the SMS but crashed before committing its offset). Without protection, the user gets **two OTPs / two "you've been charged" alerts** — confusing at best, alarming at worst.

The fix: **idempotency keys.** The producer attaches a unique `idempotency_key` to each logical notification; before sending, the consumer **checks whether that key was already processed**:

```
on message(key, payload):
    if redis.exists("sent:" + key):   # already sent → skip
        return
    send via provider
    redis.set("sent:" + key, ttl=24h) # record it
```

The dedup record lives in **Redis with a TTL** (e.g., 24h — long enough to cover retry windows, short enough to bound memory). This turns at-least-once delivery into **effectively exactly-once** *user-visible* behavior — the practical pattern (at-least-once + idempotent consumer) recommended in the message-queue guide. (Set the key atomically to avoid a race between the check and the set.)

---

## 9. Preferences, Templates & Status Tracking

### User Preferences Service

Stores **per-channel opt-ins** per user. The API **checks preferences before sending** and skips channels the user opted out of. This isn't just UX — it's **legal compliance** (CAN-SPAM, TCPA, GDPR): sending promotional messages to users who opted out can carry real penalties. Preferences must be authoritative and checked on every send.

### Template Service

Messages are **templates with placeholders** (`Hi {{name}}, your OTP is {{otp}}`), stored in a DB and rendered at send time. This **separates content from delivery logic** — non-engineers can edit copy, and you get consistency and **localization** (templates keyed by language, picking the user's locale). Never hard-code message text in the workers.

### Status Tracking

Each notification moves through a **state machine**: `PENDING → SENT → DELIVERED / FAILED`. Store `(notif_id, status, timestamp)` updates in **Cassandra** — chosen because status events are **write-heavy and time-series-like** (the same reasoning as the WhatsApp and KV-store guides). Crucially, "sent" (handed to the provider) isn't "delivered": providers like SendGrid/FCM **post delivery webhooks back** to you asynchronously, which you reconcile to advance the status to DELIVERED or FAILED. This gives you observability into actual delivery, not just submission.

---

## 10. Scaling & Provider Management

- **Shard by `user_id`** — distribute load across worker fleets so processing scales horizontally (the sharding principle from the database-scaling guide).
- **Per-channel rate limiting** — respect each provider's quota (Twilio/SendGrid throttle or ban you for exceeding limits). The rate limiter here protects *the provider relationship*, not just your own system — a bidirectional concern (rate-limiter guide).
- **Geo-routing** — send via **regional providers** for lower latency and better deliverability (and sometimes compliance/data-residency).
- **Provider failover** — abstract providers behind a common interface so you can **fail over to a backup** (e.g., a second SMS provider) if the primary is down — critical for "never lose a critical notification."
- **Scheduling** — future/scheduled notifications go through a delay queue or scheduler that releases them at the target time into the normal pipeline.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. How do you guarantee an OTP arrives within 5 seconds while a 10M promo blast is running?**
Separate priority queues/topics: transactional has a dedicated fast lane with its own workers and aggressive retries, so it's never stuck behind promotional traffic. Promotional is lower-priority, throttled, and can be deferred to quiet hours. Match reliability/latency guarantees to message criticality.

**Q2. How do you prevent duplicate notifications?**
Idempotency keys: the producer attaches a unique key; the consumer checks a Redis dedup store (with ~24h TTL) before sending and records the key after. This makes at-least-once queue delivery behave as exactly-once from the user's perspective. Set the key atomically to avoid check-then-set races.

**Q3. Why async/queue-based instead of sending synchronously?**
So senders don't block on slow third-party providers, so promo bursts are buffered instead of overwhelming the system, so failed sends can be retried/replayed independently, and so senders are decoupled from channel/provider specifics. It's the foundation of both responsiveness and resilience.

**Q4. How do you handle provider failures?**
Exponential backoff with jitter, a max-retry cap, and a dead-letter queue for permanent failures (for investigation, never silent drops). Abstract providers behind a common interface so you can fail over to a backup provider when the primary is down.

**Q5. What's the difference between "sent" and "delivered," and how do you track it?**
"Sent" = handed to the provider; "delivered" = the provider confirmed it reached the device. Providers post asynchronous delivery webhooks back, which you reconcile to advance the status (PENDING → SENT → DELIVERED/FAILED), stored in Cassandra (write-heavy, time-series). This gives true delivery observability.

**Q6. Why Cassandra for status, and how do you store it?**
Status updates are high-volume, append-heavy, and time-ordered — a wide-column store like Cassandra (LSM, consistent hashing) handles the write rate well, and partitioning by notification/user makes status lookups fast. The same write-heavy reasoning as the messaging-store designs.

**Q7. How do you respect both user choice and the law?**
A Preferences Service is checked on every send; opted-out channels are skipped. This is mandatory for compliance (CAN-SPAM/TCPA/GDPR) — promotional sends to opted-out users carry legal risk. Transactional messages (security, account) may bypass marketing opt-outs per regulation, which is a policy nuance worth calling out.

**Q8. How do you scale and avoid getting throttled by providers?**
Shard work by user_id across worker fleets, and apply per-channel rate limiting that respects each provider's quota (the limiter protects the provider relationship, not just you). Geo-route to regional providers for latency and deliverability.

**Q9. How would you support scheduled notifications?**
A scheduler / delay queue holds messages until their send time, then releases them into the normal priority pipeline. This keeps scheduling separate from the delivery path.

**Q10. What's the consistency model and is it acceptable?**
Effectively at-least-once delivery made exactly-once-visible via idempotency, with eventually-consistent status (webhooks arrive asynchronously). That's the right trade: guaranteeing delivery (with dedup) matters far more than instant status consistency, and transactional vs promotional get appropriately different guarantees.

---

## 12. Quick Glossary

- **Channel** — a delivery medium: email, SMS, push, in-app.
- **Transactional vs promotional** — critical/individual messages vs marketing/bulk.
- **Provider** — third-party delivery service (SendGrid, Twilio, FCM/APNs).
- **Fan-out** — delivering one logical notification across multiple channels/recipients.
- **Idempotency key** — a unique token ensuring a logical notification is sent once.
- **Dedup store** — a (Redis, TTL'd) record of processed keys preventing duplicates.
- **Exponential backoff** — increasing retry delays (1s, 2s, 4s…) to ease pressure on a failing provider.
- **Jitter** — randomized delay added to avoid synchronized retry storms.
- **Dead-Letter Queue (DLQ)** — where messages go after exhausting retries, for investigation.
- **Priority queue / topic** — separating traffic so urgent messages aren't stuck behind bulk.
- **Template** — a message with placeholders rendered at send time; supports localization.
- **Delivery webhook** — async callback from a provider confirming delivery status.
- **Status state machine** — PENDING → SENT → DELIVERED/FAILED lifecycle.
- **Preferences service** — authoritative per-channel opt-in/opt-out, checked before sending.
- **Provider failover** — switching to a backup provider when the primary fails.

---

*Reference document. Contributions and corrections welcome.*
