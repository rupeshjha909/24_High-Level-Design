# Designing Pastebin — Detailed HLD (presentation-ready, full request/response)

> An interview-ready HLD for a text-sharing service like Pastebin — structurally a URL shortener that stores **large content**. This expanded version adds the **full request/response contract for every endpoint**, detailed **data-model schemas**, step-by-step **read/write/pre-signed flows**, status-code and error conventions, caching/CDN, expiration, access control, abuse, capacity math (verified), failure modes, and a senior Q&A — everything you'd walk an interviewer through end to end.

> 💡 **The one idea:** it's a URL shortener + a **metadata/content split** — small structured **metadata in a DB**, large **content in object storage (S3)**, DB holds only a **pointer**. State the shortener parallel, then spend your time on the storage divergence and the API/flows that follow from it.

---

## Table of Contents
1. [Approach & the URL-shortener connection](#1-approach--the-url-shortener-connection)
2. [Requirements](#2-requirements)
3. [Capacity estimation (verified)](#3-capacity-estimation-verified)
4. [The defining decision: split metadata from content](#4-the-defining-decision-split-metadata-from-content)
5. [Architecture](#5-architecture)
6. [API design — full contract for every endpoint](#6-api-design--full-contract-for-every-endpoint)
7. [Data models & schemas](#7-data-models--schemas)
8. [Key generation](#8-key-generation)
9. [Storage decisions (polyglot)](#9-storage-decisions-polyglot)
10. [Write flow (content-first), step by step](#10-write-flow-content-first-step-by-step)
11. [Read flow, step by step](#11-read-flow-step-by-step)
12. [Large content: pre-signed URL flows](#12-large-content-pre-signed-url-flows)
13. [Caching & CDN](#13-caching--cdn)
14. [Expiration & cleanup](#14-expiration--cleanup)
15. [Access control, abuse & optional features](#15-access-control-abuse--optional-features)
16. [Failure modes & handling](#16-failure-modes--handling)
17. [Senior follow-up Q&A](#17-senior-follow-up-qa)
18. [TL;DR](#18-tldr)
19. [Glossary](#19-glossary)

---

## 1. Approach & the URL-shortener connection

Pastebin is structurally **a URL shortener that stores large content instead of a URL.** Reuse that skeleton — short base62 key, a key-generation service, extreme read:write skew, a cache-heavy read path — and spend your time on the **one genuine difference: the content is large.** That forces the central decision (**where to store big text blobs**), and everything interesting (object storage, the metadata/content split, expiration of bulky data, access control, pre-signed URLs) flows from it.

> 💡 Leading with the analogy ("this is a URL shortener that stores blobs — let me reuse that and focus on the content-storage divergence") is the focused, senior move.

---

## 2. Requirements

**Functional**
- **Create:** submit text → receive a short URL.
- **Read:** anyone with the URL can read the paste (subject to visibility).
- **Optional:** expiration, syntax highlighting, password protection, private/unlisted pastes, edit history, delete.

**Non-functional**
- **High read:write ratio (~100:1)** — overwhelmingly a serving system.
- **Low write latency** — creating feels instant.
- **Durability** — never lose a stored paste.
- **Availability** — reads must stay up; degrade gracefully.
- Full-text **search is out of scope** (clarify).

---

## 3. Capacity estimation (verified)

**Assumptions:** 1M pastes/day, 100M reads/day (100:1), ~10 KB avg paste.

```
Write QPS  = 1M / 86,400        ≈ 12/sec   (peak 5× ≈ 58/sec)
Read QPS   = 100M / 86,400      ≈ 1,157/sec (peak 5× ≈ 5,787/sec)
Content    = 1M × 10 KB         = 10.2 GB/day → 3.74 TB/yr → 18.7 TB/5yr
Metadata   = 1.825B rows × 300B ≈ 548 GB over 5yr (fits a KV store easily)
Read BW    = 1,157 × 10 KB      ≈ 11.9 MB/s avg (59 MB/s peak) → CDN/cache offloads most
Cache      = ~50k hot pastes    ≈ 0.5 GB Redis (tiny)
Key space  = 62^7 = 3.52T; 5yr pastes = 1.825B = 0.05% of space
```

**Takeaways:** writes are trivial; reads dominate → **caching/CDN central**. The new dimension vs. the shortener is **~19 TB of content over 5 years, growing unbounded** — precisely why content goes to **object storage**, not the DB, while metadata stays tiny (~0.5 TB) and fast.

> 💡 The estimation justifies the whole design: read-heavy + immutable → cache/CDN; big unbounded content → S3; tiny metadata → a fast KV store; 62^7 keys → never recycle.

---

## 4. The defining decision: split metadata from content

- **Metadata** (short key, content location, expiry, owner, flags) — **small, structured, queried constantly** → **database**.
- **Content** (the paste text, 10 KB–MBs) — **large, opaque, write-once** → **object storage (S3)**; the DB holds only a **pointer** (the S3 key).

Putting big blobs in the primary DB bloats it, slows queries, inflates cost, and strains backups/replication. S3 is purpose-built for cheap (~$23/TB/mo), durable (~11 nines), infinitely scalable blobs. This **metadata-in-DB, blob-in-object-store** pattern (same as media in WhatsApp) is the heart of Pastebin.

---

## 5. Architecture

```
                              ┌────────────► [CDN]  (public/hot content, static)
                              │
[Client] ─► [Load Balancer] ─► [API Server (stateless)] ─► [Cache (Redis)]
                                   │      │        │
                                   │      │        └────► [Object Store (S3)]   ← paste CONTENT (blobs)
                                   │      └─────────────► [Metadata DB (KV)]    ← key → S3 loc, expiry, owner, flags
                                   └────────────────────► [Key Gen Service]     ← unique short keys
                                                          [User-Index DB (SQL)] ← "list my pastes"
                                                          [Async workers]        ← cleanup, abuse scan, versioning
```

- **API server** — stateless behind an LB; orchestrates KGS, metadata DB, cache, S3.
- **KGS** — issues unique base62 keys (pre-generated).
- **Metadata DB** — one small record per paste (KV/wide-column for point lookups).
- **User-Index DB** — relational, for "list my pastes."
- **S3** — content blobs; **CDN** fronts public/hot content.
- **Redis** — caches hot paste content + metadata.

---

## 6. API design — full contract for every endpoint

### 6.0 Conventions
- **Base:** `https://api.pastebin.example/v1` ; public short URL: `https://pb.example/{key}`.
- **Auth:** `Authorization: Bearer <token>` (optional for public create; required for private/owner actions). Anonymous allowed for public pastes.
- **Idempotency:** creates accept `Idempotency-Key: <uuid>` — a retry with the same key returns the original result (no duplicate paste).
- **Rate limits:** per-IP and per-user on create (e.g., 20/min anon, 100/min authed); `429` with `Retry-After` when exceeded.
- **Size limit:** inline create ≤ **512 KB**; larger (up to, say, **10 MB**) must use the **pre-signed upload** flow (§12).
- **Errors** (uniform envelope):
```json
{ "error": { "code": "PASTE_NOT_FOUND", "message": "No paste with that key", "requestId": "..." } }
```
- **Status codes:** `200` OK, `201` Created, `302` redirect (to CDN/pre-signed), `400` bad request, `401` unauthenticated, `403` forbidden, `404` not found, `410` gone (expired), `413` payload too large, `429` rate-limited, `500` server error.

---

### 6.1 Create paste (inline, small content) — `POST /v1/pastes`
**Request**
```
POST /v1/pastes
Authorization: Bearer <token>            # optional
Content-Type: application/json
Idempotency-Key: 4f1c-...                # optional but recommended
```
```json
{
  "content": "def hello():\n    print('hi')",
  "title": "hello.py",                   // optional
  "syntax": "python",                    // optional, client-side highlight hint
  "visibility": "public",                // public | unlisted | private
  "password": "hunter2",                 // optional; if set → password-protected
  "expiresIn": "7d"                      // optional: 10m|1h|1d|7d|30d|never (or expiresAt ISO)
}
```
**Response `201 Created`**
```json
{
  "key": "AbC1234",
  "url": "https://pb.example/AbC1234",
  "visibility": "public",
  "size": 32,
  "syntax": "python",
  "createdAt": "2026-07-20T10:00:00Z",
  "expiresAt": "2026-07-27T10:00:00Z"
}
```
**Errors:** `413` (content > 512 KB → use pre-signed), `400` (invalid visibility/expiry), `429` (rate-limited), `401` (private paste but no auth).

> 💡 The server never trusts the client for the key or the S3 location — it mints the key (KGS) and derives the S3 key. The client only supplies content + options.

---

### 6.2 Create large paste (pre-signed upload) — `POST /v1/pastes/upload-url` then `PUT` to S3 then `POST /v1/pastes/confirm`
See §12 for the full flow. Summary:
1. `POST /v1/pastes/upload-url` → server mints a key + returns a **pre-signed S3 PUT URL**.
2. Client `PUT`s the content **directly to S3** (bypasses the API server — no big body through your service).
3. `POST /v1/pastes/confirm { key, size, ... }` → server writes metadata (content already durably in S3).

---

### 6.3 Read paste — `GET /{key}` (or `GET /v1/pastes/{key}`)
**Request**
```
GET /v1/pastes/AbC1234
Authorization: Bearer <token>            # required only if private
X-Paste-Password: hunter2                # required only if password-protected
Accept: application/json                 # or text/plain for raw
```
**Response `200 OK`** (JSON form)
```json
{
  "key": "AbC1234",
  "title": "hello.py",
  "content": "def hello():\n    print('hi')",
  "syntax": "python",
  "visibility": "public",
  "createdAt": "2026-07-20T10:00:00Z",
  "expiresAt": "2026-07-27T10:00:00Z"
}
```
(For `Accept: text/plain`, return the raw content body.)
**Errors:** `404` not found, `410` gone (expired — lazily purged), `401` (private, no auth), `403` (not owner / wrong password), `413`-adjacent: for very large content the server returns a **`302` redirect to a pre-signed download URL / CDN** instead of inlining (§12).

---

### 6.4 Read metadata only — `GET /v1/pastes/{key}/meta`
Returns everything except `content` (size, syntax, visibility, timestamps, owner, version). Useful for the UI to decide whether to prompt for a password or stream a large blob.

---

### 6.5 List my pastes — `GET /v1/users/me/pastes?limit=20&cursor=...`
**Request** (auth required)
```
GET /v1/users/me/pastes?limit=20&cursor=eyJvZmZzZXQiOjIwfQ==&sort=createdAt:desc
Authorization: Bearer <token>
```
**Response `200`**
```json
{
  "items": [
    { "key": "AbC1234", "title": "hello.py", "visibility": "public",
      "createdAt": "2026-07-20T10:00:00Z", "expiresAt": "2026-07-27T10:00:00Z" }
  ],
  "nextCursor": "eyJvZmZzZXQiOjQwfQ=="   // null when no more
}
```
> 💡 This is the *relational* query ("my pastes, newest first, paginated") that the **User-Index DB (SQL)** serves — not the KV metadata store, which is optimized for single-key point lookups.

---

### 6.6 Delete paste — `DELETE /v1/pastes/{key}`
Auth required; owner-only. `204 No Content` on success; `403` if not owner; `404` if absent. Deletes metadata immediately; content is removed via S3 delete or lifecycle (§14).

---

### 6.7 Edit / new version — `POST /v1/pastes/{key}/versions`
Creates a **new immutable version** (a new S3 object) and links it in metadata; the key can resolve to `latest` or a specific `?version=3`. `201` with the new version number. (See §15.)

---

## 7. Data models & schemas

### 7.1 Metadata store (KV / wide-column — DynamoDB / Cassandra)
Point-lookup by `key` on every read → the dominant access pattern.
```
Table: paste_meta
  key            (PK)  string   -- "AbC1234"
  s3_key               string   -- "pastes/Ab/C1/AbC1234/v1"
  size                 int      -- bytes
  syntax               string   -- "python" | null
  title                string   -- optional
  visibility           string   -- public | unlisted | private
  password_hash        string   -- bcrypt/argon2, null if none
  owner_id             string   -- null if anonymous
  content_hash         string   -- sha256 (for dedup, optional)
  latest_version       int      -- for edit history
  created_at           timestamp
  expires_at           timestamp  -- null = never; also a TTL attribute
```
- **TTL attribute** on `expires_at` lets the store auto-evict expired metadata (DynamoDB TTL / Cassandra TTL).
- **S3 key layout** uses a sharded prefix (`pastes/Ab/C1/AbC1234/`) to spread objects across S3 partitions and avoid hot prefixes.

### 7.2 User-Index (relational — PostgreSQL)
```
Table: user_pastes
  user_id     (PK part)  string
  created_at  (sort)     timestamp
  paste_key              string
  title                  string
  visibility             string
  INDEX (user_id, created_at DESC)     -- "list my pastes", paginated
```

### 7.3 Cache (Redis)
```
paste:content:{key}  → raw content   (only small/hot pastes; TTL e.g. 1h)
paste:meta:{key}     → metadata JSON  (TTL, so private/expiry checks stay fresh)
```

### 7.4 Object store (S3)
```
s3://pb-content/pastes/{shard}/{key}/v{n}        content blob (immutable, write-once)
+ lifecycle rule: expire objects tagged ttl=... automatically
```

> 💡 **Three stores, each for its strength (polyglot):** KV for O(1) key→location lookups, SQL for "my pastes" relational queries, S3 for cheap durable blobs. Name why each is chosen.

---

## 8. Key generation

Identical to the URL shortener:
- **Base62 keys** (`[a-zA-Z0-9]`); **7 chars = 62⁷ ≈ 3.52 trillion** — 5 years of pastes use **0.05%** of the space (verified).
- **Pre-generate via a KGS** (collision-free, fast at request time) or **counter + base62 with range allocation**. Random keys keep unlisted/private pastes **non-enumerable**.
- The key doubles as the **content locator** — content lives in S3 under a path derived from the key (`pastes/{shard}/{key}/v1`).

---

## 9. Storage decisions (polyglot)

| Data | Where | Why |
|:--|:--|:--|
| key → S3 location + flags | **DynamoDB / Cassandra** | O(1) point lookup at scale (the dominant read) |
| paste **content** | **S3** | cheap (~$23/TB/mo), ~11-nines durable, infinitely scalable |
| user's pastes index | **PostgreSQL** | relational "list my pastes", joins with accounts |
| hot content + metadata | **Redis** | sub-ms reads for popular pastes |
| public/hot content | **CDN** | static immutable content served at the edge |

---

## 10. Write flow (content-first), step by step

### Inline create — `POST /v1/pastes`
```
Client            API Server              KGS      S3            Meta DB      User-Index
  │  POST content   │                       │        │              │             │
  │────────────────►│ validate size/limits  │        │              │             │
  │                 │ rate-limit / idem check│        │              │             │
  │                 │  get key ─────────────►│        │              │             │
  │                 │◄──────── "AbC1234" ────│        │              │             │
  │                 │  PUT content ───────────────────►│  (1) CONTENT FIRST         │
  │                 │◄───────────── 200 ───────────────│              │             │
  │                 │  INSERT metadata ───────────────────────────────►│  (2)        │
  │                 │  INSERT user index (if owner) ──────────────────────────────►│ (3)
  │◄── 201 {url} ───│                        │        │              │             │
```
**Ordering matters (senior detail):** write **content to S3 *first*, metadata second.** If metadata were written first and you crashed, it would point to nonexistent content — a broken paste. Content-first means a crash leaves at worst an **orphaned S3 object** (cheap, cleanable by a sweeper), never a dangling pointer.

> 💡 **Idempotency:** the `Idempotency-Key` is checked before minting a key — a retried create returns the original paste instead of creating a duplicate (and re-uploading the blob). Store `idem_key → paste_key` with a short TTL.

---

## 11. Read flow, step by step

### `GET /v1/pastes/{key}`
```
1. Redis GET paste:content:{key}         → HIT → (check meta cache for expiry/visibility) → return
2. MISS → Meta DB point-lookup by key
      - not found            → 404
      - expires_at < now     → 410 Gone (lazily delete meta; content reclaimed by lifecycle)
      - visibility=private   → require owner auth (else 401/403)
      - password_hash set    → verify X-Paste-Password (else 403)
3. Small/hot content → S3 GET → populate Redis (content + meta) → return inline
4. Large content     → return 302 → pre-signed S3 GET URL (or CDN URL) → client fetches directly (§12)
```
Because pastes are **immutable**, cached content never goes stale → cache aggressively; public pastes can be fronted by a **CDN**.

> 💡 **Why check `expires_at` on read (not only via a sweeper)?** So an expired paste returns `410`/`404` **immediately**, even if the background cleanup hasn't run yet. Read-time check = correctness now; sweeper/lifecycle = reclaiming space later.

---

## 12. Large content: pre-signed URL flows

Never stream multi-MB blobs *through* your API servers — let the client talk to S3 directly via **pre-signed URLs** (time-limited, signed, scoped to one object).

**Upload (large create):**
```
Client                         API Server                         S3
  │ POST /v1/pastes/upload-url    │                                 │
  │──────────────────────────────►│ mint key (KGS)                  │
  │                               │ presign PUT s3://…/AbC1234/v1    │
  │◄── { key, uploadUrl, expiresIn }│                                │
  │  PUT content ──────────────────────────────────────────────────►│  (direct to S3)
  │◄──────────────────────────── 200 ──────────────────────────────│
  │ POST /v1/pastes/confirm {key,size,visibility,expiresIn,...}      │
  │──────────────────────────────►│ INSERT metadata (content-first  │
  │◄── 201 { url } ───────────────│  guarantee: blob already in S3)  │
```
**Download (large read):** API verifies access, then returns `302` to a **pre-signed GET URL** (private) or a **CDN URL** (public). The client downloads directly from S3/CDN — your API servers move no bulk bytes.

> 💡 **Pre-signed URLs are the trick that keeps the API tier cheap and stateless.** Big bytes flow client↔S3; the API only handles small JSON (auth, metadata, minting URLs). This also enforces access control (private pastes get short-lived signed URLs; the bucket is never public).

---

## 13. Caching & CDN

- **Redis** caches small/hot paste **content** + **metadata** (short TTL so private/expiry flags stay correct). ~0.5 GB for ~50k hot pastes (verified) — tiny.
- **CDN** fronts **public, immutable** content — a hot paste is served from the edge without touching your origin. Immutability means no invalidation headaches (a paste's content never changes; edits create new versions/keys).
- **Client cache** for a paste just created/viewed.

> 💡 **Immutability is the caching superpower:** since a paste's content is write-once, cached/CDN copies are always valid — you can cache with long TTLs and never worry about staleness. Only *metadata* (expiry/visibility) needs short TTLs.

---

## 14. Expiration & cleanup

Reclaim **both** metadata and content:
- **Read-time check** — `expires_at < now` → return `410`/`404` and lazily delete the metadata row.
- **S3 lifecycle policies (elegant)** — tag objects with a TTL; **S3 auto-expires** the content. Offloads bulky cleanup entirely.
- **DB TTL / light sweeper** — the metadata store's TTL attribute (DynamoDB/Cassandra TTL) evicts expired rows; a small sweeper handles stragglers and the user-index.

Given 62⁷ keys and only 0.05% used in 5 years, **recycling expired keys isn't worth the complexity** (same as the shortener).

---

## 15. Access control, abuse & optional features

- **Visibility & passwords.** Content sits in S3, but **access control is enforced at the app layer** — the API checks ownership/password before returning content or issuing a pre-signed URL. **Never make the bucket public for private pastes.** Access matrix:
  | visibility | who can read |
  |:--|:--|
  | public | anyone (CDN-served) |
  | unlisted | anyone **with the key** (non-enumerable random key) |
  | private | owner only (auth required) |
  | password | anyone with the key **and** the password |
- **Abuse.** Pastebins get used to dump stolen data/credentials/malware. Mitigate with **content size limits**, **creation rate limiting**, **async abuse scanning** (scan new blobs, flag/takedown), and abuse-report + takedown tooling — same class as phishing on a URL shortener.
- **Syntax highlighting.** Client-side (highlight.js) — the server stores/serves raw text, language-agnostic.
- **Edit history.** Each edit = a **new immutable S3 object**; metadata tracks `latest_version` and version→s3_key links; UI diffs versions. Fits the write-once model.
- **Dedup (optional).** Store `content_hash`; identical pastes can **share one S3 object** (ref-counted) — saves storage on repeatedly-pasted dumps.

---

## 16. Failure modes & handling

| Failure | Effect | Handling |
|:--|:--|:--|
| Crash after S3 put, before metadata | Orphaned S3 object | Cheap; sweeper reclaims unreferenced objects. **Content-first ordering guarantees no dangling pointer.** |
| Crash after metadata, before user-index | Paste readable but not in "my pastes" | Reconcile index from metadata async; minor |
| Metadata DB down | Reads/writes fail | KV store is HA (replicated); cache serves hot reads meanwhile |
| S3 slow/unavailable | Content fetch fails | Serve from cache/CDN; retries; S3 is ~11-nines so rare |
| Expired paste still cached | Stale read | Short **metadata** TTL + read-time `expires_at` check catches it |
| Abusive/huge content | Storage/abuse risk | Size limit (`413`), rate limit (`429`), async scan + takedown |
| Duplicate create on retry | Two pastes | `Idempotency-Key` returns the original |
| Hot paste (viral) | Origin load | CDN absorbs it; content is static |

---

## 17. Senior follow-up Q&A

**Q1. How is this different from a URL shortener?** Same skeleton (base62 key, KGS, read-heavy, cache-first) but stores **large content** → **content in S3, metadata + pointer in the DB.**

**Q2. Why not store content in the DB?** Blobs bloat the DB, slow queries, inflate cost, burden backups. S3 is cheap, ~11-nines durable, infinitely scalable. Metadata stays small/fast (~0.5 TB vs ~19 TB of content).

**Q3. Order of writes, and why?** **Content to S3 first, metadata second.** Metadata-first + crash = dangling pointer (broken paste). Content-first = at worst a cheap orphaned object.

**Q4. How do you serve large pastes without overloading the API?** **Pre-signed S3 URLs** (upload and download) — big bytes go client↔S3/CDN; the API only handles small JSON + access checks.

**Q5. How do private/password pastes work if content is in S3?** App-layer access control before returning content or issuing a **short-lived pre-signed URL**; bucket never public.

**Q6. Keep reads fast?** Cache-first (immutable → never stale), CDN for public content, metadata point-lookups in a KV store, pre-signed/CDN for large blobs.

**Q7. Expiration of big content?** Read-time `expires_at` check (`410`) + **S3 lifecycle** auto-expiry for content + DB TTL/sweeper for metadata. Don't recycle keys.

**Q8. Idempotent creates?** `Idempotency-Key` header → `idem_key → paste_key` mapping; retries return the original paste, no duplicate blob.

**Q9. Which databases (polyglot) and why?** KV/wide-column for key→location point lookups; SQL for "list my pastes" relational queries; S3 for blobs. Each does what it's best at.

**Q10. Edit history?** New immutable S3 object per version + version links in metadata; UI diffs. Immutable versions are trivially durable.

**Q11. Abuse?** Size limits, creation rate limiting, async content scanning, takedown tooling.

**Q12. Save storage on duplicate content?** Content-address by `sha256`; identical pastes share one ref-counted S3 object.

---

## 18. TL;DR

**Design:** a URL shortener + a **metadata/content split**. Small metadata (key → S3 location, expiry, owner, flags) in a **KV DB**; large content in **S3**; **CDN/Redis** front reads; a **KGS** mints base62 keys; a **SQL user-index** powers "my pastes."

**APIs (contract):** `POST /v1/pastes` (inline create, `Idempotency-Key`, ≤512 KB), `POST /v1/pastes/upload-url`+`/confirm` (large, pre-signed), `GET /v1/pastes/{key}` (read, access-checked, `302`→CDN/pre-signed for large), `/{key}/meta`, `GET /v1/users/me/pastes` (paginated), `DELETE`, `/versions` (edit history). Uniform error envelope; `410` for expired.

**Flows:** write **content-first** (S3 then metadata) to avoid dangling pointers; read **cache → metadata (expiry/visibility/password checks) → S3/CDN**; large content via **pre-signed URLs** so bulk bytes never traverse the API tier.

**Ops:** immutable content → cache/CDN hard (no staleness); expire via read-time check + S3 lifecycle + DB TTL; never recycle keys; guard abuse with size/rate limits + scanning.

### Verified numbers
```
Write ~12/s (peak 58) | Read ~1,157/s (peak 5,787) | 100:1
Content 10.2 GB/day → 3.74 TB/yr → 18.7 TB/5yr   |   Metadata ~0.5 TB/5yr
Read BW ~11.9 MB/s (peak 59) → CDN offloads   |   Hot cache ~0.5 GB Redis
62^7 = 3.52T keys; 5yr pastes 1.825B = 0.05% used
```

### One-line philosophy
> **Pastebin is a URL shortener whose payload is big: mint a base62 key, store the small metadata (key→location, expiry, owner, flags) in a fast KV DB and the large immutable content in S3, and let the DB hold only a pointer. Write content-first so a crash never leaves a dangling pointer, serve reads cache/CDN-first (immutability makes caching trivial), move large blobs client↔S3 via pre-signed URLs so the API tier stays thin, enforce access control at the app layer, and expire via read-time checks + S3 lifecycle.**

---

## 19. Glossary

- **Paste / short key** — stored text and its base62 id (also locates content in S3).
- **Metadata / pointer** — small DB record (key, S3 location, expiry, owner, flags).
- **Content / blob** — the large paste text in object storage.
- **KGS** — pre-generates unique keys for fast, collision-free creation.
- **Polyglot persistence** — multiple stores, each for its best-fit data (KV / SQL / S3).
- **Pre-signed URL** — time-limited signed URL for direct, authorized S3 upload/download.
- **Content-first ordering** — store the blob before the metadata to avoid dangling pointers.
- **S3 lifecycle policy** — auto-expiry of objects by TTL tag.
- **Idempotency-Key** — header making retried creates return the original paste.
- **Immutable content** — write-once data; ideal for caching/CDN and versioned edits.

---

*Reference document — presentation-ready. Contributions and corrections welcome.*
