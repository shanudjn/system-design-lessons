# Learning record — Lesson 0045: Webhooks & Outbound Event Delivery

**Date:** 2026-08-01
**File:** `lessons/0045-webhooks-outbound-delivery.html`
**Curriculum spine item:** #45 (Webhooks & outbound event delivery)
**Worked example:** the e-commerce platform's outbound webhook system — Lesson 33's outbox emits 5,000 domain events/s, ~2 subscriptions each → ~10,000 outbound POSTs/s to 50,000 endpoints we neither control nor trust.

## What this lesson covered

The framing reversal: every prior delivery lesson pushed to a server WE own; a webhook pushes to a server we DON'T own and can't fix. That flip means we can't BLOCK on the receiver (their downtime becomes ours) and can't DROP (the webhook is their only signal). Followed the four-move spine:

- **Estimate** — Inline delivery (POST from the event pipeline's own thread) welds your availability to strangers'. A dead endpoint accepts the connection and hangs to a 30 s timeout (L07 partial failure — can't fail fast). Little's Law: even healthy at 300 ms, 10,000/s holds L = 3,000 in-flight threads; let 1% go dark → 100/s × 30 s = **3,000** threads frozen on dead endpoints alone, and they accumulate (new dead calls arrive as fast as old ones time out) → checkout pipeline dies from a merchant's expired TLS cert. Fix = decouple: event pipeline durably enqueues a delivery job and returns in µs; a separate worker fleet makes the slow flaky calls (L05/09).
- **Model** — the four properties a stranger forces:
  - **At-least-once** — retry until a 2xx, because a silent drop is invisible to the receiver (they can't detect it). Consequence: the ambiguous timeout (receiver processed it, only the response was lost) means retries can duplicate; no exactly-once on the wire (L09).
  - **Idempotency key** — stable `event_id` in a header (`Webhook-Id`), identical across retries; the published contract to integrators is L13 pointed outward ("store it, ignore repeats"). A fresh id per attempt would defeat their dedup.
  - **HMAC signature** — `HMAC-SHA256(secret, t + "." + body)` in `Webhook-Signature: t=…, v1=…`; per-subscription shared secret, constant-time compare. Gives authenticity (only secret-holder can produce it → forged bodies fail) + integrity (flip a byte → mismatch). Signing the timestamp too → reject anything older than ~5 min = replay protection. NOT a static bearer token (which proves nothing about the body and forges anything if leaked). (L30)
  - **Backoff + jitter + deadline** — 1,2,4,8…s doubling, capped at 1 h, over a 24 h window. Sum 1+2+…+2048 = 4095 s ≈ 68 min for the first ~12 attempts, then hourly for ~23 h ≈ **~35 attempts total**. Jitter (×0.5–1.5) breaks the recovering-fleet thundering herd (L07/26). Exhaustion → **DLQ** + merchant dashboard replay (never silent-drop, never infinite-hammer).
- **Trace** — (A) clean: fan-out → enqueue → worker claims (visibility timeout L05) → sign → POST 30 s → 2xx → DELIVERED. (B) failing: 503/timeout → backoff retries → after 24 h → DLQ + dashboard "Replay all". (C) the ambiguous timeout: worker sees no 2xx → MUST retry → receiver gets it twice → with dedup a harmless no-op, without it a duplicate side effect (second email/charge). This is WHY the idempotency key isn't optional.
- **First bottleneck** — the shared delivery fleet. Healthy baseline L = 10,000 × 0.3 s = 3,000 concurrent. One big merchant at 500 deliveries/s goes dark (hangs to 30 s) → L = 500 × 30 = **15,000** in-flight slots from ONE endpoint = 5× the whole healthy fleet → head-of-line blocking (L09 poison message) + noisy neighbor (L40) at once, everyone's webhooks go late. Fixes: **per-endpoint circuit breaker** (5 consecutive fails → open, park deliveries, probe occasionally → dead endpoint drops from 15,000 slots to ~0, the top lever, L07); **per-endpoint concurrency bulkhead** (cap in-flight per subscription, L07/40); separate first-attempt vs retry queues (L09). Second wall = **ordering**: independent retries reorder events (created backs off while updated succeeds) → don't promise order, make events **self-describing** (version/sequence + timestamp so the receiver skips a superseded event, L35/37) rather than per-key partitions (which rebuild head-of-line on purpose).

Four traps: inline/synchronous delivery; no signature or a static token; assuming exactly-once / a fresh id per retry; one shared pool with no per-endpoint isolation.

Arithmetic double-checked: 5,000 × 2 = 10,000 POSTs/s; 10,000 × 0.3 s = 3,000 concurrent; 1% of 10,000 = 100/s × 30 s = 3,000 frozen; 500/s × 30 s = 15,000 (5× 3,000); 1+2+4+8+16+32+64+128+256+512+1024+2048 = 4095 s ≈ 68.25 min over 12 doublings; ~23 hourly attempts after → ~35 total.

## Reuses / threads pulled forward
L05/09 producer→queue→workers + at-least-once + DLQ + visibility timeout + partitioning; L07 partial failure/timeout/backoff+jitter/circuit breaker/bulkhead; L13 idempotency key + ambiguous timeout + exactly-once effect not on the wire; L26 polling-is-waste (why push) + lockstep reconnect storm (why jitter); L30 HMAC authenticity/integrity/replay + constant-time compare; L33 outbox as the durable event source; L35/37 let-the-reader-resolve-ordering (self-describing versioned events); L40 noisy-neighbor (one endpoint can't own the shared fleet).

## What it sets up next
**Lesson 0046 — Distributed job scheduling** (spine #46): run thousands of cron jobs across a fleet so each fires ONCE at the right time — leader/lease to avoid double-fire (L10/22), sharding the schedule for throughput, missed-fire catch-up after an outage, clock skew across nodes (L35), and idempotent jobs so a retry is harmless (L13). Trade: timeliness vs exactly-once execution.

Spine still has #46 (distributed job scheduling) queued — course will not run dry.
