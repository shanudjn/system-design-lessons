# Learning record — Lesson 0005: Async Work (queues + workers)

## What this lesson covered
Took Lesson 03/04's villain — the enterprise bulk API minting 2,000,000 links in
one call — and moved it off the request path onto a queue drained by workers.

- **Estimate** — 5 ms/row real work.
  - Inline: 1 worker = 200 rows/s → 2M ÷ 200 = 10,000 s ≈ **2.78 h**, well past
    a ~60 s LB/proxy timeout → structurally impossible (client retries stack).
  - 20 workers = 4,000 rows/s → 2M ÷ 4,000 = **500 s ≈ 8 min 20 s**; API returns
    `202 + job_id` in ~5 ms. Queue holds 2M × 200 B = **400 MB** at peak.
- **Model** — producer (API) → durable queue → worker pool. API writes a job row
  (status/done/total) and enqueues; client polls `GET /jobs/{id}`. Safety
  mechanism = **visibility timeout**: on receive the msg is hidden (e.g. 30 s),
  deleted only on ack; a crashed worker that never acks → msg reappears for retry.
- **Trace** — three paths:
  - (A) **At-least-once** → duplicates (worker creates link then crashes pre-ack →
    redelivered → created twice). Fix = **idempotency key** `(customer_id, row#)`
    + `INSERT ... ON CONFLICT DO NOTHING`, NOT "switch to exactly-once" (expensive
    distributed txn; most "exactly-once" is at-least-once + dedup anyway).
  - (B) **Poison message** (always-failing malformed row) loops forever → fix with
    **retry limit (~5) + DLQ**. Worked: 50/2M poison → 250 wasted attempts,
    1,999,950 succeed, 50 parked in DLQ. Exponential backoff between tries.
  - (C) **Fan-out** notifications need ordering. Global order kills parallelism →
    **per-key ordering via partition key = customer_id**: one customer ordered,
    different customers parallel; cross-customer order not needed.
- **Next bottleneck** — **consumer lag / backpressure**. Worked: A enqueues 2M @
  t0 (drains by 500 s); B adds 1M @ t100 → backlog 2.6M → finishes t750 s.
  Scaling signal = queue depth ÷ drain_rate (age of oldest msg) → **autoscale
  workers** (40 → 8,000/s) or **shed load** at producer. A queue smooths bursts,
  it cannot invent throughput.

## Trade-offs named
- decoupling + elastic throughput vs added latency (eventual result, polling) +
  a new failure surface (the broker, stuck messages)
- at-least-once + idempotency (cheap, robust) vs true exactly-once (expensive)
- retry limit: too low → transient blips lost to DLQ; too high → poison wastes
  capacity longer (~3–5 + backoff is the middle)
- ordering vs throughput: total order = 1 consumer (no parallelism); per-key
  order = ordered where it matters, parallel where it doesn't

## What it sets up next
- **Lesson 0006 — Consistency & replication:** workers concurrently bump a shared
  `done` counter / like-counter → lost updates under concurrent writes.
- **Lesson 0007 — Designing for failure:** timeouts, retries+backoff, idempotency
  promoted to core discipline.
- **Lesson 0008 — Rate limiting:** the load-shedding hand-waved here (token vs
  leaky bucket) to protect the queue from permanent overload.

## Curriculum bookkeeping
- Marked spine #5 ✅ (Lesson 0005). Spine still has #6–#13 queued (consistency,
  failure, rate limiting, message queues deep-dive, leader election, CAP,
  indexing/search, idempotency) — plenty of runway, no new topics needed yet.
