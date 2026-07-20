# Designing an Online Chess System (chess.com / lichess): A Senior Interview Guide

> A practical, interview-focused reference for a real-time multiplayer chess platform — **matchmaking, live move exchange, server-authoritative game state, the chess clock, ratings, spectators, and anti-cheat** at scale. This guide builds the architecture up piece by piece, traces the full life of a game, goes deep on the two genuinely hard mechanisms (how a move is validated and delivered between two players, and the **server-authoritative clock / flag-fall problem** most designs botch), nails down the data contracts, and covers matchmaking, rating, spectator fan-out, reconnection, sharding, and failure modes — with verified capacity, clock, and Elo math and a senior follow-up bank.

> 💡 **The one idea:** a chess game is **authoritative server state, not a message stream.** The server holds the canonical board, **validates every move** (you cannot trust the client — that's cheating), owns the **clock**, and pushes the resulting state to both players. Get *server authority* and *the clock* right and the rest is standard real-time plumbing.

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
5. [The Game State Machine](#5-the-game-state-machine)
6. [Hard Problem 1: How a Move Is Validated and Delivered](#6-hard-problem-1-how-a-move-is-validated-and-delivered)
7. [Hard Problem 2: The Chess Clock (server-authoritative time)](#7-hard-problem-2-the-chess-clock-server-authoritative-time)
8. [Matchmaking](#8-matchmaking)
9. [Data Contracts: Request Fields, Payloads & DB Schemas](#9-data-contracts-request-fields-payloads--db-schemas)
10. [Spectators & Fan-Out](#10-spectators--fan-out)
11. [Rating (Elo / Glicko)](#11-rating-elo--glicko)
12. [Reconnection & Anti-Cheat](#12-reconnection--anti-cheat)
13. [Scaling Summary](#13-scaling-summary)
14. [Failure Modes & Handling](#14-failure-modes--handling)
15. [Senior Follow-Up Questions (with Answers)](#15-senior-follow-up-questions-with-answers)
16. [Quick Glossary](#16-quick-glossary)

---

## 1. How to Approach This in an Interview

What makes chess distinctive — and where you should spend your time — is that it's a **server-authoritative real-time game with a clock.** Unlike a chat system (where the server relays whatever you send), the chess server **owns the truth**: it holds the board, **validates every move**, and decides the clock — because a client you trust is a client that cheats. Two mechanisms carry the design: (1) the **move round-trip** (client proposes → server validates against authoritative state → server pushes confirmed state to both players and any spectators), and (2) the **clock** (who owns time, how a "flag fall" is detected fairly despite network lag, without running a live timer for every one of a million games).

A good structure:
1. **Clarify requirements** — matchmaking, time controls, live play, ratings, spectators, anti-cheat.
2. **Estimate scale** — concurrent *games* and the **move rate** (~hundreds of thousands/sec), plus millions of live connections.
3. **Establish server-authoritative game state** — the canonical board + move validation lives on the server; the client is a renderer.
4. **Go deep on the two hard mechanisms** — the validated move round-trip and the server-authoritative clock.
5. **Cover matchmaking, ratings, spectators, reconnection, anti-cheat, sharding.**

> 💡 **Senior signal:** say up front — *"The server is authoritative: it validates every move and owns the clock; the client only renders and proposes moves. The subtle part isn't relaying moves — it's the clock: deciding time fairly under network lag, and detecting flag-falls for a million concurrent games without a live timer each."*

---

## 2. Requirements

### Functional
- **Matchmaking** — pair players by rating + chosen time control (bullet/blitz/rapid); also friend challenges.
- **Play** — make legal moves, alternating turns, with a **clock** per player (base time + increment).
- **Game end** — checkmate, resignation, timeout (flag), draw (stalemate, agreement, repetition, 50-move, insufficient material).
- **Ratings** — update after each rated game (Elo/Glicko).
- **Spectate** — watch live games; **game history / PGN**; **reconnection**.

### Non-Functional
- **Low latency** — a move must feel instant (<100 ms round-trip).
- **Server-authoritative correctness** — the server validates moves and owns the clock (**anti-cheat**).
- **Fair timing** — flag-falls decided consistently despite network lag.
- **High availability** — a dropped connection must not lose the game.
- **Scale** — millions of concurrent games/connections; ~hundreds of thousands of moves/sec.

> 💡 **The split that matters:** *live play* is latency-critical and server-authoritative; *history/ratings/analysis* are batchy and eventually consistent. The design effort goes into the live-play path.

---

## 3. Capacity Estimation (verified)

**Assumptions:** 10M DAU; 5 games/user/day; ~6-min avg game; ~80 plies/game; rush-hour concentration.

```
Games/day        = 50,000,000
Concurrent games = avg ~208,000, PEAK ~1,000,000
MOVE rate        ≈ 0.22 ply/s per game → PEAK ~231,000 moves/sec   ← the real-time msg load
WebSocket conns  ≈ 2.6M at peak  (≈2.1M players + ~0.5M spectators) → gateway fleet ~6–26 nodes
Live game state  = 1M games × ~2KB ≈ 2.1 GB in memory  (shard across game servers)
Storage          = 50M games/day × ~1.5KB ≈ 77 GB/day → ~28 TB/yr  (move log is tiny; PGN reconstructs)
```

**Takeaways that shape the design:**
- **Move throughput (~231K/s) is the real-time load** — moderate, and served from **in-memory game state**, not a database round-trip per move.
- **~2.6M live connections** → a **WebSocket gateway/game-server fleet**.
- **Game state is tiny** (a board is ~70 bytes of FEN + a short move log) → **1M games fit in ~2 GB of RAM**, sharded across game-server nodes.
- **Storage is modest** — the durable record is a **move log** (event-sourced), from which any game/PGN is reconstructed.

> 💡 The numbers say: keep each live game **in memory on one node** (moves are validated and clocked against RAM state, ~sub-ms), persist only the **move log** for durability/history, and scale by **sharding games across nodes**. No database is on the per-move hot path.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

An online chess system is different from a chat app in one decisive way: **the server is the authority on the game, not a relay.** When you "make a move," you aren't sending a message to your opponent — you're **proposing** a move to the server, which checks it against the **canonical board it holds in memory** (is it your turn? is the move legal? does it leave you in check?), updates the board and the **clocks**, and only then pushes the *confirmed* new state to both players (and any spectators). This server authority is what makes the game **cheat-resistant** (a hacked client can't force an illegal move or a favorable clock) and **consistent** (there's exactly one truth). A live game is therefore a small **stateful object pinned to one game-server node** — the board, whose turn it is, and the two clocks — updated in memory (sub-millisecond) and backed by a durable **move log** so it survives crashes and reconnections. Around this sit **Matchmaking** (pair players by rating + time control), a **WebSocket gateway** (hold the live connections and route moves), a **Rating** service (update Elo/Glicko when a game ends), **spectator fan-out** (broadcast confirmed moves to watchers), and **anti-cheat** (server validation live + statistical engine-detection offline). And because each game is independent, the system **shards games across nodes** trivially.

### 4.2 The diagram

```
 player A ── WebSocket ──┐                       ┌── WebSocket ── player B
                         ▼                       ▼
                  ┌───────────────────────────────────┐
                  │   Gateway (WebSocket fleet)          │  registry(user→node) + pub/sub backplane
                  └───────┬──────────────────┬──────────┘
             join queue   │                  │ move (propose)
                          ▼                  ▼
                 ┌──────────────┐   ┌───────────────────────────┐  validate + clock (in RAM)
                 │ Matchmaking   │   │  Game Server (authoritative)│──────────► [Move Log / Game DB]
                 │ (rating+TC     │   │  board(FEN), turn, clocks   │            (durable; event-sourced)
                 │  queues)       │   │  MOVE VALIDATOR (rules)     │
                 └──────┬────────┘   └───────┬──────────┬─────────┘
                  create game                 │ confirmed move       │ on game end
                        ▼                      ▼ (push both players)  ▼
                 [assign to a Game Server]   [Spectator fan-out]   ┌──────────────┐
                 (shard by game_id)          (pub/sub → watchers)   │ Rating (Elo/  │
                                                                    │ Glicko), async│
                                                                    └──────────────┘
```

The key visual idea: a move is a **propose → validate-in-RAM → confirm → push** cycle owned by the **Game Server** that holds that game; the durable **move log** is written for recovery/history but is *not* on the latency-critical path; spectators and rating hang off to the side.

### 4.3 Each component, in detail

**① Client (the board UI).** A **renderer + proposer**, not an authority. It displays the board and a *local, approximate* clock (for smooth UX), lets the user drag a piece, and sends the **proposed move** over its WebSocket. It applies only what the server **confirms** — if the server rejects the move (illegal / not your turn), the client rolls back. It never decides legality or time.

**② Gateway (WebSocket fleet).** Terminates the ~2.6M live connections and routes each player's move to the **Game Server that owns their game**, via a `user→node` (or `game→node`) registry + pub/sub backplane. Dumb on purpose; heartbeats + reconnect-resume handle mobile churn.

**③ Matchmaking Service.** Players enter a **queue keyed by (time control, rating bucket)**; it pairs the closest-rated waiting players, widening the rating window over time, and on a match **creates a game** and assigns it to a Game Server (§8).

**④ Game Server (the authority — the heart).** Holds each live game **in memory**: the board (FEN + position history for repetition), whose turn it is, and the two **clocks**. For each proposed move it runs the **Move Validator** (legal-move generation / rules engine), and if legal, applies it, **updates the clocks** (§7), detects end conditions (checkmate/stalemate/draw/flag), appends to the **move log**, and pushes the confirmed state to both players + spectators. A game is **pinned to one node** so its state has a single owner (no distributed consensus per move).

**⑤ Move Validator (rules engine).** Server-side legal-move logic: is it this player's turn, is the move legal for that piece, does it leave the king in check, castling/en-passant/promotion rules, and end-state detection. **This is the anti-cheat foundation** — the client's claim is never trusted.

**⑥ Move Log / Game DB (durable, event-sourced).** The append-only list of moves (+ clock times) is the **source of truth for history and recovery** — replaying it reconstructs any position or the PGN. Metadata (players, result, time control, ratings) in a relational/wide-column store. Written per move for durability but off the hot path (async/batched).

**⑦ Rating Service.** On game end, computes Elo/Glicko updates and persists new ratings — asynchronous (not on the move path).

**⑧ Spectator fan-out.** Broadcasts each confirmed move to watchers via pub/sub; featured games (top GMs) can fan out to huge audiences via a broadcast/CDN tier (§10).

**⑨ Anti-cheat.** Live: the Move Validator blocks illegal moves. Offline: statistical **engine-detection** (move-quality vs rating, timing patterns) runs in a batch pipeline (§12).

### 4.4 The end-to-end life of a game

Here is exactly what happens from **"Play" to a finished, rated game**, step by step:

```
MATCH:
1.  Player A clicks "Play (Blitz 5+3)" → Matchmaking enqueues A in the (blitz, ~1500) bucket.
2.  Player B (rating ~1520) is waiting there → Matchmaking PAIRS them, creates game G
        (assigns colors, starting position, clocks 300+3), and ASSIGNS G to Game Server node N (shard by game_id).
3.  Both apps are told "game G on node N" → both open/route a WebSocket to N; N loads G into memory.

A MOVE (the propose → validate → confirm → push cycle):
4.  It's White's (A's) turn; A drags e2→e4 → client sends PROPOSE {game:G, move:"e2e4", client_ts}.
5.  Game Server N: is it A's turn? is e2e4 legal on the authoritative board? → YES.
        - UPDATE the board (now Black to move), UPDATE clocks (deduct A's elapsed think time, add increment — §7)
        - detect end conditions (check/mate/draw?) → none
        - APPEND the move (+ clock) to the move log (durable, async)
        - PUSH confirmed state {move, new FEN or SAN, clocks, whose turn} to A and B (and spectators).   ← §6
6.  B sees the move + updated clocks instantly; B replies; repeat 4–5, alternating.

END:
7.  A plays a move that is checkmate (or B resigns / B's flag falls — §7).
        Game Server marks G finished with the result, writes the final move-log + metadata.
8.  Rating Service (async) computes Elo/Glicko deltas and updates both players' ratings.
9.  The game is now a durable PGN in history; the in-memory game object is freed from node N.
```

The single most important thing to notice: **step 5 is entirely in memory on one node** — validation and clocking happen against RAM state in sub-milliseconds; the durable write is a side-effect for recovery, *not* a blocking step. That's what keeps a move feeling instant while the game stays cheat-proof and crash-recoverable.

### 4.5 Why this split? (the design rationale)

- **Server authoritative, client a renderer** — if the client decided legality or time, cheating would be trivial (force illegal moves, fake the clock). Server authority is the whole anti-cheat and consistency foundation.
- **Game pinned to one Game Server (in memory)** — a chess game is a tiny, highly-interactive shared state mutated ~twice a second; putting it on **one owning node** avoids per-move distributed consensus and keeps validation/clocking sub-millisecond. Games are independent, so this shards perfectly.
- **Move log (event-sourced) separate from live state** — the durable truth is the *sequence of moves*; replaying it reconstructs any position and survives crashes/reconnects — but it's written off the hot path so it never adds move latency.
- **Rating asynchronous** — it only matters at game end and is a pure function of the result; keeping it off the move path keeps play snappy.
- **Gateway separate and dumb** — holding millions of connections is a different problem from game logic; isolating it lets it scale and restart freely.

### 4.6 Where the load actually goes

- **Moves:** ~**231K/s** at peak — served from **in-memory game state** (validate + clock + push), ~sub-ms each. The database is **not** per-move (only an async move-log append). This is very manageable.
- **Connections:** ~**2.6M** WebSockets → a **gateway/game-server fleet of ~6–26 nodes**; connection-holding is the bigger infra concern than move CPU.
- **Live state:** ~**2 GB** for 1M games — trivial; the constraint is *ownership* (each game on one node), not memory.
- **Spectators (the deceptive spike):** a normal game has ~0 watchers, but a **top-GM game can draw hundreds of thousands** — a fan-out hotspot handled by a broadcast tier, not the game server itself (§10).
- **Storage:** ~**28 TB/yr** of move logs — modest and highly compressible.

> 💡 **The senior framing:** *"Move throughput is moderate and served from RAM — each game is a tiny in-memory object on one owning node, validated and clocked in sub-ms, with the move log persisted off the hot path. The real infra load is holding ~2.6M live connections and handling spectator fan-out spikes on featured games. Games shard trivially because each is independent."*

---

## 5. The Game State Machine

A game is a durable state machine (event-sourced by its move log):

```
MATCHING ─► IN_PROGRESS ─► FINISHED ─► RATED
                 │
                 ├─► (each move) alternate turn, update clocks, check end conditions
                 └─► end via: CHECKMATE | RESIGN | TIMEOUT(flag) | DRAW(stalemate/agreement/
                                                                   repetition/50-move/insufficient)
   ABORTED  (e.g., a player never made a first move)
```

- **MATCHING → IN_PROGRESS**: matchmaking pairs players, game created on a Game Server.
- **IN_PROGRESS**: alternating validated moves; clocks tick for the player to move; end-conditions checked after each move.
- **FINISHED → RATED**: result recorded; ratings updated (async).

> 💡 Every transition is **server-decided and persisted to the move log**, so a crashed Game Server reloads the game by replaying moves, and disputes are auditable.

---

## 6. Hard Problem 1: How a Move Is Validated and Delivered

The move round-trip is the core loop, and where server authority + cross-node delivery meet. *When A drags a piece, how does a legal, confirmed move reach B (who may be on a different node), and how is cheating prevented?*

### The naive (wrong) approach
"A sends the move; the server forwards it to B; each client updates its own board." This is a **relay**, and it's broken for a game: a hacked client A could send an **illegal move** (or claim a capture that didn't happen), and B's client would apply it — trivial cheating. There's also **no single source of truth**, so the two boards can diverge.

### The key realization: propose, don't send
A move is a **proposal to the authority**, not a message to the opponent. The **Game Server owns the canonical board**; A's client sends "I want to play e2e4," and the server **validates it against that board** before anything is confirmed. Only the server's *confirmed* state is applied — by both clients. This single shift (relay → propose/validate/confirm) is what makes the game cheat-proof and consistent.

### The mechanism, end to end
```
1. A's client → PROPOSE {game:G, move:"e2e4", client_ts}  (over A's WebSocket)
2. Gateway routes it to the GAME SERVER that owns G (registry game→node / pub-sub).
3. Game Server (in memory):
     - is it A's turn?  is e2e4 legal on G's authoritative board?  (Move Validator)
     - illegal / not your turn → REJECT (tell A to roll back); nothing else changes.
     - legal → apply it: new board, flip turn, UPDATE CLOCKS (§7), detect end conditions,
               append to the move log (durable, async).
4. Game Server PUSHES the CONFIRMED state to BOTH players (and spectators):
     {move, resulting position, clocks, whose turn, [result if game ended]}.
5. Both clients apply the confirmed state. A's optimistic render is now server-confirmed;
   B sees the move + updated clocks.
```

### Cross-node delivery (A and B may be on different gateway nodes)
Both players' connections don't have to live on the same node — but the **game does** (it's pinned to one Game Server). So delivery is the same pattern as chat's cross-gateway problem: the Game Server produces the confirmed move and **publishes it**; the **gateway nodes holding A's and B's sockets are subscribed** (via a `user→node` registry + pub/sub backplane) and do the final socket write. A WebSocket is pinned to its process, so the push must happen on the node owning each socket — the backplane carries the confirmed move there.

> 💡 **The senior one-liner:** *"A move is a proposal to an authoritative Game Server, not a message to the opponent. The server owns the board, validates the move (illegal or out-of-turn → rejected), updates the board and clocks in memory, and pushes the confirmed state to both players — routed to whatever gateway nodes hold their sockets via a pub/sub backplane. Server authority is the anti-cheat and consistency foundation; the client only renders and proposes."*

---

## 7. Hard Problem 2: The Chess Clock (server-authoritative time)

This is the genuinely chess-specific hard mechanism — the one most designs botch. Each player has a clock (e.g., **5 min + 3 s increment**); the player to move burns time; if it hits zero, they **flag** (lose on time). *Who owns the clock, how is a flag-fall decided fairly despite network lag, and how do you do this for a million games without a live timer each?*

### Why the client can't own the clock
If clients ran the clock, a cheater would simply stop it. And two clients + network lag would **disagree** on how much time is left. So the **server is authoritative on time**, exactly as it is on the board. The client shows a smooth *local approximation* for UX, but the server decides.

### The key realization: time is computed lazily, not ticked
You do **not** run a per-game background timer decrementing a clock every 100 ms — that's a million live timers, wasteful and unscalable. Instead, the server **stores each clock's remaining time + a timestamp of when the current turn started**, and computes elapsed time **only at two moments**:
1. **On each move** — deduct `(server_receive_time − turn_start_time)` from the mover's clock, check for flag, then add the increment and start the opponent's turn.
2. **For a flag while no move comes** — schedule **one timer** at `turn_start + mover_remaining`; if it fires before a move arrives, the mover has flagged.

```
on move by P at server time T:
    P.remaining -= (T - turn_start)          # time ran while it was P's turn
    if P.remaining <= 0:  → P FLAGGED (loss on time)
    P.remaining += increment                 # increment after completing the move
    turn_start = T ;  switch turn to opponent
    (re)schedule the single flag-timer at turn_start + opponent.remaining
```
(Verified: 300+3 blitz — after 4 plies clocks are W=295, B=299; white thinking 301 s → **white flagged at −1.0**. And flag detection needs just **one scheduled timer** per game at the current mover's deadline, not continuous ticking.)

### Lag fairness
Because the server timestamps **when it receives** the move, a player isn't charged for the server→client→server network round-trip in a way that steals their time — good implementations credit a small **lag compensation** (e.g., don't deduct the measured network latency, up to a cap). The principle: **the server's receive-time is truth**, with a bounded lag credit so a laggy connection doesn't unfairly flag a player who moved in time.

### Why this scales
Because time is **computed on events** (a move, or one scheduled deadline) rather than **ticked continuously**, a Game Server handles thousands of games with almost no timing overhead — the clock is just arithmetic on two numbers plus a single scheduled check per game.

> 💡 **The senior one-liner:** *"The server owns the clock. I don't tick a timer per game — I store each clock's remaining time plus a turn-start timestamp, deduct elapsed on each move (server receive-time is truth, with a bounded lag credit so network delay doesn't steal time), and schedule exactly one flag-timer at the current mover's deadline. That's cheat-proof, fair under lag, and scales to a million games because it's event-driven arithmetic, not continuous ticking."*

---

## 8. Matchmaking

Players pick a **time control** (bullet 1+0, blitz 5+3, rapid 10+0, …) and enter a **queue keyed by (time control, rating bucket)**:
```
1. Enqueue player P in bucket (blitz, floor(rating/100)) with P.rating and enqueue_time.
2. Pair P with the closest-rated waiting player in nearby buckets.
3. Widen the acceptable rating window as wait time grows (start ±50, expand to ±200…),
   so nobody waits forever in a thin bucket.
4. On a pair: create the game (assign colors, starting clocks), assign it to a Game Server
   (shard by game_id), tell both clients "game G on node N."
```
Friend challenges bypass the queue (direct game creation). Anti-abuse: prevent re-queue sniping, rating manipulation.

> 💡 Matchmaking is a **rating-bucketed queue with a widening window** — closest rating for a fair game, widened over time so wait stays bounded. It's a small service; the heavy lifting is the live game.

---

## 9. Data Contracts: Request Fields, Payloads & DB Schemas

### Part A — Key client↔server messages (over the WebSocket)
**Propose a move** (client → Game Server):
| Field | Type | Purpose |
|:--|:--|:--|
| `type` | enum | `MOVE` / `RESIGN` / `OFFER_DRAW` / `CLAIM_DRAW` / `JOIN` / `PING` |
| `game_id` | string | which game |
| `move` | string | UCI/SAN, e.g. `"e2e4"`, `"e7e8q"` (promotion) |
| `move_seq` | int | client's expected ply number → **idempotency/ordering** (reject stale/duplicate) |
| `client_ts` | epoch ms | for lag estimation (server time is authoritative) |

**Confirmed move** (Game Server → both players + spectators):
```json
{ "type":"MOVE_CONFIRMED", "game_id":"G", "ply":9, "move":"e2e4", "san":"e4",
  "fen":"rnbqkbnr/…", "turn":"black",
  "clocks":{"white_ms":295000,"black_ms":299000},
  "result":null }            // e.g. "1-0 checkmate" when the game ends
```
**Rejected move** → `{ "type":"MOVE_REJECTED","game_id":"G","reason":"illegal|not_your_turn|stale" }` (client rolls back).

### Part B — Inter-service payloads
- **Matchmaking → Game Server (create game):** `{ game_id, white_id, black_id, time_control:{base_ms,inc_ms}, start_fen }`.
- **Game Server → Move Log (append, async):** `{ game_id, ply, move, clock_white_ms, clock_black_ms, server_ts }`.
- **Game Server → pub/sub `game:{id}` (spectator + cross-node player push):** the `MOVE_CONFIRMED` envelope.
- **Game Server → Rating (on end):** `{ game_id, white_id, black_id, result, time_control }`.

### Part C — DB schema per store
**① Live game — in-memory on the owning Game Server (authoritative, ephemeral):**
```
game:{id} → { fen, turn, white_clock_ms, black_clock_ms, turn_start_ts,
              position_history[] (for repetition), draw_offer }
```
**② Move log / games — durable, event-sourced (source of truth for history):**
```
games( game_id PK, white_id, black_id, time_control, result, rated,
       white_rating_before, black_rating_before, started_at, ended_at )
moves( game_id, ply, move, clock_white_ms, clock_black_ms, server_ts,
       PRIMARY KEY (game_id, ply) )        -- replaying moves reconstructs any position / PGN
INDEX (white_id, started_at DESC)           -- a player's game history
```
**③ Players / ratings — SQL:**
```
players( user_id PK, username, rating_blitz, rating_rapid, rd (Glicko deviation), games_played )
```
**④ Matchmaking queue — Redis:** `ZSET queue:{time_control}:{bucket}` (member=user, score=enqueue_ts).
**⑤ Connection registry — Redis:** `conn:{user_id} → gateway_node`  and  `game:{game_id} → game_server_node` (route moves/pushes).

> 💡 **The contract in one line:** *"Clients PROPOSE moves (with a `move_seq` for idempotency); the authoritative Game Server validates against in-memory state, updates board + clocks, and pushes a CONFIRMED envelope (new FEN, clocks, turn) to both players and spectators via pub/sub. The durable truth is an event-sourced move log (game_id, ply, move, clocks) that reconstructs any PGN; ratings update async. Server time and server validation are authoritative — every field exists for authority, ordering, or reconstruction."*

---

## 10. Spectators & Fan-Out

A normal game has ~0 watchers, but a **top-GM game can draw hundreds of thousands** — a fan-out hotspot. Handle it like other real-time broadcasts:
- Confirmed moves are **published to a per-game channel**; spectator connections **subscribe** and receive the same `MOVE_CONFIRMED` stream players get (read-only).
- For **featured games**, add a **broadcast/CDN tier** (a fan-out fabric or edge-cached move stream) so the single Game Server isn't writing to 200k sockets directly — it publishes once, the fan-out tier multiplies.
- Spectators can **join late** and get the current position (FEN) + then the live stream.

> 💡 The game server pushes to two *players* directly; **massive spectator audiences go through a fan-out/broadcast tier**, so a viral game doesn't overload the node owning it.

---

## 11. Rating (Elo / Glicko)

On a **rated** game's end, update both players' ratings — **asynchronously**, off the move path:
```
Elo:  Ea = 1 / (1 + 10^((Rb - Ra)/400))         # A's expected score
      Ra' = Ra + K * (Sa - Ea)                    # Sa = 1 win / 0.5 draw / 0 loss
```
(Verified: 1500 beating 1700 → expected 0.24, new **1515** (+15); 1500 losing to 1300 → expected 0.76, new **1485** (−15).) **Glicko/Glicko-2** is preferred in practice (adds a rating *deviation* RD capturing uncertainty, so provisional/inactive players move faster). Ratings are per time-control (separate blitz/rapid/bullet ratings).

> 💡 Rating is a **pure function of the result**, computed once at game end and persisted async — never on the move path. Elo to explain the idea; Glicko for the real system (it models uncertainty).

---

## 12. Reconnection & Anti-Cheat

**Reconnection.** Because the game is durable (in-memory on the owning node + move log), a dropped player **reconnects, pulls the current state** (FEN + clocks + whose turn), and resumes. The **clock keeps running** during a disconnect (that's the rule — your time ticks whether or not your socket is up), though a short grace/abort window applies to first moves. If the owning Game Server crashed, the game **reloads by replaying its move log** onto a new node.

**Anti-cheat (two layers).**
- **Live (synchronous):** the server-authoritative Move Validator makes illegal moves *impossible*, and the server clock makes time-cheating impossible. This is the foundation.
- **Offline (asynchronous):** **engine-detection** — statistical analysis of move quality vs the player's rating, centipawn-loss patterns, timing regularity, and correlation with known engine lines — runs in a batch pipeline; suspicious accounts are flagged/reviewed. You can't catch strong engine use at move-time, so it's a data pipeline problem.

> 💡 Server authority prevents *rule* cheating outright; *engine* cheating is caught statistically offline. Reconnection is cheap because the move log is the durable truth — replay to rebuild.

---

## 13. Scaling Summary

- **Shard games across Game Servers by `game_id`** — each game is an independent in-memory object on one owning node; scales linearly.
- **Keep live game state in memory** (validate + clock in sub-ms); persist an **event-sourced move log** off the hot path.
- **Server-authoritative everything** (moves, clock) — the anti-cheat + consistency foundation.
- **Clock is event-driven** (deduct on move + one scheduled flag-timer), not continuously ticked → a million games cost almost nothing to time.
- **WebSocket gateway fleet** (~6–26 nodes) with a `user→node` / `game→node` registry + pub/sub backplane for cross-node move delivery.
- **Spectator fan-out via a broadcast tier** for featured games (viral-proof).
- **Ratings async**; **matchmaking** = rating-bucketed queues with a widening window.
- **Reconnection** by replaying the move log; **anti-cheat** live (validator) + offline (engine-detection pipeline).

---

## 14. Failure Modes & Handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Game Server crashes | live games on it lost from RAM | **replay the move log** onto a new node; clocks resume from last persisted times |
| Player disconnects | can't move | clock keeps running; reconnect pulls state + resumes; grace/abort on first move |
| Illegal/forged move from client | cheating attempt | server Move Validator **rejects**; client rolled back (impossible to apply) |
| Client clock disagrees with server | dispute | **server time is authoritative**; client clock is display-only, with lag credit |
| Flag while opponent idle | who wins on time? | **one scheduled flag-timer** at the deadline decides; server-authoritative |
| Featured game spectator spike | node overload | **broadcast/CDN fan-out tier** — publish once, multiply at the edge |
| Duplicate/stale move on retry | double-apply / disorder | `move_seq`/ply idempotency — stale or duplicate plies rejected |
| Rating service down | ratings delayed | game result is durable; rating update retried async (off the play path) |
| Both offer/claim draw races | inconsistent end | server serializes end-condition checks on the owning node (single writer) |

---

## 15. Senior Follow-Up Questions (with Answers)

**Q1. Why is the server authoritative, not a relay?** Because a trusted client cheats — it could force illegal moves or fake its clock. The server owns the board and clock, **validates every move**, and pushes confirmed state; the client only renders and proposes. That's the anti-cheat + consistency foundation.

**Q2. Where does a game live, and why?** In memory on **one owning Game Server** (pinned by `game_id`), because a game is a tiny, highly-interactive shared state mutated ~twice/second; a single owner avoids per-move distributed consensus and keeps validation/clocking sub-ms. Games are independent → shard trivially.

**Q3. How does a move get from A to B if they're on different nodes?** A **proposes** to the Game Server owning the game; the server validates and produces a confirmed move, then **publishes** it; the gateway nodes holding A's and B's sockets are subscribed (registry + pub/sub backplane) and do the final writes. A socket is pinned to its process, so the push happens on each owning node. (§6)

**Q4. How do you handle the clock — who owns time and how do you scale it?** The **server** owns time. It stores each clock's remaining time + a turn-start timestamp, **deducts elapsed on each move** (server receive-time is truth, with a bounded lag credit), and schedules **one flag-timer** at the mover's deadline. No continuous ticking → a million games are cheap to time. (§7)

**Q5. How is a flag (timeout) detected fairly under lag?** The server timestamps when it *receives* the move and deducts that; a bounded lag credit prevents network delay from stealing time. A flag while nobody moves is caught by the single scheduled deadline timer. Server time decides — never the client.

**Q6. How do you persist and recover games?** An **event-sourced move log** (game_id, ply, move, clocks) is the durable truth; replaying it reconstructs any position/PGN and reloads a crashed game onto a new node. Live state is in RAM for speed; the log is written off the hot path.

**Q7. How does matchmaking work?** Rating-bucketed queues per time control; pair the closest-rated waiting players; **widen the window** as wait grows so nobody waits forever. On a pair, create the game and assign it to a Game Server.

**Q8. How do you scale to huge spectator counts on a featured game?** The game server publishes each move **once** to a per-game channel; a **broadcast/CDN fan-out tier** multiplies it to hundreds of thousands of watchers, so the owning node isn't writing to 200k sockets directly.

**Q9. How do ratings work and where do they run?** Elo/Glicko as a **pure function of the result**, computed **async at game end** (never on the move path); separate ratings per time control; Glicko adds a deviation term for uncertainty.

**Q10. How do you prevent cheating?** Live: server-authoritative validation makes illegal moves/time-cheating impossible. Offline: statistical **engine-detection** (move quality vs rating, timing) in a batch pipeline flags suspicious accounts — you can't catch engine use at move-time.

**Q11. What's the consistency model?** Strong within a game (single owning node, server-authoritative moves + clock, serialized end conditions); eventual for ratings/history/analysis. Per-game strong consistency is cheap because each game is a single-writer object.

**Q12. Why not validate moves on the client for lower latency?** You can *optimistically render* on the client for smoothness, but the **server must confirm**, or cheating is trivial and the two boards can diverge. Optimistic render + server confirm (rollback on reject) gives both speed and safety.

---

## 16. Quick Glossary
- **Server-authoritative** — the server owns the board and clock and validates every move; the client renders/proposes.
- **Propose → validate → confirm → push** — the move round-trip: client proposes, server validates in RAM, pushes confirmed state.
- **Move Validator** — server-side legal-move/rules engine; the live anti-cheat foundation.
- **Game state (FEN)** — the canonical board + turn, held in memory on the owning Game Server.
- **Move log (event sourcing)** — append-only moves that reconstruct any position/PGN; the durable truth.
- **Clock / flag-fall** — per-player time (base + increment); running out = loss on time (flag).
- **Lazy clock** — remaining-time + turn-start timestamp; deduct on move + one scheduled flag-timer (no continuous ticking).
- **Lag compensation** — bounded credit so network latency doesn't steal a player's time; server receive-time is truth.
- **Matchmaking** — rating-bucketed queues per time control with a widening window.
- **Elo / Glicko** — rating updates from the result (Glicko adds a deviation for uncertainty).
- **Spectator fan-out** — broadcasting confirmed moves to watchers; a broadcast/CDN tier for featured games.
- **Pub/sub backplane** — routes a confirmed move to the gateway nodes holding each socket.
- **Shard by game_id** — each independent game pinned to one owning Game Server.

---

*Reference document. Contributions and corrections welcome.*
