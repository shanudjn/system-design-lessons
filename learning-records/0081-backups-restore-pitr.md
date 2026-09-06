# Lesson 0081 — Backups, Restore & Point-in-Time Recovery: Rewinding the Orders DB to One Second Before the DROP

**File:** `lessons/0081-backups-restore-pitr.html`
**Curriculum spine:** topic #81 (advanced batch) — now ✅
**Date:** 2026-09-06

## The one worked system
The orders DB from L79: **10 TB, 10B rows (~1 KB each), 16 shards, 40k writes/s**.
The system of record (ledger L25, source every downstream copies from L33). The
disaster: a 2 a.m. `DROP TABLE orders` on prod, and separately a bad deploy that
quietly writes garbage for six hours. Adversary = the failures durability was
assumed away for eighty lessons (fat-fingered command, corrupting bug, dead disk,
region loss, ransomware).

## What it covered (the four moves)
- **Estimate** — the two dials, priced. **RPO** (max data loss, backward, set by
  capture frequency): daily-full-only worst case = up to 24 h × 40k/s =
  **3,456,000,000 ≈ 3.5B writes lost**. **RTO** (max downtime, forward, set by
  restore speed): 10 TB @ 500 MB/s = 20,000 s ≈ **5.6 h** on one stream →
  parallel per-shard (625 GB/shard ÷ 500 MB/s = 1,250 s ≈ **21 min**) for the base
  copy. The dials pull on different machinery → size independently.
- **Model** — three blocks: full (self-contained, slow), incremental (cheap/fast,
  needs the chain), and **WAL archiving** (log shipping the write-ahead log L33/35
  off-box → a continuous **replay stream**). **PITR** = restore most recent base +
  replay WAL forward, STOP at `recovery_target` (02:00:02.999, before the DROP).
  WAL frequency = RPO dial (stream → ~0); incremental frequency = RTO dial (fresh
  base shrinks the replay window). Replay is slower than a sequential copy (re-runs
  txns + maintains indexes L12).
- **Trace** — (A) clean `DROP` → PITR: base ~25 min + replay, stop before the DROP,
  RPO ~1 s. (B) corrupting deploy → **replication is NOT a backup**: replicas +
  warm standby copy every committed write in ms, so all copies are equally corrupt
  (replication protects machines, not correctness); only a point-in-time copy goes
  back before 08:00 → restore-to-the-side then reconcile good writes (L25/32). (C)
  sharded restore → parallel = fast, but each shard's WAL ends at a different point;
  restoring by each shard's WALL CLOCK is a trap (skew L35) → restore all shards to
  a GLOBAL marker/LSN so no cross-shard txn (L32) is half-done (L11/23/35).
- **First bottleneck** — **an untested backup is not a backup** (Schrödinger's):
  corrupt files, WAL-chain gap, missing shard, stale runbook, or a restore that
  blows the RTO budget — all hidden until the disaster. Only a scheduled **restore
  drill** (game day L51) that measures actual RPO/RTO counts. Walls: 3-2-1 +
  immutable/WORM offsite (L30/75, survives ransomware & rogue admin), tiered
  retention (L20/39, ~1 TB/day WAL), warm standby (L10) for seconds-RTO (the other
  end of the RTO dial; still not a backup).

## Key idea to carry
RPO and RTO are **two independent dials**. Every backup decision buys down one or
the other, never for free: capture more often (WAL stream, frequent incrementals)
→ lower RPO; restore faster (parallel shards, fresh base, warm standby) → lower
RTO. Neither substitutes for the other (a 5 s RPO can still be a 9 h RTO; a 30 s
standby can have copied the corruption). And the whole thing is fiction until a
restore drill proves it.

## Reuses
L33/35 (WAL/binlog as durable ordered log → archived & replayed), L79 (16 shards →
parallel restore + cross-shard consistency), L35 (clock skew → logical marker not
wall clock), L06 (replication protects machines not correctness), L25/32
(compensation/reconciliation for a partial rewind), L20/39 (object storage &
tiering price retention), L30/75 (immutable/WORM copies), L10 (promote standby),
L51 (chaos/game days = the restore drill).

## Trade named
Recovery guarantees vs storage & operational cost.

## Sets up next
**Lesson 82 — Data modeling: normalization vs denormalization:** we've protected,
sharded (L79), and serialized (L80) the orders data but never questioned the shape
of the table itself. 3NF for write integrity vs denormalized/precomputed for read
speed (L15/29), the read:write ratio as the deciding bet, embedding vs referencing
in document stores, and the update-anomaly tax. Trade: write correctness & storage
vs read latency.

## Note on the spine
Spine still has topics #82–85 queued (normalization vs denormalization, full-text/
relevance pipelines, graph/recommendation traversal, multi-region data placement).
No new topics needed this run.
