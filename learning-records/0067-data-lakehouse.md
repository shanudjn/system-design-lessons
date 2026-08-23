# Learning record — Lesson 0067: Data Lakes & the Lakehouse (Open Table Formats)

**Date:** 2026-08-23
**File:** `lessons/0067-data-lakehouse.html`
**Spine topic:** #67 — Data lakes & the lakehouse (open table formats)
**Trade named:** openness & cheap shared storage vs query performance & metadata-management complexity

## What this lesson covered
Takes L29's warehouse dataset (11 TB, 11B rows, 3 years, partitioned by day) and stores it as **raw Parquet files on
object storage** (L20) — cheap and open to many engines (Spark/Trino/Flink/the warehouse), owned by no vendor (L58's
anti-lock-in). The catastrophe that comes with it: **a directory of files is NOT a table** — no transactions, no atomic
multi-file swap, and finding "the table right now" means LISTing a bucket of millions of files. The lesson builds the
missing **open table format** (Iceberg/Delta/Hudi) → a **lakehouse** (warehouse guarantees on lake storage).

The one idea the whole lesson rests on: **reduce an atomic change over MANY immutable files to an atomic swap of ONE
tiny pointer** (the catalog pointer to the current metadata). L14's rename-don't-mutate / L20's commit-metadata-last /
L33's outbox, scaled up to an entire table. Every feature (snapshots, time travel, safe compaction, schema evolution,
optimistic concurrency) is a **corollary** of that single reduction.

Four-move spine:
- **Estimate** — two killer numbers for "directory = table." (1) Planning cost: ~1.58M files (1,095 days × 1,440
  files/day, one small ~7 MB file/minute streaming) → LIST is 1,000 keys/page = ~1,580 paginated calls, then open every
  footer for min/max stats: **1.58M × ~1 ms ≈ 1,580 s ≈ 26 minutes** just to find files, before L29's 0.5 s scan begins.
  (2) No atomic multi-file change: compacting a day's 1,440 files → 78 (write 78, delete 1,440 = two non-atomic steps) →
  a query in the gap sees rows missing/doubled → **silently wrong revenue**, not an error (L32/33 boundary hazard).
  Arithmetic: 11 TB ÷ 1,095 = ~10 GB/day; 10 GB ÷ 1,440 min = ~7 MB/file; 10 GB ÷ 128 MB = ~78 compacted files;
  1,095 × 78 ≈ 86,000 files total (~18× fewer).
- **Model** — keep data as immutable Parquet; layer a **metadata tree**: catalog pointer (the ONE atomic pointer) →
  **snapshot** metadata (schema by **column-ID** not position, L24; list of manifests) → **manifest** (per-file row
  count, partition, per-column min/max stats) → Parquet data files. Planning becomes a sub-second in-memory manifest
  scan: partition-prune (1.58M → ~1,440 for one day, L29) then file-skip by stats (~1,440 → ~60, L12's index idea on
  files) — NO LIST, NO footer stampede (stats precomputed). **Commit** = write new files + new metadata, then a single
  **compare-and-swap** on the catalog pointer (L06). Steps 1-2 change nothing visible; only the CAS publishes → readers
  see the whole old or whole new snapshot, never a mix = ACID atomicity + snapshot isolation from one CAS. Falls out
  free: time travel + instant rollback (old snapshots kept), safe compaction, schema evolution (metadata-only over
  11 TB), optimistic concurrent writers (loser retries), crash safety (orphan files unreferenced → GC, L20). Trade: a
  closed warehouse co-designs storage+compute for peak speed; the lakehouse gives that up for openness + cheap object
  storage + storage/compute separation, paid in metadata-management complexity.
