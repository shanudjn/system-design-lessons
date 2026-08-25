# Lesson 0069 — Hot / Warm / Cold Path & the Serving-vs-Analytics Split

**File:** `lessons/0069-hot-warm-cold-path.html`
**Spine topic:** #69 (the "pipeline reliability → read fan-out" arc, after L68)
**Date:** 2026-08-25

## Worked example
The one reliable `order.placed` event from L68 (2,000/s, ~1 KB) read by **three questions that demand three different machines**:
- "Where's my order O-77?" — point lookup, ~2 ms, ~1 s fresh.
- "Revenue in the last minute, per product?" — live windowed aggregate, ~10 ms, ~2 s fresh.
- "Revenue by region by day, last quarter?" — billions-row historical scan, seconds, ~15 min stale OK.

## What it covered
- **Why one store can't serve all three.** Opposite shapes AND opposite failure modes (L29): a point lookup on a scan engine is slow (L12 selectivity trap reversed); a big historical scan on the serving store evicts its hot cache + trips L27's knee → "where's my order?" times out → L07 retry storm. The big query destroys the small one when they share machinery.
- **Estimate / sizing.** HOT serving store (30-day retention) = 172.8M/day × 30 = 5.18B rows ≈ **5.2 TB**, KV/doc, point-indexed (L48). WARM speed layer state scales with keys×windows not the firehose ≈ **~50 MB** (L64). COLD warehouse = 3 yrs ≈ 189B rows ≈ **189 TB raw → ~52 TB** columnar (L29/67). Sizes span a **millionfold** range — each keeps only what its question needs (L39 temperature).
- **Model = one write, three read models.** CQRS (L37) taken to its conclusion: fan the durable log (L09/33 outbox) to three idempotent consumers (L13/68), so replay rebuilds any view and a 4th view is free to add (L57 backfill). A **read-path router** (L37/49) classifies each query and matches its freshness *need* to a path's *offer*.
- **"Consistent enough."** The three views are eventually consistent with each other (L06/11), each with its own consumer lag (L09). Define freshness per question: read-your-writes for "my order" (L15), hours-stale fine for "last quarter." Authority is per-question: speed layer for "right now," warehouse for a stable/audited report; the log is the ultimate source of truth.
- **Trace.** One order appears at t+1 s (hot), t+2 s (warm), t+15 min (cold, next batch), then reconciles — batch layer supersedes the fast-approximate speed layer.
- **First bottleneck = the duplicated-logic tax.** Lambda runs the same "revenue" definition in the stream job *and* the batch job → silent drift → dashboard and report disagree. Fix: kappa (one streaming pipeline, replay the log to correct — pays in replay cost) or a shared aggregation library. Plus: N pipelines each with its own lag/backfill/failure, and a router whose mis-classification is an outage.
- **Deepest point.** Freshness / query-latency / cost-per-byte is a **triangle** no single store wins → write the event in three shapes and route each question to its corner. This is CQRS + tiered storage + lambda/kappa seen as one pattern.

## Numbers used (all checked)
- 2,000/s × 86,400 = 172.8M orders/day.
- Hot: 172.8M × 30 = 5.184B rows ≈ 5.2 TB.
- Cold: 2,000 × 86,400 × 365 × 3 = 189.2B rows ≈ 189 TB raw; ÷3.6 ≈ 52 TB compressed.
- Quarter scan: 172.8M × 90 = 15.55B rows.
- Fan-out derived writes: 2,000/s × 3 sinks = 6,000 writes/s.

## Threads reused
L68 (the reliable event now read three ways), L37 (CQRS one-write-many-reads), L29 (columnar warehouse, serving-vs-analytics split, lambda vs kappa), L64 (speed layer + windowing), L09 (durable log, consumer lag, replay), L13/33 (idempotent consumers → rebuildable views), L39 (data temperature vs storage cost), L48 (fast serving store), L15 (read-your-writes / precompute), L06/11 (eventual consistency), L27 (latency knee), L07 (retry storm), L12 (selectivity trap), L57 (backfill a new view).

## Sets up next
- **Lesson 70 — Idempotent, resumable data APIs (uploads & long jobs):** resumable multipart uploads (L20/25 chunk+checksum+commit-last), long-job status polling (L05), safely retryable mutations (L13/18) over flaky mobile networks. Trade: robustness on bad networks vs protocol & server-state complexity.
- Then Lesson 71 — quorum reads/writes & tunable consistency (the R+W>N dial from the inside).
- Spine still has 70–75 queued, so no new topics needed this run.
