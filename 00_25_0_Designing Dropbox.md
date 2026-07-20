# Designing Dropbox (File Storage & Sync): A Senior Interview Guide

> A practical, interview-focused reference for designing a file storage and sync service — built around one central idea: **chunking files into content-addressed blocks**, which unlocks delta sync, deduplication, resumable uploads, and cheap versioning. This guide builds the architecture up piece by piece, traces the full life of an upload and a cross-device sync, goes deep on the one hard mechanism (how the system *recognizes* chunks and *detects* a change), nails down the data contracts, and covers sync, conflict resolution, sharing, and the encryption-vs-dedup tension — with verified capacity math and a senior follow-up bank.

> 💡 **The question this guide answers in depth (see §5):** *How does the system actually recognize chunks, and how does it know which chunk changed when you edit a file?* Short version: every chunk is named by the hash of its own bytes, so a file is just a **list of chunk-hashes** (a "recipe"). When you edit, the client re-chunks and re-hashes; any chunk whose bytes changed gets a **different hash**, and comparing the new recipe to the old one instantly reveals exactly which chunks are new. That's the whole trick — explained slowly, with worked (verified) examples, below.

---

## Table of Contents
1. [How to Approach This in an Interview](#1-how-to-approach-this-in-an-interview)
2. [Requirements](#2-requirements)
3. [Capacity Estimation (verified)](#3-capacity-estimation-verified)
4. [Core Architecture (In Depth)](#4-core-architecture-in-depth)
   - [4.1 The big picture in one paragraph](#41-the-big-picture-in-one-paragraph)
   - [4.2 The diagram](#42-the-diagram)
   - [4.3 Each component, in detail](#43-each-component-in-detail)
   - [4.4 The end-to-end life of a file (upload + cross-device sync)](#44-the-end-to-end-life-of-a-file-upload--cross-device-sync)
   - [4.5 Why this split? (the design rationale)](#45-why-this-split-the-design-rationale)
   - [4.6 Where the load actually goes](#46-where-the-load-actually-goes)
5. [The Central Mechanism: Chunking + Content Addressing (deep dive)](#5-the-central-mechanism-chunking--content-addressing-deep-dive)
6. [Metadata vs Content Separation](#6-metadata-vs-content-separation)
7. [Upload, Download & the Hash Handshake](#7-upload-download--the-hash-handshake)
8. [Data Contracts: Request Fields, Payloads & DB Schemas](#8-data-contracts-request-fields-payloads--db-schemas)
9. [Sync & Conflict Resolution](#9-sync--conflict-resolution)
10. [Deduplication & the Encryption Tension](#10-deduplication--the-encryption-tension)
11. [Sharing, Versioning & Optimizations](#11-sharing-versioning--optimizations)
12. [Scaling Summary](#12-scaling-summary)
13. [Senior Follow-Up Questions (with Answers)](#13-senior-follow-up-questions-with-answers)
14. [Quick Glossary](#14-quick-glossary)

---

## 1. How to Approach This in an Interview

Dropbox is a storage-and-sync problem, and almost everything good about the design flows from **one decision: split files into content-addressed chunks.** That single idea unlocks efficient sync, deduplication, resumable uploads, and cheap version history. So the strong approach is to establish requirements and scale, then **lead with chunking** and show how each hard requirement (don't re-upload whole files, don't store data twice, survive interrupted uploads) falls out of it.

The other recurring theme — shared with Pastebin — is **separating small metadata from large content**: metadata in a transactional database, content (chunks) in object storage. After chunking and the metadata/content split, the remaining depth is **sync** (detecting changes, propagating to other devices, handling offline) and **conflict resolution**.

> 💡 **Senior signal:** deriving *both* delta sync and dedup from a single content-addressing mechanism, knowing the **insertion problem** and why real systems use **content-defined chunking**, and knowing why arbitrary-file conflict resolution is punted to "conflicted copies" rather than auto-merged. Say up front: *"I'll lead with content-addressed chunking — it's the one mechanism the whole design hangs off — then the metadata/content split, then sync and conflicts."*

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
- **Efficient sync** — small changes must not re-upload the whole file (bandwidth is a first-class constraint).
- **Massive scale** — billions of files, exabytes of data.

---

## 3. Capacity Estimation (verified)

```
Users            = 500M
Avg per user     = 1 GB
Raw storage      = 500M × 1 GB = 500 PB           (before deduplication)
After dedup      ≈ 100–167 PB unique              (at 3:1–5:1; popular/shared files stored once)
A 1 GB file      = 256 × 4 MB chunks → recipe = 8 KB of hashes   (metadata is tiny)
Files            ≈ 100B files → 100B metadata rows (relational, small per row)
Chunks (pre-dedup) ≈ 128 billion 4 MB blocks in object storage
Daily traffic    = petabytes of uploads/downloads
```

**Takeaways that shape the design:**
- 500 PB is enormous → **object storage** for content is the only sane choice, and **deduplication** materially cuts real bytes stored (raw uploads ≫ unique bytes).
- Petabytes of daily transfer make **bandwidth efficiency (delta sync — transfer only what changed)** a requirement, not a nicety.
- **Metadata is small** (an 8 KB recipe for a 1 GB file), so it lives comfortably in a relational DB while content scales independently to exabytes.
- ~11-nines durability → lean on object storage's built-in replication/erasure coding rather than rolling your own.

> 💡 The estimation justifies the two pillars: **object storage + dedup** for the 500 PB of content, and a **small metadata DB** for the recipes. The whole architecture is those two plus a sync mechanism.

---

## 4. Core Architecture (In Depth)

### 4.1 The big picture in one paragraph

A file-sync system is fundamentally different from a normal upload service because of one requirement: **a small edit must cost a small transfer, and identical content must never be stored twice — even across different users.** A naive "upload the file to a bucket" design fails both: editing one line of a 1 GB file would re-upload a gigabyte, and a popular file everyone has would be stored a million times. The escape from both problems is the same single idea: **cut every file into chunks and name each chunk by the hash of its own bytes.** Once you do that, a file is just an ordered **list of chunk-hashes (a "recipe")**, and two superpowers appear for free — you can tell *exactly* which chunks changed (their hashes changed) so you transfer only those (**delta sync**), and you can store each unique chunk once under its hash so duplicates cost nothing (**dedup**). Everything in the architecture serves this: the **Client/file-watcher** chunks and hashes and diffs; the **Metadata Service** stores the recipes and versions (small, transactional); the **Block Service** stores/serves the chunks in **object storage** keyed by hash and answers "which hashes are you missing?"; and the **Notification** and **Sharing** services handle cross-device sync and permissions.

### 4.2 The diagram

```
                                   ┌───────────────────────────┐
                                   │      Metadata DB (Postgres) │
                                   │  files → [chunk hashes],    │
                                   │  versions, sharing, folders │
                                   └─────────────▲───────────────┘
                                                 │ recipes / versions
 [Client A]                                      │
   file watcher ──chunk+hash+diff──► [Metadata Service]
        │                                        │ notify other devices
        ├──"which of these hashes do you have?"─► [Block Service] ──► [Object Store (S3/GCS)]
        │        (uploads only MISSING chunks)                         ▲ chunks keyed by HASH
        │                                                              │
        └──────────────────────────────────► [Notification Service] ──┘ (push sync events)
                                             [Sharing Service] ─► permissions / share links
 [Client B] ◄── "file X changed → new recipe [H1,H2',H3]" ── pulls only H2' ──► [Object Store]
```

The key visual idea: the **recipe** (list of hashes) flows through the **Metadata Service**; the **chunk bytes** flow through the **Block Service** to/from object storage; and the **"which hashes are missing?" handshake** is what makes both delta sync and dedup happen in one step. Other devices learn of changes via the **Notification Service** and pull only the changed chunks.

### 4.3 Each component, in detail

**① Client + file watcher (the smart part).** The desktop/mobile client does most of the cleverness locally. It **watches the filesystem** for changes; on a change it **re-chunks** the file, **re-hashes** each chunk (SHA-256), and **diffs** the new recipe against the last-synced one to find changed chunks (§5). It runs the **"which hashes do you have?"** handshake, uploads only missing chunks, and reassembles/downloads on the way in. It also **verifies integrity** on download by re-hashing chunks. On offline edits it **queues** changes and syncs on reconnect. The client is where chunking, hashing, and diffing live — the server never has to inspect file bytes to detect changes.

**② Metadata Service.** Owns the **recipes and versions** — the mapping `file → ordered [chunk hashes] + name/owner/folder/version`. Every file operation that isn't raw bytes (rename, move, share, new version) is a small **transactional** update here. It's the source of truth for *what files exist and what chunks compose them*, but it never stores the bytes themselves.

**③ Block Service.** The chunk gateway. It **stores and retrieves chunks in object storage keyed by hash**, and answers the pivotal question **"which of these hashes are missing?"** (often fronted by a **Bloom filter** to answer "definitely don't have it" instantly). Because chunks are content-addressed and immutable, this service is essentially a **content-addressed key-value store** — self-verifying (the hash is an integrity check) and dedupable by construction.

**④ Object Store (S3/GCS).** Where the chunk bytes actually live, keyed by hash. Cheap, exabyte-scale, ~11-nines durable via replication/erasure coding — you inherit durability rather than engineering it. This holds the ~128 billion blocks; the DB never does.

**⑤ Notification Service.** When a file changes, it tells the user's **other devices** "file X changed → here's the new recipe," so they pull the diff. Uses long-poll or WebSocket for near-real-time sync. Split out because its push traffic pattern differs from the upload/download path.

**⑥ Sharing Service.** Manages permissions and share links: a `(file_id, user_id, permission)` model, tokens for links, and an **access check on every fetch** (object storage is never directly public — every chunk fetch is gated, as in Pastebin).

**⑦ Metadata DB (PostgreSQL).** Relational + ACID, because renames/moves/shares/versioning must be consistent and folder hierarchies and sharing are inherently relational. Stores recipes and file/folder/share rows — all small.

### 4.4 The end-to-end life of a file (upload + cross-device sync)

Here is exactly what happens when **Alice saves a 1 GB file on Laptop A, later edits one paragraph, and it syncs to her Phone B**:

```
UPLOAD (first save, 1 GB file):
1.  File watcher on A notices the new file → client CHUNKS it (≈256 × 4 MB) and HASHES each
        → recipe = [H1, H2, …, H256].
2.  Client → Block Service: "Which of [H1…H256] do you already have?"
3.  Block Service (checks its hash index / Bloom filter) → "I'm missing all 256" (new file).
4.  Client uploads the 256 chunks to object storage (parallel; resumable if interrupted).
5.  Client → Metadata Service: "file report.pdf = [H1…H256], version 1."
6.  Metadata Service records it and tells the Notification Service → Phone B is told to sync.

EDIT (change one paragraph → touches ONE chunk):
7.  Watcher on A sees the change → client RE-CHUNKS + RE-HASHES → new recipe
        = [H1, H2', H3, …, H256]   (only H2 became H2').
8.  Client → Block Service: "Which of [H1, H2', H3, …] do you have?"
9.  Block Service: "I'm missing only H2'."          ← delta detected, no diff logic needed
10. Client uploads ONLY chunk H2' (a few MB, not a gigabyte).
11. Client → Metadata Service: "report.pdf version 2 = [H1, H2', H3, …]."
        (Version 1's recipe is kept → cheap history: only H2' is extra bytes.)

SYNC to Phone B:
12. Notification Service pushes to B: "report.pdf changed → recipe now [H1, H2', H3, …]."
13. B diffs the new recipe against what it has locally → it already has H1,H3,…; only H2' is new.
14. B → Block Service: fetch H2' → reassembles version 2 locally.
        (B transferred one chunk, not the whole file.)
```

The single most important thing to notice: **steps 8–10 (the hash handshake) do delta sync and dedup at once**, and **step 11 makes versioning nearly free** (a version is just another recipe that reuses unchanged chunks). No component ever needed "change tracking" — the changed hash *is* the change signal.

### 4.5 Why this split? (the design rationale)

Each separation exists for a concrete reason — be ready to justify them:

- **Metadata (recipes) separate from content (chunks)** — recipes are tiny, relational, and need ACID (renames/shares/versions must be consistent); chunk bytes are huge, immutable, and need cheap exabyte storage. Different shape, different store — Postgres vs object storage. Keeps the DB small no matter how much content accumulates.
- **Block Service separate from Metadata Service** — moving bytes and managing recipes scale on different axes; isolating the byte path lets object storage/CDN handle bulk transfer while the metadata tier stays light.
- **Client does the chunking/hashing/diffing** — pushing this to the edge means the server never inspects file bytes to detect changes; it just answers "which hashes are missing?" This is what makes delta detection free on the server.
- **Notification Service separate** — cross-device push has a different traffic pattern from upload/download; isolating it keeps the transfer path clean.
- **Sharing Service separate** — permission checks gate every fetch; keeping this as its own concern centralizes access control (object storage is never public).

### 4.6 Where the load actually goes

A senior is expected to know *which* part is hard. The math (verified):

- **Content bytes:** ~**500 PB raw → ~100–167 PB unique** after dedup — this is the whole cost center, and it lives entirely in **object storage**, not your DB.
- **Chunk count:** ~**128 billion 4 MB blocks** — the Block Service is a massive content-addressed KV store; the hash index / Bloom filter that answers "do you have this?" is the hot lookup.
- **Bandwidth:** petabytes/day — but **delta sync collapses it**: a 1-paragraph edit of a 1 GB file transfers ~one chunk (verified: fixed-chunk in-place edit changed **1 of 6** chunks), so real transfer ≪ raw change volume.
- **Metadata:** an **8 KB recipe per 1 GB file**, ~100B rows — small and relational; not the bottleneck. Renames/moves are pure metadata ops (no byte movement).
- **The insertion trap (the deceptive one):** naive **fixed-size** chunking makes an *insert* re-upload everything (verified: a 3-byte front-insert left **0 of 49** chunks surviving), which would blow the bandwidth budget — so the real load hinges on using **content-defined chunking** (verified: **153 of 154** chunks survived the same insert). Getting chunking right is where the bandwidth savings actually come from.

> 💡 **The senior framing:** *"The cost is 500 PB of content → object storage with dedup (~128B blocks keyed by hash). The metadata is tiny (8 KB recipes). The real engineering subtlety isn't storage — it's making delta sync survive insertions, which needs content-defined chunking; otherwise every insert re-uploads the whole file."*

---

## 5. The Central Mechanism: Chunking + Content Addressing (deep dive)

This is the part most explanations skip. Let's go slowly and concretely — no jargon left unexplained. (Every example below is verified in code.)

### Step 1 — What "chunking" actually does
When you save a file, the client **cuts it into pieces** called **chunks** (like slicing a loaf). Say a chunk is 4 MB; a 20 MB file becomes 5 chunks.
```
FILE (20 MB):  [========================================]
CHUNKS:        [ chunk1 ][ chunk2 ][ chunk3 ][ chunk4 ][ chunk5 ]
```

### Step 2 — What a "hash" is, and why it becomes the chunk's name
A **hash** is a short fingerprint of some bytes. Feed a chunk into **SHA-256** → a fixed-length string like `a1b2c3d4…`. Two properties:
1. **Same bytes in → same fingerprint out, always.** Identical chunks → identical hash.
2. **Change one byte → a completely different fingerprint.** No "small change → small change."

Because of (1), we use the hash **as the chunk's name (ID)**. That's **content addressing**: *the chunk's name is derived from its content.* Two identical chunks anywhere get the same name and are recognized as the same thing. **That is how chunks are "recognized."**

### Step 3 — A file is just a list of chunk-hashes (the "recipe")
```
File "report.pdf"  →  recipe = [ H1, H2, H3, H4, H5 ]     where Hi = hash(chunk i)
```
The chunk bytes live in object storage under those hash-names; the DB stores only the small **recipe** plus file info. The recipe *is* the file, to the system.

### Step 4 — How it knows a chunk changed: re-hash and compare the recipes
No magic change-tracking. On edit the client (1) **re-chunks**, (2) **re-hashes**, (3) **compares** new recipe to old, hash by hash. Untouched chunk → same bytes → same hash → looks unchanged. Touched chunk → different bytes → **different hash** → stands out. **Different hash = changed chunk.**

**Worked example (verified).** Change one byte inside chunk #2:
```
Old recipe:  [ 7486da8f, 7486da8f, 7486da8f, 7486da8f, 7486da8f, 7486da8f ]
New recipe:  [ 7486da8f, e79668a6, 7486da8f, 7486da8f, 7486da8f, 7486da8f ]
                          ^^^^^^^^  only the 2nd hash differs → 1 of 6 changed
```
So the client re-uploads **only that one chunk**. One paragraph of a 1 GB file → one small chunk over the wire, not a gigabyte. (Verified: exactly one chunk differed.)

### Step 5 — Detection over the network: "which of these hashes do you NOT have?"
The client needn't compute the diff itself. It hands the server the **new recipe** and asks *"which do you already have, which are missing?"* The server indexes all known chunks by hash, answers instantly, and the client uploads **only the missing ones**.
```
Client: "I have [7486da8f, e79668a6, 7486da8f, …]. Which are new to you?"
Server: "Missing [e79668a6]."          ← the one changed chunk
Client: uploads only chunk e79668a6.
```
This one question does **two jobs at once**: **delta sync** ("only send what changed") *and* **deduplication** ("if I already have this hash — even from another user — don't send it"). Both come free from naming chunks by content.

### Step 6 — The catch nobody mentions: the "insertion problem"
Cut chunks at **fixed positions** (every 4 MB) and it works great for edits that **replace** bytes in place. But **insert** bytes near the beginning and every following byte **shifts**, so every boundary lands on new bytes → **every chunk gets new content → every hash changes → the whole file looks changed.** Delta sync collapses to re-uploading everything.
```
Fixed-size chunking, insert 3 bytes at the front:
survived (hash still present): 0 of 49 chunks ✗   (verified)
```

### Step 7 — The fix: content-defined chunking (boundaries that move with the content)
Instead of cutting every fixed 4 MB, cut boundaries **based on the content itself**. The client slides a small window computing a cheap **rolling hash** and declares a boundary whenever that value hits a pattern (e.g., low bits all zero). This is **Content-Defined Chunking (CDC)**; the rolling-hash idea comes from **rsync**. Because boundaries are tied to the **bytes around them**, they **move with the content** on insert/delete — boundaries before and after your edit stay put, so only the chunk(s) near the edit change.
```
Content-defined chunking (verified, same 3-byte front insert):
in-place middle edit → survived 153 / 154 chunks
front insert         → survived 153 / 154 chunks
(vs FIXED-size front insert: 0 / 49 survived)
```

> 💡 **The senior one-liner for this whole section:** *"Each chunk is named by the SHA-256 of its own bytes, so a file is just an ordered list of chunk-hashes. To detect a change I re-chunk, re-hash, and compare hash lists — a changed chunk has a changed hash, so it stands out; then I ask the server which hashes it's missing and upload only those. The gotcha is that fixed-size boundaries break on insertions (everything shifts, verified 0/49 survive), so real systems use content-defined chunking with a rolling hash — rsync-style — so boundaries move with the content and only the edited region re-uploads (verified 153/154 survive)."* Saying the insertion problem and CDC unprompted separates a memorized answer from an understood one.

---

## 6. Metadata vs Content Separation

As with Pastebin, split small structured data from large blobs:
- **Metadata** — file name, owner, folder path, version, and the **ordered list of chunk hashes** (the recipe). In **PostgreSQL** (relational, ACID): renames/moves/shares/versioning need transactions, and hierarchies/sharing are relational. A file record ≈ `{name, owner, [chunk hashes], version, …}`.
- **Content** — the **chunks**, in **object storage (S3/GCS)**, keyed by hash.

A "file" in the DB is just a recipe; the bytes live cheaply in object storage. The DB stays small and fast (never holds bytes) while content scales to exabytes — the metadata-in-DB, blob-in-object-store pattern.

---

## 7. Upload, Download & the Hash Handshake

### Upload — the "which hashes are missing?" handshake
```
1. Client splits the file into chunks and hashes each (H1, H2, H3…).
2. Client → Block Service: "Do you already have H1, H2, H3?"
3. Server replies with the MISSING hashes only.
4. Client uploads only the missing chunks to object storage.
5. Client → Metadata Service: "File F = [H1, H2, H3]" (the recipe).
6. Metadata Service records it and notifies the user's other devices.
```
Steps 2–3 are the heart: asking which hashes the server lacks delivers **delta sync and dedup simultaneously**. (A **Bloom filter** can front the check to answer "definitely not present" instantly.)

### Download
```
1. Client → Metadata Service: get file F's recipe.
2. Client → Block Service: fetch the chunks (parallel).
3. Reassemble locally from the ordered chunks; re-hash each to VERIFY integrity (free corruption check).
```
Popular shared files can be served via **CDN** (static, read-heavy, immutable chunks).

---

## 8. Data Contracts: Request Fields, Payloads & DB Schemas

The sections above describe the *working*; this one nails down the concrete **data contracts**.

### Part A — Key client↔server requests

**Hash handshake** (client → Block Service):
```json
// request: "which of these do you have?"
{ "file_id":"f_88", "hashes":["H1","H2p","H3","…"] }
// response: only the missing ones
{ "missing":["H2p"] }
```
**Chunk upload** (client → Block Service): `PUT /blocks/{hash}` with the raw (optionally compressed/encrypted) chunk bytes; server verifies `sha256(body) == {hash}` before storing (self-verifying, rejects corruption/mismatch).

**Commit recipe** (client → Metadata Service):
```json
{ "path":"/docs/report.pdf", "chunks":["H1","H2p","H3","…"],
  "base_version":1, "client_mtime":1730000000123 }
→ { "file_id":"f_88", "version":2 }
```
`base_version` lets the server detect a concurrent change (someone else committed v2 already) → conflict handling (§9).

**Sync poll** (client → Notification/Metadata): "what changed for me since cursor C?"
```json
→ { "changes":[ { "file_id":"f_88","path":"/docs/report.pdf",
                  "version":2,"chunks":["H1","H2p","H3","…"] } ],
    "next_cursor":"C2" }
```

### Part B — DB schema per store

**① Files / recipes — PostgreSQL** (the recipe is the file):
```
files( file_id PK, owner_id, path, name, current_version, created_at, updated_at )
file_versions( file_id, version, chunk_hashes text[],   -- the ORDERED recipe
               size_bytes, created_at, device_id,
               PRIMARY KEY (file_id, version) )
INDEX (owner_id, path)      -- list a folder / resolve a path
```
Each **version is just another recipe** (a row with a different `chunk_hashes` array) → cheap history (§11).

**② Chunk index — content-addressed KV (backs the Block Service):**
```
blocks( hash PK, size_bytes, refcount, storage_url, created_at )
  -- refcount = how many recipes reference this chunk (for garbage collection)
```
Bytes live in object storage at `storage_url`; this table (or a Bloom filter over it) answers "do you have this hash?".

**③ Sharing — PostgreSQL:**
```
shares( file_id, shared_with_user_id, permission,  -- 'view' | 'edit'
        PRIMARY KEY (file_id, shared_with_user_id) )
share_links( token PK, file_id, permission, expires_at )
```

**④ Devices / sync cursors — track what each device has seen:**
```
device_sync( device_id PK, user_id, last_cursor, last_seen_at )
```

**⑤ Object store** — chunk bytes keyed by hash: `s3://blocks/{hash[:2]}/{hash[2:4]}/{hash}` (sharded prefix to spread across partitions).

> 💡 **The contract in one line:** *"The client sends a recipe (list of content-hashes) and asks which the server lacks; it uploads only those, then commits the recipe with a `base_version` for conflict detection. Recipes and versions live in Postgres (tiny, relational, ACID); chunk bytes live in object storage keyed by hash with a refcount for GC. Every field exists for delta sync, dedup, versioning, or conflict detection."*

---

## 9. Sync & Conflict Resolution

### Sync
- The client keeps a **local index** (the last-synced recipe) and **watches** the filesystem; on a change it re-chunks, re-hashes, diffs against the index, and uploads only changed chunks.
- For *other* devices it **listens** (long-poll/WebSocket) for change notifications; on a push it fetches the diff (new chunk hashes + updated recipe) and applies it — near-real-time sync.
- **Offline:** the client **queues** local changes and, on reconnect, pushes them and runs conflict resolution.

### Conflict resolution
When two devices edit the same file concurrently (especially across an offline gap), the `base_version` mismatch reveals it. Dropbox's deliberate choice is **simple and safe: keep both versions**, saving one as `filename (Device A's conflicted copy)`. This never loses data and never guesses wrong — **auto-merging arbitrary binary files is impossible.**

True **real-time collaborative merge** (Google Docs-style) is a different, harder problem needing **Operational Transformation (OT)** or **CRDTs**, which only work for *structured* documents whose operations can be reconciled — not arbitrary files. Knowing this distinction (conflicted-copy for general files vs OT/CRDT for structured docs) is a senior point.

---

## 10. Deduplication & the Encryption Tension

### Why dedup matters
With content-addressed chunks, identical content is stored once. At scale this is dramatic: 500 PB raw → ~100–167 PB unique (everyone stores the same OS images, popular/shared files). Block-level dedup via chunk hashes is a major cost lever — the *same hash-comparison mechanism* from §5, applied across all users rather than across versions of one file. A **refcount** per chunk enables garbage collection when the last referencing recipe is deleted.

### The encryption-vs-dedup tension (a key senior insight)
A fundamental conflict between **E2E encryption** and **cross-user dedup:**
- **Server-side dedup needs the server to recognize identical content** — to compute/compare hashes of the actual (plaintext) chunks.
- **E2E** means the server only sees **ciphertext**, and the *same* file encrypted by two users with *different* keys yields *different* ciphertext — so the server **can't tell they're identical** and **can't dedup across users.**

So you choose: **dedup (Dropbox model — server sees content, big savings)** or **E2E privacy (Mega.nz model — no cross-user dedup).** Encryption options: **at rest** (server-side in object storage), **in transit** (HTTPS), **E2E** (client encrypts before upload; strongest privacy, defeats dedup). *Nuance: **convergent encryption** — derive the key from the content hash so identical content yields identical ciphertext — restores dedup under encryption, at the cost of some confirmation-of-file attacks. The middle ground worth naming.*

---

## 11. Sharing, Versioning & Optimizations

### Sharing & permissions
A **share table** maps `(file_id, user_id, permission)`; links carry a **token**; the **server checks access on every request** (object-store content is never directly public — gate every fetch, as in Pastebin).

### Versioning (cheap, thanks to chunking)
Each version is just a **different recipe**. Unchanged chunks are shared across versions (dedup), so a new version only stores the chunks that actually changed — keeping full history is nearly free (the "only the changed hash differs" property from §5).

### Optimizations
- **Content-defined chunking (rolling hash / rsync)** — variable boundaries that survive insertions/deletions, so edits re-upload only the affected region (§5, Step 7). The production-grade upgrade over fixed chunks.
- **Sub-chunk delta (rsync)** — transfer only the changed *bytes within* a chunk for small edits.
- **CDN** for popular shared files. **Compression** for text/code before storage/transfer.
- **Bloom filter** fronting the "do you have this hash?" check for instant "definitely not present."

---

## 12. Scaling Summary

- **Content in object storage, keyed by hash** — cheap, ~11-nines durable, exabyte-scale; ~128B blocks; you inherit durability.
- **Dedup by content hash** — 500 PB raw → ~100–167 PB unique; refcount for GC.
- **Metadata in PostgreSQL** — tiny recipes (8 KB per 1 GB file); relational for shares/hierarchy/versions; shard by user/file_id as it grows.
- **Delta sync via the hash handshake** — transfer only missing chunks; the single biggest bandwidth lever.
- **Content-defined chunking** — makes delta sync survive insertions (the real subtlety).
- **Push sync via long-poll/WebSocket**; **CDN** for popular files; **Bloom filter** for fast have-check.
- **Conflicts → conflicted copies** for arbitrary files; OT/CRDT only for structured collaborative docs.

---

## 13. Senior Follow-Up Questions (with Answers)

**Q1. What's the single most important design decision?**
Chunking files into content-addressed blocks (SHA-256). A file becomes metadata + an ordered list of chunk hashes; this one mechanism delivers delta sync, dedup, resumable + parallel uploads, and cheap versioning.

**Q2. How does delta sync work — why not re-upload the whole file?**
Each chunk is identified by its content hash, so editing part of a file changes only the affected chunk(s). The client re-chunks, re-hashes, compares to the previous recipe, and uploads only changed chunks. Edit one paragraph of a 1 GB file → upload one chunk (verified: 1 of 6 changed on an in-place edit).

**Q3. How exactly does the system recognize a chunk and detect a change?**
Each chunk is *named* by the SHA-256 of its bytes (content addressing), so identical bytes → identical name and a one-byte change → a totally different name. To detect changes: re-chunk, re-hash, compare hash lists; a changed hash is the changed chunk. Then ask the server "which hashes are you missing?" and upload only those. (§5)

**Q4. What breaks if you insert bytes mid-file, and how do you fix it?**
With **fixed-size** chunks, an insertion shifts every following byte so every hash changes — delta sync degenerates to re-uploading everything (verified: front-insert → 0 of 49 chunks survived). Fix: **content-defined chunking** with a **rolling hash** (rsync-style) so boundaries move with the content (verified: 153 of 154 survived the same insert). (§5, Steps 6–7)

**Q5. How do delta sync and dedup come from the same mechanism?**
The "do you already have these hashes?" handshake: client sends chunk hashes, server returns only missing ones, client uploads only those. New-only = delta sync; "already have this hash (even from another user)" = dedup. Both fall out of naming chunks by content.

**Q6. Metadata vs content — where and why?**
Metadata (name, owner, recipe, version) in PostgreSQL — needs ACID + relational queries for renames/sharing/hierarchy. Content (chunks) in object storage keyed by hash — cheap, durable, exabyte-scale. The DB stays small (8 KB recipes); content scales independently.

**Q7. How do you handle conflicts?**
Arbitrary files: keep both as conflicted copies — never lose data, never guess a merge (auto-merging binaries is impossible). `base_version` detects the concurrent edit. Real-time merge needs OT/CRDTs and only works for structured docs.

**Q8. How does sync propagate to other devices?**
The client watches the FS and uploads diffs; the server notifies other devices via long-poll/WebSocket; they fetch the changed chunk hashes + updated recipe and apply them. Offline edits are queued and synced (with conflict resolution) on reconnect.

**Q9. Why does E2E encryption conflict with dedup?**
Cross-user dedup needs the server to recognize identical content, but E2E means it only sees ciphertext, and the same file under different keys yields different ciphertext — so it can't dedup. Choose dedup (Dropbox) or E2E (Mega); convergent encryption is a middle ground with security caveats.

**Q10. How is version history so cheap?**
A version is just a different recipe; unchanged chunks are shared via dedup, so a new version only stores the chunks that changed. Full history costs little extra.

**Q11. How do you make uploads resilient/fast and verify integrity?**
Chunking enables **resumable** uploads (resume from the last good chunk) and **parallel** uploads (chunks independent). Each chunk's hash is a fingerprint, so the client re-hashes downloads to verify no corruption. CDC, sub-chunk rsync deltas, and compression cut bandwidth further.

**Q12. How do you achieve 11-nines durability?**
Store chunks in object storage that replicates/erasure-codes across machines and zones — inherit its durability. Metadata is in a replicated, backed-up relational DB.

**Q13. How do you garbage-collect chunks no file references?**
Keep a **refcount** per chunk (how many recipes reference it); when a version/file is deleted, decrement; when it hits zero, the chunk is eligible for GC (with care around concurrent uploads referencing the same hash — dedup means a chunk you're about to delete could be re-referenced).

---

## 14. Quick Glossary
- **Chunk / block** — a piece of a file (e.g., 4 MB), the unit of storage and sync.
- **Hash (SHA-256)** — a fixed-length fingerprint of bytes; same bytes → same hash, any change → totally different hash.
- **Content addressing** — naming a chunk by the hash of its content, so its identity = its hash.
- **Recipe (chunk list)** — the ordered list of chunk-hashes that reconstitutes a file; stored as metadata.
- **Delta sync** — transferring only the chunks (or bytes) that changed.
- **Deduplication** — storing identical content once, keyed by hash.
- **Hash handshake** — client asks the server which chunk hashes it lacks; uploads only those.
- **Fixed-size chunking** — cutting at fixed byte positions; simple, breaks on insertions (everything shifts).
- **Content-defined chunking (CDC)** — boundaries placed by content via a rolling hash, so they move with edits (insert/delete safe).
- **Rolling hash** — a cheap hash over a sliding window, used by CDC/rsync to find boundaries.
- **Refcount** — count of recipes referencing a chunk; enables garbage collection.
- **Metadata** — file name, owner, version, and the recipe (in a relational DB).
- **Block/Object store** — where chunks live, keyed by hash (S3/GCS).
- **Resumable upload** — continuing an interrupted upload from the last good chunk.
- **File watcher** — client component detecting local filesystem changes.
- **Conflicted copy** — saving both versions when a merge can't be done automatically.
- **OT / CRDT** — techniques for real-time collaborative merge of structured documents.
- **E2E encryption** — client-side encryption; maximizes privacy but prevents server dedup.
- **Convergent encryption** — key derived from content hash, restoring dedup under encryption.
- **rsync algorithm** — sub-chunk delta transfer of only changed bytes within a chunk, using a rolling hash.

---

*Reference document. Contributions and corrections welcome.*
