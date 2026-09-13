# Lesson 0088 — Notification Systems End to End

**File:** `lessons/0088-notification-systems.html`
**Curriculum spine:** topic 88 (advanced batch, queued after L87).
**Worked example:** the L86/L87 marketplace (5M DAU). One event = **order.shipped** for user U-8842 ("Alice"), traced end to end across four channels (push, email, SMS, in-app). Job: right message, right channel, exactly once, decent hour, without training the user to disable notifications.

## What it covered
- **The core idea:** a send is an **irreversible side effect on a scarce, non-renewable resource** (the user's attention + your permission to reach them). Unlike a DB write, a buzz can't be recalled and the opt-out it triggers is ~permanent. So the system is built to **decide whether to send at all** — most stages are filters that DROP.
- **Estimate — volume vs cost:**
  - Volume: 2M orders/day × 3 lifecycle events = 6M/day = **~70/s avg, ~300/s peak**; a marketing campaign is a burst (4M queued at once, ~13,000× the stream) → queue + priority lane (L09).
  - **Cost is the whole story:** push ~$0/instant; email ~$0.0001 (4M = $400, ~5k/s → 13 min); SMS ~$0.0075 (4M = **$30,000**, ~100/s → **11 hours**); in-app ~$0. SMS = **75× email, 50× slower** → can never be bulk → **push→email→SMS fallback ladder** (reach vs cost).
- **Model — the pipeline (filters that mostly say no):**
  - **Dedup (L13/L73):** atomic claim on `hash(user+event_id+type+channel)` in a TTL'd store BEFORE the send (at-least-once upstream, L68); must be atomic or concurrent redelivery double-buzzes. TTL trap (L13).
  - **Preference & routing:** per-user, per-category, per-channel; route = preference ∩ fallback ladder; security can't be fully disabled.
  - **Inbox rate limit (L08):** per-user token bucket (cap 3, refill 1/4h); overflow → **digest**, not dropped. Priority tiers (critical / transactional / marketing) decide what bypasses.
  - **Quiet hours:** timezone-aware hold of non-urgent traffic (Alice in IST, 22:00–08:00). **Render:** template+data per channel/locale; SMS segment math — GSM-7 160/153, one emoji → UCS-2 67/seg → 150-char msg = **1 → 3 segments = 3× cost**.
  - **Delivery (L09/68):** retry TRANSIENT (429/5xx/timeout, backoff+jitter L07), suppress PERMANENT (hard bounce, dead token, unsubscribe) → DLQ; hard vs soft bounce. Receipts/bounces/unsubs feed back into prefs + suppression list.
- **Trace three paths:** (A) happy push — dedup new → pref ON → push → ~1s, $0, other channels correctly unused; (B) duplicate event → dedup key exists → DROP, invisible (contrast: double buzz → opt-out); (C) 2 a.m. promo → held by quiet hours AND empty bucket → released 08:00 as one morning digest email.
- **First bottleneck + walls:** the bottleneck is **not throughput but the permission to be reached**, held on two coupled sides — user (notifications-on; annoy → disable push forever) and providers (**sender reputation**; over-send to unengaged → bounces/spam reports → Gmail spam-folders you for EVERY user). Both non-renewable. Walls: **out-of-order events** (shipped before placed) → state-aware / latest-state sends (L35); **self-inflicted thundering herd** (4M instant blast → 20% tap in 2 min = ~6,700 req/s onto the 40k fleet, L27) → stagger+jitter over 15 min (~4,400/s).
- **Four traps:** send-all-channels-to-be-safe; no dedup ("queue is reliable"); retry every failure the same (burns reputation); blast the campaign all at once.
- **Interactive quiz (4 Q):** why all-channels is the wrong default (cost + permission); duplicate event needs atomic dedup before send; stagger a mass blast (self-DDoS, not cost); retry transient / suppress permanent (hard bounce wrecks sender reputation for all users).

## Reuses / threads pulled through
L87 (the shipped result we now deliver), L73 (dedup/entity resolution → dedup before an irreversible send), L13 (exactly-once effect on at-least-once delivery, TTL trap), L09/68 (queues, retries, DLQ, at-least-once), L08 (token bucket → rationing an inbox), L07/27/28 (backoff/jitter, latency knee, goodput cliff), L35 (logical/causal ordering → out-of-order lifecycle events).

## What it sets up next
**Lesson 89 — Data quality & pipeline observability:** we've trusted the numbers our pipelines emit (order counts, CTR, delivery rates); L89 asks how we know they're right. Schema/contract enforcement at ingest (L80), freshness/volume/distribution checks, the silent-bad-data failure (a null flood no alarm caught), lineage, backfill-safe reprocessing (L57). Trade: data trust & coverage vs pipeline complexity & latency.

Spine still has topics 89–90 queued, so no new topics were added this run.
