# Learning record — Lesson 0003: Scaling Writes

## What this lesson covered
Pushed the URL shortener's *write* path (ignored in 01–02 because it was read-bound)
to a 40,000 writes/sec enterprise-bulk-API peak, and applied the four moves.

- **Estimate** — found TWO separate walls, deliberately kept distinct:
  - *Throughput wall:* the single `INCR counter` row serializes every write,
    capping ~10,000/sec → 4× over at peak.
  - *Storage wall:* 6B rows × ~200 B ≈ 1.2 TB now, ~12 TB after 10× growth →
    past one disk.
- **Model (IDs)** — distributed ID generation:
  - ID blocks / ticket server (the pick): 1 trip per 1,000 IDs → counter sees
    40 req/sec not 40,000 (1,000× cut), codes stay short (~7 chars). Cost: lost
    IDs in a block if a server crashes.
  - Snowflake 64-bit bit-budget walk-through: 41-bit ms (~69.7 yr) + 10-bit
    machine (1,024) + 12-bit seq (4,096/ms → 4.1M/s per machine). Zero
    coordination but ~11-char codes → bad for a *short*ener.
  - UUIDv4: zero coordination, but random inserts wreck index locality.
- **Model (storage)** — sharding: hash vs range partitioning. Chose
  `hash(short_code) % 16` (even spread, no newest-shard hotspot) over range
  (cheap scans but write hotspot). N=16 → 2,500 writes/sec & ~75 GB per shard.
  Trade: hash kills range scans (scatter-gather), fine since we look up one code.
- **Trace** — write picks ID from a local block, then recomputes shard from the
  code; read recomputes the same shard. Routing is deterministic; shard is never
  stored on the row.
- **Next bottleneck** — `% N` resharding: going 16→17 keeps only ~1/17 of keys,
  so ~94% migrate. Cause named: the router is bound to N.

## Trade-offs named
- coordination vs code length (blocks vs Snowflake)
- even load vs cheap range scans (hash vs range partitioning)
- wasted IDs vs 1,000× less coordination (block size)

## What it sets up next
- **Lesson 0004 — Consistent hashing:** break the router's tie to N so adding a
  node moves ~1/N of keys via a hash ring + virtual nodes. Explicitly teased.
- **Lesson 0005 — Async work:** bulk link creation shouldn't block the API →
  queues + workers.

## Curriculum bookkeeping
- Marked spine #3 ✅ (Lesson 0003).
- Reordered spine: promoted **consistent hashing to #4** (direct sequel teased
  in this lesson), pushed async/consistency/failure down. Added **#13 Idempotency**
  to keep the queue full.
