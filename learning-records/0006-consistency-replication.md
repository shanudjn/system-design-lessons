# Learning record — Lesson 0006: Consistency & Replication

**Topic (spine #6):** A like-counter under concurrency — the simplest piece of
shared mutable state, used to surface every consistency problem at once.

**Worked example:** A viral short link `xZ9` gets a like button. Peak load
50,000 likes/sec and 500,000 reads/sec on one row.

## What it covered (four moves)
- **Estimate:** two ceilings — a hot row serializes at ~2,000 writes/s (atomic
  lock ~0.5 ms) vs 50,000 needed; one node serves ~10,000 reads/s vs 500,000
  needed. Framed the whole lesson as the tension between "one true copy"
  (correct, throughput-capped) and "many fast copies" (scales reads, copies
  disagree).
- **Model:** "+1" is secretly read-modify-write; the gap between read and write
  is a race condition. Worked the clock: two likes → count 101 not 102 (lost
  update), and noted it's the COMMON case under load, not a rare one.
- **Trace the paths:**
  - Path A — atomic increment (`UPDATE SET count = count + 1`) closes the gap;
    mentioned compare-and-set/optimistic locking as the general form. Trade-off:
    DB owns the arithmetic (less app flexibility) vs safety.
  - Path B — replication (leader + replicas) scales reads but adds lag; worked
    40 ms lag = ~2,000 likes behind, harmless for a badge, fatal for a balance.
    Read-your-writes broken for the author; cheap fix = route own reads to
    leader / optimistic UI.
  - Strong vs eventual table; quorums with N=3, W=2, R=2 → W+R>N forces overlap
    (pigeonhole), counter-example W=1,R=1. Named as CAP in miniature.
- **Next bottleneck:** the still-hot single row → sharded/approximate counter
  (K≈50 sub-rows, each ~1,000 w/s; read = SUM, cached/slightly stale).
  Trade exact read for scalable write; wrong trade for a bank balance.

## Recurring discipline reinforced
Match the consistency to what the number MEANS — acceptable staleness is a
property of the data, not the database. Pay only for the consistency you need.

## Sets up next
- **0007 Designing for failure** — timeouts, retries, idempotency, backpressure
  (retries against the quorum/leader; idempotency promoted to core discipline).
- **CAP in practice** and **leader election** (the single-leader write point and
  split-brain) are both teed up explicitly in the closing card.
