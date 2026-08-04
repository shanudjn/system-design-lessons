# Learning Record — Lesson 0048: Distributed Caching & Invalidation at Scale

**Date:** 2026-08-04
**File:** `lessons/0048-distributed-caching-invalidation.html`
**Curriculum topic:** #48 (Advanced batch — "the course never runs dry" queue)
**Trade named:** freshness vs cache hit rate & load

## The worked example
A product catalog read path: **10M products × 2 KB = ~20 GB** working set,
**50,000 reads/s** at peak, against a database that safely serves only
**~5,000 reads/s** (Lesson 27's utilization knee), with the catalog changing at
**~500 writes/s** (100:1 read:write). The read fleet is Lesson 27's **286 app
servers**. The point: shared + hot + *changing* is where a cache stops being a
speed trick and becomes a correctness (coherence) problem.

## The four moves
1. **Estimate — one shared copy beats 286 private ones.** Private caches
   multiply the RAM (286 × 20 GB = 5.72 TB to hold the catalog 286×) but, more
   importantly, multiply the *invalidation surface*: one write → up to 286 stale
   copies to chase. A shared tier holds **one copy per key** (consistent hashing,
   L04) → one delete to invalidate. Offload must be near-total: `1 − 5k/50k = 90%`
   minimum (zero headroom) → aim **99% hit**, misses = 500/s (1/10 of DB budget).
2. **Model — the write is the hard half.** Three policies (cache-aside /
   write-through / write-behind) on a freshness↔latency curve, plus the sharper
   fork: **delete** (idempotent, race-free, but a miss) vs **versioned overwrite**
   (no miss, but needs compare-and-set, L06, or it loses updates in the cache).
3. **Trace** — hit (99%, DB untouched); miss (populate + TTL backstop, L14 dial);
   clean write (**source of truth first, then invalidate**); and the
   **stale-repopulation race** where a slow *reader* SETs an old value back after a
   newer write — bounded only by a version stamp or a TTL (delete-vs-overwrite
   can't fix a reader-vs-writer race).
4. **First bottleneck — the near cache brings N copies back.** A per-server L1
   (20 MB) beats the network hop but recreates 286 copies → invalidation becomes a
   best-effort **pub/sub fan-out** (500 × 286 = 143k drops/s) whose only hard
   guarantee is a short **TTL**. Deleting one **hot key** (10k/s → `L=R×T=100`
   simultaneous misses, L13, on one hot shard L03) reignites L02's **thundering
   herd** → single-flight, serve-stale/SWR, or versioned overwrite.

## Four traps
1. Invalidating the cache before writing the DB (reader fills the gap with the old value).
2. Overwriting in place without a version (lost update inside the cache, L06).
3. Trusting the invalidation fan-out to be reliable (it's best-effort → keep the TTL short).
4. Deleting a hot key and expecting the DB to absorb the herd (single-flight / SWR / versioned overwrite).

## Reuse / callbacks
L02 (cache-aside, TTL/eviction, thundering herd, single-flight, SWR), L03 (hot
shard), L04 (consistent-hash routing → one copy per key), L06 (lost update +
compare-and-set), L09 (pub/sub fan-out), L13 (Little's Law sizes the herd), L14
(freshness-vs-speed, TTL-vs-purge), L27 (fleet/knee/99% offload), L47
(memtable-then-flush ≈ write-behind buffering).

## What it sets up next
Topic #49 — **API gateways & the edge**: the single front door in front of the
fleet (auth, routing, request aggregation/BFF, and enforcing cross-cutting
concerns — rate limits L08, quotas, mTLS L30 — in one place), and the mirror-image
risk of this lesson's shared tier: a fat gateway that becomes a SPOF (L07/26) and
a deploy bottleneck (L31/36). Trade: centralized cross-cutting concerns vs a
fragile shared choke point.
