# Presence & Last-Seen — System Design (HLD, Detailed)

> "Online", "typing…", "last seen 5 minutes ago" look trivial — a boolean and a timestamp — but at scale presence is one of the **hardest** parts of a chat system, and a favorite senior interview probe. The naive design (broadcast every status change to all your contacts) creates a **fan-out storm that can dwarf actual message traffic by ~100×**. This doc builds the naive approach, shows precisely why it collapses, then designs the optimized approach (heartbeats + TTL, pull-on-demand, subscribe-on-view, debounce) — with the numbers, the data model, the architecture, and the trade-offs.

> 💡 **The one sentence that frames the whole problem:** presence is **best-effort, eventually consistent, and deliberately approximate** — you trade exactness for affordability, because making it precise and real-time for billions of users is economically impossible and nobody actually needs second-perfect accuracy.

---

## Table of Contents
1. [The Problem & Requirements](#1-the-problem--requirements)
2. [Why presence is deceptively hard](#2-why-presence-is-deceptively-hard)
3. [The naive approach (eager broadcast)](#3-the-naive-approach-eager-broadcast)
4. [The fan-out storm — with numbers](#4-the-fan-out-storm--with-numbers)
5. [The key mindset shift: approximate & best-effort](#5-the-key-mindset-shift-approximate--best-effort)
6. [The optimized approach — four techniques](#6-the-optimized-approach--four-techniques)
7. [Technique 1: Heartbeats + Redis TTL (online detection)](#7-technique-1-heartbeats--redis-ttl-online-detection)
8. [Technique 2: Pull-on-demand (last-seen)](#8-technique-2-pull-on-demand-last-seen)
9. [Technique 3: Subscribe-on-view (bounded push)](#9-technique-3-subscribe-on-view-bounded-push)
10. [Technique 4: Coalesce / debounce (flap control)](#10-technique-4-coalesce--debounce-flap-control)
11. [Data model](#11-data-model)
12. [The Architecture](#12-the-architecture)
13. [End-to-end flows](#13-end-to-end-flows)
14. [Capacity estimation](#14-capacity-estimation)
15. [Edge cases](#15-edge-cases)
16. [Privacy](#16-privacy)
17. [Trade-offs & talking points](#17-trade-offs--talking-points)
18. [How to present this in the interview](#18-how-to-present-this-in-the-interview)
19. [Common mistakes to avoid](#19-common-mistakes-to-avoid)
20. [TL;DR](#20-tldr)

---

## 1. The Problem & Requirements

Show, for each user, a **presence status** — `online` / `offline` — and, when offline, a **last-seen** time ("last seen 5 minutes ago"). Visible to a user's contacts (subject to privacy). Often extended to **typing indicators** (same mechanics, shorter-lived).

### Functional
- Mark a user online when their app is connected/active.
- Mark them offline (with last-seen) when they disconnect or go idle.
- Let a contact see another user's current presence / last-seen.
- (Optional) typing indicators.

### Non-functional (this is where it gets interesting)
- **Massive scale** — billions of users, hundreds of millions concurrent.
- **High churn** — mobile clients connect/disconnect constantly (network flaps, app background/foreground).
- **Cost-bounded** — presence must not cost more than the core product (messaging).
- **Approximate is OK** — "last seen 5 min ago" being off by a minute is fine; nobody needs millisecond accuracy.

> 💡 **Lead with the non-functional reality.** The interview turning point is recognizing that presence's hardness is *churn × fan-out × scale*, and that the requirement is **"cheap and approximately right," not "precise and real-time."** Stating that reframes the whole design.

---

## 2. Why presence is deceptively hard

A status flag sounds like one boolean. The trap is the **combination** of:

1. **Fan-out** — a status change is interesting to *all* of a user's contacts (hundreds to thousands).
2. **Churn** — mobile users change connection state *constantly* (every network switch, screen-off, app-background can drop/re-establish the socket).
3. **Scale** — multiply fan-out × churn across billions of users.

`presence events ≈ users × state_changes_per_user × contacts_per_user` — three large multipliers. That product is what explodes.

> 💡 **Name the three multipliers.** "Presence is hard because it's fan-out (contacts) × churn (mobile reconnects) × scale (billions) — three multipliers stacked. Each alone is fine; multiplied, presence traffic can exceed message traffic." That single observation is most of the senior signal.

---

## 3. The naive approach (eager broadcast)

The obvious design:

```
On CONNECT  (A comes online):  push "A is online"  to ALL of A's contacts
On DISCONNECT (A goes offline): push "A is offline (last seen now)" to ALL of A's contacts
```

```
   A connects
       │
       ▼
  Presence Service
       │  fan-out to every contact of A
       ├──► contact 1   "A online"
       ├──► contact 2   "A online"
       ├──► ...
       └──► contact 1000 "A online"     ← 1000 pushes for ONE connect event
```

It's correct and real-time. It also **does not survive contact with reality** — every connect/disconnect costs `O(contacts)` pushes, and mobile generates a torrent of connect/disconnect events.

---

## 4. The fan-out storm — with numbers

Concrete (illustrative) figures:

- **100M** concurrent users, avg **200** contacts each.
- Mobile churn: ~**2 state changes per 5 minutes** per user (network flaps, bg/fg).

| Metric | Naive (broadcast to all contacts) |
|:-------|:----------------------------------|
| Presence pushes/sec | **≈ 133,000,000 /sec** |
| Message traffic (100B msgs/day) | ≈ 1,160,000 /sec |
| **Presence ÷ messages** | **≈ 115×** |

**Naive presence traffic is ~115× the actual message traffic.** You'd spend two orders of magnitude more infrastructure on "is the green dot on?" than on delivering the messages people came for. It also produces useless noise: most of those 200 contacts aren't even looking at A's chat when A's dot flickers.

> 💡 **The killer line:** "With eager broadcast, presence traffic is ~100× message traffic — you'd spend more on the green dot than on messages. And most of it is wasted, because the vast majority of contacts aren't looking at that user right now." That quantified absurdity motivates every optimization.

---

## 5. The key mindset shift: approximate & best-effort

Before optimizing the mechanics, change the **requirement**. Presence does **not** need to be:
- **Exact** — "last seen 5 minutes ago" can be off by a minute; nobody notices or cares.
- **Real-time** — a few seconds of staleness ("shows online for ~30s after they background the app") is invisible to users.
- **Strongly consistent** — different viewers can briefly see different states; that's fine.

So presence is explicitly **best-effort, eventually consistent, and approximate.** That single relaxation unlocks every optimization below — you can batch, delay, sample, expire, and pull instead of pushing eagerly, because *approximately right and cheap beats exactly right and ruinous*.

> 💡 **Say this explicitly in the room.** "I'll treat presence as best-effort and approximate — eventual consistency, a few seconds of staleness allowed. That's the relaxation that makes it affordable." Interviewers wait to hear it; it's the difference between someone who's built this and someone who hasn't.

---

## 6. The optimized approach — four techniques

The optimized design layers four ideas, each attacking a different multiplier:

| Technique | Attacks | Effect |
|:----------|:--------|:-------|
| **Heartbeats + Redis TTL** | churn / ungraceful disconnect | online status derived from a cheap timestamp; auto-expires |
| **Pull-on-demand** | fan-out (for last-seen) | compute last-seen *when someone looks*, push nothing |
| **Subscribe-on-view** | fan-out (for live online status) | push only to the few contacts actively viewing this user |
| **Coalesce / debounce** | churn | smooth flicker; don't react to every flap |

Together they turn `users × churn × contacts` into roughly `users × churn × (active viewers ≈ a handful)` for the push path, and a cheap periodic write + on-demand read for last-seen.

---

## 7. Technique 1: Heartbeats + Redis TTL (online detection)

**Problem:** how do you know someone is online — and, crucially, how do you detect they've *gone* offline when mobile disconnects are often **ungraceful** (no clean close event; the socket just dies silently)?

**Answer: heartbeats with a self-expiring key.** The client sends a lightweight heartbeat over its WebSocket every `N` seconds (say 30s). Each heartbeat refreshes a Redis key with a **TTL** of `~2–3× N`:

```
on heartbeat from user A:
    SET  presence:A  "1"  EX 60          # online flag, expires in 60s if no further heartbeat
    SET  lastseen:A  <now>               # last-seen timestamp (no TTL; persists)

is A online?  →  EXISTS presence:A       # key present = online; expired = offline
```

The TTL is the elegance: you **don't need a reliable disconnect event**. If A's phone dies, drops off wifi, or the app is killed, the heartbeats stop, the key **expires on its own**, and A naturally becomes offline ~60s later. Graceful disconnects can delete the key immediately (or just let it expire).

```
heartbeat ──► refresh presence:A (TTL 60s)
   …                                   missed heartbeats (phone offline)
   no heartbeat ──► key EXPIRES ──► A is now "offline", lastseen:A frozen at last heartbeat
```

> 💡 **Why TTL beats a disconnect event:** half-open TCP sockets look alive; mobile rarely closes cleanly. A self-expiring heartbeat key makes "offline" a *passive* outcome (absence of recent heartbeats) rather than something you must reliably observe — which you can't. The TTL also doubles as the debounce window (Technique 4): a brief blip under the TTL never registers as offline.

Alternative storage: a **Redis sorted set** `presence_zset` with `score = last_heartbeat_epoch`; "online" = members with `score > now − threshold`; periodically `ZREMRANGEBYSCORE` to prune. Either works; TTL keys are simpler, the ZSET makes "who's online" range queries easy.

---

## 8. Technique 2: Pull-on-demand (last-seen)

**Insight:** last-seen only matters **when someone looks at it.** Most of a user's contacts never open that user's chat in a given session — so proactively pushing "A was last seen at T" to all of them is almost entirely wasted.

So **don't push last-seen at all.** Store `lastseen:A` (updated by A's heartbeats and on disconnect) and **compute the answer lazily** when a viewer requests it:

```
B opens A's chat / profile:
    ts = GET lastseen:A
    if EXISTS presence:A (or now - ts < threshold):  show "online"
    else:                                            show "last seen " + humanize(now - ts)
```

One cheap read at view time replaces N proactive pushes. The write path is just the periodic heartbeat refresh; the read path is on-demand and naturally bounded by *who's actually looking*.

> 💡 **Push vs pull is the core lever.** Eager push pays `O(contacts)` on every change whether anyone cares; pull pays `O(1)` only when someone looks. For data that's read far less often than it changes — and presence on mobile changes constantly — **pull-on-demand is dramatically cheaper.** "I compute last-seen when viewed, not when it changes" is the line.

---

## 9. Technique 3: Subscribe-on-view (bounded push)

Pure pull can feel static — you want a contact *currently viewing A's chat* to see A flip to "online" live, without polling. But you only need to push to **the handful of people looking at A right now**, not all 200 contacts.

**Subscribe-on-view:** when B opens A's chat, B **subscribes** to A's presence; when B closes it, B unsubscribes. A's presence changes are pushed **only to current subscribers**:

```
B opens A's chat   →  subscribe(B → A)      (add B to A's presence subscribers)
A's presence flips →  push only to A's CURRENT subscribers   (usually 0–few, not 200)
B closes A's chat  →  unsubscribe(B → A)
```

This caps fan-out at the number of **active viewers** — typically 0 to a few — instead of A's full contact list. The math: pushes drop from `× 200 contacts` to `× ~2 active viewers` — a **~100× reduction**, bringing presence push traffic down to roughly the same order as message traffic (and much of it avoided entirely by pull-on-demand for the rest).

```
NAIVE:    A flips → push to ALL 200 contacts
OPTIMIZED: A flips → push to the 0–few contacts WITH A's CHAT OPEN
```

> 💡 **The decisive optimization.** "I only push presence to contacts actively viewing the user — subscribe on view, unsubscribe on close. Fan-out goes from all-contacts to active-viewers, ~100× less, and everyone else gets last-seen via pull-on-demand." Combining subscribe-on-view (live, for viewers) with pull-on-demand (lazy, for everyone else) is the complete answer.

---

## 10. Technique 4: Coalesce / debounce (flap control)

Mobile connections **flap** — a user switching from wifi to cellular, or backgrounding for 3 seconds, can drop and re-establish the socket. Reacting to every flap produces "online… offline… online…" noise and re-triggers fan-out.

**Smooth it:**
- **Grace period before offline** — don't mark offline the instant the socket drops; wait out the heartbeat TTL (≈60s). A blip shorter than the TTL never registers as a state change at all (the TTL *is* the debounce).
- **Debounce change events** — collapse rapid flips into at most one presence-change push per interval.
- **Coalesce** — if A flips online→offline→online within the window, emit nothing (net no change).

> 💡 **The TTL already buys you most of this.** Because online is "heartbeat within the last 60s," a 5-second blip is invisible — no key expiry, no event. You only add explicit debounce on the *push* side so a genuine flap near the boundary doesn't spam subscribers. "Brief disconnects are absorbed by the heartbeat TTL; I debounce the change events on top."

---

## 11. Data model

Small, all in a fast in-memory store (Redis), sharded by userId:

| Key | Value | TTL | Purpose |
|:----|:------|:----|:--------|
| `presence:{userId}` | `1` (or device set) | ~2–3× heartbeat (e.g. 60s) | online flag; absence ⇒ offline |
| `lastseen:{userId}` | epoch ms | none (persists) | last-seen timestamp |
| `subs:{userId}` | set of viewer ids (or their gateway nodes) | short / managed | who's currently viewing → push targets |
| `device:{userId}` | set of active connection ids | per-device TTL | multi-device aggregation |

(Alternative: a `presence_zset` of `userId → last_heartbeat` for range "who's online" queries.)

> 💡 **Why Redis, not a durable DB?** Presence is **ephemeral, high-churn, and disposable** — losing it on a restart is fine (heartbeats rebuild it in seconds). It wants O(1) reads/writes and **TTL semantics built in**. That's exactly Redis's sweet spot; putting presence in a persistent SQL table would be both slow and pointless.

---

## 12. The Architecture

```
   mobile/web clients ── WebSocket (heartbeat every 30s) ──┐
                                                            ▼
                                              ┌───────────────────────┐
                                              │   Gateway nodes        │  (hold live sockets;
                                              │  receive heartbeats    │   receive open/close-chat
                                              └───────┬───────────────┘   = subscribe/unsubscribe)
                          heartbeat → refresh         │ presence change → publish
                                   ▼                  ▼
                       ┌────────────────────┐   ┌──────────────────────┐
                       │  Redis (presence)  │   │  Pub/Sub backplane    │
                       │  presence:{u} TTL  │   │ (route change events  │
                       │  lastseen:{u}      │   │  to gateways holding  │
                       │  subs:{u}          │   │  current subscribers) │
                       └─────────┬──────────┘   └──────────┬───────────┘
                                 │ on-demand read           │ push to subscriber's gateway
                                 ▼                          ▼
                     viewer asks "is A online /        subscriber viewing A's chat
                      last seen?" → read & compute      gets live online/offline flip
```

Components:
- **Gateway** — terminates WebSockets, receives heartbeats (refresh Redis), tracks open-chat events (subscribe/unsubscribe), and delivers presence pushes to its connected subscribers.
- **Redis (presence store)** — TTL flags + last-seen + subscriber sets; sharded.
- **Pub/Sub backplane** (Redis pub/sub / NATS / Kafka) — a presence change on one gateway must reach subscribers held on *other* gateways → publish to a per-user channel that subscriber gateways listen on.

> 💡 **Same backplane as messaging.** A subscriber's socket lives on some gateway node; the user who changed state is on another. So presence changes route through the **same pub/sub backplane + registry** as chat messages — you reuse the messaging fan-out infrastructure, you don't build a second one.

---

## 13. End-to-end flows

**Coming online:** client connects → first heartbeat → Gateway `SET presence:A EX 60` + `SET lastseen:A now` → publish "A online" to A's presence channel → only gateways holding A's *current subscribers* deliver the live flip.

**Staying online:** heartbeat every 30s refreshes the TTL; `lastseen:A` advances. No fan-out (nothing changed).

**Going offline (graceful):** clean socket close → delete `presence:A` (or let it expire) → `lastseen:A` frozen → publish "A offline" to current subscribers.

**Going offline (ungraceful — phone dies):** heartbeats stop → `presence:A` **expires after TTL** → A is offline; the next viewer computing presence sees `lastseen:A` from the last heartbeat. No event needed.

**Someone views A (pull):** B opens A's chat → read `lastseen:A` + check `presence:A` → render "online" or "last seen X ago"; **and** subscribe B to A's presence so subsequent flips push live.

---

## 14. Capacity estimation

(100M concurrent, 200 contacts, ~2 state changes / 5 min, 30s heartbeat.)

| Path | Naive | Optimized |
|:-----|:------|:----------|
| Presence **pushes**/sec | ~133M (× all contacts) | ~1.3M (× ~2 active viewers) — **~100× less** |
| Last-seen delivery | pushed to all, constantly | **pull on demand** (only when viewed) |
| Redis **heartbeat writes**/sec | — | ~3.3M cheap `SET … EX` (one per user / 30s) |
| vs message traffic (~1.16M/sec) | ~115× | ~1× (comparable / manageable) |

The optimized design moves presence from "100× the cost of messaging" to "about the same order as messaging," and most of *that* is cheap Redis writes, not cross-network fan-out.

> 💡 **Tuning knobs to mention:** the heartbeat interval (longer = fewer writes but staler/slower offline detection) and the TTL multiplier (bigger = more flap tolerance but slower offline). These are the dials that trade cost vs freshness — naming them shows you understand the cost model.

---

## 15. Edge cases

| Case | Handling |
|:-----|:---------|
| Ungraceful disconnect (phone dies) | TTL expiry → offline automatically (no event needed) |
| Connection flap (wifi↔cellular) | Absorbed by the heartbeat TTL; debounce on the push side |
| Multiple devices (phone + web) | Track per-device; user is online if **any** device is; last-seen = most recent across devices |
| Brief background (app bg 5s) | Under TTL → stays "online"; no churn |
| Viewer has chat open during a flip | Gets the live push (they're a subscriber) |
| Nobody viewing during a flip | No push at all (only state in Redis); next viewer pulls |
| Typing indicator | Same mechanics, very short TTL (~5–10s), only to active viewers |
| Redis node restart | Presence lost but **self-heals** — heartbeats rebuild it within one interval |

> 💡 **Multi-device is the one to call out.** "A user is online if any device has a live heartbeat; I aggregate per-device presence (a set of active connections) and report the OR, with last-seen as the max across devices." It's the edge case interviewers most often add.

---

## 16. Privacy

Presence is sensitive (it reveals when you're active, sleeping, etc.). Real systems (WhatsApp) let users restrict it:
- **Visibility settings** — show last-seen/online to everyone / contacts only / nobody.
- **Reciprocity** — if you hide your last-seen, you can't see others' (a common policy).
- Enforce on the **read path**: when B asks for A's presence, check A's privacy setting (and the B↔A relationship) before returning anything.

> 💡 **Privacy belongs on the read path.** Since presence is computed/pulled when viewed, that's the natural place to enforce "is B allowed to see A's presence?" — another reason pull-on-demand is convenient: the authorization check sits right where the data is served.

---

## 17. Trade-offs & talking points

- **Exactness vs cost** — accept staleness/approximation to make it affordable (the core trade).
- **Push vs pull** — push (subscribe-on-view) for the live, in-view case; pull for everything else.
- **Heartbeat interval & TTL** — freshness/offline-detection speed vs write volume and flap tolerance.
- **Consistency** — eventual; different viewers may briefly disagree; acceptable.
- **Reuse vs rebuild** — presence rides the existing messaging backplane, not a parallel system.
- **Durability** — none needed; ephemeral Redis, self-healing via heartbeats.

---

## 18. How to present this in the interview

### Suggested flow
| Phase | What to do |
|:------|:-----------|
| Reframe | "Presence is best-effort, approximate, eventually consistent — that relaxation is the key." |
| Naive + math | Eager broadcast; show it's ~100× message traffic (fan-out × churn × scale). |
| Heartbeat + TTL | Online via self-expiring Redis key; handles ungraceful disconnects. |
| Pull-on-demand | Compute last-seen when viewed; push nothing for it. |
| Subscribe-on-view | Push live flips only to active viewers — ~100× fan-out cut. |
| Debounce + multi-device + privacy | Flap control; OR across devices; enforce on read path. |
| Numbers + knobs | Restate the before/after; name interval/TTL as cost dials. |

### What to say
- *"Three multipliers make this hard: contacts × churn × scale."*
- *"I'll treat presence as approximate and best-effort — that's what makes it affordable."*
- *"Online = a heartbeat-refreshed Redis key with a TTL; offline is just the key expiring, so I don't need a reliable disconnect event."*
- *"Last-seen is pull-on-demand; live online status is push, but only to contacts currently viewing the chat."*
- *"It rides the same pub/sub backplane as messages; the store is ephemeral Redis that self-heals."*

---

## 19. Common mistakes to avoid

- ❌ **Eager broadcast to all contacts** — the fan-out storm; ~100× message traffic.
- ❌ **Relying on disconnect events for offline** — mobile disconnects are ungraceful; use heartbeat **TTL expiry**.
- ❌ **Pushing last-seen proactively** — it's read far less than it changes; **pull on demand**.
- ❌ **Pushing live status to all contacts** — only push to **active viewers** (subscribe-on-view).
- ❌ **Reacting to every flap** — debounce; let the TTL absorb blips.
- ❌ **Storing presence in a durable SQL DB** — it's ephemeral/high-churn; use Redis with TTL.
- ❌ **Claiming strong consistency / real-time exactness** — it's eventually consistent and approximate; say so.
- ❌ **Forgetting multi-device** — aggregate (OR) across a user's connections.
- ❌ **Ignoring privacy** — enforce visibility on the read path.
- ❌ **Building a separate fan-out system** — reuse the messaging backplane.

---

## 20. TL;DR

### The problem
```
presence events ≈ users × state_changes(churn) × contacts(fan-out)
Mobile churn is huge → NAIVE eager broadcast ≈ 100× message traffic.
```

### The reframe
```
Presence is BEST-EFFORT, EVENTUALLY CONSISTENT, APPROXIMATE.
Approximately-right-and-cheap >> exactly-right-and-ruinous.
```

### The optimized design (four techniques)
```
1. Heartbeats + Redis TTL : online = key refreshed within TTL ; offline = key expires (no disconnect event needed)
2. Pull-on-demand         : compute last-seen WHEN VIEWED ; push nothing for it
3. Subscribe-on-view      : push live online/offline ONLY to contacts currently viewing → ~100× less fan-out
4. Coalesce / debounce    : TTL absorbs flaps ; debounce change pushes
Store: ephemeral Redis (presence:{u} TTL, lastseen:{u}, subs:{u}) ; routes over the messaging pub/sub backplane.
```

### The numbers
```
Naive presence pushes:    ~133M/sec  (~115× messages)
Optimized presence pushes: ~1.3M/sec  (~1× messages)   — 100× reduction via subscribe-on-view
Heartbeat writes:          ~3.3M/sec cheap Redis SET…EX ; last-seen reads are on-demand
```

### The four things that score points
1. **Quantify the naive fan-out storm** (contacts × churn × scale ≈ 100× messages).
2. **Reframe as best-effort/approximate** — the relaxation that enables everything.
3. **Heartbeat+TTL for online, pull-on-demand for last-seen, subscribe-on-view for live** — the trio.
4. **Ephemeral Redis + reuse the messaging backplane**; debounce flaps; handle multi-device & privacy on read.

> **One-line philosophy:** *Presence only looks like a boolean — at scale it's fan-out × churn × billions, so you make it cheap by making it approximate: derive "online" from a self-expiring heartbeat key, compute "last seen" only when someone looks, and push live changes only to the handful of people actually viewing — turning a storm that dwarfs your message traffic into a trickle that rides the infrastructure you already have.*
