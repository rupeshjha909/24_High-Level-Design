# Consistent Hashing: A Ground-Up Guide

> A practical reference on the technique that lets distributed systems add and remove servers without reshuffling all their data — why naive `hash % N` fails, how the hash ring fixes it, what virtual nodes add, and where it's used. With worked examples, code sketch, and interview prep.

---

## Table of Contents

1. [The Problem It Solves](#1-the-problem-it-solves)
2. [Why Naive Hashing Breaks](#2-why-naive-hashing-breaks)
3. [The Hash Ring](#3-the-hash-ring)
4. [Adding and Removing Servers](#4-adding-and-removing-servers)
5. [The Uneven-Load Problem & Virtual Nodes](#5-the-uneven-load-problem--virtual-nodes)
6. [Handling Replication](#6-handling-replication)
7. [Where It's Used](#7-where-its-used)
8. [Implementation Sketch](#8-implementation-sketch)
9. [Interview-Ready Insights](#9-interview-ready-insights)
10. [Quick Glossary](#10-quick-glossary)

---

## 1. The Problem It Solves

When you distribute data across **N servers** (a cache cluster, a sharded database, a CDN), you need a rule for deciding *which server owns which key*. The obvious rule is:

```
server = hash(key) % N
```

This works perfectly — until **N changes**. And N changes all the time: servers crash, you add capacity for a traffic spike, you remove idle nodes to save cost. The moment N changes, the naive scheme **reshuffles almost every key to a different server.**

In a cache, that means a near-total **cache miss storm** — suddenly almost nothing is where the formula now says it should be, so every request misses the cache and hammers the database, often hard enough to take it down (a "thundering herd"). In a sharded database, it means migrating nearly all your data.

**Consistent hashing** is the fix. Its defining property: when you add or remove a server, only about **1/N of the keys move** — just the keys that "belong" to the changed server — instead of nearly all of them. That single property is the entire reason it exists.

---

## 2. Why Naive Hashing Breaks

It's worth seeing *just how bad* `hash % N` is, because the contrast is the whole motivation.

Suppose N = 4 servers, and we have keys whose hashes are 10, 11, 12, 13:

```
hash % 4:   10→2   11→3   12→0   13→1
```

Now one server dies, so N = 3:

```
hash % 3:   10→1   11→2   12→0   13→1
```

Three of the four keys (10, 11, 13) changed servers — and we only *removed* one node. In general, changing N invalidates roughly **(N-1)/N** of all key→server mappings. The modulo ties every key's location to the *total count* of servers, so changing that count disturbs everything. Consistent hashing breaks that dependency.

---

## 3. The Hash Ring

The core idea is to map **both servers and keys onto the same circular space** — a "ring" — and let each key belong to the nearest server going one direction around the ring.

**Step by step:**

1. Imagine a ring of all possible hash values, e.g. `0` to `2³² − 1`, wrapping around so the largest value is adjacent to `0`.
2. Place each **server** on the ring at position `hash(server_id)`.
3. Place each **key** on the ring at position `hash(key)`.
4. Each key is owned by the **first server found going clockwise** from the key's position. (If you walk clockwise past the top of the ring, you wrap around to the first server.)

```
                 key1
                  ↓ (walk clockwise → lands on ServerB)
        ServerA ───────────── ServerB
              │                 │
              │                 │
        ServerD ───────────── ServerC
```

The crucial difference from `hash % N`: a key's owner depends only on **where the nearest server sits on the ring**, *not* on the total number of servers. So changing the server count only affects the region of the ring near the change — everything else stays put.

---

## 4. Adding and Removing Servers

This is where consistent hashing pays off. Because ownership is "next server clockwise," a change only disturbs **one arc** of the ring.

**Adding a server.** Place `ServerE` on the ring between `ServerD` and `ServerA`. Now only the keys that fall in the arc *between ServerD and ServerE* move — they used to walk clockwise to ServerA, but now hit ServerE first. **Every other key is untouched.** You've taken on new capacity by moving only a small slice of data.

```
   before:  ...ServerD ───────────────── ServerA...   (arc owned by A)
   after:   ...ServerD ──── ServerE ───── ServerA...   (only this slice moves to E)
```

**Removing a server.** If `ServerB` leaves (crash or decommission), only the keys it owned move — they now walk clockwise to the *next* server after B. Keys belonging to A, C, D are all unaffected.

In both cases the disturbance is roughly **1/N of the keys**, compared to the ~all-of-them churn of naive hashing. That's the whole win.

---

## 5. The Uneven-Load Problem & Virtual Nodes

There's a catch with the basic ring. If you place each server at just **one** random point, the arcs between servers come out **uneven** — by chance, one server might own a huge stretch of the ring and another a tiny sliver. That means **lopsided load**: the unlucky server gets far more keys than its peers. And when a server is removed, *all* its load dumps onto the single next server rather than spreading out.

**Virtual nodes (vnodes)** solve this. Instead of placing each physical server once, you place it at **many points** around the ring (e.g., 100–200 virtual positions per physical server, each at `hash(server_id + i)`). The ring is now finely interleaved with little arcs from every server.

```
   without vnodes:   A........B..C.................D     (uneven arcs)
   with vnodes:      A.B.D.C.A.D.B.C.A.B.D.C.A.C.B.D     (evenly mixed)
```

Benefits:

- **Even load distribution** — many small arcs per server average out to roughly equal total ownership.
- **Smoother rebalancing** — when a server is added or removed, its (or its replacement's) many small arcs are scattered around the ring, so the moved load is **spread across all the remaining servers** instead of dumped on one neighbor.
- **Heterogeneous capacity** — a more powerful machine can be given *more* vnodes, so it naturally takes a proportionally larger share of the load.

Virtual nodes are why consistent hashing works well in practice; almost every real implementation uses them.

---

## 6. Handling Replication

Real systems don't store each key on just one server — they replicate it for durability and availability. Consistent hashing extends naturally: after finding the key's primary server (first clockwise), you also store copies on the **next few distinct servers** continuing clockwise around the ring. So a key with replication factor 3 lives on the first three distinct physical servers encountered walking clockwise from the key's position.

This is exactly how Dynamo-style systems pick their replica sets, and it means replica placement inherits the same "only 1/N moves on change" property.

---

## 7. Where It's Used

- **Amazon DynamoDB** — distributes partition keys across nodes using consistent hashing.
- **Apache Cassandra** — its "token ring" *is* a consistent-hash ring; each node owns token ranges (with vnodes by default).
- **Memcached clients (ketama)** — the classic client-side library that spreads keys across cache servers so adding/removing a cache node doesn't wipe the whole cache.
- **Akamai CDN** — routing content across edge servers.
- **Discord** — routing/sharding for its real-time infrastructure.

The common thread: all of these need to add and remove nodes routinely without a catastrophic data reshuffle — precisely consistent hashing's superpower.

---

## 8. Implementation Sketch

The whole thing reduces to a **sorted structure of ring positions** plus a "find the next position ≥ my hash" lookup.

```
SETUP
  ring = sorted map { position → server }
  for each server:
      for i in 0..V (number of vnodes):
          pos = hash(server_id + ":" + i)
          ring[pos] = server

LOOKUP(key)
  h = hash(key)
  find the smallest position in ring with position >= h
  if none (h is past the last position):  wrap around → use the first position
  return ring[that position]
```

The key operation is "find the first ring entry ≥ hash(key), wrapping around if needed." That's a **ceiling lookup on a sorted set**, which is why the natural data structures are:

- **Java:** `TreeMap` — use `ceilingKey(h)`, and if it returns null, take `firstKey()`.
- **Python:** the `bisect` module on a sorted list of positions, or a `SortedDict` from the `sortedcontainers` library.
- **Go:** a sorted slice with `sort.Search`.

Lookups are O(log V·N) thanks to the sorted structure; adding/removing a server inserts/removes its V vnode entries.

---

## 9. Interview-Ready Insights

**Q: What problem does consistent hashing solve?**
With naive `hash(key) % N`, changing N (adding/removing a server) reshuffles ~all keys, causing a cache-miss storm or massive data migration. Consistent hashing moves only ~1/N of keys when the cluster changes.

**Q: How does it achieve that?**
By mapping servers *and* keys onto the same hash ring and assigning each key to the next server clockwise. A key's owner depends on its neighbor on the ring, not on the total server count — so a change only disturbs one arc.

**Q: What are virtual nodes and why do you need them?**
Placing each server at a single ring point gives uneven arcs and lopsided load, and dumps all of a removed server's load on one neighbor. Vnodes place each server at many points, which evens out load, spreads rebalancing across all nodes, and lets you weight stronger machines with more vnodes.

**Q: How much data moves when you add the (N+1)th server?**
Roughly 1/(N+1) of the keys — just the arcs the new server takes over — versus nearly everything under modulo hashing.

**Q: What data structure implements the ring?**
A sorted map / tree (Java `TreeMap`, Python `bisect`/`SortedDict`) supporting a "ceiling" lookup: find the smallest ring position ≥ hash(key), wrapping to the first position if you run off the end.

**Q: How does replication fit in?**
Store the key on the next *R* distinct servers clockwise from its position — that's the replica set, and it keeps the same minimal-movement property.

**Q: Where is it used in the real world?**
DynamoDB partitioning, Cassandra's token ring, Memcached's ketama clients, CDNs like Akamai, and Discord's routing.

---

## 10. Quick Glossary

- **Consistent hashing** — a hashing scheme where adding/removing a node moves only ~1/N of keys.
- **Hash ring** — the circular hash space (e.g., 0 to 2³²−1) onto which servers and keys are placed.
- **Clockwise assignment** — a key is owned by the first server encountered going clockwise from it.
- **Rehashing / reshuffle** — relocating keys to new servers when the cluster changes.
- **Cache-miss storm / thundering herd** — mass cache misses (from a reshuffle) overwhelming the backend.
- **Virtual node (vnode)** — one of many ring positions assigned to a single physical server for even load.
- **Shard key / partition key** — the value hashed to place data; here, hashed onto the ring.
- **Replication factor (R)** — how many distinct servers store a copy of each key.
- **Ceiling lookup** — finding the smallest ring position ≥ a given hash (the core ring operation).

---

*Reference document. Contributions and corrections welcome.*
