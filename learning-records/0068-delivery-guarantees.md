# Lesson 0068 — Delivery Guarantees & the Dead-Letter Lifecycle

**File:** `lessons/0068-delivery-guarantees.html`
**Spine topic:** #68 (Advanced batch — first of the L67-onward "pipeline reliability" arc)
**Date:** 2026-08-24

## Worked example
One `order.placed` event through a **four-hop pipeline** — ingest → payment → fulfillment → notify — at 2,000 orders/s (peak ~10,000/s), each hop a separate process behind a durable log (L9). Question: what does "delivered" mean end to end when any hop can crash, retry, or choke?

## What it covered
- **The three delivery semantics.** At-most-once (may lose), at-least-once (may duplicate), exactly-once. The two-generals argument that exactly-once *delivery* is impossible on an unreliable network (an ack can always be lost), while exactly-once *effect* — "effectively-once" — is routine.
- **Estimate.** Duplicates are not rare: 0.05% crash-before-ack at 2,000/s = 1 dup/s/hop = **~86,400 double-charges/day** at the payment hop alone, ~345,600/day across four hops. Forces dedup. At-most-once rejected because loss > duplication for orders.
- **Model.** At-least-once transport + idempotent consumers = effectively-once. Dedup lives at **every** hop, keyed by a **producer-stamped idempotency key** carried unchanged through all hops. The key trap: the effect and its dedup-key record must commit in **one atomic step** (L13/33) — record-first loses on crash, effect-first doubles on crash.
- **Poison messages & DLQ.** A deterministically-failing message retried forever burns CPU and, in an ordered partition, head-of-line-blocks everything behind it (~15,500 good orders behind one bad one over a 1+2+4+8+16 = 31 s retry ladder at 500 msg/s). Fix = bounded retries → **dead-letter queue** (quarantine with the failure reason), then **triage + replay** (safe because idempotent).
- **Trace.** Four paths: happy (effect once), duplicate (absorbed to a no-op), poison (quarantined), replay (re-injected, dedup protects partial successes).
- **First bottleneck.** Certainty is *stored in-flight state*: dedup store grows ~35 GB/day (691M keys) and needs a bounded TTL window (L8/48 dial, L21 Bloom filter); dedup lookup taxes the hot path; the un-acked backlog is backpressure (L27/28); and "exactly-once" reaches only as far as effects can be made idempotent.

## Numbers used (all checked)
- 0.05% × 2,000/s = 1 dup/s/hop; ×86,400 = 86,400/day; ×4 hops = 345,600/day.
- Dedup keys: 2,000/s × 4 hops × 86,400 s = 691.2M/day; ×50 B ≈ 34.6 GB ≈ 35 GB rolling.
- Retry ladder 1+2+4+8+16 = 31 s; 500 msg/s × 31 s = 15,500 blocked orders.

## Threads reused
L9 (durable log, at-least-once, per-partition ordering), L13 (idempotency keys), L33 (outbox / effect+marker atomic), L7 (timeout tells you nothing), L17 (trace IDs), L28 (backpressure), L27 (autoscaling), L8/48 (window/TTL, hot-path lookup), L21 (Bloom filter), L64 (speed layer, teased next).

## Sets up next
- **Lesson 69 — Hot/warm/cold path & the serving-vs-analytics split:** one event feeding a low-latency serving store, a streaming speed layer (L64), and a batch warehouse (L29) at once, with a read-path router (L37 CQRS).
- Then Lesson 70 — idempotent, resumable data APIs.
- Added a fresh advanced batch (71–75: quorum/tunable consistency, bulk/batch APIs, entity resolution, config/flag delivery, append-only audit logs) so the spine stays ahead.
