# Learning record — Lesson 0029: Data Warehousing & OLAP

**File:** `lessons/0029-data-warehousing-olap.html`
**Topic (curriculum spine #29):** Data warehousing & OLAP (batch + streaming)
**Trade named:** query freshness vs cost & complexity

## The one system
The order events from L12/L25 at scale: **10M orders/day × 3 years ≈ 11B rows**, ~50
columns, ~1 KB each → **~11 TB**. One analytics question: *"total revenue by region, by
day, for the last quarter (90 days)"* — touches only **2 columns** over **90 of 1,095
day-partitions**. This is the OLAP workload the previous 28 lessons (all serving/OLTP)
never built for. What machine answers it, and why isn't it the serving DB?

## What the lesson covers (four moves)
- **Estimate — same query, two layouts.** Row store (L12) drags whole 1 KB rows to read
  2 of 50 columns: 900M rows × 1 KB = **900 GB → 1,800 s = 30 min** @ 500 MB/s (whole-table
  scan would be ~6.1 h). Column store reads only the 2 columns (900M × 10 B = **9 GB**,
  100× less), compresses ~3.6× (→ **~2.5 GB**), scans in **5 s**, **~0.5 s across 10 nodes**
  → **360×**, all from reading fewer bytes, not a faster disk. Layout follows access pattern;
  no single layout is best at both, so you keep two differently-shaped copies.
- **Model — separate columnar warehouse, fed by a pipeline.** Never run analytics on the
  serving DB: the 900 GB scan evicts its hot cache (L02) and trips L27's utilization knee →
  L07 timeouts/retry storm. So a SEPARATE warehouse: **columnar**, **partitioned** (enables
  pruning), **denormalized** into a **star schema** (one wide fact table + small dimension
  tables; precompute the join = L15's precompute-the-read; normalize for writes, denormalize
  for reads). An **ETL/ELT** pipeline copies OLTP → warehouse on a schedule → the warehouse
  is a *copy*, and therefore *stale* (the seed of the first wall).
- **Trace — load + scan.** Load: an order commits to OLTP in ms (L25), enters analytics only
  at the next batch (CDC extract → transform/denormalize → load into the day-partition,
  columnar-encoded on write) — up to ~24 h stale. Scan: **prune** (90 of 1,095 partitions,
  zero bytes from the other 1,005) → **project** (2 of 50 column files) → **scan** (decompress)
  → **aggregate-and-merge** partials across nodes (L21's local-compute-then-central-merge).
  Fast OLAP is mostly the art of NOT reading data: prune, then project, then compute.
- **First bottleneck — staleness, and three traps.** "Batch more often" loses: batch cost
  scales with history × frequency, and the 6.1 h full re-scan can't fit a 5-min window → need
  a layer whose cost scales with NEW data. **Lambda** = batch layer (complete/correct/stale) +
  speed/stream layer (fresh/approximate/recent), merged at read — pays in SAME logic maintained
  twice (drift bugs). **Kappa** = one streaming pipeline, reprocess history by REPLAYING the
  L09 retained log through the same job — pays in expensive replay. Traps: (1) point lookup on
  a scan engine = L12's selectivity trap REVERSED (send point reads to OLTP); (2) over-partition
  by minute → 1,095 × 1,440 ≈ 1.58M small files → pruning drowns in metadata (L20); (3) late/
  out-of-order events break streaming windows → **watermark** holds windows open for stragglers
  (L17's tail/late arrival), why streaming is "approximate" until batch reconciles.

## Threads reused
- **L12** — 500 MB/s scan math; the selectivity trap (here reversed: scan engines are bad at
  point lookups).
- **L15** — precompute-the-read (denormalization / star schema).
- **L21** — local partial aggregates then a central merge; scales cleanly across nodes.
- **L20** — the small-file problem, now on the analytics side (over-partitioning).
- **L17** — late/out-of-order arrivals → watermarks in streaming aggregates.
- **L09** — the retained log as the replayable source of truth (kappa's backfill).
- **L02/L07/L27** — why analytics must not share the serving DB (cache eviction, knee, retry storm).

## What it sets up next
**Lesson 0030 — Secrets, keys & encryption (at rest / in transit).** The warehouse now holds
a copy of every payment (L25) and customer record — one breach exposes 3 years at once. Next:
envelope encryption, key rotation without re-encrypting everything, the KMS as a coordination
point (recap L10), and TLS termination placement. Trade shifts to **security blast-radius vs
operational friction.**

Spine still has #30 (secrets/encryption) and #31 (deployments/rollout) queued — course does
not yet need new topics appended.
