# Learning Record — Lesson 0026: Real-Time Delivery (WebSocket / push at scale)

**Date:** 2026-07-13
**File:** `lessons/0026-realtime-delivery.html`
**Spine topic:** #26 — Real-time delivery (WebSocket/push at scale)
**Trade named:** statefulness vs elastic scale

## The one worked example
A real-time delivery layer for Lesson 25's wallet: tell 10M phones "You got paid $50"
the instant the ledger moves, in ~100 ms, without them asking. Direct pickup of L25's
teaser ("the ledger settled the money; now tell the phones").

## What it covers (the four moves)
- **Estimate.** 10M concurrent sockets @ ~20 KB ≈ 200 GB across ~40 gateways (250k
  each) — idle connections are cheap. The killer contrast: polling every 5 s = 10M/5 =
  **2,000,000 req/s** to carry only ~232 real events/s (10M transfers/day ≈ 116/s × 2
  people notified) = **~8,600× waste**, and still 5 s stale. That gap is the entire
  reason to hold live connections.
- **Model.** Split a dumb, **disposable gateway** (holds the raw WebSocket — an HTTP
  request *upgraded* into a persistent full-duplex socket) from the durable truth in a
  shared **connection registry** (`user → gateway`, heartbeat-renewed TTL). **Presence**
  = "is there a live registry entry?" for free. Off-gateway state is what makes a
  stateful layer survivable.
- **Trace.** (A) Bob connects — LB picks any gateway, the registry write is the only
  durable effect. (B) Alice pays Bob — dispatcher does registry lookup → forward to that
  one gateway (L09 pub-sub) → one frame down the open socket (~100 ms), fanned out to
  all devices (L15). (C) Bob offline — event does NOT vanish; degrade to store-and-forward
  mobile push (APNs/FCM) + durable inbox (L15). The live socket is an accelerator, never
  the system of record (L25's disposable-fast-path discipline).
- **First bottleneck.** Correlated motion: a gateway with 250k sockets dies → all 250k
  reconnect in ~1 s (~6,410/s per survivor = 51× baseline, ~20 cores of TLS handshake, a
  lockstep metastable storm — L02/L07 thundering herd). Fix with **backoff + jitter**
  (L07, → ~1.7× baseline, breaks the lockstep), **capacity headroom** (L07/L23's N−1),
  and **sticky-but-disposable** routing. Secondary walls flagged: slow-consumer
  backpressure (bound the per-conn send buffer) and the registry as a hot dependency
  (shard/replicate, L04/L06).

## Threads reused (course is cumulative)
L02 (thundering herd), L07 (timeout / backoff+jitter / unbounded-queue backpressure),
L09 (pub-sub forwarding), L15 (fan-out + durable inbox), L23 (N−1 headroom), L25 (fast
path is a disposable accelerator, never the truth).

## Quiz (4 Q)
1. Why push over polling (the 2M req/s vs ~232 events/s waste + staleness).
2. How an event finds Bob's specific socket (registry lookup → forward → write, not
   broadcast, not a fixed hash).
3. The reconnect-storm fix (backoff + jitter + headroom, NOT reconnect faster, NOT
   resurrect sockets).
4. Why "stateful but disposable" matters (truth off the gateway → a death costs a
   reconnect, not a message; the gateway is not stateless).

## What it sets up next
**Lesson 27 — Autoscaling & capacity planning.** We insisted on "headroom" so N−1
gateways survive; next: *how much* headroom, and how to add gateways *before* the storm.
Little's Law (L13) returns, the ~70% utilization latency knee, reactive vs predictive
scaling, and the cold-start tax. Trade shifts to **cost vs headroom for spikes**.
