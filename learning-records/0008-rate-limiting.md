# Learning record — Lesson 0008: Rate Limiting

**Topic (spine #8):** Rate limiting — token bucket vs leaky bucket, distributed
counters. Who may do how much, enforced at the front door before any real work.

**Worked example:** The URL-shortener free tier (60 requests/min per API key).
One runaway script key (`k_42`) firing flat-out, gated before it touches db/queue.

## What it covered (four moves)
- **Estimate:** a limit is TWO numbers — sustained rate (1 req/s) + burst (~60).
  Sized against the downstream ceiling (20,000 req/s from L07); ~100k keys × 1/s
  = 100k worst case = 5× over → per-key limit needs a global cap too. Trade-off:
  false throttling (too low) vs resource starvation (too high/loose).
- **Model:** fixed-window counter is cheap (one int + reset) but has the
  **boundary burst** bug — 60 at 12:00:59 + 60 at 12:01:00 = 120 in ~2s, promised
  60 (2×). Fixes: sliding-window log (exact, costly), sliding-window counter
  (cheap, good default), token bucket (burst by design).
- **Trace the paths:**
  - Path A — **token bucket**: capacity B = biggest burst, refill r = sustained
    average. B=60, r=1/s. Worked the runaway: t=0 burst 60 (bucket→0), t=0.1
    req 61 throttled, t=10 → 10 tokens back. Lazy O(1) impl: store
    (tokens, last_refill_ts), add (now−ts)×r capped at B on each check.
  - Path B — **leaky bucket**: fixed-rate drain, smooth output, drops overflow.
    Trade-off: token bucket = smoothness for burst tolerance (bursty APIs);
    leaky bucket = burst tolerance for smoothness (fragile downstream), adds
    burst latency.
- **Next bottleneck:** ONE limiter across M=10 servers. Local per-server buckets
  multiply the limit ×M (60→600, worse as autoscaling grows M). Fixes:
  (1) divide budget (uneven traffic wastes share); (2) **centralized atomic
  counter** (Redis, Lua/INCR) — accurate but beware the **lost-update** race
  (L06) and it's now on every request's hot path → ~100k ops/s ceiling vs 200k
  fleet = 2× over. Escapes: shard by key (consistent hashing, L04) — celebrity
  key still pins one shard; or **approximate** local buckets reconciled each sec
  (bounded overshoot for scale = the exact-vs-scalable bargain of L06's sharded
  counter).

## Recurring discipline reinforced
Counting a limit correctly at scale is itself a small distributed-systems
problem. Every control names its trade-off; distributed limiting's master dial
is **accuracy vs scalability** — pick the overshoot you can tolerate.

## Sets up next
- **0009 Message queues deeper** — at-least-once vs exactly-once, ordering,
  dead-letter queues (idempotency + backpressure now core from L05/L07).
- **Leader election & CAP in practice** — the single leader / single Redis we
  keep leaning on: split-brain, quorums, what a partition really forces.
