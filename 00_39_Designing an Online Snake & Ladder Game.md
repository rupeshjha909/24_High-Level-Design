# Designing an Online Snake & Ladder Game: A Senior Interview Guide

> A practical, interview-focused reference for a real-time multiplayer Snake & Ladder platform — **rooms/matchmaking, turn-based play, server-authoritative provably-fair dice, real-time broadcast to all players, turn timers, and reconnection** at scale. The game logic is trivial; the design interest is that it's an **online turn-based game where the server must *generate* the randomness** (the dice) in a way neither the server nor a player can rig. This guide builds the architecture up piece by piece, traces the full life of a game, goes deep on the two genuinely hard mechanisms (broadcasting a turn to every player in a room, and **provably-fair server-authoritative dice**), nails down the data contracts, and covers rooms, turn timers, reconnection, sharding, and failure modes — with verified capacity, fairness, and uniformity math and a senior follow-up bank.

> 💡 **The one idea:** Snake & Ladder is **authoritative server state with server-generated randomness.** The board rules are trivial — the hard part is that the **dice must be produced by the server** (a client that rolls its own dice cheats), yet be **unpredictable, un-riggable, and verifiable after the fact** (provably fair). Get *server authority* and *provably-fair dice* right and the rest is standard turn-based real-time plumbing.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified)](#3-capacity-estimation-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a game](#44-the-end-to-end-life-of-a-game)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Game / Turn State Machine](#5-the-game--turn-state-machine)
6. [Hard Problem 1: How a Turn Is Made and Broadcast to Every Player](#6-hard-problem-1-how-a-turn-is-made-and-broadcast-to-every-player)
7. [Hard Problem 2: Provably-Fair Server-Authoritative Dice](#7-hard-problem-2-provably-fair-server-authoritative-dice)
8. [Rooms & Matchmaking](#8-rooms--matchmaking)
9. [Data Contracts: Request Fields, Payloads & DB Schemas](#9-data-contracts-request-fields-payloads--db-schemas)
10. [Turn Timer & Reconnection](#10-turn-timer--reconnection)
11. [Scaling Summary](#11-scaling-summary)
12. [Failure Modes & Handling](#12-failure-modes--handling)
13. [Senior Follow-Up Questions (with Answers)](#13-senior-follow-up-questions-with-answers)
14. [Quick Glossary](#14-quick-glossary)

---

## 1. How to Approach This in an Interview

Snake & Ladder looks trivial — roll a die, move a token, jump on snakes/ladders, first to 100 wins. As a *single-player* app it's a weekend project. What makes it an **HLD** question is turning it into a **real-time, multiplayer, cheat-resistant online game**: 2–4 players in a room, each taking turns, seeing each other's moves live — and the twist that the server must **generate the dice**, because a client that rolls its own dice will roll a six every time. So the interview is really testing two things: (1) **real-time turn-based multiplayer** (rooms, turn management, broadcasting each move to everyone in the room, reconnection), and (2) **server-authoritative, provably-fair randomness** (how the server produces a die roll that it can't rig against a player and a player can't predict or fake — and that both can *verify* afterward).

A good structure:
1. **Clarify requirements** — rooms vs matchmaking, player count, turn timer, provably-fair dice, spectators.
2. **Estimate scale** — concurrent *games/rooms* and the **roll rate**, plus millions of live connections.
3. **Establish server-authoritative state + server-generated dice** — the board, whose turn, and the RNG all live on the server; the client renders and requests.
4. **Go deep on the two hard mechanisms** — the turn broadcast to a room, and provably-fair dice.
5. **Cover rooms/matchmaking, turn timers, reconnection, sharding.**

> 💡 **Senior signal:** say up front — *"The board logic is trivial; the design problem is real-time multiplayer plus **server-generated randomness**. The client can't roll its own dice — it would cheat — so the server generates each roll, but it must be **provably fair**: neither the house nor a player can rig or predict it, and both can verify it afterward. That commit-reveal scheme is the interesting part."*

---

## 2. Requirements

### Functional
- **Rooms** — create/join a private room (share code) or **quick-match** into a public room (2–4 players).
- **Turn-based play** — on your turn, request a **dice roll**; your token moves; **snakes** send you down, **ladders** send you up; first to reach exactly 100 wins.
- **Provably-fair dice** — the server generates rolls; players can verify no rigging.
- **Live updates** — every player sees each roll/move in real time; **turn timer** (auto-skip/auto-roll on timeout).
- **Reconnection**, game history, optional spectators, leaderboards.

### Non-Functional
- **Low latency** — a roll/move feels instant (<150 ms).
- **Server-authoritative + provably fair** — the server owns state and the RNG; **anti-cheat**.
- **High availability** — a dropped player mustn't corrupt or stall the room.
- **Scale** — millions of concurrent rooms/connections; hundreds of thousands of rolls/sec.

> 💡 The split: *live play* is latency-critical and server-authoritative; *history/leaderboards* are batchy and eventually consistent. Effort goes into live play + fair dice.

---

## 3. Capacity Estimation (verified)

**Assumptions:** 10M DAU; 8 games/user/day; ~4-min games; ~3 players/game; ~87 rolls/game.

```
Games/day        = 80,000,000
Concurrent games = avg ~222,000, PEAK ~1,000,000
ROLL rate        ≈ 0.36 rolls/s per game → PEAK ~362,000 rolls/sec   ← the real-time load
WebSocket conns  ≈ 3,000,000 at peak (3 players × 1M games) → gateway fleet ~6–30 nodes
Live game state  = 1M games × ~250 B ≈ 0.25 GB in memory   (shard by game_id)
Storage          = 80M games/day × ~0.5 KB ≈ 41 GB/day → ~15 TB/yr  (result + optional roll log)
```

**Takeaways that shape the design:**
- **Roll throughput (~362K/s) is the real-time load** — moderate, served from **in-memory room state**, not a DB per roll.
- **~3M live connections** → a **WebSocket gateway/game-server fleet**.
- **Game state is tiny** — 2–4 token positions + whose turn + board config ≈ 250 bytes → **1M games in ~0.25 GB RAM**, sharded across nodes.
- **Storage is modest** — the durable record is the result (+ optionally a roll log for provably-fair verification / replay).

> 💡 The numbers say: keep each live game **in memory on one owning node** (roll + move + broadcast in sub-ms), persist only the result and a small roll log, and scale by **sharding rooms across nodes** — no DB on the per-roll hot path.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

An online Snake & Ladder system is a **server-authoritative turn-based game** — the same shape as online chess, with one twist: the source of a turn isn't the player's *chosen* move but a **server-generated random dice roll.** When it's your turn, you don't send "I rolled a 4" — you send "**roll for me**," and the **server** produces the die value (from its own RNG), applies the board rules (advance the token, then apply any snake/ladder, check for a win), and pushes the **confirmed** result to every player in the room. Server authority is what makes the game cheat-proof (a hacked client can't fake a six or move out of turn), and server-generated randomness is what makes the *dice* cheat-proof — but that raises a trust question the design must answer: how do players know the *house* isn't rigging the dice? The answer is a **provably-fair commit-reveal scheme** (the server commits to a secret seed before the game and reveals it after, so every roll can be recomputed and verified). A live game is a tiny **stateful object pinned to one game-server node** (token positions, whose turn, the RNG seeds), updated in memory and backed by a durable record. Around it sit **Rooms/Matchmaking** (create/join or quick-match), a **WebSocket gateway** (hold connections, broadcast turns to the room), a **turn timer** (auto-skip idle players), **reconnection** (rejoin from durable state), and **leaderboards** (async). Rooms are independent, so the system **shards by game/room id** trivially.

### 4.2 The diagram

```
 player1 ─WS─┐   player2 ─WS─┐   player3 ─WS─┐     (2–4 players per room)
             ▼               ▼               ▼
        ┌───────────────────────────────────────┐
        │        Gateway (WebSocket fleet)         │  registry(user→node) + pub/sub backplane
        └────────┬──────────────────────┬─────────┘
        join/quick-match                 │ "roll for me" (on my turn)
                 ▼                        ▼
        ┌────────────────┐   ┌────────────────────────────────┐  roll + move + win-check (in RAM)
        │ Room /          │   │  Game Server (authoritative)     │──────────► [Game DB / roll log]
        │ Matchmaking     │   │  positions, turn, board config,  │            (durable; provably-fair record)
        │ (rooms, codes,  │   │  RNG seeds (commit-reveal)        │
        │  quick-match)   │   └───────┬───────────────┬─────────┘
        └──────┬─────────┘            │ confirmed turn │ on game end
         create room                   ▼ (broadcast to  ▼
               ▼                        all players)   ┌──────────────┐
     [assign room to a Game Server]  [room fan-out]     │ Leaderboard / │
     (shard by game_id)              (pub/sub → N players)│ Stats (async)│
                                                          └──────────────┘
```

The key visual idea: a turn is a **request-roll → server generates die → apply rules → broadcast** cycle owned by the **Game Server** holding that room; the durable **roll log** records the provably-fair record but is off the latency-critical path.

### 4.3 Each component, in detail

**① Client (the board UI).** A **renderer + requester**, not an authority. It shows the board and tokens, and on the player's turn sends "**roll**." It applies only the server's **confirmed** result (die value + new positions + whose turn next). It never generates the die or decides moves — it can only *ask* to roll and animate what the server returns. It may also send the **client seed** used in the provably-fair scheme (§7).

**② Gateway (WebSocket fleet).** Terminates the ~3M live connections and routes each player's action to the **Game Server owning their room**, and broadcasts confirmed turns back to all players in the room, via a `user→node` / `room→node` registry + pub/sub backplane. Dumb by design; heartbeats + reconnect handle churn.

**③ Room / Matchmaking Service.** Handles **create private room** (returns a join code), **join by code**, and **quick-match** (drop a player into an open public room, or open a new one). When a room is full/started, it **creates the game** and assigns it to a Game Server (§8).

**④ Game Server (the authority — the heart).** Holds each live room **in memory**: token positions, whose turn, the board config (snakes/ladders map), and the **RNG seeds** for provably-fair dice. On "roll," it **generates the die** (§7), applies the rules (advance, snake/ladder jump, overshoot rule, win check), advances the turn, appends to the **roll log**, and **broadcasts** the confirmed turn to all players. The room is **pinned to one node** so its state has a single owner.

**⑤ RNG / Provably-Fair module.** Generates each roll deterministically from `(serverSeed, clientSeed, nonce)` via HMAC, with a **commit-reveal** wrapper so rolls are unpredictable, un-riggable, and verifiable (§7). The anti-cheat foundation for randomness.

**⑥ Game DB / roll log (durable).** Stores the game result and the **provably-fair record** (committed hash, seeds revealed at end, ordered rolls) so any player can verify the dice weren't rigged, and games can be replayed. Written off the hot path.

**⑦ Turn timer.** Each turn has a deadline; if the player doesn't roll in time, the server **auto-rolls or skips** them, so one idle/disconnected player can't stall the room (§10). Same lazy-timer trick as the chess clock — schedule one timer per active turn.

**⑧ Leaderboard / Stats.** Async updates (wins, coins) on game end — off the play path.

### 4.4 The end-to-end life of a game

Here is exactly what happens from **"Play" to a finished game**, step by step:

```
ROOM SETUP:
1.  Player quick-matches (or creates a room, shares a code) → Room service groups 2–4 players.
2.  Room service creates game G, assigns board config + turn order, and — for provably-fair dice —
        the server picks a secret serverSeed and sends its COMMITMENT hash to all players (§7).
        Assigns G to a Game Server node N (shard by game_id); all clients route WS to N; N loads G in RAM.

A TURN (the request-roll → generate → apply → broadcast cycle):
3.  It's Player 1's turn → Player 1's client sends ROLL {game:G, nonce:k, [clientSeed]}.
4.  Game Server N (authoritative):
        - is it Player 1's turn?  → yes
        - GENERATE the die: value = f(serverSeed, clientSeed, nonce) → 1..6   (§7)
        - APPLY rules: newPos = pos + value; if overshoot > 100 → stay;
                       else apply snake/ladder jump; check win (== 100)
        - append {turn, die, from, to} to the roll log (durable, async)
        - advance the turn to the next player
        - BROADCAST the confirmed turn to ALL players in the room:
              {player, die, from, to, jumped?, nextTurn, [winner]}                 ← §6
5.  Every player sees the die and the token move instantly; the next player rolls; repeat 3–4.
        (If a player doesn't roll before the turn deadline → auto-roll/skip — §10.)

END:
6.  A player lands exactly on 100 → Game Server marks G finished (winner), writes the final record,
        and REVEALS serverSeed so every player can VERIFY all rolls were fair (§7).
7.  Leaderboard/Stats update async; the in-memory game object is freed from node N.
```

The single most important thing to notice: **step 4 is entirely in memory on one node** — generate + apply + broadcast in sub-milliseconds; the durable roll-log write is a side-effect for verification/replay, not a blocking step. That keeps a roll instant while the game stays cheat-proof, fair, and recoverable.

### 4.5 Why this split? (the design rationale)

- **Server generates the dice (client requests)** — if the client rolled, it would cheat (always a six). Server-generated randomness is mandatory; the client can only *ask* to roll.
- **Provably-fair commit-reveal** — server authority stops *players* cheating, but players must also trust the *house* isn't rigging against them. Commit-reveal makes the dice verifiable by both, so neither side can cheat — essential for money/competitive play.
- **Room pinned to one Game Server (in memory)** — a game is a tiny, turn-by-turn shared state; one owning node keeps roll/apply/broadcast sub-ms and avoids per-turn distributed consensus. Rooms are independent → shard trivially.
- **Roll log separate from live state** — the durable truth is the *ordered rolls + seeds* (for verification/replay), written off the hot path so it never adds turn latency.
- **Gateway separate and dumb; leaderboard async** — holding millions of connections and updating stats are different concerns from game logic; isolating them lets each scale/fail independently.

### 4.6 Where the load actually goes

- **Rolls:** ~**362K/s** at peak — served from **in-memory room state** (generate + apply + broadcast), ~sub-ms each; the database is **not** per-roll (only an async roll-log append). Very manageable.
- **Connections:** ~**3M** WebSockets → a **gateway/game-server fleet of ~6–30 nodes**; connection-holding + **room broadcast fan-out** are the bigger infra concerns than roll CPU.
- **Live state:** ~**0.25 GB** for 1M rooms — trivial; the constraint is *ownership* (each room on one node), not memory.
- **Broadcast:** each turn fans out to 2–4 players in the room — small, unlike a viral spectator spike; only truly popular *spectated* games need a broadcast tier.
- **Storage:** ~**15 TB/yr** of results + roll logs — modest and compressible.

> 💡 **The senior framing:** *"Roll throughput is moderate and served from RAM — each room is a tiny in-memory object on one owning node, and a turn is generate-die + apply-rules + broadcast in sub-ms. The real infra load is holding ~3M connections and fanning each turn to the room. Rooms shard trivially. The one subtle design piece is the provably-fair dice."*

---

## 5. The Game / Turn State Machine

A game is a durable state machine driven by turns:

```
WAITING_FOR_PLAYERS ─► IN_PROGRESS ─► FINISHED
        │                   │
        │                   ├─► (each turn) current player ROLLs → move → snake/ladder → win-check
        │                   │        turn timeout → AUTO_ROLL / SKIP (idle player)
        │                   └─► player leaves → mark left; continue if ≥2 remain, else ABORT
        └─► not enough players joined in time → ABORTED
```

- **WAITING_FOR_PLAYERS → IN_PROGRESS**: room fills (or host starts); seeds committed; turn order set.
- **IN_PROGRESS**: rounds of turns; each turn is server-generated roll → move → win check; a turn timer bounds idle players.
- **FINISHED**: a player reaches 100; result recorded; seeds revealed for verification.

> 💡 Every transition is **server-decided and (for turns) logged**, so a crashed Game Server reloads the room by replaying the roll log, and the provably-fair record is auditable.

---

## 6. Hard Problem 1: How a Turn Is Made and Broadcast to Every Player

The turn cycle is the core loop, where server authority + room fan-out meet. *When a player rolls, how does a legal, confirmed turn reach all 2–4 players (possibly on different gateway nodes), and how is cheating prevented?*

### The naive (wrong) approach
"The player rolls locally, sends the result and their new position; the server forwards it to the others." This is a **relay**, and it's broken: a hacked client sends "I rolled a 6 and moved to 100," and everyone applies it — trivial cheating. There's also **no single source of truth**, so boards can diverge.

### The key realization: request the roll, don't report it
A turn is a **request to the authority**, not a result you announce. The client sends "**roll for me**"; the **Game Server generates the die** (§7), applies the rules against the **authoritative board it holds in memory**, and only the server's **confirmed** result is applied — by all clients. This shift (relay → request/generate/apply/broadcast) makes the game cheat-proof and consistent.

### The mechanism, end to end
```
1. Player 1's client → ROLL {game:G, nonce:k, [clientSeed]}   (over its WebSocket)  — on its turn
2. Gateway routes it to the GAME SERVER that owns room G (registry room→node / pub-sub).
3. Game Server (in memory):
     - is it Player 1's turn?  (out of turn → REJECT)
     - GENERATE the die value from (serverSeed, clientSeed, nonce)  (§7)
     - APPLY rules: advance, overshoot check, snake/ladder jump, win check
     - append {turn, die, from, to} to the roll log (durable, async)
     - advance turn to the next player
4. Game Server BROADCASTS the confirmed turn to ALL players in the room:
     {player, die, from, to, jumped?, nextTurn, [winner]}
5. Every client animates the die + token move from the confirmed data; the next player may now roll.
```

### Room fan-out across nodes (players may be on different gateway nodes)
The players' sockets may live on different gateway nodes, but the **room** is pinned to one Game Server. So broadcast uses the same pattern as chat/spectator fan-out: the Game Server produces the confirmed turn and **publishes it to the room's channel**; the gateway nodes holding each player's socket are **subscribed** (via a registry + pub/sub backplane) and do the final socket writes. A socket is pinned to its process, so each push happens on the node owning that socket.

> 💡 **The senior one-liner:** *"A turn is a request to an authoritative Game Server, not a result the client announces. The server owns the board, checks it's your turn, **generates** the die itself, applies the rules, and broadcasts the confirmed turn to everyone in the room — routed to whatever gateway nodes hold their sockets via a pub/sub backplane. Server authority stops move-cheating; server-generated dice stop roll-cheating."*

---

## 7. Hard Problem 2: Provably-Fair Server-Authoritative Dice

This is the genuinely distinctive hard mechanism — the one that separates a toy from a real money/competitive game. The server must generate each die roll, but that creates a **trust problem in both directions**: a player must not be able to **predict or fake** a roll, *and* the player must be able to trust the **house isn't rigging** rolls against them. *How do you make randomness that neither side can cheat, and that anyone can verify afterward?*

### Why neither pure approach works
- **Client rolls** → the player cheats (always a six). Unacceptable.
- **Server rolls with a hidden RNG** → stops the player, but now the *house* is a black box: players can't tell if the server rigged an unlucky roll at a key moment. For competitive/real-money play, "trust us" isn't good enough.

### The key realization: commit-reveal makes randomness verifiable
Use a **provably-fair commit-reveal** scheme built from a keyed hash (HMAC-SHA256):
1. **Commit (before the game):** the server picks a secret **`serverSeed`** and sends its **hash** (`SHA256(serverSeed)`) — the **commitment** — to all players. The server is now *locked in*: it chose the seed before knowing anything about the game, so it **can't change it later** to rig outcomes.
2. **Client seed:** each player contributes a **`clientSeed`** (or the client picks one). This ensures the server couldn't have precomputed favorable outcomes for a *known* seed — the player influences the stream.
3. **Each roll** is deterministic: `die = map1to6( HMAC(serverSeed, clientSeed : nonce) )`, where `nonce` increments per roll. Unpredictable to the player (serverSeed is secret) and un-riggable by the server (committed).
4. **Reveal (after the game):** the server publishes `serverSeed`. Every player checks `SHA256(serverSeed) == commitment` and **recomputes every roll** to confirm it matches what was played.

```
before game:  commitment = SHA256(serverSeed)         → sent to players (serverSeed kept secret)
each roll k:  die_k = 1 + ( HMAC_SHA256(serverSeed, "clientSeed:k") reduced to 0..5 )
after game:   reveal serverSeed → verify SHA256(serverSeed)==commitment AND recompute die_1..die_n
```
(Verified: commitment matches the revealed seed; rolls recompute identically; the server committed *before* seeing the client seed/outcomes so it can't rig, and the player can't predict because the seed is secret until reveal.)

### Avoiding modulo bias (a real detail)
Mapping a hash byte to 1–6 with a naive `% 6` is slightly biased (256 isn't divisible by 6). Use **rejection sampling** — discard byte values ≥ 252 (the largest multiple of 6 ≤ 255) and take `% 6` of the rest — so all six faces are equally likely. (Verified over 600,000 rolls: each face ≈ 16.6%, within 1%.)

> 💡 **The senior one-liner:** *"The server generates the dice so players can't cheat, but I make it **provably fair** with commit-reveal: before the game the server publishes `SHA256(serverSeed)`, each roll is `HMAC(serverSeed, clientSeed:nonce)` mapped to 1–6 with rejection sampling to avoid modulo bias, and after the game the server reveals `serverSeed` so anyone can verify the commitment and recompute every roll. The server can't rig (it committed the seed before play) and the player can't predict (the seed is secret until reveal) — neither side can cheat, and it's all verifiable."*

---

## 8. Rooms & Matchmaking

Two entry paths:
- **Private room:** a host creates a room → gets a **join code** → shares it; friends join by code. Direct room creation, no queue.
- **Quick-match:** a player asks for a public game → matchmaking drops them into an **open room** with a free seat (grouped loosely by skill/coins if ranked), or opens a new room; the game starts when the room fills or a countdown expires.

```
quick_match(player, mode):
    room = find open public room for `mode` with a free seat        (Redis set of open rooms)
    if none: room = create new room (mode, capacity 2–4)
    add player to room
    if room full OR start-countdown elapsed:
        start game → commit serverSeed → assign to a Game Server (shard by game_id) → notify players
```

> 💡 Rooms are a lightweight lobby concern; the heavy lifting is the live game. Quick-match is "find an open room or make one"; private rooms are just direct creation with a code.

---

## 9. Data Contracts: Request Fields, Payloads & DB Schemas

### Part A — Key client↔server messages (over the WebSocket)
**Roll request** (client → Game Server), only valid on your turn:
| Field | Type | Purpose |
|:--|:--|:--|
| `type` | enum | `ROLL` / `JOIN` / `LEAVE` / `PING` |
| `game_id` | string | which room/game |
| `nonce` | int | the roll index (turn counter) → provably-fair input + **idempotency** (dup/stale rejected) |
| `client_seed` | string | player's contribution to the fair-dice stream (sent at join) |

**Confirmed turn** (Game Server → all players in the room):
```json
{ "type":"TURN", "game_id":"G", "turn":9, "player":"p1",
  "die":4, "from":21, "to":42, "jump":"ladder",   // 21→42 via ladder
  "positions":{"p1":42,"p2":16,"p3":7},
  "next_turn":"p2", "winner":null }
```
**Rejected roll** → `{ "type":"ROLL_REJECTED","reason":"not_your_turn|stale_nonce|game_over" }`.
**Fair-play envelope** (at start / end):
```json
start: { "type":"COMMIT", "game_id":"G", "commitment":"5e61e6c8…" }
end:   { "type":"REVEAL", "game_id":"G", "server_seed":"S3cr3t…", "rolls":[1,3,3,6,2,…] }
```

### Part B — Inter-service payloads
- **Matchmaking → Game Server (create game):** `{ game_id, players[], turn_order[], board_config_id, server_seed, commitment }`.
- **Game Server → roll log (append, async):** `{ game_id, turn, player, nonce, die, from, to }`.
- **Game Server → pub/sub `room:{id}` (fan-out to players/spectators):** the `TURN` envelope.
- **Game Server → Leaderboard (on end):** `{ game_id, winner, players[], mode }`.

### Part C — DB schema per store
**① Live game — in-memory on the owning Game Server (authoritative, ephemeral):**
```
game:{id} → { positions:{player→cell}, turn_order[], current_turn,
              board_config_id, server_seed, client_seeds{}, nonce, turn_deadline_ts }
```
**② Games / roll log — durable (result + provably-fair record):**
```
games( game_id PK, mode, players (json), winner, board_config_id,
       commitment, server_seed (revealed at end), started_at, ended_at )
rolls( game_id, turn, player, nonce, die, from_cell, to_cell,
       PRIMARY KEY (game_id, turn) )     -- replay + provably-fair verification
```
**③ Board configs — static, cached:** `board_config( id PK, snakes (json), ladders (json), size )` (many games share one classic board; custom boards possible).
**④ Rooms / matchmaking — Redis:** `open_rooms:{mode}` (set), `room:{code}→game_id`.
**⑤ Connection registry — Redis:** `conn:{user}→gateway_node`, `game:{id}→game_server_node`.
**⑥ Players / leaderboard — SQL:** wins, coins, rank.

> 💡 **The contract in one line:** *"Clients send ROLL (with a `nonce` for idempotency + fair-dice input) only on their turn; the authoritative Game Server generates the die, applies the rules in memory, and broadcasts a TURN envelope (die, from→to, jump, positions, next turn) to the whole room via pub/sub. The durable truth is a roll log + the commit/reveal seeds that make the dice verifiable; leaderboards update async. Every field exists for authority, ordering, or provable fairness."*

---

## 10. Turn Timer & Reconnection

**Turn timer.** Each turn has a deadline (e.g., 15 s). If the current player doesn't roll in time, the server **auto-rolls** (or skips) them, so one idle or disconnected player can't stall the room. Implemented like the chess clock: store `turn_deadline_ts` and schedule **one timer** per active turn — no continuous ticking. Repeated timeouts can drop a player from the room.

**Reconnection.** Because the game is durable (in-memory on the owning node + roll log), a dropped player **reconnects and pulls the current state** (all positions + whose turn + time left) and resumes. The game continues in their absence via auto-roll/skip, so the room never blocks. If the owning Game Server crashed, the room **reloads by replaying its roll log** onto a new node (positions are a deterministic function of the rolls).

> 💡 The turn timer keeps a room live despite idle/absent players (auto-roll/skip on one scheduled deadline), and reconnection is cheap because the roll log is the durable, replayable truth.

---

## 11. Scaling Summary

- **Shard rooms across Game Servers by `game_id`** — each room is an independent in-memory object on one owning node; scales linearly.
- **Keep live room state in memory** (generate + apply + broadcast in sub-ms); persist a small **roll log** off the hot path.
- **Server-authoritative + provably-fair dice** — the anti-cheat foundation (players can't fake rolls or moves; the house can't rig, and it's verifiable).
- **Turn timer is event-driven** (one scheduled deadline per turn), not continuously ticked → millions of rooms cost almost nothing to time.
- **WebSocket gateway fleet** (~6–30 nodes) with a `user→node`/`room→node` registry + pub/sub backplane for room broadcast across nodes.
- **Board configs cached/static; leaderboard async; matchmaking = find-open-room-or-create.**
- **Reconnection** by replaying the roll log; **spectator-heavy games** (if supported) use a broadcast tier.

---

## 12. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Game Server crashes | live rooms on it lost from RAM | **replay the roll log** onto a new node (positions are deterministic from rolls) |
| Player disconnects | can't take turn | turn timer **auto-rolls/skips**; reconnect pulls state + resumes; room never blocks |
| Client fakes a roll/move | cheating attempt | server **generates** the die + validates the turn → forged input rejected |
| Player claims dice were rigged | trust dispute | **commit-reveal**: reveal serverSeed → player verifies commitment + recomputes every roll |
| Duplicate/stale ROLL on retry | double-apply / disorder | `nonce`/turn idempotency — stale or duplicate rolls rejected |
| Idle player stalls room | game hangs | turn deadline → auto-roll/skip; repeated timeouts drop the player |
| Not enough players join | empty room | start-countdown → ABORT or fill with bots (product choice) |
| Leaderboard service down | stats delayed | result is durable; leaderboard update retried async (off the play path) |
| Room broadcast to different nodes | player misses a turn | pub/sub backplane + registry route the TURN to each player's gateway node; reconnect re-syncs |

---

## 13. Senior Follow-Up Questions (with Answers)

**Q1. Why can't the client roll its own dice?** It would cheat (always roll a six / pick favorable outcomes). The **server generates** every die and validates the turn; the client can only *request* a roll and render the confirmed result. (§6–§7)

**Q2. If the server rolls, how do players trust it isn't rigged?** A **provably-fair commit-reveal** scheme: the server publishes `SHA256(serverSeed)` before the game (locking in its seed), each roll is `HMAC(serverSeed, clientSeed:nonce)`, and the server reveals `serverSeed` at the end so anyone can verify the commitment and recompute every roll. Server can't rig (committed early), player can't predict (seed secret until reveal). (§7)

**Q3. How do you avoid bias mapping a hash to 1–6?** **Rejection sampling** — discard byte values ≥ 252 (largest multiple of 6 ≤ 255), then `% 6` — so all faces are equally likely. A naive `% 6` is slightly biased. (Verified: ~16.6% each over 600k rolls.)

**Q4. Where does a game live, and why?** In memory on **one owning Game Server** (pinned by `game_id`), because a room is a tiny turn-by-turn shared state; one owner keeps roll/apply/broadcast sub-ms and avoids per-turn consensus. Rooms are independent → shard trivially.

**Q5. How does a turn reach all players if they're on different nodes?** The player **requests** a roll to the owning Game Server; the server generates + applies + produces a confirmed TURN, then **publishes** it to the room's channel; the gateway nodes holding each player's socket are subscribed (registry + pub/sub) and do the final writes. (§6)

**Q6. How do you stop one idle/disconnected player from freezing the game?** A **turn timer** (one scheduled deadline per turn) auto-rolls or skips the current player on timeout; repeated timeouts drop them. The game keeps moving. (§10)

**Q7. How do you persist and recover games?** The **roll log** (game_id, turn, die) is the durable truth; positions are a deterministic function of the rolls, so replaying it reconstructs the game and reloads a crashed room onto a new node. (§10, §12)

**Q8. How does matchmaking/rooms work?** Private rooms via a join code (direct creation); quick-match = find an open public room with a free seat or create one, start when full or countdown elapses. (§8)

**Q9. What's the consistency model?** Strong within a game (single owning node, server-authoritative rolls + turns, serialized win check); eventual for leaderboards/history. Per-game strong consistency is cheap because each room is a single-writer object.

**Q10. Where's the real load?** Not roll CPU (~362K/s, served from RAM). It's holding ~**3M live connections** and fanning each turn to the room; plus the provably-fair record. Rooms shard by game_id so it scales horizontally.

**Q11. Could you add bots?** Yes — a bot is just a "player" the Game Server rolls for on a timer; useful to fill rooms or replace a dropped player. The server already generates the dice, so a bot needs no special path.

**Q12. Is a per-roll DB write needed?** No — rolls apply to in-memory state (sub-ms) and are appended to the roll log **async** for durability/verification. The DB is never on the per-roll hot path.

---

## 14. Quick Glossary
- **Server-authoritative** — the server owns the board, generates the dice, and validates turns; the client renders/requests.
- **Request-roll → generate → apply → broadcast** — the turn cycle: client requests, server generates the die and applies rules, broadcasts confirmed turn.
- **Provably-fair (commit-reveal)** — server commits `SHA256(serverSeed)` before play, reveals `serverSeed` after, so every roll is verifiable; neither side can cheat.
- **serverSeed / clientSeed / nonce** — inputs to `HMAC` producing each roll; server's is secret-then-revealed, client's influences the stream, nonce increments per roll.
- **Rejection sampling** — discarding out-of-range hash values to map uniformly to 1–6 (avoids modulo bias).
- **Roll log (event sourcing)** — ordered rolls that reconstruct positions and enable fairness verification; the durable truth.
- **Room** — a game instance of 2–4 players, pinned to one Game Server.
- **Turn timer** — one scheduled deadline per turn; auto-roll/skip on timeout so idle players don't stall the room.
- **Room fan-out** — broadcasting a confirmed turn to all players via pub/sub across gateway nodes.
- **Pub/sub backplane** — routes a confirmed turn to the gateway nodes holding each player's socket.
- **Shard by game_id** — each independent room pinned to one owning Game Server.

---

*Reference document. Contributions and corrections welcome.*
