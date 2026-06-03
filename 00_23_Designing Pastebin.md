# Designing Pastebin: A Senior Interview Guide

> A practical, interview-focused reference for designing a text-sharing service like Pastebin — closely related to a URL shortener, but with one defining twist: storing large content. Covers the metadata/content split, object storage, key generation, read/write flows, expiration, and access control. With capacity math, trade-offs, and a senior follow-up bank.

---

## Table of Contents

1. [How to Approach This (and the URL-Shortener Connection)](#1-how-to-approach-this-and-the-url-shortener-connection)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [The Defining Decision: Split Metadata from Content](#4-the-defining-decision-split-metadata-from-content)
5. [Architecture](#5-architecture)
6. [Key Generation](#6-key-generation)
7. [Storage Decisions (Polyglot)](#7-storage-decisions-polyglot)
8. [Read & Write Flows](#8-read--write-flows)
9. [Expiration & Cleanup](#9-expiration--cleanup)
10. [Access Control, Abuse & Optional Features](#10-access-control-abuse--optional-features)
11. [Senior Follow-Up Questions (with Answers)](#11-senior-follow-up-questions-with-answers)
12. [Quick Glossary](#12-quick-glossary)

---

## 1. How to Approach This (and the URL-Shortener Connection)

Pastebin is, structurally, **a URL shortener that stores large content instead of a URL.** Almost everything from the URL-shortener guide carries over directly: a short base62 key, a key-generation service, an extreme read:write skew, and a cache-heavy read path. So the smart interview move is to **state that parallel explicitly** ("this is very similar to a URL shortener; let me reuse that skeleton") and then **spend your time on the one genuine difference: the content is large.**

That difference forces the central design decision — **where to store potentially large text blobs** — and everything interesting (object storage, the metadata/content split, expiration of bulky data, access control on content) flows from it. Leading with the URL-shortener analogy and then drilling into the storage divergence is exactly the focused, senior approach interviewers want.

---

## 2. Requirements

### Functional

- **Create:** paste text → receive a short URL.
- **Read:** anyone with the URL can read the paste.
- **Optional:** expiration, syntax highlighting, password protection, private pastes, edit history.

### Non-Functional

- **High read:write ratio (~100:1)** — like the shortener, it's overwhelmingly a *read/serving* system.
- **Low write latency** — creating a paste should feel instant.
- **Durability** — never lose a paste once stored.
- Search is **usually out of scope** (clarify with the interviewer).

---

## 3. Capacity Estimation

**Assumptions:** 1M new pastes/day; 100M reads/day (the 100:1 ratio); ~10 KB average paste.

```
Write QPS   = 1M / 10⁵ s/day        ≈ ~12 writes/sec
Read QPS    = 100M / 10⁵ s/day       ≈ ~1,200 reads/sec
Storage/day = 1M × 10 KB             = 10 GB/day
Storage/yr  = 10 GB × 365            ≈ 3.6 TB/year
```

**Takeaways:**
- Writes (~12/sec) are trivial; reads (~1.2K/sec) dominate → **caching is central** (same as the shortener).
- The new dimension vs. the shortener: **3.6 TB/year of *content***, growing unbounded. A URL shortener stored ~500 bytes/record; Pastebin stores ~10 KB+ each. That volume of bulky blobs is precisely why content goes to **object storage**, not the primary database.

---

## 4. The Defining Decision: Split Metadata from Content

The single most important design choice — and the senior signal — is to **separate the small metadata from the large content:**

- **Metadata** (short key, content location, expiry, owner, flags) is **small, structured, and frequently queried** → store in a **database**.
- **Content** (the actual paste text, 10 KB–MBs) is **large, opaque, and write-once** → store in **object storage (S3)**, with the DB holding only a **pointer** (the S3 URL/key).

Why not just store the content in the database? Putting large blobs in your primary DB **bloats it, slows queries, balloons cost, and strains backups/replication.** Object storage is purpose-built for cheap, durable blobs (storage-types and distributed-filesystem guides), so you let it do the heavy lifting and keep the DB lean and fast. This **metadata-in-DB, blob-in-object-store** pattern is the same one used for media in the WhatsApp design — and it's the heart of Pastebin.

---

## 5. Architecture

```
[Client] ──► [Load Balancer] ──► [API Server] ──► [Cache (Redis)]
                                      │                  │
                                      ▼                  ▼
                                 [Key Gen Svc]     [Object Store (S3)]  ← paste content
                                      │
                                      ▼
                                 [Metadata DB]   ← key → S3 location, expiry, owner
```

- **API server** (stateless, behind a load balancer) handles create/read, talking to key-gen, the metadata DB, the cache, and S3.
- **Key Gen Service** issues unique short keys.
- **Metadata DB** stores the small record per paste.
- **S3** stores the actual content blobs.
- **Redis** caches hot pastes to keep reads fast.

---

## 6. Key Generation

Identical to the URL shortener (see that guide for depth):

- **Base62 short keys** (`[a-zA-Z0-9]`); **7 chars = 62⁷ ≈ 3.5 trillion** combinations — vastly more than needed.
- **Pre-generate keys via a KGS** (Key Generation Service) so generation is collision-free and fast at request time (or use a counter + base62 with range allocation). Random keys also make pastes non-enumerable, protecting unlisted/private ones.

One nuance vs. the shortener: the key here **identifies where the content lives** — you store the content in S3 *at* that key (`s3://pastes/AbC1234`), so the short key is both the public identifier and the storage locator.

---

## 7. Storage Decisions (Polyglot)

A textbook case of **polyglot persistence** (SQL-vs-NoSQL guide) — three stores, each for what it's best at:

| Data | Where | Why |
|------|-------|-----|
| Short key → S3 location mapping | **DynamoDB / Cassandra** | Fast key-value point lookup at scale (the dominant read) |
| Paste **content** | **S3 / blob storage** | Cheap, durably stored, infinitely scalable; not query-able and not needed to be |
| User's pastes index | **PostgreSQL** | Relational queries ("list my pastes," joins with accounts) |

### Why S3 for content?

- **Cheap** (~$23/TB/month) — crucial given unbounded content growth.
- **Extremely durable** (~11 nines) — satisfies the "never lose a paste" requirement without you engineering replication yourself.
- **Infinitely scalable** — no capacity planning for the blob layer.
- The **DB stores only metadata + the S3 pointer**, staying small and fast regardless of how much content accumulates.

This is the metadata/content split made concrete: the hot, queried, small data lives in fast databases; the bulky, immutable content lives cheaply in object storage.

---

## 8. Read & Write Flows

### Read — `GET /AbC1234`

```
1. Check cache (Redis) for the paste → HIT → return content
2. MISS → look up metadata in DB (get S3 location, check not expired)
3. Fetch content from S3
4. Populate cache, return content
```

Because pastes are **immutable**, cached content never goes stale — ideal for aggressive caching (and popular pastes can even be fronted by a **CDN**, since the content is static and read-heavy — CDN guide). Cache the content for small/hot pastes; for very large pastes, you might cache only metadata and stream from S3 (or hand out a pre-signed S3 URL).

### Write — `POST /paste`

```
1. Receive content
2. Generate a short key (KGS)
3. Upload content to S3 at s3://pastes/<key>     ← content FIRST
4. Insert metadata { key, s3_url, expires_at, owner } into the DB
5. Return the short URL
```

**Ordering matters (a senior detail):** write the **content to S3 *before* the metadata to the DB.** If you did it the other way and crashed in between, the metadata would point to nonexistent content — a broken paste. Content-first means a crash leaves at worst an orphaned S3 object (cheap, cleanable), never a dangling pointer.

---

## 9. Expiration & Cleanup

Pastes often expire, and you must reclaim **both** the metadata and the content:

- **Background sweeper** — a job periodically scans the DB for expired pastes and deletes them from S3 and the DB. Full control, but it's work you operate.
- **S3 lifecycle policies (the elegant option)** — tag objects with a TTL and let **S3 auto-expire** them. This offloads content cleanup entirely to S3. You still need a DB-side TTL/sweep (or check `expires_at` on read and lazily purge) to clear the metadata, but the bulky content cleanup is handled for you.

A common combined approach: check `expires_at` on read (so expired pastes return 404 immediately) **and** rely on S3 lifecycle for content reclamation **plus** a light DB sweep for metadata. Given the huge key space, recycling expired keys generally isn't worth the complexity (same conclusion as the shortener).

---

## 10. Access Control, Abuse & Optional Features

Senior-level concerns to raise unprompted:

- **Private / password-protected pastes.** The content sits in S3, but **access control happens at the app layer**: the API checks ownership/password before returning the content (or before issuing a **pre-signed S3 URL** for direct download). Never make the S3 bucket public for private pastes; gate every fetch.
- **Abuse.** Pastebins are notoriously used to dump **stolen data, credentials, and malware**. Mitigations: **content size limits**, **rate limiting** on creation (rate-limiter guide), scanning/abuse-reporting, and takedown tooling — the same class of concern as phishing on a URL shortener.
- **Syntax highlighting.** Do it **client-side** (e.g., highlight.js) so the server just stores/serves raw text — keeps the backend simple and language-agnostic.
- **Edit history / diffing.** Store each edit as a **new version** (a new S3 object, with metadata linking versions); the UI computes/shows diffs between versions. Treating versions as immutable objects fits the write-once model.

---

## 11. Senior Follow-Up Questions (with Answers)

**Q1. How is this different from a URL shortener?**
Structurally it's the same (short base62 key, KGS, read-heavy, cache-first), but it stores **large content** instead of a URL. That forces the key decision: store the bulky content in **object storage (S3)** and keep only **metadata + a pointer** in the database.

**Q2. Why not store paste content in the database?**
Large blobs bloat the DB, slow queries, inflate cost, and burden backups/replication. S3 is cheap (~$23/TB/mo), ~11-nines durable, and infinitely scalable — purpose-built for blobs. Keeping content out of the DB keeps metadata lookups fast no matter how much content accumulates.

**Q3. In what order do you write content and metadata, and why?**
Content to S3 **first**, then metadata to the DB. If metadata were written first and you crashed, it would point to missing content (a broken paste). Content-first means a crash leaves only a cheap, cleanable orphaned S3 object — never a dangling pointer.

**Q4. How do you keep reads fast?**
Cache-first reads from Redis; pastes are immutable so cached content never goes stale. Hot/popular pastes can be served from a CDN since the content is static. Cache content for small/hot pastes; for large ones, cache metadata and stream from S3 or hand out a pre-signed URL.

**Q5. How do you generate keys?**
Same as the shortener: base62 (7 chars ≈ 3.5T combinations) via a KGS (or counter + base62 with range allocation). Random keys keep unlisted pastes non-enumerable. The key also locates the content in S3.

**Q6. How do you handle expiration of large content?**
Check `expires_at` on read (return 404 + lazily purge), and reclaim content via S3 lifecycle policies (auto-expire by TTL tag) plus a light DB sweep for metadata. S3 lifecycle offloads the bulky cleanup; recycling keys isn't worth it given the huge key space.

**Q7. How do private/password-protected pastes work given content is in S3?**
Access control is enforced at the application layer — the API verifies ownership/password before returning content or issuing a short-lived pre-signed S3 URL. The bucket is never public for private pastes; every fetch is gated.

**Q8. What are the abuse risks and mitigations?**
Pastebins are used to dump stolen data and malware. Mitigate with content size limits, creation rate limiting, scanning/abuse reporting, and takedown tooling — analogous to phishing defenses on a URL shortener.

**Q9. How would you support edit history?**
Store each edit as a new immutable S3 object with metadata linking the versions; show diffs in the UI. Immutable versions fit the write-once content model and make history trivially durable.

**Q10. Which databases and why (polyglot)?**
Key→location mapping in a KV/wide-column store (DynamoDB/Cassandra) for fast point lookups at scale; content in S3; the user's pastes index in PostgreSQL for relational queries. Each store does what it's best at — textbook polyglot persistence.

---

## 12. Quick Glossary

- **Paste** — a stored block of text identified by a short key.
- **Short key** — base62 identifier in the URL; also locates content in S3.
- **KGS (Key Generation Service)** — pre-generates unique keys for fast, collision-free creation.
- **Metadata** — small structured record (key, content location, expiry, owner) stored in a DB.
- **Content / blob** — the large paste text stored in object storage.
- **Object storage (S3)** — cheap, durable, scalable blob store; holds paste content.
- **Pointer** — the S3 location stored in the metadata DB.
- **Polyglot persistence** — using multiple databases, each for the data it suits.
- **Pre-signed URL** — a time-limited URL allowing direct, authorized S3 access.
- **Immutable content** — write-once data; ideal for caching and CDN serving.
- **S3 lifecycle policy** — automatic expiry of objects by TTL tag, offloading cleanup.
- **Content-first write ordering** — store the blob before the metadata to avoid dangling pointers.

---

*Reference document. Contributions and corrections welcome.*
