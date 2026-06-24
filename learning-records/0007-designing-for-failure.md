# Learning record — Lesson 0007: Designing for Failure

**Topic (spine #7):** Timeouts, retries, idempotency, and backpressure — how a
system should behave *during* failure, traced through one timed-out call.

**Worked example:** The app server's `recordLike(xZ9)` call to the leader
(normally ~10 ms) times out after 50 ms with no reply. One call, every decision.

## What it covered (four moves)
- **Estimate:** set the timeout from the dependency's real latency
  (p50 10 ms, p99 50 ms, p99.9 200 ms → timeout ~200 ms). Worked the trap of a
  1 s default: 200-thread pool does 20,000 calls/s healthy (100/thread) but
  collapses to 200 calls/s when every call hangs 1 s (1/thread) — a 100× drop,
  taking down the whole app. Trade-off: false failures (too short) vs thread
  exhaustion (too long).
- **Model:** a timeout is a **partial failure** — three truths (never arrived /
  done-but-ACK-lost / still running) the caller can't distinguish. Blind retry
  on the "done" case duplicates: worked the clock, count 100→101→(retry)→102 for
  one click. Retries are only safe on idempotent ops.
- **Trace the paths:**
  - Path A — **idempotency key**: client-generated UUID reused across retries;
    server applies a key once, no-ops repeats (same trick as Lesson 05's
    at-least-once queue dedup). Cost: storing applied keys (TTL, hot-path lookup);
    key must identify the logical action, not the attempt.
  - Path B — **retry storm / metastable failure**: 80% leader + "retry 3×" →
    offered load → ~4× (80k vs 25k ceiling) → self-sustaining death spiral.
    Fixes: exponential backoff, jitter (desync the herd, à la Lesson 02
    stampede), retry budget (cap ~10% → amplification ~1.1×).
  - Path C — **circuit breaker**: CLOSED → OPEN (failures >50% of last 20) →
    HALF-OPEN trial → CLOSED. Fail fast (~0 ms) vs hang 200 ms × everything.
    Trade-off: availability for stability.
- **Next bottleneck:** work arriving faster than the leader drains →
  **backpressure**. Unbounded queue hides the collapse (latency → minutes,
  serving the dead); bounded queue (cap 10k / 20k drain = 0.5 s worst wait) +
  **load shedding** (honest 429). Trade completeness for survival.

## Recurring discipline reinforced
You can't prevent failure — design what happens during it. Never wait forever,
never retry blindly, always prefer a fast honest "no" to a slow collapse. Every
control names its trade-off.

## Sets up next
- **0008 Rate limiting** — token bucket vs leaky bucket, distributed counters
  (who/how-much, vs backpressure's when).
- **Message queues deeper** (exactly-once, DLQs) now that idempotency is core;
  **CAP in practice** and **leader election** (the single-leader write point,
  split-brain) still teed up.
