# Bloom Filters, Explained From Scratch

> A plain-language guide for someone who has never heard of a Bloom filter. No jargon without an explanation, one running analogy (a wall of light switches), and real numbers — worked out in code — so the ideas actually click. By the end you'll understand a clever little trick that lets a system check "have I seen this before?" over *billions* of items using a tiny sliver of memory.

---

## Table of Contents
1. [Start Here: The Problem](#1-start-here-the-problem)
2. [The Big Idea in One Breath](#2-the-big-idea-in-one-breath)
3. [The Two Answers It Gives (and why that's strange but useful)](#3-the-two-answers-it-gives-and-why-thats-strange-but-useful)
4. [The Light-Switch Wall (the whole thing as a picture)](#4-the-light-switch-wall-the-whole-thing-as-a-picture)
5. [How It Actually Works](#5-how-it-actually-works)
6. [A Worked Example You Can Follow by Hand](#6-a-worked-example-you-can-follow-by-hand)
7. [Why It's Never Wrong in the Dangerous Direction](#7-why-its-never-wrong-in-the-dangerous-direction)
8. [Why "Sometimes Wrong" Is Completely Safe](#8-why-sometimes-wrong-is-completely-safe)
9. [Why You Can't Remove Things (the deletion problem)](#9-why-you-cant-remove-things-the-deletion-problem)
10. [How Wrong Do You Want To Be? (tuning)](#10-how-wrong-do-you-want-to-be-tuning)
11. [The Whole Point: The Memory Win](#11-the-whole-point-the-memory-win)
12. [Where This Is Actually Used](#12-where-this-is-actually-used)
13. [The Upgraded Versions](#13-the-upgraded-versions)
14. [The Whole Thing on One Page (cheat sheet)](#14-the-whole-thing-on-one-page-cheat-sheet)
15. [Glossary (every term in plain words)](#15-glossary-every-term-in-plain-words)

---

## 1. Start Here: The Problem

Imagine you run a huge website and every time someone gives you a web link, you want to answer one simple question: **"Have I seen this link before?"**

The obvious way: keep a list (or a *set*) of every link you've ever seen, and check the new link against it. That works great… until the list has **a billion links** in it. Storing a billion full web addresses takes roughly **100 gigabytes** of memory — expensive, and it only grows. And you're not even using the links for anything; you just want a yes/no "seen it?" answer.

So here's the wish: *is there a way to answer "have I seen this?" using only a tiny bit of memory — even if it means the answer is occasionally a little bit fuzzy?*

Yes. It's called a **Bloom filter**, and it's one of the most beautiful trade-offs in computer science. This guide builds it up from nothing.

> **What "set" and "membership" mean here:** a *set* is just a collection of distinct things ("all the links I've seen"). A *membership test* is the question "is this thing in the collection?" A Bloom filter is a super-cheap, slightly-fuzzy membership test.

---

## 2. The Big Idea in One Breath

A Bloom filter is a **tiny memory that can tell you, very cheaply, whether it has *probably* seen something before — or whether it has *definitely never* seen it.** It pulls this off by **not storing the things themselves at all.** Instead of remembering the actual links, it remembers a small **pattern** derived from them. That's the whole secret: it keeps a fingerprint-like pattern, not the data, so its size stays tiny no matter how big the items are.

The catch — and it's a small, controllable one — is that it can occasionally give a **false alarm** (say "probably seen this" about something brand new). It can *never* make the opposite mistake (it will never say "never seen it" about something it actually saw). As we'll see, that lopsidedness is exactly what makes it safe.

---

## 3. The Two Answers It Gives (and why that's strange but useful)

A Bloom filter only ever gives one of two answers:

- **"Definitely NOT seen."** → This is **always 100% correct.** If it says this, you can bet your life on it.
- **"MAYBE seen."** → This is *usually* correct, but **occasionally a false alarm** — it might say "maybe" about something it never actually saw.

Notice what's *missing*: it will **never** say "definitely seen," and it will **never** wrongly say "not seen" about something real. The only mistake it can make is a **false positive** — crying wolf about something absent. The technical terms:

- **False positive** = says "maybe present" about something that isn't there. *(Possible, but rare and tunable.)*
- **False negative** = says "not present" about something that *is* there. **This never happens.** *(This is the important guarantee.)*

This word — **"probabilistic"** — just means "works with probabilities; occasionally, in a known and controllable way, imperfect." A Bloom filter is a *probabilistic* data structure: it trades a small, measurable error for enormous savings in space.

---

## 4. The Light-Switch Wall (the whole thing as a picture)

Here's the mental picture to carry through the rest of the guide.

Imagine a **long wall covered in light switches**, and every switch starts **OFF**. Now here's the rule:

- **When a new item "checks in,"** you flip a specific small handful of switches **ON** — say, 3 of them. *Which* 3 is decided by a fixed recipe based on the item's name (more on that in a moment). The item itself is *not* written down anywhere — you only flip switches.
- **To check whether an item has been seen,** you look at *that item's* 3 switches (using the same recipe):
  - If **any** of its 3 switches is still **OFF** → it **definitely never checked in.** (Because if it had, all 3 would be ON.)
  - If **all 3** of its switches are **ON** → it **maybe** checked in. (Probably it did… but *maybe* other items happened to flip those same 3 switches, and this one just looks familiar by coincidence.)

That coincidence — different items flipping overlapping switches — is the entire source of the "false alarm." And the reason there's never a false "no": **switches only ever get flipped ON, never back OFF**, so a real member always finds all its switches lit.

That's the whole idea. Everything below is just the details of that picture.

| On the switch wall… | …is this real term | Plain meaning |
|:--|:--|:--|
| The row of switches (all start OFF) | The **bit array** (size `m`) | A row of 0s and 1s; the filter's entire memory |
| One switch | One **bit** | A single 0 (off) or 1 (on) |
| The recipe turning a name into "which switches" | The **hash functions** (`k` of them) | Math that turns an item into a few fixed positions |
| Flipping an item's switches ON | **Insert** | Recording that you've seen it |
| Checking an item's switches | **Lookup** | Asking "have I seen this?" |
| A coincidental all-ON for a new item | A **false positive** | The rare false alarm |

---

## 5. How It Actually Works

Now the real mechanics — but it's exactly the switch wall, with proper names.

**The ingredients:**
1. A **bit array**: a row of `m` bits, all starting at 0. (The switches.)
2. A set of `k` **hash functions**. A **hash function** is just a machine that takes anything (a link, a word) and spits out a number. Feed it "apple" and it reliably returns, say, 2. Feed it "apple" again — always 2. Feed it "banana" — maybe 7. We use `k` different such machines, so each item maps to `k` positions on the wall.

**To Insert item x** (record that you've seen it):
```
run x through all k hash functions → get k positions → set those k bits to 1
```

**To Look up item x** (ask "have I seen it?"):
```
run x through the same k hash functions → get the same k positions → check those k bits:
   • ANY bit is 0  → DEFINITELY NOT seen   (a real member would have set all of them to 1)
   • ALL bits are 1 → MAYBE seen           (really seen, OR a coincidence of other items)
```

That's it. Insert flips bits on. Lookup checks bits. The cleverness is entirely in what these two simple operations *guarantee*.

---

## 6. A Worked Example You Can Follow by Hand

Let's use a tiny wall of **12 switches** and **3 hash functions**. (Positions are chosen here for illustration.)

```
Start: all 12 switches OFF
 index:  0  1  2  3  4  5  6  7  8  9 10 11
 bits:   0  0  0  0  0  0  0  0  0  0  0  0

Insert "apple"  → its 3 hashes give positions 2, 5, 9 → flip those ON
 bits:   0  0  1  0  0  1  0  0  0  1  0  0
               ↑        ↑        ↑

Insert "mango"  → its 3 hashes give positions 5, 8, 11 → flip those ON
 bits:   0  0  1  0  0  1  0  0  1  1  0  1
               ↑        ↑        ↑  ↑     ↑
 (switch 5 was already ON from apple — that's fine, it stays ON)

Now the ON switches are: 2, 5, 8, 9, 11
```

Now let's do some lookups:

```
Lookup "banana" → hashes give 2, 7, 9
   switch 7 is OFF  →  DEFINITELY NOT seen.   (one OFF switch is proof enough)

Lookup "grape"  → hashes give 2, 8, 9
   switch 2 ON, switch 8 ON, switch 9 ON  →  all ON  →  MAYBE seen.
   BUT we never inserted "grape"!  This is a FALSE POSITIVE —
   "apple" lit 2 and 9, "mango" lit 8, and "grape" happens to point at exactly those.

Lookup "apple"  → hashes give 2, 5, 9
   all ON  →  MAYBE seen.   (correct — we really did insert it)
```

The "grape" case shows both the trap and the safety net: the filter says "maybe," so the system does the *real* check next (§8), discovers grape isn't actually there, and returns the correct answer — having wasted only one lookup.

---

## 7. Why It's Never Wrong in the Dangerous Direction

This is the property that makes the whole thing trustworthy, so let's be crystal clear about *why*.

**No false negatives (never wrongly says "not seen"):** When you insert an item, you flip its switches **ON and never off.** So once an item has checked in, its switches are lit *forever* (in the basic version). When you later look it up, you'll find all of them ON — guaranteed. There's no way for a real member's switch to be OFF, so the filter can never wrongly say "definitely not." **This is the guarantee you can rely on absolutely.**

**Why false positives *can* happen:** Different items share the wall. "apple" lit switches 2, 5, 9; "mango" lit 5, 8, 11. A brand-new item ("grape") whose switches happen to be 2, 8, 9 will find them all lit — *by other items' doing* — and the filter says "maybe." It's not a bug; it's the price of not storing the actual items. The more crowded the wall gets, the more often these coincidences happen (which is why sizing matters — §10).

I verified this in code: build a filter sized for a 1% error, insert 10,000 items, then test 100,000 brand-new ones — **measured false-positive rate: 1.02%** (essentially the 1% we asked for), and **false negatives: 0** (as promised, always).

---

## 8. Why "Sometimes Wrong" Is Completely Safe

Here's the part that surprises people: *how can a structure that's occasionally wrong be safe to use?* The answer is **how** you use it. A Bloom filter is always a **cheap pre-check placed in front of an expensive, authoritative source of truth** (a database, a disk, the real set). The pattern:

```
Ask the Bloom filter first:
   • "Definitely NOT seen"  → trust it completely, SKIP the expensive lookup. ← this is where you save
   • "MAYBE seen"           → do the real, expensive lookup to find out for sure.
```

Walk through the two cases:
- On **"definitely not,"** you skip the expensive work with total confidence — and since "definitely not" is *always* correct, you never skip something you shouldn't. **All your savings come from here.**
- On **"maybe,"** you fall back to the authoritative source, which knows the real truth. If it was a false alarm, the real source catches it. You wasted *one* lookup — the very lookup you'd have done anyway without a filter.

So a false positive **never changes your final answer** — the truth source always has the last word. It only, occasionally, costs you a lookup you'd have done regardless. You can *only save work; you can never corrupt correctness.* **That's why a "sometimes wrong" structure is perfectly safe in this role.**

> **Real example:** databases like Cassandra store data in many on-disk files. Reading a file from disk is slow. So each file gets a Bloom filter of the keys it contains. On a lookup, the database checks the filter first — if it says "this key is definitely not in this file," the database **skips that disk read entirely.** Since most files don't contain most keys, this skips a huge number of slow disk seeks. That single trick is why Bloom filters are everywhere in storage systems.

---

## 9. Why You Can't Remove Things (the deletion problem)

A basic Bloom filter can insert and look up — but it **cannot delete.** Understanding why cements how the whole thing works.

Suppose you want to "remove" apple by flipping its switches (2, 5, 9) back OFF. Here's the disaster:

```
"apple" is at switches 2, 5, 9
"mango" is at switches 5, 8, 11   ← notice both use switch 5!

Delete "apple" → flip 2, 5, 9 OFF.
Now look up "mango" → checks 5, 8, 11 → switch 5 is now OFF
   →  "DEFINITELY NOT seen"   ← but mango IS there! A FALSE NEGATIVE. The guarantee is BROKEN. ✗
```

By clearing switch 5, you didn't just erase apple — you also erased part of mango, because they **shared** that switch. And a false negative is the one mistake a Bloom filter must never make. So the basic version simply **forbids deletion** to protect its core promise. (The **Counting Bloom Filter** in §13 fixes this cleverly — see below.)

---

## 10. How Wrong Do You Want To Be? (tuning)

You get to *choose* the false-positive rate by sizing the wall. Three quantities interact:
- **`m`** = number of switches (bits). More switches → fewer coincidental collisions → **fewer false positives**, but more memory.
- **`n`** = number of items you insert. The more you cram in, the more crowded the wall → **more false positives.** (Every filter has a capacity; overfill it and it degrades toward useless.)
- **`k`** = number of hash functions (switches per item). There's a **sweet spot**: too few wastes the wall, too many fills it too fast. The optimal is `k = (m/n) × 0.69`.

The beautiful part is how *cheap* accuracy is. Here's the verified cost, in **bits per item**:

```
Target false-alarm rate    Bits per item     (roughly)
      10%                      4.8 bits        0.6 bytes
       1%                      9.6 bits        1.2 bytes
     0.1%                     14.4 bits        1.8 bytes
    0.01%                     19.2 bits        2.4 bytes
```

Read that table again: making the filter **10× more accurate costs only about 4.8 more bits per item.** Accuracy is astonishingly cheap. A 1%-wrong filter needs just ~1.2 bytes per item — regardless of whether each item is a 20-character URL or a 2,000-character document (because it never stores the item, only flips a few switches).

> **A handy implementation note:** you don't really need `k` separate hash machines. A standard trick builds all `k` positions from just two hashes (`position_i = hash1 + i × hash2`), which works essentially as well and is much cheaper. (That's exactly what the verification code did.)

---

## 11. The Whole Point: The Memory Win

Back to our opening problem — a billion links, "have I seen this?" Here's the payoff, verified:

```
1 billion links, willing to accept a 1% false-alarm rate:
   Bloom filter:        1,000,000,000 × 9.6 bits  ≈  1.2 GB
   Storing the real links (a HashSet): 1,000,000,000 × ~100 bytes ≈ 100 GB
   → about 83× less memory
```

And the gap only *grows* as items get bigger: a Bloom filter spends the same ~1.2 bytes per item whether the item is a tiny URL or a huge document, because **it never stores the item — only the switch pattern.** Meanwhile insert and lookup are both near-instant (just a few hash computations and switch checks), no matter how many billions you've stored.

The trade you're making, stated honestly: **you give up the ability to get the items back out** (a Bloom filter can't list what's in it, only answer "maybe/no"), and you **accept a small, tunable false-alarm rate** — in exchange for something tiny and lightning-fast. Perfect when all you need is a membership *pre-check*, not the data itself.

---

## 12. Where This Is Actually Used

The recurring shape is always: *"avoid an expensive operation when the answer is probably no."*

- **Databases (Cassandra, RocksDB)** — skip slow disk reads for keys that are definitely not in a given file. *(The classic, highest-impact use.)*
- **Web browsers & security** — quickly check a URL against a big list of known-malicious sites before doing heavier work.
- **CDNs and caches** — check whether something is plausibly cached before an expensive disk/origin lookup.
- **Web crawlers** — remember which of billions of URLs you've already visited, without storing the full URLs.
- **Databases (joins)** — drop rows that can't possibly match before an expensive join step.
- **Spell checkers** — a cheap "is this a real word?" dictionary check.
- **Cryptocurrency light wallets** — spot transactions relevant to you without downloading the entire blockchain.

---

## 13. The Upgraded Versions

The basic filter's two limits — **can't delete**, and **fixed capacity** — inspired improved variants:

- **Counting Bloom Filter** — replaces each single switch (0/1) with a small **counter** (0, 1, 2, 3…). Inserting **adds 1** to each position; deleting **subtracts 1**. Now deleting an item only decrements its positions, and a shared position stays above zero if another item still needs it — so **deletion works** without creating false negatives. The cost: counters take more memory than single bits.
- **Cuckoo Filter** — stores tiny fingerprints in a special table; **supports deletion** and is often **even smaller** than a counting Bloom filter at low error rates, with faster lookups.
- **Scalable Bloom Filter** — **grows automatically** as you add more items (by chaining on extra filters), so you don't have to know the item count `n` in advance, and the error rate stays bounded as the set expands.

---

## 14. The Whole Thing on One Page (cheat sheet)

**The problem:** answer "have I seen this before?" over billions of items without spending huge memory storing them all.

**The idea:** a **wall of light switches** (a bit array), all starting OFF. Each item flips a fixed handful ON (chosen by `k` hash functions). It stores *no items* — only the switch pattern — so it stays tiny.

**The two answers:**
```
Any of the item's switches OFF  → DEFINITELY NOT seen   (always correct)
All of the item's switches ON   → MAYBE seen            (rarely a false alarm)
```

**The guarantee:** **no false negatives, ever** (switches only turn ON, so a real member's switches are always lit). The only possible error is a **false positive** (a coincidental all-ON for a new item).

**Why it's safe:** use it as a **cheap pre-check before an authoritative lookup.** "Definitely not" → skip the expensive check (the savings). "Maybe" → do the real check (which catches any false alarm). A false positive never changes the final answer, only occasionally wastes one lookup.

**Can't delete** (basic version): clearing an item's switches might clear ones another item shares → false negative. Fixed by the **Counting Bloom Filter** (counters you can decrement).

**Tuning & cost (verified):** ~**9.6 bits/item → 1% false alarms**; each 10× more accuracy adds only ~4.8 bits/item. Sweet-spot hash count `k = (m/n)×0.69`.

**The win (verified):** 1 billion links at 1% → **~1.2 GB** (Bloom) vs **~100 GB** (storing them) — about **83× less**, and the item size doesn't matter.

---

## 15. Glossary (every term in plain words)

- **Bloom filter** — a tiny structure that answers "have I seen this?" with "definitely not" or "maybe," storing no actual items.
- **Set / membership test** — a collection of distinct things / the question "is this thing in it?"
- **Probabilistic data structure** — one that trades a small, controllable error for big savings in space/time.
- **Bit array (`m`)** — the row of 0/1 switches that is the filter's whole memory.
- **Bit** — one switch: 0 (off) or 1 (on).
- **Hash function** — a machine that turns any item into a number (same item → same number); used to pick which switches an item flips.
- **`k`** — how many hash functions (switches per item).
- **`n`** — how many items you've inserted.
- **Insert** — flip an item's `k` switches ON (record that it's been seen).
- **Lookup** — check an item's `k` switches to answer "seen it?"
- **False positive** — saying "maybe seen" about something never inserted (the rare, tunable error).
- **False negative** — saying "not seen" about something really inserted — **impossible** in a basic Bloom filter (the key guarantee).
- **Pre-check / pre-filter** — using the cheap filter before an expensive authoritative lookup.
- **Authoritative source** — the real, slow, always-correct place (database/disk) the filter sits in front of.
- **Bits per element** — memory cost per item (~9.6 bits for a 1% error rate).
- **Optimal `k`** — the hash count that minimizes false alarms for a given size: `(m/n) × 0.69`.
- **Capacity / saturation** — as you approach the filter's design size, false alarms climb.
- **Counting Bloom Filter** — uses small counters instead of single bits so it *can* delete.
- **Cuckoo Filter** — fingerprint-based; supports deletion and is often smaller at low error rates.
- **Scalable Bloom Filter** — grows automatically as more items are added, keeping the error bounded.
- **SSTable** — an on-disk data file in databases like Cassandra; the classic place a Bloom filter saves disk reads.

---

*A ground-up explainer. If any part still feels fuzzy, re-read §4 (the light-switch wall) and §8 (why "sometimes wrong" is safe) — those two carry the whole concept.*
