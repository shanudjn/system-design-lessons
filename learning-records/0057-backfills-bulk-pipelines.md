# Learning record — Lesson 0057: Bulk Data Pipelines & Backfills

**Topic:** Reprocessing history at scale — re-running a transform over billions of
rows on infrastructure that is busy serving live traffic. Generalizes L56's
one-user log reprocessing into a routine capability.
**Worked example:** Backfill a new `currency` column across **2 billion historical
`orders` rows** (the L12/24/25 table) on a live DB serving 50k reads/s + 5k
writes/s, with zero downtime — the leftover work from L24's expand/contract split.

## What this lesson covered
- **Estimate — why one big `UPDATE` detonates three ways.**
  - 200 B/row (L12) × 2B rows = **400 GB** to read+rewrite. At a production-safe
    10k rows/s → 2B ÷ 10k = 200,000 s ≈ **55.6 h**; batch 5,000 → **400,000 batches**.
  - (1) One giant transaction holds locks+undo for all 400 GB → fail at 95% =
    all-or-nothing rollback, 50 h lost (twice, counting rollback). (2) Flat-out
    scan saturates the DB serving live traffic → past L27 knee → L07 retry storm.
    (3) A 55 h run WILL be interrupted → no resume = start over.
  - Checkpoint converts "lose 50 h" into "lose ≤1 batch = 5,000 rows ÷ 10k/s = 0.5 s".
- **Model — the four-part kit, each disarming one detonation.**
  1. **Chunk** — 5,000-row independently-committed batches (~1 MB undo each). Page
     by **cursor not offset** (L18): `WHERE id > :last_id ORDER BY id LIMIT 5000` =
     O(limit) B-tree seek (L12), constant at any depth. OFFSET re-reads+discards the
     prefix → O(N²); the last batch alone scans ~2B rows.
  2. **Checkpoint** — save the cursor after each commit; resume from `id > last_id`.
     Commit+checkpoint is itself L33's dual-write → put the cursor update IN THE SAME
     TXN as the batch, or rely on idempotency.
  3. **Idempotent re-runs** (L13) — the committed-but-uncheckpointed batch that WILL
     re-run must be a no-op. `WHERE currency IS NULL` self-guards for free; a transform
     that increments/inserts/appends-to-ledger (L25) needs a UNIQUE idempotency key.
  4. **Throttle** (L08/28) — adaptive on **replica lag** (L06): backfill writes
     replicate; outrun the replica → lag climbs → read-your-writes breaks → back off.
     L27 controller pointed at lag; L28 backpressure with the batch job as the thing
     pushed back.
- **Trace.** (A) happy batch: seek → transform → small txn (+ checkpoint atomically)
  → sleep-to-rate, ×400k. (B) crash at hour 50: read checkpoint, resume; in-flight
  batch either never committed (redone) or committed-pre-checkpoint (re-run skips the
  already-non-NULL rows) — idempotency makes "did it commit?" a non-question. (C)
  spike: replica lag crosses 1 s → throttle cuts rate to ~0 → recovers → ramps. Guest
  yields to host; production never notices.
- **First bottleneck — structural.** Backfill + production fight over ONE machine's
  disk/CPU/replication → speed capped by production's HEADROOM, not worker count.
  Parallelize by key-range (L03): 8 workers = 8× throughput (55 h → 6.9 h) but 8× load
  → spends headroom faster, doesn't escape the wall. Real escape = move the heavy
  history SCAN onto a replica / offline copy (L29, built for full-history scans),
  write only results back throttled. Two more walls: backfill DURATION = the mixed-state
  window (partially-populated column; readers coalesce/fallback until cutover L24/L31);
  moving target (new rows mid-sweep → ascending-PK sweep handles it, else 2nd pass /
  dual-write bridge L24). Deepest: a backfill is a SCHEDULING problem in a data
  problem's clothes.

## Trade named
Throughput vs impact on the live system. The backfill's speed is a loan against
production's spare capacity, repaid the instant production needs it back.

## Reuses
L24 expand/contract backfill-step + cutover + mixed-state, L18 cursor-not-offset,
L12 B-tree seek + row/table sizes, L13 idempotency + L33 atomic-commit/dual-write,
L06 replica lag, L28 backpressure + L27 knee/control-loop, L08 rate limit, L03
shard-by-key-range parallelism, L29 offline/warehouse copy, L07 retry storm, L25
ledger append, L46 durable resumable job.

## Sets up next
**Lesson 0058 — Multi-cloud & vendor portability:** we ran a heavy job on one busy
database; next, what happens when "one database" becomes "two clouds." The two walls
that make this harder than "deploy it twice" — **data gravity** (moving petabytes
between providers is slow, and the **egress bill** L14 is brutal) and
**lowest-common-denominator services vs managed lock-in** — plus where a control
plane can span clouds and where it can't. Trade: portability & resilience vs
complexity & cost.
