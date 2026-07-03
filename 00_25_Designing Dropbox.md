# Designing Dropbox (File Storage & Sync): A Senior Interview Guide

> A practical, interview-focused reference for designing a file storage and sync service — built around one central idea: chunking files into content-addressed blocks, which unlocks delta sync, deduplication, resumable uploads, and cheap versioning. Covers the metadata/content split, sync, conflict resolution, sharing, and the encryption-vs-dedup tension. With capacity math, trade-offs, and a senior follow-up bank.

> 💡 **The question this guide now answers in depth (see §4A):** *How does the system actually recognize chunks, and how does it know which chunk changed when you edit a file?* Short version: every chunk is named by the hash of its own bytes, so a file is just a **list of chunk-hashes** (a "recipe"). When you edit, the client re-chunks and re-hashes; any chunk whose bytes changed gets a **different hash**, and comparing the new recipe to the old one instantly reveals exactly which chunks are new. That's the whole trick — explained slowly, with a worked example, below.

---

## Table of Contents

1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Central Insight: Chunking + Content Addressing](#4-the-central-insight-chunking--content-addressing)
   - [4A. How Chunks Are Recognized & How a Change Is Detected (in simple language)](#4a-how-chunks-are-recognized--how-a-change-is-detected-in-simple-language)
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

The most important design decision: **split each file into chunks (e.g., 4 MB) and identify each chunk by the hash of its content (SHA-256).** A file then becomes *metadata + an ordered list of chunk hashes.* This **content addressing** (a chunk's hash *is* its identity — the same idea as Git, IPFS, and Merkle trees from the Merkle/gossip guide) unlocks four capabilities at once:

- **Delta sync** — edit one paragraph of a 1 GB file and only the **one changed chunk** needs re-uploading; the other chunks' hashes are unchanged. Massive bandwidth savings.
- **Deduplication** — a chunk is stored under its hash, so if that exact content already exists (same user or another), it's **stored once**. Identical files/regions cost no extra storage.
- **Resumable uploads** — track which chunks succeeded; a failed upload **resumes from the last good chunk** instead of restarting.
- **Parallel uploads** — chunks are independent, so upload them **concurrently** for speed.

All four fall out of the same mechanism. The content-addressed chunk store is effectively a **key-value store keyed by content hash** (KV-store and object-storage guides) — immutable, dedupable, and self-verifying (the hash doubles as an integrity check). Lead with this in the interview; everything else builds on it.

---

## 4A. How Chunks Are Recognized & How a Change Is Detected (in simple language)

This is the part most explanations skip. Let's go slowly and concretely — no jargon left unexplained.

### Step 1 — What "chunking" actually does

When you save a file, the client software **cuts the file into pieces** called **chunks** (think: slicing a loaf of bread). Say a chunk is 4 MB. A 20 MB file becomes 5 chunks.

```
FILE (20 MB):  [========================================]
CHUNKS:        [ chunk1 ][ chunk2 ][ chunk3 ][ chunk4 ][ chunk5 ]
                 4MB       4MB       4MB       4MB       4MB
```

### Step 2 — What a "hash" is, and why it becomes the chunk's name

A **hash** is a short fingerprint computed from some bytes. Feed a chunk's bytes into **SHA-256** and out comes a fixed-length string like `a1b2c3d4…`. Two key properties:

1. **Same bytes in → same fingerprint out, every time.** Identical chunks always produce the identical hash.
2. **Change even a single byte → a completely different fingerprint.** There's no "small change → small change"; any difference scrambles the whole hash.

Because of property (1), we use the hash **as the chunk's name (its ID)**. This is what "**content addressing**" means: *the chunk's address/name is derived from its content.* You don't name a chunk "chunk3" — you name it by its fingerprint. So two identical chunks anywhere in the world automatically get the same name and are recognized as the same thing. **That is how chunks are "recognized."**

### Step 3 — A file is just a list of chunk-hashes (the "recipe")

After chunking and hashing, the file is stored as an **ordered list of chunk hashes** — call it the **recipe**:

```
File "report.pdf"  →  recipe = [ H1, H2, H3, H4, H5 ]
where H1 = hash(chunk1), H2 = hash(chunk2), ...
```

The actual chunk bytes live in object storage under those hash-names; the database only stores the small recipe (the list of hashes) plus file info (name, owner, version). The recipe *is* the file, as far as the system is concerned.

### Step 4 — How it knows a chunk changed: re-hash and compare the recipes

Now the core question: **you edit the file. How does the system know which chunk changed?**

It doesn't need any magic "change tracking." It simply:
1. **Re-chunks** the edited file into pieces again.
2. **Re-hashes** each piece.
3. **Compares the new recipe to the old recipe**, hash by hash.

Any chunk you *didn't* touch produces the **same bytes → same hash** → it appears unchanged in the recipe. The chunk you *did* touch produces **different bytes → different hash** → it stands out immediately. **Different hash = changed chunk.** That's the entire detection mechanism.

**Worked example (verified in code).** Take a file split into 6 chunks and change one byte inside chunk #2:

```
Old recipe:  [ c34ab6ab, 2f858775, acea9bbe, b1eedd29, 0c6edc90, bdd77372 ]
New recipe:  [ c34ab6ab, 2ec435b4, acea9bbe, b1eedd29, 0c6edc90, bdd77372 ]
                          ^^^^^^^^
                          only the 2nd hash differs
```

Five hashes are byte-for-byte identical; only the second changed. So the client knows **exactly one chunk** was modified and re-uploads **only that one**. Editing one paragraph of a 1 GB file → one small chunk goes over the wire, not a gigabyte. (This was verified: exactly one chunk differed.)

### Step 5 — Detection over the network: "which of these hashes do you NOT have?"

The client doesn't even need to figure out the diff by itself. It just hands the server the **new recipe** (the list of hashes) and asks: *"which of these do you already have, and which are missing?"* The server keeps all known chunks indexed by hash, so it answers instantly, and the client uploads **only the missing ones**.

```
Client: "I have hashes [c34ab6ab, 2ec435b4, acea9bbe, b1eedd29, 0c6edc90, bdd77372]. Which are new to you?"
Server: "I'm missing [2ec435b4]."          ← the one changed chunk
Client: uploads only chunk 2ec435b4.
```

(Verified: server reported exactly one missing hash.) This one question does **two jobs at once**: it gives you **delta sync** ("only send what changed") *and* **deduplication** ("if I already have this hash — even from another user — don't send it again"). Both come free from naming chunks by their content.

### Step 6 — The catch nobody mentions: the "insertion problem"

There's a subtle weakness in the simple approach, and mentioning it is a strong senior signal.

If you cut chunks at **fixed positions** (every 4 MB), the scheme works great for edits that **replace** bytes in place (the change stays inside one chunk). But watch what happens if you **insert** bytes near the beginning — say you add a few characters at the top of a document. Every byte after the insertion **shifts over**, so every chunk boundary now lands on different bytes → **every chunk gets new content → every hash changes → the whole file looks "changed."** Your clever delta sync collapses to re-uploading everything.

```
Fixed-size chunking, insert 1 byte at the front:
Old:  [ c34ab6ab, 2f858775, acea9bbe, b1eedd29, 0c6edc90, bdd77372 ]
New:  [ e5e0332c, ea6e214b, db7d215a, 325aecc3, 1bc45eb7, afc4d898, f67ab10a ]
        ↑ ALL different — 0 of 6 chunks survived the shift ✗
```

(Verified: a single front-insertion changed all 6 chunks; zero survived.)

### Step 7 — The fix: content-defined chunking (boundaries that move with the content)

Instead of cutting every fixed 4 MB, cut the boundaries **based on the content itself**. The client slides a small window over the bytes computing a cheap **rolling hash**, and declares a chunk boundary whenever that rolling value hits a chosen pattern (e.g., "the last few bits are all zero"). This is called **Content-Defined Chunking (CDC)**, and the rolling-hash idea comes from the **rsync** algorithm.

Why this fixes insertion: because boundaries are tied to the **bytes around them**, not to absolute positions, they **move along with the content** when you insert or delete. The boundaries before your edit stay put, the boundaries after your edit stay put — only the chunk(s) right around the edit change. Everything else keeps its old hash and is recognized as unchanged.

```
Content-defined chunking:
- edit one word in the middle → shared unchanged chunks: 15 / 17
- INSERT bytes at the front  → shared unchanged chunks: 15 / 17
  (only the chunks near the change differ; the rest survive)
```

(Verified: with content-defined boundaries, 15 of 17 chunks survived *both* an in-place edit and a front-insertion — versus 0 of 6 surviving under fixed chunking.)

> 💡 **The senior one-liner for this whole section:** *"Each chunk is named by the SHA-256 of its own bytes, so a file is just an ordered list of chunk-hashes. To detect a change I re-chunk, re-hash, and compare hash lists — a changed chunk has a changed hash, so it stands out; then I ask the server which hashes it's missing and upload only those. The gotcha is that fixed-size boundaries break on insertions (everything shifts), so real systems use content-defined chunking with a rolling hash — rsync-style — so boundaries move with the content and only the edited region re-uploads."* Saying the insertion problem and CDC unprompted is what separates a memorized answer from an understood one.

---

## 5. Metadata vs Content Separation

As with Pastebin, split the small structured data from the large blobs:

- **Metadata** — file name, owner, folder path, version, and the **ordered list of chunk hashes** (the recipe) that makes up the file. Stored in **PostgreSQL** (relational, ACID). It needs transactions and relational queries: renames/moves/shares/versioning must be consistent, and sharing and folder hierarchies are relational. A file record is essentially `{name, owner, [chunk hashes], version, ...}`.
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

- **Client** runs a **file watcher** that detects local changes, chunks/hashes them, and diffs against the last synced state (using exactly the re-chunk-and-compare mechanism from §4A).
- **Metadata Service** manages the file→chunks recipes and versions.
- **Block Service** stores/retrieves chunks in object storage by hash, and answers the "which hashes are missing?" question.
- **Notification Service** tells a user's *other* devices when something changed, so they sync.
- **Sharing Service** handles permissions and share links.

---

## 7. Upload, Download & the Hash Handshake

### Upload — the elegant "which hashes are missing?" handshake

```
1. Client splits the file into chunks and hashes each (H1, H2, H3…).
2. Client → Block Service: "Do you already have H1, H2, H3?"
3. Server replies with the MISSING hashes only.
4. Client uploads only the missing chunks to object storage.
5. Client → Metadata Service: "File F = [H1, H2, H3]" (the recipe).
6. Metadata Service records it and notifies the user's other devices.
```

Step 2–3 is the heart of the design (the mechanics are detailed in §4A, Steps 5): by asking the server which chunk hashes it lacks, the client **uploads only new content** — this single handshake delivers **both delta sync and deduplication** simultaneously. (A Bloom filter can front the "do you have this hash?" check to answer "definitely not present" instantly — Bloom-filter guide.)

### Download

```
1. Client → Metadata Service: get file F's chunk list (recipe).
2. Client → Block Service: fetch the chunks (H1, H2, H3) from object storage.
3. Reassemble the file locally from the ordered chunks.
```

Chunks can be fetched in parallel, and popular shared files can be served via **CDN** (CDN guide). Because each chunk's hash *is* a fingerprint of its bytes, the client can also **verify integrity** on download: re-hash each downloaded chunk and confirm it matches the requested hash (a free corruption check).

---

## 8. Sync & Conflict Resolution

### Sync

- The client maintains a **local index** of file state (the last-synced recipe) and **watches** the filesystem for changes; on a change it re-chunks, re-hashes, diffs against that index, and uploads only the changed chunks.
- For *other* devices, the client **listens** (long-poll or WebSocket — protocols guide) for change notifications; on a push, it fetches the diff (the new/changed chunk hashes and updated recipe) and applies it locally — near-real-time sync.
- **Offline:** the client **queues local changes** and, on reconnect, pushes them and runs conflict resolution.

### Conflict resolution

When two devices edit the same file concurrently (especially across an offline gap), you have a conflict. Dropbox's deliberate choice is **simple and safe: keep both versions**, saving one as `filename (Device A's conflicted copy)`. This never loses data and never guesses wrong, because **automatically merging arbitrary binary files is impossible.**

True **real-time collaborative merge** (Google Docs-style) is a different, harder problem requiring **Operational Transformation (OT)** or **CRDTs** (the conflict-free types from the KV-store guide) — which only work for *structured* documents whose operations can be reconciled, not arbitrary files. Knowing this distinction — conflicted-copy for general files vs. OT/CRDT for structured collaborative docs — is a senior point.

---

## 9. Deduplication & the Encryption Tension

### Why dedup matters

With content-addressed chunks, identical content is stored once. At scale this is dramatic: a petabyte of *raw* uploads might be only ~200 TB of *unique* stored bytes (everyone stores the same popular files, OS images, shared documents). Block-level dedup via chunk hashes is a major cost lever — and it's the *same hash-comparison mechanism* from §4A, just applied across all users instead of across versions of one file.

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

Each version is just a **different chunk-list recipe.** Because unchanged chunks are shared (dedup), keeping version history is **cheap** — a new version only stores the chunks that actually changed (exactly the "only the changed hash differs" property from §4A). This is a lovely consequence of content addressing: history is nearly free.

### Optimizations

- **Content-defined chunking (rolling hash / rsync)** — variable-size boundaries that survive insertions and deletions, so edits re-upload only the affected region (see §4A, Step 7). This is the production-grade upgrade over fixed-size chunks.
- **Sub-chunk delta (rsync algorithm)** — transfer only the changed *bytes within* a chunk, not the whole chunk, squeezing bandwidth further for small edits.
- **CDN** for popular shared files (static, read-heavy content).
- **Compression** for text/code files before storage/transfer.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. What's the single most important design decision?**
Chunking files into content-addressed blocks (hashed with SHA-256). A file becomes metadata + an ordered list of chunk hashes. This one mechanism delivers delta sync, deduplication, resumable uploads, and parallel uploads — and makes version history cheap.

**Q2. How does delta sync work — why not re-upload the whole file?**
Because each chunk is identified by its content hash, editing part of a file changes only the affected chunk(s). The client re-chunks, re-hashes, and compares to the previous recipe; unchanged chunks keep their hashes and are skipped; only the changed chunk(s) re-upload. Edit one paragraph of a 1 GB file → upload one chunk. (Full mechanics in §4A.)

**Q3. How exactly does the system recognize a chunk and detect that it changed?**
Each chunk is *named* by the SHA-256 hash of its bytes (content addressing), so identical bytes always get the identical name and a single-byte change produces a totally different name. To detect changes, the client re-chunks the file, re-hashes each chunk, and compares the new list of hashes to the old one: a chunk whose hash is unchanged is unchanged; a chunk with a new hash is the modified one. Then the client asks the server "which of these hashes are you missing?" and uploads only those. (See §4A, Steps 1–5.)

**Q4. What breaks if you insert bytes in the middle of a file, and how do you fix it?**
With **fixed-size** chunks, an insertion shifts every following byte, so every chunk boundary lands on new content and *every* hash changes — delta sync degenerates to re-uploading everything (verified: one front-insert changed all chunks). The fix is **content-defined chunking (CDC)**: use a **rolling hash** (rsync-style) to place boundaries based on the surrounding bytes, so boundaries move with the content and only the chunks near the edit change (verified: 15/17 chunks survived an insertion). (§4A, Steps 6–7.)

**Q5. How do delta sync and dedup come from the same mechanism?**
The "do you already have these hashes?" handshake. The client sends its chunk hashes; the server returns only the missing ones; the client uploads only those. New-only uploads = delta sync; "already have this hash (even from another user)" = dedup. Both fall out of naming chunks by content.

**Q6. Where do you store metadata vs content, and why?**
Metadata (name, owner, chunk-hash recipe, version) in PostgreSQL — it needs ACID transactions and relational queries for renames, sharing, and hierarchy. Content (chunks) in object storage keyed by hash — cheap, durable, exabyte-scale. The DB stays small; content scales independently.

**Q7. How do you handle conflicts?**
For arbitrary files, keep both versions as conflicted copies — never lose data, never guess a merge (auto-merging binary files is impossible). Real-time collaborative merge needs OT or CRDTs and only works for structured documents, not general files.

**Q8. How does sync propagate to other devices?**
The client watches the local filesystem and uploads diffs; the server notifies the user's other devices via long-poll/WebSocket; those devices fetch the changed chunk hashes + updated recipe and apply them. Offline edits are queued and synced (with conflict resolution) on reconnect.

**Q9. Why does E2E encryption conflict with deduplication?**
Cross-user dedup requires the server to recognize identical content, but E2E means it only sees ciphertext, and the same file encrypted with different keys yields different ciphertext — so it can't dedup. You choose dedup (Dropbox) or E2E privacy (Mega). Convergent encryption (key derived from content hash) is a middle ground with its own security caveats.

**Q10. How is version history so cheap?**
A version is just a different chunk-list recipe; unchanged chunks are shared across versions via dedup. A new version only stores the chunks that changed, so keeping full history costs little extra storage.

**Q11. How do you make uploads resilient and fast, and verify integrity?**
Chunking enables resumable uploads (resume from the last successful chunk after a failure) and parallel uploads (chunks are independent). Because each chunk's hash is a fingerprint of its bytes, the client can re-hash downloaded chunks to verify they weren't corrupted. Content-defined chunking, sub-chunk rsync deltas, and compression further cut bandwidth.

**Q12. How do you achieve 11-nines durability?**
Store chunks in object storage that replicates/erasure-codes across machines and zones (storage-types/distributed-filesystem guides) — you inherit its durability rather than engineering replication yourself. Metadata is in a replicated, backed-up relational DB.

---

## 12. Quick Glossary

- **Chunk / block** — a piece of a file (e.g., 4 MB), the unit of storage and sync.
- **Hash (SHA-256)** — a short fixed-length fingerprint of some bytes; same bytes → same hash, any change → totally different hash.
- **Content addressing** — naming a chunk by the hash of its content, so its identity = its hash.
- **Recipe (chunk list)** — the ordered list of chunk-hashes that reconstitutes a file; stored as metadata.
- **Delta sync** — transferring only the chunks (or bytes) that changed.
- **Deduplication** — storing identical content once, keyed by hash.
- **Hash handshake** — client asks the server which chunk hashes it lacks; uploads only those.
- **Fixed-size chunking** — cutting chunks at fixed byte positions; simple, but breaks on insertions (everything shifts).
- **Content-defined chunking (CDC)** — placing chunk boundaries based on the content itself via a rolling hash, so boundaries move with edits (insert/delete safe).
- **Rolling hash** — a cheap hash over a sliding window of bytes, used by CDC and rsync to find boundaries.
- **Metadata** — file name, owner, version, and the ordered chunk-hash recipe (in a relational DB).
- **Block/Object store** — where chunks live, keyed by hash (S3/GCS).
- **Resumable upload** — continuing an interrupted upload from the last good chunk.
- **File watcher** — client component detecting local filesystem changes to sync.
- **Conflicted copy** — saving both versions when a merge can't be done automatically.
- **OT / CRDT** — techniques for real-time collaborative merge of structured documents.
- **E2E encryption** — client-side encryption; maximizes privacy but prevents server dedup.
- **Convergent encryption** — key derived from content hash, restoring dedup under encryption.
- **rsync algorithm** — sub-chunk delta transfer of only changed bytes within a chunk, using a rolling hash.

---

*Reference document. Contributions and corrections welcome.*
