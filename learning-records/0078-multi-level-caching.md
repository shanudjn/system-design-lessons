# Lesson 0078 — Multi-Level Caching: One Write Down Three Layers

**File:** `lessons/0078-multi-level-caching.html`
**Curriculum spine:** topic #78 (advanced batch) — now ✅
**Date:** 2026-09-03

## The one worked system
An e-commerce product page over a cache hierarchy: per-server in-process **L1**
(~100 ns, NOT shared → 200 independent copies), shared **Redis L2** (~0.5 ms),
**DB** as the source of truth (~10 ms), and a **CDN** at the edge (L14). Two data
types on the same page behave oppositely: a **price** (read-heavy, correctness-
critical) and a **"viewers now" counter** (write-heavy, loss-tolerant).

## What it covered (the four moves)
- **Estimate** — the layered read: 0.9/0.09/0.01 hit split → `0.00009 + 0.045 +
  0.100 = ~0.145 ms` average, ~69× vs a raw 10 ms DB read, DB sees only 1% (400
  rps). Write policies by ack latency: write-through `0.5+10 = ~10.5 ms`,
  write-back `~0.5 ms` (21× faster), write-around `~10 ms`. Coalescing headline:
  a 2,000 incr/s hot counter = 2,000 DB writes/s under write-through (L6 hot row)
  vs **1/s** under write-back (2,000×).
- **Model** — the hierarchy (each tier trades a weakness below for one more copy
  to keep coherent) and the three write policies as ONE decision "how far down
  before ACK": write-through (spend latency, buy durability — prices), write-back
  (spend durability/RPO, buy latency + coalescing — counters), write-around
  (spend a first-read miss, buy a clean cache — imports). Plus inclusive
  (L1⊆L2, simple, duplicated) vs exclusive (one tier each, +capacity, complex).
- **Trace** — (A) price change write-through + pub/sub invalidate; (B) counter
  write-back coalesce; (C) crash 0.7 s after flush (counters lost = shrug; prices
  would VANISH → why policy is per-type); (D) 5M-row import write-around leaves
  the hot set untouched.
- **First bottleneck** — **coherence** across 200 private L1 copies a write never
  touched → short TTL vs pub/sub invalidation (+ TTL backstop) vs versioned keys.
  Then: write-back dirty = un-replicated in-memory truth (RPO = flush interval →
  replicate/log/flush-often, never money); synchronized cross-tier expiry
  stampede (→ single-flight + jitter + serve-stale); CDN = farthest copy
  (→ purge vs versioned URLs).

## Key idea to carry
A cache is a *copy* of the truth; a **write changes the truth**, so it
invalidates every copy at once. Reads are the easy half (a stale read = one wrong
answer); the write is the hard half — decide per data type how far down it must
travel and how the copies you didn't touch will learn the truth moved. Deepest:
caching is the craft of handling the moment the "truth won't change" bet loses.

## Reuses
L2 (single cache, herd, single-flight, TTL/eviction, read-through/cache-aside),
L14 (CDN edge, purge vs immutable URLs), L48 (invalidation race + pub/sub),
L6 (hot-row write wall, coalescing), L26 (push invalidation, disposable fast
path), L7 (RPO/durability), L27 (the 200-server fleet).

## Trade named
Read/write latency vs durability & coherence.

## Sets up next
**Lesson 79 — Sharding strategies & resharding live:** we kept assuming "the
database" is one logical box, but at 50M products it must be split. Directory vs
range vs hash sharding, resharding a hot shard live (L24 online-migration shape),
cross-shard queries + scatter-gather tax (L19), shard key as a one-way door.
Trade: even load & locality vs query flexibility.
