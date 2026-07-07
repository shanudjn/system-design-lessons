# Learning record — Lesson 0020: Object Storage

**Date:** 2026-07-07
**File:** `lessons/0020-object-storage.html`
**Spine topic:** #20 — Object / blob storage (now ✅)

## The one system
`PUT(key, bytes)` / `GET(key)` over **100 PB** of dashcam clips & photos that must
survive dying disks and be fetched by key. Two promises: durability (disks die at
scale) and it must fit on disk we can afford.

## What the lesson covered
- **Estimate — the two obvious designs fail.**
  - Object count: 100 PB / 100 MB avg = **10⁹ = 1 billion objects**. A filesystem
    drowns in a billion-entry tree (path walks, inode limits, per-lookup seeks) →
    want a **flat key→blob namespace** (give up rename/paths/partial writes; gain
    hash-the-key sharding, L03/L04).
  - Naïve durability = 3× replication → **300 PB** raw for 100 PB, survives only **2**
    failures. +200% overhead.
  - Trade: **durability per raw byte** — copies are simple + fast-repair but scale
    linearly ((f+1)× to tolerate f failures).
- **Model — four ideas.**
  - **Split metadata plane from data plane.** Metadata (key→chunks→fragment locs) ≈
    1 KB × 1B = **1 TB**, vs 100 PB data = **100,000×** gap. Pay strong consistency
    (quorums, L06/L11, in-RAM) on the 1 TB; cheap append-only durability on 100 PB.
  - **Chunk** big objects into 64 MB pieces (parallel upload, resume, spread). Trade:
    parallelism/repair granularity vs metadata overhead.
  - **Erasure coding RS(10,4):** split a 64 MB chunk into 10 data + 4 parity = 14
    fragments of 6.4 MB, any 10 rebuild it, scatter over 14 failure domains.
    - Worked table: 3× repl = 300 PB (survive 2); 5× repl = 500 PB (survive 4);
      **RS(10,4) = 140 PB (+40%, survive 4)**. Matching EC's 4-tolerance with copies
      needs 500 PB → **~3.5×** more; saves 300−140 = **160 PB ≈ $3.2M/mo ≈ $38M/yr**.
    - Durability ~11 nines because losing an object needs 5 of 14 gone *before repair*;
      lever is **MTTR vs MTBF** (fast repair shrinks the danger window).
    - Trade: EC = disk saved for **parity CPU + 10× repair amplification** (rebuild 1
      fragment by reading k=10 = 64 MB). Replication still wins hot/latency-critical.
- **Trace — write & read.** Write: chunk → RS code → scatter 14 frags → **commit
  metadata LAST** (the atomic point-of-no-return; pre-commit crash = orphan frags the
  sweeper reclaims — L13's durable-effect-then-record). Read: metadata lookup → fetch
  **any 10** (prefer the 10 data frags, no decode) → decode only if a data frag is
  missing. EC read fans out to 10 disks → tail latency (L17). Trade: single-fetch
  simplicity vs fan-out.
- **Next bottleneck — the small-file problem.** 4 KB objects: 100 PB / 4 KB = **25
  trillion** objects → 1 TB index balloons to **25 PB** (¼ the data), EC fragments
  become 400 B splinters. Costs are **per-object, not per-byte**. Fix: **pack** many
  small objects into 1 GB container blobs (EC the container), keep a compact in-RAM
  `key→(container, offset, len)` index (1 seek/read), deletes = **tombstone +
  compaction** (L15). Trade: per-object cost vs per-byte cost.

## Reuses / callbacks
- L19 dashcam data (what we now have to store) — the direct handoff.
- L03/L04 key-sharding (the flat namespace).
- L06/L11 quorum consistency (the metadata plane).
- L13 durable-commit discipline (write the pointer last).
- L15 tombstones (deletes in a packed/log-structured store).
- L17 fan-out tail (an EC read touches 10 disks).

## What it sets up next
- **#21 Analytics & counting at scale** — HyperLogLog / Count-Min / Bloom: "how many
  *unique* cars uploaded today across 25 trillion objects?", trading exactness for
  fixed memory (the same approximate-to-scale move L06's sharded counter and L19's
  hot cell forced).

Spine had only #21 left, so this run **added 5 new advanced topics** (#22–#26:
distributed locks/leases, multi-region active-active, schema migrations at scale,
payments/ledgers, real-time WebSocket delivery) so the course never runs dry.
