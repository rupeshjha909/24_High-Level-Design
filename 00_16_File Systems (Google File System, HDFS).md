# Distributed File Systems (GFS & HDFS): A Ground-Up Guide

> A practical reference for how large-scale distributed file systems work — the GFS master/chunkserver design and the insight that makes a single master viable, why chunks are huge, the workload assumptions that drive every trade-off, the relaxed consistency model, HDFS as its open-source descendant, and the modern shift to disaggregated storage. With trade-offs and interview prep.

---

## Table of Contents

1. [Why a Distributed File System?](#1-why-a-distributed-file-system)
2. [The Big Idea: Separate Metadata from Data](#2-the-big-idea-separate-metadata-from-data)
3. [GFS Architecture](#3-gfs-architecture)
4. [Why These Design Choices?](#4-why-these-design-choices)
5. [Read & Write (Append) Flows](#5-read--write-append-flows)
6. [The Relaxed Consistency Model](#6-the-relaxed-consistency-model)
7. [Failure Handling](#7-failure-handling)
8. [HDFS: The Open-Source Descendant](#8-hdfs-the-open-source-descendant)
9. [Modern Distributed File Systems & the Cloud Shift](#9-modern-distributed-file-systems--the-cloud-shift)
10. [Interview-Ready Insights](#10-interview-ready-insights)
11. [Quick Glossary](#11-quick-glossary)

---

## 1. Why a Distributed File System?

A single machine — no matter how large — cannot store petabytes, nor can it serve the read/write throughput that big-data and web-scale workloads demand. A **distributed file system (DFS)** spreads files across many machines while presenting them as one logical file system. The motivations:

- **Capacity beyond one machine** — store petabytes by pooling the disks of thousands of commodity servers.
- **Fault tolerance** — at that scale, **hardware failure is the norm, not the exception** (with thousands of disks, something is always dying), so the system must tolerate constant failures without losing data or stopping.
- **Parallel reads & high throughput** — spreading a file across many machines lets many readers pull different parts in parallel, optimizing for **aggregate throughput** rather than single-request latency.

The defining design philosophy, pioneered by Google's GFS (2003 paper) and copied by HDFS: **build reliability in software on top of cheap, unreliable commodity hardware**, assuming failure and engineering around it — rather than buying expensive "reliable" hardware (which is the storage analog of the distributed-replication-over-RAID argument from the storage-types guide).

---

## 2. The Big Idea: Separate Metadata from Data

The single most important architectural insight in GFS/HDFS — and the thing to lead with in an interview — is the **separation of the control plane from the data plane**:

- A **master** (GFS) / **NameNode** (HDFS) holds only **metadata**: the directory tree and the mapping of *file → chunks → which servers hold them*. It never touches the actual file bytes.
- Many **chunkservers** (GFS) / **DataNodes** (HDFS) hold the **actual data**, split into chunks.
- **Clients ask the master *where* the data is, then read/write the data *directly* from the chunkservers.**

This separation is what makes the whole thing scale. Because the master is off the data path — it just answers "where is it?" — it isn't a throughput bottleneck even though it's a single node, and the bulk data flows in parallel directly between clients and the many chunkservers. Keep this control-plane/data-plane split in mind; it explains nearly every other design choice.

---

## 3. GFS Architecture

```
[Client] ──(1) "where is file X, chunk 5?"──► [Master]   (single; metadata only, in RAM)
   │                                              │
   │  ◄──(2) chunk handle + replica locations─────┘
   │
   └──(3) read/write data directly──► [ChunkServer A] [ChunkServer B] [ChunkServer C]
                                         (each 64 MB chunk replicated 3×)
```

- **Master (single):** holds all metadata **in memory** for speed — the namespace, file→chunk mapping, and chunk→location data. Assigns chunks to chunkservers and orchestrates replication.
- **Chunkservers (many):** store **64 MB chunks** on local disks; each chunk is **replicated 3×** across different servers for durability.
- **Clients:** consult the master for locations (and cache them), then exchange data **directly** with chunkservers — the master is never in the data path.

---

## 4. Why These Design Choices?

Every GFS decision flows from its **workload assumptions** — and recognizing this co-design is a strong senior signal. GFS was built for Google's specific workload: **huge files, append-mostly writes (logs, crawled web pages), and large sequential reads, prioritizing throughput over latency.** Given that:

### Large 64 MB chunks (vs. typical KB-sized FS blocks)

- **Shrinks metadata** — fewer chunks per file means the master can hold *all* metadata in RAM (the master's scalability ceiling is metadata size).
- **Reduces client↔master chatter** — a client gets a big chunk's location once and reads a lot from it.
- **Suits large sequential reads.** The cost: **terrible for many small files** (metadata bloat, wasted space, hot chunks) — the famous "small files problem."

### Single master

- **Simplicity.** One authority for metadata avoids the complexity of distributed consensus for the namespace. It works *because* the master is metadata-only and off the data path (Section 2), and because clients cache locations to avoid hammering it.
- The cost: the master is a **single point of failure** (addressed in Section 7) and a metadata-scalability limit.

### Clients talk directly to chunkservers

- Keeps the master out of the high-bandwidth data path, so **data throughput scales with the number of chunkservers**, not the master.

The lesson: **knowing your access pattern lets you make aggressive trade-offs.** GFS is *not* a general-purpose file system — it deliberately sacrifices small-file performance, low latency, and strong consistency to excel at its actual workload.

---

## 5. Read & Write (Append) Flows

### Read

1. Client asks the **master**: "where is chunk 5 of file X?"
2. Master returns the **chunk handle and replica locations** (clients cache this to avoid repeat lookups).
3. Client reads the data from the **nearest chunkserver** directly.

### Write / Append

Because the workload is append-mostly, appends are the important path:

1. The master designates one replica as the **primary** for the chunk (grants it a lease).
2. The client **pushes the data to all replicas**, pipelined (data flows server-to-server in a chain to use bandwidth efficiently, decoupled from control).
3. The client tells the **primary** to commit; the primary **serializes the order** of concurrent appends and tells the secondaries to apply them in that same order, ensuring all replicas agree on ordering.

Separating the **data flow** (pipelined to all replicas) from the **control flow** (primary serializes the mutation order) is a key technique — bandwidth is used efficiently while a single primary keeps replicas consistent in ordering.

---

## 6. The Relaxed Consistency Model

A deliberate and often-overlooked GFS trade-off: it provides a **relaxed (weak) consistency model**, not the strong consistency of a local filesystem. Specifically, its **record append** guarantees data is written **at least once** atomically, but this can leave **duplicates** and **padding** in the file, and concurrent writes can produce regions that are "consistent but undefined."

GFS pushes the burden of handling this to **applications**: they're expected to tolerate duplicates (e.g., via record checksums and unique IDs to dedupe) and to use append rather than random overwrite. This is the **CAP-style trade-off** applied to a filesystem — GFS sacrifices strong consistency for **availability, simplicity, and throughput at scale**, betting (correctly, for its workload) that applications can cope with weaker guarantees more cheaply than the system can provide strong ones.

---

## 7. Failure Handling

At GFS scale, failures are routine, so recovery is automatic:

- **Chunkserver failure** — chunkservers send **heartbeats** to the master. When one stops, the master notices its chunks are now under-replicated and **re-replicates** them elsewhere to restore the 3× factor. Data survives because every chunk has 3 copies.
- **Data integrity** — chunkservers checksum chunks to detect silent disk corruption and report bad chunks for re-replication.
- **Master failure** — originally the master was a **single point of failure**. Mitigations: the master logs metadata changes to a replicated **operation log**, and **shadow/backup masters** can take over (read-only shadows for availability; a new master can be reconstructed from the log). This SPOF is the design's weakest point — and exactly what HDFS later hardened.

---

## 8. HDFS: The Open-Source Descendant

**HDFS (Hadoop Distributed File System)** is essentially an open-source reimplementation of GFS, and it's the storage backbone of the Hadoop ecosystem.

The mapping is nearly one-to-one:

| GFS | HDFS |
|-----|------|
| Master | **NameNode** (metadata) |
| Chunkserver | **DataNode** (data blocks) |
| Chunk (64 MB) | **Block (128 MB)** |
| 3× replication | 3× replication |

- **Larger 128 MB blocks** (vs GFS's 64 MB) — further reducing metadata and favoring large sequential I/O.
- **High-throughput, batch-oriented** — optimized for large scans, **not** low-latency random access; ideal for analytics, not for serving interactive requests.
- **Used by:** Hadoop MapReduce, Hive, Spark, and the broader big-data stack.
- **NameNode HA evolution:** like GFS, the NameNode began as a single point of failure. Modern HDFS runs **active/standby NameNodes** sharing an edit log via **JournalNodes**, with **ZooKeeper**-based automatic failover — removing the original SPOF.

A famous inherited weakness: the **"small files problem"** — millions of tiny files explode NameNode metadata (each file/block needs a metadata entry held in RAM), so HDFS strongly prefers fewer, larger files.

---

## 9. Modern Distributed File Systems & the Cloud Shift

The landscape has broadened beyond GFS-style designs:

- **Ceph** — a unified system providing **object, block, and file** interfaces (covering all three storage types from the storage-types guide) over one distributed cluster, with no single metadata bottleneck (it uses an algorithm, CRUSH, to compute data placement rather than a central master).
- **GlusterFS** — a **POSIX-compliant** distributed filesystem that scales linearly, presenting a standard filesystem interface.
- **AWS S3** — **object storage**, now the dominant choice for cloud-scale unstructured data and big-data lakes.

### The big architectural shift: disaggregating compute and storage

HDFS was built around **data locality** — co-locate compute (MapReduce tasks) with the data (DataNodes) to avoid moving petabytes over the network. The modern cloud pattern **disaggregates** them: store data in **object storage (S3)** and run **separate, elastic compute** (Spark, query engines) against it. The trade-off:

- *HDFS/locality:* moving compute to data minimizes network transfer — great when network was the bottleneck.
- *S3/disaggregation:* compute and storage scale **independently** (spin up compute on demand, pay only for what you use; storage persists cheaply), and fast cloud networks have made "fetch from object storage" acceptable. This decoupling is why much of the industry has moved from HDFS toward S3-backed data lakes.

So the trajectory runs from "a single master coordinating co-located chunkservers" (GFS/HDFS) toward "elastic compute over cheap, decoupled object storage" (the cloud model) — though GFS's core ideas (chunking, replication, metadata/data separation, building reliability on commodity hardware) remain foundational.

---

## 10. Interview-Ready Insights

**Q: What's the core architectural idea of GFS/HDFS?**
Separating the **control plane** (a master/NameNode holding only metadata, off the data path) from the **data plane** (many chunkservers/DataNodes holding replicated data). Clients ask the master *where* data is, then transfer it *directly* with chunkservers — so the single master isn't a throughput bottleneck and data flows scale with the number of chunkservers.

**Q: Why such large chunks/blocks (64/128 MB)?**
To shrink metadata (so the master can hold it all in RAM — its scalability limit), reduce client↔master interaction, and suit large sequential reads. The cost is the "small files problem": many tiny files bloat metadata and waste space.

**Q: Why is a single master acceptable, and what's the risk?**
It's acceptable because the master is metadata-only and off the data path, and clients cache locations. The risk is that it's a SPOF and a metadata-scaling limit — mitigated by operation logs and shadow masters in GFS, and by active/standby NameNodes with JournalNodes + ZooKeeper failover in modern HDFS.

**Q: Describe GFS's consistency model.**
Relaxed/weak: record append guarantees at-least-once atomic appends but allows duplicates and padding, and concurrent writes can leave "undefined" regions. Applications must tolerate this (checksums, unique IDs, prefer append). It trades strong consistency for availability, simplicity, and throughput — the CAP trade-off applied to a filesystem.

**Q: How does it handle failures?**
Heartbeats detect dead chunkservers; the master re-replicates under-replicated chunks to maintain 3 copies; checksums catch corruption. Every chunk having 3 replicas across machines is what makes constant hardware failure survivable (the distributed-replication-over-RAID principle).

**Q: How do GFS design choices reflect its workload?**
They're co-designed: huge files, append-mostly, large sequential reads, throughput over latency → large chunks, append-optimized writes, relaxed consistency, single master. The lesson: knowing your access pattern lets you make aggressive, otherwise-unacceptable trade-offs. GFS is a special-purpose, not general-purpose, filesystem.

**Q: HDFS vs the modern S3-based approach?**
HDFS co-locates compute with data (locality) to avoid moving petabytes. The cloud disaggregates them: data in S3, elastic compute (Spark) on top, scaling independently and paying per use. Fast networks made this viable, which is why the industry has largely shifted from HDFS to S3-backed data lakes.

---

## 11. Quick Glossary

- **Distributed file system (DFS)** — files spread across many machines presented as one filesystem.
- **Control plane / data plane** — the metadata-coordination layer vs. the bulk-data layer.
- **Master / NameNode** — the single node holding metadata (namespace, file→chunk→location).
- **Chunkserver / DataNode** — nodes storing the actual data chunks/blocks.
- **Chunk / Block** — the unit a file is split into (GFS 64 MB, HDFS 128 MB).
- **Replication factor** — copies of each chunk (typically 3) across different machines.
- **Heartbeat** — periodic signal chunkservers send so the master knows they're alive.
- **Re-replication** — restoring the replica count after a chunkserver fails.
- **Primary replica / lease** — the replica that serializes the order of mutations for a chunk.
- **Pipelined write** — pushing data through replicas in a chain to use bandwidth efficiently.
- **Relaxed consistency** — weak guarantees (at-least-once append, possible duplicates/padding).
- **Small files problem** — many tiny files bloating master/NameNode metadata.
- **Data locality** — co-locating computation with the data to avoid network transfer.
- **Disaggregated storage** — separating compute from storage (e.g., Spark over S3).
- **Ceph / GlusterFS** — modern distributed storage (unified multi-interface / POSIX-compliant).

---

*Reference document. Contributions and corrections welcome.*
