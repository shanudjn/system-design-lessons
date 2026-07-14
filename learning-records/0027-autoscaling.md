# Learning record — Lesson 0027: Autoscaling & Capacity Planning

**File:** `lessons/0027-autoscaling.html`
**Title:** Autoscaling & Capacity Planning: Sizing a Fleet Before the Spike Hits
**Date authored:** 2026-07-14

## What this lesson covered
Cashes the "headroom" IOU from Lesson 26 by sizing and steering the stateless
API tier behind the connection gateways. Worked system: **40,000 req/s** peak,
**40 ms** service time per request, **8 cores/server** (→ 200 req/s capacity).

- **Estimate (two ways that must agree):**
  - *Floor from Little's Law (L13):* busy cores = λ·S = 40,000 × 0.04 = **1,600
    cores = 200 servers** at an impossible 100% utilization — the physical wall,
    not an operating point.
  - *Target from the knee:* model a server as a queue, response time
    `T = S/(1−ρ)`. The curve is flat then vertical: 40 ms request → 80 ms @50%,
    **133 ms @70%**, 200 ms @80%, 400 ms @90%, 800 ms @95%, **4,000 ms @99%**.
    Operate at the **knee (~70%)** → each server carries 140 req/s → **286
    servers**. Cross-checks via cores (1,600/0.7 = 2,286/8 = 286).
  - The extra 86 servers (43%, ≈ **$377k/yr** @ $0.50/hr) buy *distance from the
    cliff*, not throughput. Trade: **latency vs cost, decided at the knee.**
- **Model — the autoscaler as a feedback loop:** measure ρ → compare to target
  ρ*=70% → `desired = current × ρ/ρ*` = `load ÷ (C × ρ*)` → act → cooldown.
  Design lives in the hidden parts: scale-out fast / scale-in slow; wide band +
  cooldown to avoid **flapping** (L10's leadership flap, now in servers). Trade:
  **responsiveness vs stability.**
- **Trace three loads** on the one comparison that decides all — does load rise
  slower or faster than a server boots:
  - *A — morning ramp:* slow climb, reactive target-tracking stays a step ahead
    (the default, and why reactive works for most services).
  - *B — sudden 2× spike:* reactive CANNOT catch it. Fleet runs
    ρ = 80,000/(286×200) = **140%** for ~3 min, backlog grows 22,800 req/s ≈
    **4.1M** requests, latency leaves the knee → timeouts → L07 retry storm.
  - *C — known 8:30 surge:* beat only by **predictive/scheduled pre-scaling**
    ahead of the event. Trade: **react vs predict.**
- **First bottleneck — the ~3-min cold-start lag** = a control loop's **dead
  time** (minutes vs seconds-long spikes). Where the 180 s goes (provision /
  app start / warm-up / health check / metric window). You can't zero boot time,
  so attack the *gap*: (1) spike-buffer **headroom** (run 60% → 333 servers,
  absorbs +16% to 46,620/s instantly = one boot-window of slack), (2) **warm
  pools** (pay idle, take traffic in seconds), (3) **predictive** pre-scale,
  (4) **faster warm-up** (smaller images, snapshot/restore). **Scale-to-zero**
  is the opposite extreme of the same dial (cost vs cold-start latency).
- **Core trade named:** cost vs headroom for spikes — you can't set headroom to
  zero because the boot-time lag means capacity ordered now isn't capacity had
  now.

## Reuses / threads pulled through
- L13 Little's Law (`L = λW`) to size the fleet floor.
- L07 timeout / retry storm / metastable collapse = what happens past the knee.
- L10 flapping = the autoscaler's thrash around the threshold.
- L26 "headroom" and the reconnect storm as a concrete spike source.

## Interactive quiz (4 Q)
Why 200 servers is the floor not the target (the knee); why a perfectly-tuned
reactive autoscaler still loses to a 2× spike (provision ≠ have); how to handle
a *predictable* surge (pre-scale, don't react); which levers reduce cold-start
*damage* (attack the gap, not the boot time).

## What it sets up next
Lesson 0028 — **Backpressure & flow control end-to-end.** This lesson proved
scaling can't beat a spike that outruns boot time, so for those ~3 minutes
something must give: not "queue everything" (the 4.1M-request backlog that
collapses) but *saying no gracefully*. Next: bounded queues, credit-based flow
control, and load shedding across a whole call graph (recap L05/L07/L09) —
deciding WHERE to put the "no" (admission control at the edge). Trade shifts to
**throughput vs tail latency & stability.**