- **Trace** — (A) READ: catalog → metadata_042 (pin it) → manifest filter (partition + region stats) → read only
  region+amount cols of ~60 files; a compaction committing 043 mid-query doesn't disturb the pinned 042 (snapshot
  isolation). Planning sub-second, not 26 min. (B) WRITE/compaction: write 78 files → metadata_043 → CAS(042→043)
  atomic; concurrent appender racing = first CAS wins, loser re-reads 043 & retries on top → 044 (optimistic, no lock,
  L06); crash after writing files = orphans no snapshot points to → GC (L20). (C) TIME TRAVEL: bad ETL commits 045 →
  rollback = CAS pointer back to 044 (instant, no data moved, L31 blue-green flip); audit = query "AS OF 044 vs 045"
  (L38); **L56 caveat** — time travel KEEPS deleted data until snapshots are expired + files GC'd (right-to-be-forgotten
  forces expiry, the same op that reclaims storage).
- **First bottleneck** — the hard part didn't vanish, it **moved into the metadata**. (1) Manifests bloat: 1.58M entries
  = hundreds of MB to read for planning; thousands of snapshots pile up → compact data AND metadata, expire old
  snapshots, cache manifests (L20/29 small-file & over-partition traps one level up). (2) The commit is **one
  serialization point per table** — all writers CAS the same pointer (L66's "one logical counter everyone contends on")
  → high commit rate = CAS-retry storm (L07) → batch into fewer, larger commits (L09/47, which also makes bigger files +
  less metadata); single-writer ceiling per table (L03). (3) Commit interval = L29's freshness-vs-cost dial reborn
  (often = fresh but small files + contention; rarely = stale but cheap), now SAFE to tune because every change is
  atomic; true sub-second freshness still needs a paired speed layer (L29 lambda/kappa, L64).

Deepest point: the lakehouse is "rename, don't mutate" (L14) scaled to a whole table — atomicity over N files reduced
to atomicity over 1 pointer, one indirection to an immutable precomputed description of which immutable files are the
table now.

## Reuses (how it threads the course)
L20 (object storage, immutable blobs, commit-metadata-last, small-file problem, GC of unreferenced), L29 (columnar
Parquet, partition pruning, star schema, freshness-vs-cost batch dial, over-partitioning), L12 (per-file stats as an
index / file skipping), L06 (compare-and-swap, optimistic concurrency), L14 (rename-don't-mutate / versioned immutable),
L24 (schema evolution by column ID), L31 (rollback as a pointer flip), L32 (isolation across a boundary), L33 (outbox /
dual-write hazard), L38 (snapshot log as event history), L56 (time travel vs right-to-be-forgotten), L58 (open =
anti-lock-in), L64 (pair with a stream speed layer), L66 (single serialization point), L07/09/47 (retry storm & batching
the expensive op).

## Interactive quiz
4 questions: (1) why planning on a raw directory is a 26-min disaster (LIST + 1.58M footers; no centralized precomputed
stats), rejecting "Parquet can't store stats" and "object storage too slow to read 11 TB"; (2) the non-atomic
compaction race → silently wrong answer, fixed by immutable files + one CAS, rejecting "compaction just slow, use a
faster box" and "schema mismatch"; (3) two concurrent writers = first CAS wins, loser re-reads & retries (optimistic
concurrency), rejecting "both succeed independently, no conflict" and "second writer blocks on a lock"; (4) the moved
bottleneck = metadata layer + single commit pointer, master dial = commit interval (compaction/expiry keep it healthy),
rejecting "durability, add replicas" and "no bottleneck, scales without tuning."

## What it sets up next
Next in the queued spine: #68 **delivery guarantees & the dead-letter lifecycle** — threading L09 (at-least-once), L13
(idempotent effects), L33 (outbox) across a WHOLE pipeline: where dedup lives, poison-message quarantine, DLQ
triage/replay, proving a message was handled. Then #69 hot/warm/cold path & serving-vs-analytics split (L37 CQRS router
picking a path per query), #70 idempotent resumable data APIs (L05/13/18/20 uploads & long jobs over flaky networks).
The spine still has 3 queued topics (68-70) after this lesson, so this run did NOT need to add new ones — the course
won't run dry.
