# Learning record — Lesson 0054: Billing & Metering Systems

**Topic:** Turning a usage firehose into an invoice that is correct to the cent.
**Worked example:** A cloud platform metering ~50B billable events/day (API calls,
storage GB-hours, egress GB) across ~1M accounts, a ~$50M/month book.

## What this lesson covered
- **Estimate.** Firehose: 50B events/day = ~579k/s, ~200 B each → 10 TB/day raw,
  which must be *retained* (evidence for disputes) → 3.65 PB/yr audit. The error
  argument: a quiet 0.5% metering error on a $50M/mo book = $250k/mo = **$3M/yr**
  mischarged — invisible on a dashboard, seven figures on an invoice. Metering
  error is **asymmetric**: undercount = silent revenue leakage; overcount =
  overbilling → disputes, refunds, churn, regulatory exposure. This is why the
  cheap approximate counters of L21 are disqualified for the *billed* number.
- **Model — four stages, four guards:**
  1. **Ingest** — an at-least-once firehose (L09) delivers duplicates; emitter
     stamps a unique `event_id` (L13) and ingest does an atomic insert-if-absent
     (L25 `UNIQUE`, L06 CAS) → counted-once. Durable-append-before-ACK guards drops.
  2. **Aggregate** — EXACT usage counters per (account, meter, hour), never a
     sketch. A sketch may run *alongside* only for the live "usage so far"
     dashboard (exact-slow bill vs approx-fast view, the L29/L53 two-shapes split).
  3. **Rate** — tiered + **point-in-time** pricing on event-time buckets. Worked
     calc: 12.5M calls → free(1M) + 9M×$0.50 + 2.5M×$0.40 = $0 + $4.50 + $1.00 =
     **$5.50**. A mid-cycle price change forces rating each unit at the price in
     effect *then* (L53 PIT, L35 event-time).
  4. **Reconcile** — every derived number is a sum of the durable raw log, so
     recompute independently and check against the invoice and the L25 ledger
     (billed-but-uncollected / collected-but-unbilled = drift). HOLD a wrong
     invoice before it ships; don't send-then-refund.
- **Trace.** (A) one event → line item; (B) a retried duplicate the `event_id`
  claim drops before double-charging (exactly-once *effect* on at-least-once
  *delivery*); (C) month-end close held behind a **watermark** + grace window
  (L29) so a straggler ingested next month bills to THIS cycle by event-time
  (L35); later stragglers → compensating line on the next invoice (L25/L32).
- **First bottleneck — the dedup memory.** "Never counted twice" = remember every
  `event_id` for the whole retry horizon = 5.6 TB (7-day window × 800 GB/day of
  16 B keys). The L13 **TTL trap** in dollars: forget too soon → a late duplicate
  double-charges; keep forever → unbounded growth → bound by the max retry
  horizon. You may NOT shrink it with a sketch (approximate dedup = approximate
  bill). A **Bloom filter** over the FULL window cuts *lookups* the safe way:
  "definitely not seen" (no false negatives) → accept; "maybe" → exact check; its
  only error (a false positive) costs one extra lookup, never a wrong bill (L21
  Bloom-as-guard) — but aging a key out early makes "not seen" a lie → double-charge.
- **Secondary walls.** Hot-account counter → EXACT sharded counter (L03/06/21/25,
  merged total still exact). A cycle is closed-with-correction, never truly sealed.

## Trade named
Billing accuracy vs metering cost & latency. Exactness is bought with storage
(keep every event), dedup memory (remember every id), and latency (close after
stragglers). Approximation is a *feature* for a dashboard and a *defect* for a bill.

## Reuses
L13 idempotency + TTL trap, L21 exact-vs-approximate + Bloom-as-guard, L25
double-entry ledger + compensating entries, L09 at-least-once delivery, L29
watermark + batch-exact/stream-fast, L35 event-time vs processing-time, L53
point-in-time correctness, L03/04/06/48 hot-key/sharding.

## Sets up next
**Lesson 0055 — Edge computing & compute-at-the-edge:** push not just cached bytes
(L14) but *code* to the edge PoPs (L50). Cold-start vs locality, state at the edge
(L11/23 eventual consistency), what can/can't run far from its data. Trade:
latency & offload vs consistency & operational reach.
