# Learning record — Lesson 0031: Deployments & Progressive Rollout

**File:** `lessons/0031-deployments-rollout.html`
**Topic (curriculum spine #31):** Deployments & progressive rollout
**Trade named:** release velocity vs blast radius

## The one system
Lesson 27's API tier: **286 servers, 40,000 req/s**, target utilization below the knee. A new
version `v2` is built and passed tests — but tests never cover the exact mix of live traffic,
real data, and production load. Ship `v2` to every server without an outage, with a fast, safe
way back to `v1`. Core distinction the lesson rests on: **deploy** (code *running* on servers)
≠ **release** (behavior *reaching* users) — keeping them separate is the strongest lever.

## What the lesson covers (four moves)
- **Estimate — big-bang blast radius.** Blast radius = rate × time-to-recover. Big-bang: a bug
  meets **40,000 × 300 s = 12,000,000** requests (100% of users) before a ~5-min rollback lands.
  Same bug as a **1% canary**: 400 req/s × 300 s = **120,000** (100× cut), and with automated
  analysis reacting in ~60 s → **~24,000** (500× cut), and it never promotes past 1%. Trade:
  release velocity vs blast radius — every safer strategy trades deploy speed or spend for a
  smaller blast.
- **Model — three strategies + the flag.** Two dials: shrink the exposed slice, or shrink the
  time-to-undo. **Rolling** (batches of ~29; no extra fleet, but capacity dip 286/257 ≈ 1.11×,
  exposure grows per wave, slow backward rollback, mixed versions live). **Blue-green** (two full
  fleets, flip the LB → rollback in seconds; 2× transient fleet ≈ $143 for a 1 h flip; but the
  flip is all-at-once → optimizes rollback SPEED, not blast radius). **Canary + health-gated
  promotion** (1%→5%→25%→50%→100%, gated on error-rate/p99 vs baseline, L17; smallest blast
  radius, costs metrics + bake time + automation). **Feature flag** (orthogonal): decouple
  deploy from release — dark-launch code off, flip on for 1% of *users*, rollback = a ms flag
  flip, no redeploy; works for changes you can't route around; composes with the others.
- **Trace — healthy vs poisoned.** Path A (good `v2`): deploy to 1% → bake → gate passes →
  ramp → **drain** old instances (finish in-flight, close connections, L26) before retiring `v1`;
  worst case at each stage is bounded (400 req/s at 1%). Path B (bad `v2`): error rate 0.1%→4%,
  p99 doubles → gate breached → **auto-rollback** to 0%, ~24,000 requests burned, 99% untouched
  — BUT `v2` already **wrote to the shared DB** in its new shape, and the LB flip doesn't un-write
  it → the wall.
- **First bottleneck — two versions, one shared state.** Every progressive strategy runs old +
  new **simultaneously** over unversioned shared state (DB schema, cache, queue message formats
  L09, API contract L18). Hazard 1: mixed versions must interoperate (v2 writes a field v1 can't
  parse → v1 breaks — L18 additive-only, L24 expand/contract between two live versions). Hazard 2:
  the DB is not in the deploy — code flips back in seconds, a schema/data change can't (L24
  migration, L30's one-way DROP). So rollback is only as fast as the slowest-to-reverse layer;
  safety requires every neighbor pair backward/forward compatible → **expand → deploy → bake →
  contract**. Four traps: (1) coupling a destructive migration to the deploy (code rolls back,
  schema doesn't → stranded); (2) a thin/unrepresentative canary that never triggers the bug
  (green ≠ proof; need enough traffic × bake time × representative slice); (3) dropping in-flight
  work instead of **draining**/graceful shutdown (L26); (4) feature flags as permanent 2^N test
  debt (give each an owner + expiry, delete after rollout).

## Threads reused
- **L27** — 286-server / 40,000 req/s fleet + utilization knee (capacity dip during rolling).
- **L30** — the all-at-once blast radius, now in *release* form (this lesson's framing).
- **L24** — expand/contract migrations = the compatibility wall + Trap 1.
- **L18** — additive-only API contract (mixed-version interop).
- **L17** — p99 / error-rate metrics = the health gate's signal.
- **L26** — connection draining / graceful shutdown at cutover.
- **L25 / L13** — compensations + idempotency, the bridge to next lesson.

## What it sets up next
**Lesson 0032 — Distributed transactions & sagas.** Having shipped one service safely, next make
many services commit all-or-nothing without a shared DB transaction: **2PC** (a coordinator that
can block the whole set, L10 with a darker failure mode) vs the **saga** (per-step
**compensating transactions**, L25, when a later step fails), leaning on idempotency (L13) for
safe retries. Trade shifts to **atomicity vs availability across service boundaries.**

Spine refilled: this record's lesson was the last queued topic, so NOTES.md now appends a fresh
batch — #32 distributed transactions/sagas, #33 change data capture & outbox, #34 service
discovery & health checking, #35 logical clocks & causal ordering, #36 cell-based architecture —
so the course never runs dry.
