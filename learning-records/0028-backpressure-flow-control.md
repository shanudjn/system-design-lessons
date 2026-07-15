# Learning record — Lesson 0028: Backpressure & Flow Control

**File:** `lessons/0028-backpressure-flow-control.html`
**Topic (curriculum spine #28):** Backpressure & flow control end-to-end
**Trade named:** throughput vs tail latency & stability

## The one system
The request path from recent lessons: edge (LB/gateway, L26) → API tier (stateless
workers, L27) → slow downstream (ledger DB writer, L25). During L27's 2× spike the edge
is offered **80,000 req/s** but the downstream sustains only **40,000 req/s** — a fast
producer, a slow consumer, for the ~3-minute window before new servers boot (L27). What
gives?

## What the lesson covers (four moves)
- **Estimate — "do nothing" detonates twice.** Unbounded queue grows at 40k/s → at
  ~20 KB in-flight each, ~144 GB in 180 s → OOM crash (a crash sheds 100% of load). And
  **goodput** collapses: once wait crosses the 1 s client timeout (~1 s into overload,
  when queue hits 40k), every response is stale-on-arrival → throughput stays 40k/s but
  goodput → ~0. Doing nothing is worse than doing less.
- **Model — bound the queue, then choose the "no".** Size the bound from the timeout:
  `depth < timeout × rate = 40,000` → pick 20,000 → max wait 0.5 s, memory fixed at
  400 MB, goodput back to 40k/s. A full bounded queue either **blocks** the producer
  (credit-based flow control — TCP receive window / HTTP-2 window / Reactive Streams
  `request(n)`; the "no" as an advertised number) when you control the producer, or
  **drops** (honest 429, L07) when you don't. Rule: backpressure propagates inward as
  blocking, terminates at the edge as shedding.
- **Trace — three responses, measured by wasted work.** (A) unbounded → crash +
  retry-storm metastable (L07); (B) bounded but shed at the slow end → survives, goodput
  40k/s, but the 40k/s rejected already burned edge+API-tier CPU (~200 stolen cores at
  5 ms each); (C) bounded + **edge admission control** → downstream fullness travels
  upstream as a credit/health signal, edge 429s the excess for ~free, goodput preserved,
  graph runs at L27's ~70% knee. Best place to say no = the first place that can.
- **First bottleneck — where the "no" goes, three traps.** (1) an unbounded queue hiding
  at ANY hop reopens the OOM and swallows the backpressure signal → bound every hop;
  (2) blocking a shared user-facing thread pool re-creates L07's exhaustion → bulkhead +
  non-blocking edge; (3) blind shedding wastes goodput → drop retries before first-tries,
  low tier before paid, stale head via FIFO→LIFO, never health checks.

## Threads reused
- **L13 Little's Law** — sizing the bounded queue (L = λW; wait = depth ÷ rate).
- **L07** — timeout, honest 429, retry storm, metastable collapse, thread-pool exhaustion.
- **L05 / L09** — consumer lag & partition queues as where backpressure lives.
- **L27** — the spike that outruns boot time is the reason flow control is needed at all.

## What it sets up next
**Lesson 0029 — Data warehousing & OLAP (batch + streaming).** We've now protected the
serving/write path end to end (L01–L28). Next we pivot to the read/analytics path: asking
big aggregate questions over the whole event firehose (L21's 25-trillion stream) —
columnar storage, partition pruning, the OLTP-vs-OLAP split, and the lambda/kappa (batch
vs streaming) reconciliation. Trade shifts to **query freshness vs cost & complexity.**

Spine still has #29 (OLAP), #30 (secrets/encryption), #31 (deployments/rollout) queued —
course does not yet need new topics appended.
