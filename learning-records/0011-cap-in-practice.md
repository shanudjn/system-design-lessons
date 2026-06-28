# Learning record — Lesson 0011: CAP in Practice

**Date:** 2026-06-28
**File:** `lessons/0011-cap-in-practice.html`
**Title:** CAP in Practice: What a Partition Forces You to Give Up

## What this lesson covered
Took ONE concrete system — a single account balance ($100) held on three
replicas across three zones — cut the trans-atlantic cable, and traced what
the partition forces. Followed the four-move spine:

- **Estimate** — a single 30 s partition on a 15k req/s store touches the
  isolated zone's ~5,000 req/s × 30 s = **150,000 requests**, several times a
  month. Partitions are a Tuesday, not an edge case. Established that **P is not
  optional** (the network imposes it), so the honest CAP is "choose C or A, and
  only while a partition is happening."
- **Model** — recovered the quorum rule `W+R>N` from L06: W=2,R=2,N=3 → read/write
  sets of 2 in a pool of 3 must overlap (pigeonhole, same as L10's majority
  proof) → linearizable read. Showed the split leaves the **majority side**
  {Z2,Z3} fully consistent+available and forces the dilemma onto the **minority**
  {Z1}, which can't reach quorum.
- **Trace the paths** — (A) healthy: both C and A for free, so "CP/AP" describes
  behavior *under partition*; (B) **CP** choice — minority refuses the non-quorum
  write (errors, but the double-spend is mathematically impossible; L10's
  "minority steps down" applied to data); (C) **AP** choice — both sides answer,
  diverge, and pay a reconciliation bill ($100 − $160 = −$60/account → $60k for
  1,000 accounts under last-write-wins). Right for a mergeable shopping cart
  (set-union), wrong for a scalar balance.
- **Next bottleneck** — the choice CAP *hides*: **PACELC**. Even with no
  partition you trade Latency vs Consistency — a strongly-consistent cross-region
  write waits ~90 ms for a remote quorum ack vs ~1 ms local-then-replicate. That
  everyday tax usually matters more than the rare partition. Per-data table
  (balance/cart/last-seen/username) shows you mix CP and AP in one product. Wall:
  "AP is only as good as your ability to reconcile" → idempotency / CRDTs.

## Trade-offs named
- P is forced; the real choice is C **vs** A, only during a partition
- CP: correctness **vs** minority uptime (honest 503 over a confident lie)
- AP: uptime everywhere **vs** divergence you must reconcile (LWW loses data;
  merge/CRDT keeps it but only for mergeable data types)
- PACELC "Else": latency (~1 ms local) **vs** consistency (~90 ms quorum)

## Callbacks to earlier lessons
L10 "minority steps down" sacrifice (now for data, not leadership); L06 quorum
`W+R>N` and sharded/approximate counter; L07 "honest failure over a confident
lie" (503/429) reflex.

## What it sets up next
- **Lesson 12 — Idempotency:** the machinery that makes AP's "reconcile later"
  and L07's retries safe — dedup keys, exactly-once *effects* (not delivery),
  replayed write as a no-op, link to CRDT merges.
- DB indexing & search (B-trees, inverted indexes) queued after.
