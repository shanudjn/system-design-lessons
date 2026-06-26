# Lesson 0009 — Message Queues: The Log, Exactly-Once & Ordering

**Published:** 2026-06-26
**File:** `lessons/0009-message-queues.html`
**Spine topic:** Advanced #9 (Message queues), the deeper dive teed up by Lesson 0008.

## What it covered
Took ONE concrete stream — every redirect on the URL shortener emits a 200-byte
click event, 100,000/sec at peak — and traced it end to end through a **log-based**
message queue (Kafka-shaped), reusing the four-move spine:

- **Estimate** — 20 MB/s ingest, 8.64B events/day, ~12.1 TB over a 7-day retention
  window. One consumer ~10k events/s ⇒ need ≥10 partitions; chose 20 (5k/s each).
  Trade-off named: **partitions = parallelism vs order** (order only within a partition).
- **Model** — log vs traditional mailbox queue: events are *kept* for the retention
  window, each **consumer group** tracks its own **committed offset** per partition,
  enabling replay and multiple independent readers. The key idea: the **danger gap**
  between doing the work and committing the offset.
- **Trace paths** —
  - **Delivery semantics:** commit-before-work = at-most-once (loses on crash);
    commit-after-work = at-least-once (duplicates on crash). **No exactly-once on the
    wire** (dropped-ack / two-generals); "exactly-once" = at-least-once + idempotency
    key (dedup by event_id) OR transactional write of output+offset = **effectively-once**.
  - **Ordering:** partition-by-key (`hash(code) % N`) gives per-key order; cost is a
    **hot partition** (celebrity code = 30k/s on one lane that caps at 10k/s). Commutative
    work (click counts) can partition randomly and spread freely.
- **Next bottleneck** — read side: **consumer lag** (worked: 150k producer vs 100k
  consumer = 50k/s lag, 3M backlog/min, 150s to drain; the number to alarm on),
  **poison message** blocking a partition → bounded retries + **DLQ**, **rebalance**
  storm → coordination.

## Reused / reinforced
- Idempotency keys (Lessons 05 & 07) applied to the consumer side.
- Hot-key problem (Lessons 03 & 06) reappears as the hot partition.
- DLQ + at-least-once first introduced in Lesson 05, now deepened.

## What it sets up next
- **Lesson 10 — Leader election & coordination:** the rebalance generalized —
  heartbeats, quorums, electing ONE leader, split-brain.
- **CAP in practice:** what a partition between consumers and broker actually forces
  you to give up.
