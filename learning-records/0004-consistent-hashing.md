# Learning record — Lesson 0004: Consistent Hashing

## What this lesson covered
Resolved Lesson 0003's cliffhanger: adding a 17th shard with `hash % N` remaps
~94% of 6B rows. Applied the four moves to make membership changes cost ~1/N.

- **Estimate** — priced both migrations on the same hardware change (16→17):
  - Naive `% N`: a key stays only if `hash%16 == hash%17` (~1/17), so 94.1% move
    = 0.941 × 1.2 TB ≈ **1.13 TB**; at ~1 GB/s ≈ **19 min** of ambiguous fleet shuffle.
  - Consistent hashing: only the new node's arc moves (~1/17) = 6B/17 ≈ 353M rows
    ≈ **71 GB**; at ~1 GB/s ≈ **71 s**. ~16× less data / time.
- **Model** — the hash ring: range `0…2³²−1` bent into a circle; hash BOTH nodes
  and keys into it; a key is owned by the **first node clockwise**. Toy ring
  0..999, shards A@120 B@370 C@610 D@850, key "8tWk"@410 → C (smallest point ≥410,
  wrap if none). Lookup = binary search, O(log n). Adding E@500 splits arc
  (370→610]: only (370,500] moves C→E; everything else untouched.
- **Trace** — two bugs of one-point-per-node: (1) uneven arcs → load skew;
  (2) a dead node dumps its whole arc on the single clockwise neighbor (~2×).
  Fix = **virtual nodes**: hash each shard at ~150 points ("shard-C#1"…). Balance
  improves ~**1/√V** (V=100 → ~10%, V=256 → ~6%; real systems ~100–256). On
  failure a dead node's 150 arcs merge into many different neighbors: each of 15
  survivors gains only (1/16)/15 ≈ 0.42% (6.25%→6.67%), no neighbor doubles.
- **Next bottleneck** — named two things the ring does NOT solve:
  (a) hot key — it balances key COUNT not request HEAT; one viral code hits one
  shard (handle via cache/replication), and (b) rebalancing is heavy background
  work that must stay off the request path → queues/workers.

## Trade-offs named
- memory + lookup complexity (sorted ring, binary search) vs ~16× smaller migration
- vnode count: more vnodes = smoother load + gentler failure vs more ring entries
  (N×V points), memory, slower lookups (sweet spot ~100–256)
- even-by-count ≠ even-by-traffic (the hot-key limitation)

## What it sets up next
- **Lesson 0005 — Async work (queues + workers):** the bulk API and the 71 GB
  rebalance shouldn't run inline → push to a queue, drain with workers
  (at-least-once, retries, ordering, DLQ). Explicitly teased.
- **Lesson 0006 — Consistency & replication:** a dead shard's keys must already
  live elsewhere; like-counter under concurrency.

## Curriculum bookkeeping
- Marked spine #4 ✅ (Lesson 0004). Spine still has #5–#13 queued; no need to add
  new topics yet (plenty of runway).
