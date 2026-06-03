# How to Scale a Database: A Ground-Up Guide

> A practical reference for scaling a database past a single machine — vertical scaling, read replicas, sharding, partitioning, leader election, and indexing — organized around the one question that decides everything: *what is the bottleneck — reads, writes, or data size?* With trade-offs and interview prep.

---

## Table of Contents

1. [The First Question: What's the Bottleneck?](#1-the-first-question-whats-the-bottleneck)
2. [Vertical Scaling (Scale-Up)](#2-vertical-scaling-scale-up)
3. [Read Replicas (Replication)](#3-read-replicas-replication)
4. [Sharding (Horizontal Partitioning)](#4-sharding-horizontal-partitioning)
5. [Partitioning (Vertical & Horizontal)](#5-partitioning-vertical--horizontal)
6. [Leader Election](#6-leader-election)
7. [Indexing](#7-indexing)
8. [The Decision Tree](#8-the-decision-tree)
9. [Interview-Ready Insights](#9-interview-ready-insights)
10. [Quick Glossary](#10-quick-glossary)

---

## 1. The First Question: What's the Bottleneck?

Scaling a database isn't one technique — it's a toolbox, and choosing the right tool starts with diagnosing **what is actually overloaded.** There are three distinct bottlenecks, and each has a different answer:

- **Read load** too high? → **Read replicas** and **caching** (spread/offload reads).
- **Write load** too high? → **Sharding** (split writes across machines).
- **Data size** too big for one machine? → **Sharding / partitioning** (split the data).

This framing prevents the classic mistake of reaching for sharding (the most painful option) when you merely have a read problem that a replica or a cache would solve cheaply. The techniques also form a rough **escalation order** — exhaust the simpler, less invasive ones before the drastic ones (mirroring the 0-to-million scaling guide): vertical scaling → read replicas → caching → sharding. Each step buys headroom and exposes the next bottleneck.

---

## 2. Vertical Scaling (Scale-Up)

Buy a **bigger machine** — more CPU, more RAM, faster SSD.

- **Pros:** the easiest option — **no code or architecture change**, just a larger box. Always the first thing to try because it's nearly free in engineering effort.
- **Cons:** hits a **hardware ceiling** quickly (there's a biggest machine you can buy), top-end hardware gets disproportionately expensive, and it's still **one machine** — a single point of failure with no added fault tolerance.

Vertical scaling buys time, not a final answer. When you outgrow it, you move to *horizontal* techniques (more machines).

---

## 3. Read Replicas (Replication)

The answer to a **read-heavy** workload. Keep one **primary (master)** for writes and add **replicas (slaves)** that copy its data and serve reads.

```
[Writer] ──► [Primary]
                 │ replicates
        ┌────────┼────────┐
        ▼        ▼        ▼
   [Replica 1][Replica 2][Replica 3]   ← reads spread across these
```

Writes go to the primary; reads fan out across replicas, multiplying read capacity. Since most applications are read-heavy, this is often the highest-leverage scaling step.

### The replication-lag catch

Replication is usually **asynchronous**, so replicas trail the primary slightly (**replication lag**). A read from a replica right after a write might return **stale** data — replicas are **eventually consistent**. This is the **CAP trade-off in practice** (CAP guide): you gain read scale and availability at the cost of immediate consistency. The common workaround for "read-your-own-writes" is to **route reads that need freshness to the primary** (or use sticky/monotonic reads).

### Replication modes — a durability vs. latency vs. availability spectrum

| Mode | Guarantee | Pros | Cons |
|------|-----------|------|------|
| **Async** | primary doesn't wait for replicas | fast writes, high availability | **data loss** possible if the primary crashes before replicas catch up |
| **Semi-sync** | at least one replica confirms | bounded data-loss risk | slightly slower writes |
| **Sync** | all replicas confirm before ack | **no data loss** | slow writes; a replica failure can **block writes** (availability cost) |

This is the same durability/latency tension as Kafka's `acks`/ISR (message-queue guide): the more replicas you wait for, the safer the data but the slower and less available the write. Most systems pick **async or semi-sync** as a pragmatic middle.

---

## 4. Sharding (Horizontal Partitioning)

When **write throughput maxes out the single primary** (replicas don't help writes — every write still hits the primary) or the **data no longer fits one machine**, you **shard**: split the data across multiple independent databases, each handling its own subset of writes and storage.

The defining decision is the **shard key** — the attribute that decides which shard a row lives on. This choice governs everything: how evenly load spreads, which queries are cheap, and whether you get hotspots (the same critical decision as a partition key in Kafka/Cassandra — message-queue and KV-store guides). A bad shard key creates a **hot shard** that bottlenecks the whole system.

### Sharding strategies

**1. Range-based** — assign contiguous ranges to shards (`user_id 1–1M → shard1`, etc.).
- *Pros:* simple; **range queries are efficient** (a range lives on one shard).
- *Cons:* **hotspots** if data is skewed — e.g., sharding by date/sequential ID sends all recent (hottest) activity to one shard.

**2. Hash-based** — `shard = hash(key) % N`.
- *Pros:* **even distribution** (hashing scatters keys).
- *Cons:* **range queries become scatter-gather** (data is spread everywhere), and changing N (`% N`) **reshuffles almost everything** — the exact problem consistent hashing solves.

**3. Consistent hashing** — places shards and keys on a ring so only **~1/N keys move** when adding/removing a shard (see the consistent-hashing guide). **Best for elastic clusters** that grow and shrink.

**4. Geographic** — shard by region (`us-east`, `eu-west`, `ap-south`).
- *Pros:* **data locality** (users hit a nearby shard) and **compliance** (data residency / GDPR).
- *Cons:* cross-region queries/joins are slow.

**5. Directory-based** — a **lookup table** maps `key → shard`.
- *Pros:* **most flexible** (you can move data and just update the mapping).
- *Cons:* extra indirection, and the directory itself becomes critical infrastructure to keep available.

### Sharding's challenges (why it's the last resort)

Sharding is powerful but introduces the hardest problems in distributed data:

- **Cross-shard joins** — data that needs joining may live on different shards; you resort to **application-level joins** or **denormalization** (store data together to avoid the join).
- **Cross-shard transactions** — no single ACID transaction spans shards; you need **distributed transactions** (slow) or the **Saga pattern** (microservices guide) — eventual consistency with compensating actions.
- **Resharding** — changing the shard scheme means moving huge amounts of data; **painful, so plan the scheme up front** (and consistent hashing eases it).
- **Hotspots** — mitigate by splitting hot shards or replicating hot keys.

Because of all this, sharding is the **nuclear option** — exhaust vertical scaling, replicas, and caching first.

---

## 5. Partitioning (Vertical & Horizontal)

"Partitioning" splits a table, and the term is used two ways — distinguish them clearly (a common interview point):

### Vertical partitioning — split by **columns**

Break a wide table into narrower ones by column groups:

```
User  →  user_basic(id, name, email)
      +  user_profile(id, bio, photo_url)
      +  user_preferences(id, settings)
```

Useful when some columns are **large or rarely accessed** (e.g., a big `bio`/photo) — keep the hot, small columns in one table for fast access and isolate the cold, bulky ones.

### Horizontal partitioning — split by **rows, within the same database**

Divide a table's rows into partitions (e.g., by date: one partition per month) **inside a single DB**. This is **different from sharding**: horizontal partitioning keeps everything on **one machine/DB** (a single-server optimization for query pruning and maintenance), while **sharding spreads partitions across multiple machines.** Sharding is essentially horizontal partitioning taken across servers.

---

## 6. Leader Election

In a replicated system, **exactly one node must be the leader (primary)** at a time — the one that accepts writes and coordinates. If the leader fails, the system needs **automatic failover** to elect a new one. The danger if this goes wrong is **split-brain**: two nodes both believing they're leader, accepting conflicting writes, and corrupting data.

### Algorithms

- **Raft** — designed to be understandable; used by **etcd, Consul, CockroachDB**.
- **Paxos** — the classic (and famously subtle) consensus algorithm; **Google Chubby**.
- **ZooKeeper-based** — coordination service used by older Kafka, HBase.
- **Bully algorithm** — simple election (highest-ID node wins); common in academic settings.

### How Raft works (high level)

1. All nodes start as **followers**.
2. If a follower hears no **heartbeat** from a leader for a timeout, it becomes a **candidate** and **requests votes** from the others.
3. A candidate that wins a **majority** of votes becomes the **leader** and starts sending heartbeats to assert its authority.
4. A recovered old leader, seeing a newer term, **steps down** to follower.

The crucial mechanism is the **majority (quorum)** requirement: because a candidate needs more than half the nodes to win, **two leaders can't be elected simultaneously** (two overlapping majorities are impossible) — this prevents split-brain. It's the same quorum/overlap logic as the `R + W > N` rule in the KV-store guide, and it's why these systems need an **odd number of nodes** (3, 5) to form clear majorities. During a partition, only the side with a majority can elect a leader — the minority side stops accepting writes, a deliberate **choice of consistency over availability** (CAP).

---

## 7. Indexing

Before scaling out, make each query efficient. An **index** is an auxiliary structure that speeds up reads — but at a cost: **every write must also update the indexes** (write amplification) and indexes consume storage. So indexing is fundamentally a **read-speed vs. write-speed/storage trade-off**: index what you query, but don't over-index.

### Index types (and where each fits)

- **B-Tree** — the default; balanced tree supporting **equality and range** queries efficiently. What most relational DBs use (and the contrast to LSM from the KV-store guide — B-trees favor reads, LSM favors writes).
- **Hash index** — **O(1) equality** lookups, but **no range queries** (hashing destroys order).
- **LSM tree** — **write-optimized** (Cassandra, RocksDB); see the KV-store guide.
- **Inverted index** — maps **term → list of documents**; the basis of full-text search (Elasticsearch) and the autocomplete guide's search infrastructure.
- **Bitmap index** — efficient for **low-cardinality** columns (gender, status) where few distinct values exist.
- **GIN / GiST (PostgreSQL)** — specialized indexes for full-text, geospatial, and JSONB data.

### Index design rules

1. **Index columns used in `WHERE`, `JOIN`, and `ORDER BY`** — those are what the DB filters/sorts on.
2. **Leftmost-prefix rule for composite indexes** — an index on `(a, b, c)` helps queries filtering on `a`, `(a, b)`, or `(a, b, c)`, but **not** on `b` alone — it can only use the index left-to-right.
3. **More indexes = slower writes** — every insert/update maintains every affected index, so each index taxes write throughput.
4. **Covering indexes** — if an index contains *all* columns a query needs, the DB answers from the index alone (an **index-only scan**), skipping the table lookup entirely — a powerful optimization.

---

## 8. The Decision Tree

A practical order of operations, tying the techniques together:

```
Is the workload read-heavy?
   ├─ YES → add Read Replicas (and/or a cache)
   └─ NO  → ↓
Is the dataset large / writes maxed out?
   ├─ YES → Shard / Partition
   └─ NO  → ↓
Is the DB still struggling?
   ├─ YES → add a Caching Layer (Redis)
   └─ STILL? → consider NoSQL or specialized stores (search, time-series, graph)
```

The spirit of the tree: **diagnose the bottleneck, apply the cheapest technique that addresses it, re-measure, escalate only if needed.** Don't shard for a read problem; don't add replicas for a write problem; cache aggressively before re-architecting; and switch data stores (SQL-vs-NoSQL guide) only when your access pattern genuinely outgrows the relational model.

---

## 9. Interview-Ready Insights

**Q: How do you approach scaling a database?**
First diagnose the bottleneck — reads, writes, or data size — because each has a different fix: replicas/cache for reads, sharding for writes/size. Then escalate from cheapest to most invasive: vertical scaling → read replicas → caching → sharding. Don't shard for a read problem.

**Q: Read replicas vs sharding?**
Replicas copy the whole dataset to scale **reads** (writes still hit one primary); sharding splits data into subsets to scale **writes and storage**. Add replicas first (simple); shard last (it brings cross-shard joins, distributed transactions, and resharding pain).

**Q: Async vs sync replication?**
Async: fast, available writes, but possible data loss if the primary crashes before replicas catch up. Sync: no data loss, but slow and a replica failure can block writes. Semi-sync (wait for one replica) is the common middle. It's a durability-vs-latency-vs-availability trade-off, like Kafka's acks/ISR.

**Q: How do you choose a shard key, and what goes wrong?**
Pick a key that spreads load evenly and keeps commonly-queried data together. A poor key creates a hot shard (e.g., sharding sequential/date keys concentrates recent traffic). Hash-based gives even spread but breaks range queries and reshuffles on resize; consistent hashing fixes the resize problem; range-based eases range queries but risks hotspots.

**Q: Sharding vs partitioning?**
Vertical partitioning splits a table by columns; horizontal partitioning splits by rows within one DB; sharding spreads partitions across multiple machines. Sharding is horizontal partitioning taken across servers — and brings the distributed-systems hard problems with it.

**Q: Why is leader election needed and how does Raft avoid split-brain?**
A replicated system needs exactly one leader for writes; failover must elect a new one automatically. Raft uses majority-vote elections — since two overlapping majorities are impossible, two leaders can't coexist, preventing split-brain. During a partition only the majority side elects a leader; the minority stops writing (consistency over availability).

**Q: What's the trade-off with indexing?**
Indexes speed reads but slow writes (every write maintains them) and use storage. Index columns in WHERE/JOIN/ORDER BY, respect the leftmost-prefix rule for composite indexes, avoid over-indexing, and use covering indexes for index-only scans.

---

## 10. Quick Glossary

- **Vertical scaling (scale-up)** — a more powerful single machine.
- **Horizontal scaling (scale-out)** — more machines working together.
- **Read replica** — a copy of the primary serving read queries.
- **Replication lag** — delay before a replica reflects the primary's latest writes.
- **Async / semi-sync / sync replication** — wait for zero / one / all replicas before acking a write.
- **Sharding** — splitting data across multiple databases/machines by a shard key.
- **Shard key** — the attribute deciding which shard a row lives on.
- **Hot shard** — an overloaded shard from uneven key distribution.
- **Scatter-gather** — querying all shards and merging results (needed when data is spread).
- **Vertical partitioning** — splitting a table by columns.
- **Horizontal partitioning** — splitting a table by rows within one database.
- **Leader election** — choosing the single node that accepts writes; needs automatic failover.
- **Split-brain** — two nodes both acting as leader, risking conflicting writes.
- **Quorum / majority** — more than half the nodes; prevents two simultaneous leaders.
- **Raft / Paxos** — consensus algorithms for leader election and agreement.
- **Index** — auxiliary structure speeding reads at the cost of write speed and storage.
- **B-Tree / Hash / LSM / inverted / bitmap index** — index types for different access patterns.
- **Leftmost-prefix rule** — a composite index is usable left-to-right only.
- **Covering index** — an index containing all columns a query needs (index-only scan).

---

*Reference document. Contributions and corrections welcome.*
