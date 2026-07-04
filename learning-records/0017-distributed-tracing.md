# Lesson 0017 — Distributed Tracing & Observability

**Published:** 2026-07-04
**File:** `lessons/0017-distributed-tracing.html`
**Spine topic:** #17 (Distributed tracing & observability)

## The worked system
The Lesson 16 search backend in production: one query fans out across ~20
service calls (gateway → auth → parser → retrieval → re-ranker → feature
store). The mean response is a healthy 44 ms, but 1% of requests take ~950 ms
and no single service's dashboard shows anything wrong. Goal: find the slow hop.

## What it covered (four moves)
- **Estimate — why not keep everything:** 50k req/s × 20 spans × 500 B =
  1M spans/s = 500 MB/s = **43.2 TB/day**. Unaffordable → sample (1% → 432 GB/day).
  Names the core trade-off: **observability cost vs coverage**.
- **Model — a request is a tree of spans:** span = `{trace_id, span_id,
  parent_id, name, start, duration}`; a trace = all spans with one trace ID,
  reassembled from parent pointers. **Trace-context propagation** (the
  `traceparent` header) is what makes it distributed — the weakest-instrumented
  hop is a blind spot. Also why **metrics** (aggregates, no per-request
  identity) can't do this job — the **three pillars** division of labor.
- **Trace the paths:** (a) the mean lies — 99×35 + 950 → 44 ms hides a 950 ms
  tail, so alarm on **p99** not the mean; (b) read the **waterfall** to expose
  an **N+1** of 100 serial 8 ms feature-store calls (800 ms of serial waiting)
  → batch into one 8 ms call → request 950 ms → 158 ms (~6×); (c) **tail
  amplification**: `1 − 0.99²⁰ ≈ 18%` of requests hit at least one slow hop,
  so fan-out makes the whole-request p99 far worse than any one service's.
- **Bottleneck — sampling drops the trace you needed:** **head-based** sampling
  decides blind at ingress, keeping a slow trace only `1% × 1% = 1-in-10,000` →
  **tail-based** sampling buffers the whole trace and keeps slow/errored ones
  (pays memory + coordination). Plus the metrics-side cliff: **cardinality** —
  high-cardinality ids (user, request) belong on traces, never on metric labels.

## Trade-offs named
observability cost vs coverage · instrumentation cost vs blind spots ·
the mean hides the tail · head sampling cheap-but-blind vs tail sampling
smart-but-buffered · metric granularity vs cardinality cost.

## Callbacks used
- L16 fan-out search is the system being instrumented.
- L07's "set timeouts from p99 not a lazy default" reappears as "alarm on p99."
- The "narrow cheap, perfect expensive" instinct → "record cheap aggregates,
  keep expensive traces sparingly."

## What it sets up next
- **Lesson 18 — API design & pagination:** the trace exposed an N+1 (100 round
  trips where 1 batch would do); that lives in the API shape. Next: REST vs RPC
  vs GraphQL, and offset vs cursor pagination (why offset breaks under inserts),
  the N+1 problem head-on.
- Later: geo/proximity systems, object/blob storage, approximate counting.
