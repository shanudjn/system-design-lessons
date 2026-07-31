# Learning record — Lesson 0044: Time-Series & Metrics Storage

**Date:** 2026-07-31
**File:** `lessons/0044-time-series-storage.html`
**Curriculum spine item:** #44 (first item of the "Time-series & metrics storage" queued batch)
**Worked example:** the observability backend for a 10,000-host platform — 10M active series × a 10-second scrape = 1,000,000 data points/second, stored and kept for years.

## What this lesson covered

Followed the four-move spine (estimate → model → trace → first bottleneck):

- **Estimate** — Naive "one row per reading" with inline labels ≈ 216 MB/s ≈ 18.7 TB/day (the same ~200 B label string written 1M×/s). Two fixes: (1) SPLIT the series (labels) from the samples (L20 plane split) — store each label set once, a sample becomes a bare 16 B (timestamp 8 B + value 8 B) → 1.38 TB/day ≈ 505 TB/yr; (2) COMPRESS the samples for their shape — **delta-of-delta** timestamps (regular 10-s interval → delta-of-delta 0 → ~1 bit) + **XOR** values (slow-moving gauge → mostly-zero XOR → ~1.3 B) crush 16 B → ~1.5 B/point (Gorilla's ~1.37) → 130 GB/day ≈ 47 TB/yr, an ~11× cut. Worked the encoding by hand (t0 full 64 b, then ~1 bit each; v0 full, then 1 bit if unchanged / ~2 B on a small change).
- **Model** — two halves:
  - **Series index** = an inverted index on labels (L12/16): posting list per `label=value`, intersect for a multi-matcher query → the handful of series to read. Small IF cardinality is sane.
  - **Sample store** = per-series samples appended in time order into the current **chunk** (2-h window); once the window passes the chunk is **immutable** → compress hard, tier (L39), roll up, and DROP whole partitions at retention (O(1), L24 partition-and-drop). Chunking by time = L29 partition pruning for range reads.
  - **Rollups** = downsample raw 10s → 1m → 1h, storing min/max/sum/count. A 1-hour rollup replaces 360 raw points; at 4 aggregates it's ~90× smaller per unit time. Tiered (raw 7 d / 1-min 30 d / 1-h 2 y) ≈ 4 TB vs ~95 TB all-raw-2-y ≈ 24× cut. Rollup is lossy + irreversible (the 2-min spike is gone once averaged) — L17's tail-hides-in-the-mean as a storage decision.
- **Trace** — (A) a sample landing: resolve series (create = costly index insert + fresh RAM buffer; existing = ~a couple bytes+bits), append to in-mem chunk + WAL (L38 durability), flush immutable when the window closes; (B) "p99 latency, service=search, last 6 h": label index → ~40 series (not 10M), time-partition prune → ~3 chunks each (L29), decode only those few MB; a 90-day range reads rollups instead.
- **First bottleneck** — **cardinality, not volume.** Point volume is bounded (series ÷ interval) and compresses away; series count is a **product** of label cardinalities. `{service=50, region=5, status=15, endpoint=200}` = 750,000 series (fine); add `user_id` (10M) → 750,000 × 10,000,000 = **7.5 trillion** series → ~750 TB index (at ~100 B/series) + unbounded per-series RAM buffers, while the sample rate still looks fine. Fix = keep identifiers OFF labels (L17 "ids on traces, never metric labels"; exemplars to link a metric to a trace), cap/allowlist labels at ingest, pre-aggregate the dimensions actually queried.

Arithmetic double-checked: 10M/10 = 1M pts/s; 1M×216 B ≈ 216 MB/s ≈ 18.7 TB/day; 1M×16 B = 16 MB/s = 1.38 TB/day = 505 TB/yr; 1M×1.5 B = 1.5 MB/s = 129.6 GB/day ≈ 47 TB/yr; delta-of-delta of a constant interval = 0; 120-point chunk timestamps ≈ 64+14+118 ≈ 196 bits ≈ 25 B ≈ 0.2 B each; 1-h rollup = 3600/10 = 360 points → /4 aggregates = 90×; 50×5×15×200 = 750,000; ×10M = 7.5×10¹²; ×100 B = 750 TB.

## Reuses / threads pulled forward
L12/16 inverted index (label lookup), L17 observability firehose + "ids on traces not metric labels" (the cardinality wall), L20 metadata/data-plane split, L21 lossy summary (rollups), L24 partition-and-drop, L29 columnar encoding + partition pruning, L38 append-only log/WAL, L39 tiers/retention, L40 noisy-neighbor (a bad label exhausts a shared resource).

## What it sets up next
**Lesson 0045 — Webhooks & outbound event delivery** (spine #45): deliver "your order shipped" to thousands of third-party endpoints you don't control — at-least-once + retries/backoff (L07/09), a signature so the receiver can trust it (L30), an idempotency key so the receiver can dedup (L13), the slow/dead endpoint (circuit breaker + DLQ, L07/09), and ordering. Trade: delivery reliability vs the burden pushed onto the receiver.

Spine still has #45 (webhooks/outbound delivery) and #46 (distributed job scheduling) queued — course will not run dry.
