# Storage Types (Block, File, Object) & RAID: A Ground-Up Guide

> A practical reference for the three storage abstractions — block, file, and object — what each exposes and when to use it, plus RAID levels for disk redundancy and why modern systems prefer distributed replication. With trade-offs and interview prep.

---

## Table of Contents

1. [The Core Question: How Is Data Accessed?](#1-the-core-question-how-is-data-accessed)
2. [Block Storage](#2-block-storage)
3. [File Storage](#3-file-storage)
4. [Object Storage](#4-object-storage)
5. [Comparison & How to Choose](#5-comparison--how-to-choose)
6. [RAID: Disk-Level Redundancy](#6-raid-disk-level-redundancy)
7. [RAID vs Distributed Replication](#7-raid-vs-distributed-replication)
8. [Interview-Ready Insights](#8-interview-ready-insights)
9. [Quick Glossary](#9-quick-glossary)

---

## 1. The Core Question: How Is Data Accessed?

The three storage types — block, file, and object — aren't really about *where* data lives; they're about **the abstraction through which you access it.** Each sits at a different level, trading **control and performance** against **simplicity and scale.**

- **Block storage** gives you **raw blocks** — like a bare hard drive. You bring your own filesystem. Maximum control and performance, minimum convenience.
- **File storage** gives you **someone else's filesystem over the network** — folders and files you mount and share. Convenient and familiar.
- **Object storage** gives you **no filesystem at all** — just blobs addressed by a key over HTTP. Maximum scale and lowest cost, least control over layout.

The progression is "lower-level and faster" → "higher-level and more scalable." Picking the right one is mostly about matching this abstraction to your access pattern, which the rest of this guide walks through.

---

## 2. Block Storage

**What it is:** raw **block devices** — the storage presents fixed-size blocks with no filesystem of its own, exactly like a physical or virtual hard drive. The **host** attaches the device and **formats it** with whatever filesystem it wants, then reads/writes blocks directly.

- **No filesystem on the storage itself** — the OS/application on the host imposes structure and manages the data layout.
- **Low latency, high IOPS** — direct block access with minimal overhead makes it the fastest of the three, and it supports **random, in-place updates** (overwrite a block without rewriting the whole object).
- **Examples:** AWS EBS, Azure Managed Disks, iSCSI SANs.
- **Use case:** **databases** and **VM disks** — anything that needs fast random access, fine-grained in-place writes, and full control over data layout.

**Why databases need it:** a database manages its own on-disk structures (pages, indexes, write-ahead logs) and needs fast random reads/writes with in-place updates — exactly what raw blocks provide and what object storage (replace-whole-blobs over HTTP) cannot.

---

## 3. File Storage

**What it is:** a **hierarchical filesystem** — the familiar tree of folders and files — exposed over the network via protocols like **NFS** (Unix) or **SMB** (Windows). Clients **mount** it and use it like a local drive, but it's shared.

- **Hierarchy** — folders and files, paths, permissions; the model everyone already knows.
- **Concurrent multi-client access** — many machines can mount the same share at once, making it a natural **shared filesystem**.
- **Examples:** AWS EFS, NFS, GlusterFS, Azure Files.
- **Use case:** shared content across servers, user **home directories**, CMS/upload directories, legacy apps that expect a POSIX filesystem.

The trade-off: more convenient and shareable than block storage, but the filesystem layer and network protocol add overhead, and it doesn't scale as limitlessly or cheaply as object storage.

---

## 4. Object Storage

**What it is:** a **flat namespace** of **objects**, where each object = **data + metadata + a globally unique key**. There are no folders (the "folder" in an S3 key is just part of the key string), and you access objects over **HTTP/REST** (the S3 API).

- **Flat keys, not a tree** — you `GET`/`PUT` an object by its key, like a giant distributed HashMap for blobs (it's essentially a **key-value store for large objects** — see the KV-store guide — accessed over HTTP).
- **Objects are typically immutable** — you replace a whole object rather than editing it in place. This immutability is what enables the massive scale and durability.
- **Effectively infinite scale, very cheap** — designed to scale to exabytes with high durability (many replicas/erasure coding behind the scenes), at the lowest cost per byte of the three.
- **Rich metadata** travels with each object.
- **Examples:** AWS S3, Google Cloud Storage, Azure Blob, MinIO.
- **Use case:** backups, media files (images/video), data lakes, static website hosting, and as a **CDN origin** (see the CDN guide) — i.e., large, unstructured, write-once-read-many data.

The trade-off: HTTP access and no in-place edits make it **higher-latency** and unsuitable for things like database storage, but unbeatable for storing and serving large blobs at scale.

---

## 5. Comparison & How to Choose

| | **Block** | **File** | **Object** |
|-|-----------|----------|------------|
| Access | Raw block device | NFS / SMB | HTTP / REST |
| Hierarchy | None (host adds FS) | Tree (folders) | Flat (keys) |
| Mutability | In-place block writes | File edits | Replace whole object |
| Performance | Highest (low latency, high IOPS) | Medium | Lower (HTTP) |
| Scale | Per-volume limit | Multi-PB | ~Exabyte |
| Cost | High | Medium | Low |
| Use case | Databases, VM disks | Shared filesystems | Media, backups, data lakes |

**How to choose:**
- Need **fast random access and in-place updates** (a database, a VM's disk)? → **Block.**
- Need a **shared filesystem** many machines mount, with a familiar folder hierarchy? → **File.**
- Need to store **huge amounts of unstructured blobs** cheaply and serve them over HTTP (media, backups, static assets)? → **Object.**

A useful heuristic: the more your workload looks like "a program managing its own data structure with frequent small writes," the more you want **block**; the more it looks like "store and fetch whole files/blobs," the more you want **object**, with **file** in between for shared-filesystem semantics.

---

## 6. RAID: Disk-Level Redundancy

**RAID (Redundant Array of Independent Disks)** combines multiple physical disks into one logical unit for **performance, redundancy, or both.** Three primitives underlie all the levels:

- **Striping** — split data across disks so reads/writes happen in parallel → **speed**, but no protection.
- **Mirroring** — keep identical copies on multiple disks → **redundancy**, but you pay in capacity.
- **Parity** — store computed redundancy data so a failed disk can be **reconstructed** from the others → redundancy with less capacity overhead than mirroring, at a write-time computation cost.

| Level | Technique | Pros | Cons |
|-------|-----------|------|------|
| **RAID 0** | Striping | Fast (parallel I/O), full capacity | **No redundancy** — one disk fails = all data lost |
| **RAID 1** | Mirroring | Redundancy; simple; fast reads | **50% capacity** lost to the mirror |
| **RAID 5** | Striping + single parity (3+ disks) | Good capacity/redundancy balance; survives 1 failure | **Slow writes** (parity recompute); risky rebuilds |
| **RAID 6** | Striping + double parity | Survives **2** simultaneous failures | More space overhead; even slower writes |
| **RAID 10** | Stripe **+** mirror (1+0) | Fast **and** safe; quick rebuilds | **Expensive** (50% capacity to mirroring) |

**Notes that matter:**
- **RAID 0 is not redundancy** — it's pure speed, and it *increases* failure risk (any disk dying loses everything). Don't be fooled by it being "RAID."
- **RAID 5's decline:** as disks grew huge, rebuilding a failed disk takes many hours, during which a *second* failure (or unreadable sector) destroys the array — so **RAID 6** (double parity) became preferred for large arrays.
- **The parity write penalty:** every write in RAID 5/6 must read old data/parity and recompute — why parity levels have slower writes than mirroring.

---

## 7. RAID vs Distributed Replication

A crucial modern point: **RAID protects against *disk* failure within a *single machine* — but not against the machine, rack, or datacenter failing.** If the server holding the RAID array dies, catches fire, or loses power, the redundancy is useless.

Large-scale and cloud systems therefore prefer **distributed replication**: store multiple copies of data on **different machines** (and different racks/availability zones). For example, **HDFS replicates each block 3× across nodes** rather than relying on RAID, and the distributed key-value stores from the KV-store guide replicate each key to N nodes via consistent hashing.

```
RAID:           [ one machine ]  →  protects against a DISK dying
                 disk disk disk      (but not the machine dying)

Replication:    [machine A] [machine B] [machine C]  →  protects against a whole
                 copy        copy        copy            MACHINE / rack / DC dying
```

The trade-off and reasoning:
- RAID is **simpler and storage-efficient** for a single server, still common in on-prem/NAS setups.
- Distributed replication survives **whole-machine and site failures**, scales horizontally, and treats individual disks and servers as **disposable** — which is the cloud-scale philosophy (and ties directly to the CAP/availability story: more copies on more machines = higher availability).

So: RAID for single-box disk resilience; **distributed replication for system-level resilience at scale.** Modern designs lean on the latter (sometimes with simple RAID or just JBOD underneath each node).

---

## 8. Interview-Ready Insights

**Q: Block vs file vs object — the core distinction?**
It's the access abstraction. Block = raw blocks, you bring the filesystem (fastest, in-place writes — databases/VMs). File = a network-mounted filesystem tree shared by many clients (NFS/SMB — shared storage). Object = flat key-addressed blobs over HTTP, immutable, infinitely scalable and cheap (media, backups, data lakes).

**Q: Why can't you run a database on object storage?**
Databases need fast random access and **in-place updates** to their own on-disk structures; object storage is HTTP-accessed and replaces whole immutable objects, with higher latency. Databases need **block** storage's low-latency, fine-grained block writes.

**Q: Why is object storage so cheap and scalable?**
A flat namespace and immutable objects (replace, don't edit) let providers distribute and replicate blobs across vast clusters without filesystem-tree or in-place-update complexity. It's essentially a distributed key-value store for blobs over HTTP.

**Q: Explain the main RAID levels.**
RAID 0 stripes for speed with no redundancy (risky). RAID 1 mirrors for redundancy (50% capacity cost). RAID 5 stripes with single parity (survives 1 failure, slow writes). RAID 6 double parity (survives 2). RAID 10 mirrors then stripes (fast and safe, expensive). The primitives are striping (speed), mirroring (redundancy), parity (efficient redundancy).

**Q: Why is RAID 0 misleading as "RAID"?**
It provides no redundancy at all — it's pure performance striping, and it actually *increases* data-loss risk since any single disk failure loses the whole array.

**Q: Why do modern distributed systems prefer replication over RAID?**
RAID only protects against disk failure within one machine; it can't save you if the machine, rack, or datacenter dies. Distributed replication (e.g., HDFS 3×, KV-store N replicas across nodes) survives whole-machine and site failures, scales horizontally, and treats disks/servers as disposable — the right model at cloud scale.

**Q: When is RAID still the right choice?**
Single-server / on-prem / NAS scenarios where you want disk-level resilience and storage efficiency on one box. At scale, you replicate across machines instead (sometimes with simple RAID or JBOD under each node).

---

## 9. Quick Glossary

- **Block storage** — raw block devices; host applies its own filesystem (AWS EBS).
- **File storage** — network-mounted hierarchical filesystem shared by clients (NFS/SMB, EFS).
- **Object storage** — flat, key-addressed, immutable blobs over HTTP (S3, GCS).
- **IOPS** — input/output operations per second; a measure of storage throughput for small ops.
- **In-place update** — modifying data where it sits (block/file) vs. replacing a whole object.
- **Metadata** — descriptive data stored alongside an object.
- **RAID** — Redundant Array of Independent Disks; combines disks for speed/redundancy.
- **Striping (RAID 0)** — splitting data across disks for parallel performance.
- **Mirroring (RAID 1)** — keeping identical copies for redundancy.
- **Parity (RAID 5/6)** — computed redundancy enabling reconstruction of a failed disk.
- **Rebuild** — reconstructing a replaced disk's data from the rest of the array.
- **Distributed replication** — storing copies across separate machines/racks/zones.
- **HDFS** — Hadoop Distributed File System; uses 3× block replication across nodes.
- **JBOD** — "Just a Bunch Of Disks"; individual disks with no RAID, common under replicated systems.

---

*Reference document. Contributions and corrections welcome.*
