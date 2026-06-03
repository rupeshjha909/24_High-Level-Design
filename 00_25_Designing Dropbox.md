# Designing Dropbox (File Storage & Sync): A Senior Interview Guide

> A practical, interview-focused reference for designing a file storage and sync service — built around one central idea: chunking files into content-addressed blocks, which unlocks delta sync, deduplication, resumable uploads, and cheap versioning. Covers the metadata/content split, sync, conflict resolution, sharing, and the encryption-vs-dedup tension. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Central Insight: Chunking + Content Addressing](#4-the-central-insight-chunking--content-addressing)
5. [Metadata vs Content Separation](#5-metadata-vs-content-separation)
6. [Architecture](#6-architecture)
7. [Upload, Download & the Hash Handshake](#7-upload-download--the-hash-handshake)
8. [Sync & Conflict Resolution](#8-sync--conflict-resolution)
9. [Deduplication & the Encryption Tension](#9-deduplication--the-encryption-tension)
10. [Sharing, Versioning & Optimizations](#10-sharing-versioning--optimizations)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This in an Interview

Dropbox is a storage-and-sync problem, and almost everything good about the design flows from **one decision: split files into content-addressed chunks.** That single idea unlocks efficient sync, deduplication, resumable uploads, and cheap version history. So the strong approach is to establish requirements and scale, then **lead with chunking** and show how each hard requirement (don't re-upload whole files, don't store data twice, survive interrupted uploads) falls out of it.

The other recurring theme — shared with the Pastebin design — is **separating small metadata from large content**: metadata in a transactional database, content (chunks) in object storage. After chunking and the metadata/content split, the remaining depth is **sync** (detecting changes, propagating to other devices, handling offline) and **conflict resolution**. Senior signal: deriving delta sync *and* dedup from a single content-addressing mechanism, and knowing why arbitrary-file conflict resolution is punted to "conflicted copies" rather than auto-merged.

---

## 2. Requirements

### Functional

- **Upload, download, delete** files.
- **Sync across devices.**
- **Share** files/folders with permissions.
- **Version history.**
- **Offline edits** that sync on reconnect.
- Optional **real-time collaboration.**

### Non-Functional

- **Durability** — never lose user data (~11 nines).
- **Availability** — ~99.99%.
- **Efficient sync** — small changes shouldn't re-upload the whole file.
- **Massive scale** — billions of files, exabytes of data.

---

## 3. Capacity Estimation

```
Users          = 500M
Avg per user   = 1 GB
Raw storage    = 500M × 1 GB = 500 PB   (before deduplication)
Daily traffic  = petabytes of uploads/downloads
```

**Takeaways:**
- 500 PB is enormous → **object storage** for content is the only sane choice, and **deduplication** materially reduces real storage (raw uploads ≫ unique bytes stored).
- Petabytes of daily transfer make **bandwidth efficiency** (delta sync — transfer only what changed) a first-class requirement, not a nicety.
- ~11-nines durability means you lean on object storage's built-in replication/erasure coding rather than rolling your own (storage-types and distributed-filesystem guides).

---

## 4. The Central Insight: Chunking + Content Addressing

The most important design decision: **split each file into fixed-size chunks (e.g., 4 MB) and identify each chunk by the hash of its content (SHA-256).** A file then becomes *metadata + an ordered list of chunk hashes.* This **content addressing** (a chunk's hash *is* its identity — the same idea as Git, IPFS, and Merkle trees from the Merkle/gossip guide) unlocks four capabilities at once:

- **Delta sync** — edit one paragraph of a 1 GB file and only the **one changed 4 MB chunk** needs re-uploading; the other chunks' hashes are unchanged. Massive bandwidth savings.
- **Deduplication** — a chunk is stored under its hash, so if that exact content already exists (same user or another), it's **stored once**. Identical files/regions cost no extra storage.
- **Resumable uploads** — track which chunks succeeded; a failed upload **resumes from the last good chunk** instead of restarting.
- **Parallel uploads** — chunks are independent, so upload them **concurrently** for speed.

All four fall out of the same mechanism. The content-addressed chunk store is effectively a **key-value store keyed by content hash** (KV-store and object-storage guides) — immutable, dedupable, and self-verifying (the hash doubles as an integrity check). Lead with this in the interview; everything else builds on it.

---

## 5. Metadata vs Content Separation

As with Pastebin, split the small structured data from the large blobs:

- **Metadata** — file name, owner, folder path, version, and the **ordered list of chunk hashes** that make up the file. Stored in **PostgreSQL** (relational, ACID). It needs transactions and relational queries: renames/moves/shares/versioning must be consistent, and sharing and folder hierarchies are relational. A file record is essentially `{name, owner, [chunk hashes], version, ...}`.
- **Content** — the actual **chunks**, stored in **object storage (S3/GCS)**, keyed by hash.

So a "file" in the database is just a recipe (an ordered list of chunk hashes); the bytes live cheaply in object storage. This keeps the metadata DB small and fast (it never holds file bytes) while content scales to exabytes in the blob store — the metadata-in-DB, blob-in-object-store pattern again.

---

## 6. Architecture

```
[Client (file watcher)] ──► [Metadata Service] ──► [Metadata DB (Postgres)]
        │
        ├──► [Block Service] ──► [Object Store (S3/GCS)]   ← chunks, keyed by hash
        │
        ├──► [Notification Service] ──► (push sync events to other devices)
        │
        └──► [Sharing Service] ──► (permissions, share links)
```

- **Client** runs a **file watcher** that detects local changes, chunks/hashes them, and diffs against the last synced state.
- **Metadata Service** manages the file→chunks recipes and versions.
- **Block Service** stores/retrieves chunks in object storage by hash.
- **Notification Service** tells a user's *other* devices when something changed, so they sync.
- **Sharing Service** handles permissions and share links.

---

## 7. Upload, Download & the Hash Handshake

### Upload — the elegant "which hashes are missing?" handshake

```
1. Client splits the file into 4 MB chunks and hashes each (H1, H2, H3…).
2. Client → Block Service: "Do you already have H1, H2, H3?"
3. Server replies with the MISSING hashes only.
4. Client uploads only the missing chunks to object storage.
5. Client → Metadata Service: "File F = [H1, H2, H3]" (the recipe).
6. Metadata Service records it and notifies the user's other devices.
```

Step 2–3 is the heart of the design: by asking the server which chunk hashes it lacks, the client **uploads only new content** — this single handshake delivers **both delta sync and deduplication** simultaneously. (A Bloom filter can front the "do you have this hash?" check to answer "definitely not present" instantly — Bloom-filter guide.)

### Download

```
1. Client → Metadata Service: get file F's chunk list.
2. Client → Block Service: fetch the chunks (H1, H2, H3) from object storage.
3. Reassemble the file locally from the ordered chunks.
```

Chunks can be fetched in parallel, and popular shared files can be served via **CDN** (CDN guide).

---

## 8. Sync & Conflict Resolution

### Sync

- The client maintains a **local index** of file state and **watches** the filesystem for changes; on a change it diffs and uploads only the changed chunks.
- For *other* devices, the client **listens** (long-poll or WebSocket — protocols guide) for change notifications; on a push, it fetches the diff (the new/changed chunk hashes and updated recipe) and applies it locally — near-real-time sync.
- **Offline:** the client **queues local changes** and, on reconnect, pushes them and runs conflict resolution.

### Conflict resolution

When two devices edit the same file concurrently (especially across an offline gap), you have a conflict. Dropbox's deliberate choice is **simple and safe: keep both versions**, saving one as `filename (Device A's conflicted copy)`. This never loses data and never guesses wrong, because **automatically merging arbitrary binary files is impossible.**

True **real-time collaborative merge** (Google Docs-style) is a different, harder problem requiring **Operational Transformation (OT)** or **CRDTs** (the conflict-free types from the KV-store guide) — which only work for *structured* documents whose operations can be reconciled, not arbitrary files. Knowing this distinction — conflicted-copy for general files vs. OT/CRDT for structured collaborative docs — is a senior point.

---

## 9. Deduplication & the Encryption Tension

### Why dedup matters

With content-addressed chunks, identical content is stored once. At scale this is dramatic: a petabyte of *raw* uploads might be only ~200 TB of *unique* stored bytes (everyone stores the same popular files, OS images, shared documents). Block-level dedup via chunk hashes is a major cost lever.

### The encryption-vs-dedup tension (a key senior insight)

There's a fundamental conflict between **end-to-end encryption** and **cross-user deduplication:**

- **Server-side dedup needs the server to recognize identical content** — i.e., to compute/compare hashes of the actual (plaintext) chunks.
- **E2E encryption** means the server only ever sees **ciphertext**, and the *same* file encrypted by two users with *different* keys produces *different* ciphertext — so the server can't tell they're identical and **can't dedup across users.**

So you must choose: **dedup (Dropbox model — server can see/compare content, big storage savings)** or **E2E privacy (Mega.nz model — server can't read content, but no cross-user dedup).** Encryption choices in general:
- **At rest** — server-side encryption in object storage.
- **In transit** — HTTPS.
- **E2E** — client encrypts before upload; strongest privacy, but defeats dedup.
- *(Nuance: **convergent encryption** — deriving the key from the content hash so identical content yields identical ciphertext — restores dedup under encryption, at the cost of some confirmation-of-file attacks. Worth mentioning as the middle ground.)*

---

## 10. Sharing, Versioning & Optimizations

### Sharing & permissions

A **share table** maps `(file_id, shared_with_user_id, permission)`. Shareable links carry a **token**; the **server checks access** on every request (the content in object storage is never directly public — gate every fetch, as in the Pastebin design). Permissions (view/edit) are enforced at the metadata/sharing layer.

### Versioning (cheap, thanks to chunking)

Each version is just a **different chunk-list recipe.** Because unchanged chunks are shared (dedup), keeping version history is **cheap** — a new version only stores the chunks that actually changed. This is a lovely consequence of content addressing: history is nearly free.

### Optimizations

- **Sub-chunk delta (rsync algorithm)** — transfer only the changed *bytes within* a chunk, not the whole 4 MB chunk, squeezing bandwidth further for small edits.
- **CDN** for popular shared files (static, read-heavy content).
- **Compression** for text/code files before storage/transfer.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. What's the single most important design decision?**
Chunking files into fixed-size, content-addressed blocks (4 MB, hashed). A file becomes metadata + an ordered list of chunk hashes. This one mechanism delivers delta sync, deduplication, resumable uploads, and parallel uploads — and makes version history cheap.

**Q2. How does delta sync work — why not re-upload the whole file?**
Because each chunk is identified by its content hash, editing part of a file changes only the affected chunk(s). The client re-uploads just those chunks; unchanged chunks keep their hashes and are skipped. Edit one paragraph of a 1 GB file → upload one 4 MB chunk.

**Q3. How do delta sync and dedup come from the same mechanism?**
The "do you already have these hashes?" upload handshake. The client sends its chunk hashes; the server returns only the missing ones; the client uploads only those. New-only uploads = delta sync; "already have this hash" = dedup. Both fall out of content addressing.

**Q4. Where do you store metadata vs content, and why?**
Metadata (name, owner, chunk-hash list, version) in PostgreSQL — it needs ACID transactions and relational queries for renames, sharing, and hierarchy. Content (chunks) in object storage keyed by hash — cheap, durable, exabyte-scale. The DB stays small; content scales independently.

**Q5. How do you handle conflicts?**
For arbitrary files, keep both versions as conflicted copies — never lose data, never guess a merge (auto-merging binary files is impossible). Real-time collaborative merge needs OT or CRDTs and only works for structured documents, not general files.

**Q6. How does sync propagate to other devices?**
The client watches the local filesystem and uploads diffs; the server notifies the user's other devices via long-poll/WebSocket; those devices fetch the changed chunk hashes + updated recipe and apply them. Offline edits are queued and synced (with conflict resolution) on reconnect.

**Q7. Why does E2E encryption conflict with deduplication?**
Cross-user dedup requires the server to recognize identical content, but E2E means it only sees ciphertext, and the same file encrypted with different keys yields different ciphertext — so it can't dedup. You choose dedup (Dropbox) or E2E privacy (Mega). Convergent encryption (key derived from content hash) is a middle ground with its own security caveats.

**Q8. How is version history so cheap?**
A version is just a different chunk-list recipe; unchanged chunks are shared across versions via dedup. A new version only stores the chunks that changed, so keeping full history costs little extra storage.

**Q9. How do you make uploads resilient and fast?**
Chunking enables resumable uploads (resume from the last successful chunk after a failure) and parallel uploads (chunks are independent). Sub-chunk rsync deltas and compression further cut bandwidth.

**Q10. How do you achieve 11-nines durability?**
Store chunks in object storage that replicates/erasure-codes across machines and zones (storage-types/distributed-filesystem guides) — you inherit its durability rather than engineering replication yourself. Metadata is in a replicated, backed-up relational DB.

---

## 12. Quick Glossary

- **Chunk / block** — a fixed-size piece of a file (e.g., 4 MB), the unit of storage and sync.
- **Content addressing** — identifying a chunk by the hash of its content (its identity = its hash).
- **Delta sync** — transferring only the chunks (or bytes) that changed.
- **Deduplication** — storing identical content once, keyed by hash.
- **Hash handshake** — client asks the server which chunk hashes it lacks; uploads only those.
- **Metadata** — file name, owner, version, and the ordered chunk-hash list (in a relational DB).
- **Block/Object store** — where chunks live, keyed by hash (S3/GCS).
- **Resumable upload** — continuing an interrupted upload from the last good chunk.
- **File watcher** — client component detecting local filesystem changes to sync.
- **Conflicted copy** — saving both versions when a merge can't be done automatically.
- **OT / CRDT** — techniques for real-time collaborative merge of structured documents.
- **E2E encryption** — client-side encryption; maximizes privacy but prevents server dedup.
- **Convergent encryption** — key derived from content hash, restoring dedup under encryption.
- **rsync algorithm** — sub-chunk delta transfer of only changed bytes within a chunk.

---

*Reference document. Contributions and corrections welcome.*
