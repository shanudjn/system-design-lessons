# Lesson 0076 — Health Checks & Graceful Degradation

**Published:** 2026-09-01
**File:** `lessons/0076-health-checks-degradation.html`
**Spine topic:** #76 (was queued in the advanced batch)

## Worked example
One product-page/checkout fleet — 286 servers, 40k req/s (reusing L27) — behind
a load balancer (L42) that probes `/health` every 2 s. The opening disaster:
a naive **deep** health check that pings the **shared** recommendations service
(worth ~2% of revenue). A 10-second latency blip makes all 286 servers fail the
probe at the same instant (correlated failure) → the LB empties the pool → 100%
checkout outage, ~480,000 failed requests. An honest local check → 0 failures.

## What it covers
- **Estimate:** the correlated-failure math — how one shared-dependency blip
  ejects the whole fleet in ~2 probe cycles; 480,000 failed requests vs 0.
- **The one rule:** a health check reports only **local, instance-unique** state.
  Ejection helps only when "bad" is uncorrelated; a shared dep makes it total.
- **Two checks, two decisions (L34):** liveness (shallow, local → restart) vs
  readiness (this instance's own resources → remove, don't restart).
- **Request-time guarding (L7):** timeout + circuit breaker + degrade for shared/
  non-critical deps — never in `/health`.
- **Feature ranking + load shedding (L28):** core checkout vs garnish recs; shed
  top-down by tier, then by request class.
- **Fail-open vs fail-closed** as a per-dependency correctness/security call:
  "which mistake can I survive?" — garnish open, money/auth closed.
- **Trace:** healthy request, recs blip (degrade), critical DB down for one box
  (eject) vs whole fleet (don't — fail-static + LB panic mode, L34/42), overload.
- **Walls:** ejecting a fleet-wide failure is useless; fail-open/closed per dep;
  degraded modes rot if untested (chaos, L51) and hide if silent (alarm, L17).
- **Deepest point:** "up" ≠ "useful"; resilience is a spectrum of degraded-but-
  serving states, built from pieces already owned (L7/17/28/34/42/51).

## Trade named
Availability of the core vs completeness of the experience.

## Sets up next
Lesson 0077 — **Blob/media processing pipelines (transcoding at scale):** one 4K
upload (L20 object storage) fanned into every resolution/codec/thumbnail — a DAG
of jobs, priority queues (viewer-waiting vs background), re-encode-on-demand vs
store-all-variants, idempotent/resumable jobs (L70). Trade: storage vs compute vs
time-to-first-play. Remaining queued spine: 77–80 (transcoding, multi-level
caching, resharding live, serialization formats).
