# Learning Record — Lesson 0051: Chaos Engineering & Fault Injection

**Date:** 2026-08-07
**File:** `lessons/0051-chaos-engineering.html`
**Curriculum topic:** #51 (Advanced batch — "the course never runs dry" queue)
**Trade named:** confidence in resilience vs the risk of the experiment itself

## The worked example
The **checkout flow** built across the whole course: a "Pay" tap crosses Lesson 49's
API gateway, hits Lesson 27's fleet behind Lesson 42's load balancer, and calls
**payment-svc** (critical, Lesson 25's ledger), **inventory-svc** (critical), and
**recommendation-svc** (a nice-to-have that must never block a purchase). It runs in
three regions (L50), sharded into cells (L36), with a defense bolted onto nearly every
edge (L07 timeouts/breakers/bulkheads, L13 idempotency, L34/42/50 health-check/failover).
The lesson's question is not "what defense to add?" but "do the defenses we already have
actually work?" — answerable only by breaking something on purpose and watching.

## The four moves
1. **Estimate — an untested defense is almost certainly broken.**
   (a) ~50 resilience mechanisms, each ~10% likely silently wrong (timeout at 30 s not 1,
   breaker that never trips, non-idempotent retry) → chance all work =
   `0.90^50 = e^(50×ln0.90) = e^-5.27 ≈ 0.5%` → **~99.5%** at least one broken now (even at
   5% wrong: `0.95^50 ≈ 7.7%` → 92% broken). (b) Blast-radius gap at 5,000 orders/s: live
   discovery = 100% outage, 30-min MTTR → `5,000×1,800 = 9,000,000` failed checkouts;
   controlled experiment = 1% blast, 30-s auto-halt → `5,000×0.01×30 = 1,500` affected (most
   retry) → **~6,000×** less damage, and you're ready for it. The bug exists either way;
   chaos only changes WHEN and HOW BIG you meet it.
2. **Model — the scientific method, not a wrecking ball.** Four parts, each load-bearing:
   - **Steady-state hypothesis** in OUTPUT terms (checkout success ≥99.9%, p99<300ms — not
     CPU; a user feels a failed purchase, not a busy core), stated as a falsifiable claim
     that steady state SURVIVES one named fault.
   - **The fault** (ladder: instance-kill → latency → resource-exhaustion → dependency-fail
     → partition/blackhole → region/cell-drop → clock-skew). LATENCY is the most revealing:
     a dead dep fails fast, a SLOW dep holds a thread/connection/slot open its whole delay →
     drains the pool → L07's whole reason to exist. "Survives a crash" ⇏ "survives slowness."
   - **Blast radius** — L31 canary applied to a fault: start at 1 instance / 1% / 1 cell (L36
     = a natural blast boundary), keep a control group, widen only after a pass, NEVER 100%.
   - **Abort/halt condition** — L31's automated rollback gate: auto-halt if success<99% for
     30 s; a human on a dashboard is backup, not primary. Independence of the abort is the crux.
3. **Trace** — (A) kill 1 payment-svc instance → L34/42 health-eject+reroute + L07 retry +
   L13 idem key → dip ~99.95% for ~3 s → HOLDS, confidence earned, widen; (B) THE PAYOFF:
   +5 s latency on recommendation-svc, EXPECT L07/49 timeout→degrade, ACTUAL no timeout set
   → aggregating BFF blocks 5 s → thread-pool freeze (L07/13 Little's Law) → checkout craters
   to 82% on 1% in 28 s → a non-critical dep sank the critical path, found cheaply; fix the
   timeout, re-run same experiment → HOLDS; (C) THE DANGER: partition inventory-svc↔DB but
   the abort+monitoring run THROUGH the partitioned path → dashboard stale-green, abort never
   arrives → "controlled experiment" becomes an incident (L36 shared-component trap, in the
   TOOLING); (D) THE LIE: run it all in staging @ 10 req/s, everything passes, proves nothing
   (no real load/skew/hot-keys/cross-region latency = L31 unrepresentative-canary trap).
4. **First bottleneck — the safety of the experiment itself.** Value unlocks only in
   production (staging lies, Path D) but production = real customers, so the practice stands on:
   (1) blast radius small AND shrinkable to zero (halt in 30 s @ 1% = ~1,500 worst case);
   (2) safety plane (monitor+abort) must NOT share fate with the target (L36 in the tooling —
   partition the net, your abort can't cross it; exhaust the DB, your monitor can't need it);
   (3) detection FASTER than harm (L17 observability, L27 knee where latency goes vertical);
   (4) the deeper, cultural wall — **blameless**: punish a failed experiment and no one runs
   the one that finds the expensive bug; the machinery only produces confidence if people will
   pull the trigger.

## Four traps
1. No steady-state hypothesis — "kill some servers and see" is vandalism; without a
   measurable output-terms healthy + a falsifiable claim you can't tell pass from fail.
2. Unbounded blast radius — starting at 100% or with no abort turns a broken defense into a
   full outage (the 9,000,000-request cost to learn what 1,500 would teach).
3. Safety plane shares fate with the target — an abort that travels the network you're
   partitioning fails exactly when needed (L36 shared-component trap in the tooling).
4. Proving it only in staging — staging finds crashes; only production reveals the failures
   that need real load/skew/dependencies (which is WHY blast radius + abort exist).

## Reuse / callbacks
L07 (timeouts/breakers/bulkheads/retries = the defenses under test; slow-vs-dead), L10/11
(partition & split-brain = the fault a network experiment injects), L13 (idempotency = safe
retry under fault injection), L17 (observability = the abort's signal, detection-vs-harm),
L27 (knee = a fault at peak cascades where staging can't), L31 (canary + automated rollback =
blast radius + abort, pointed at a fault; unrepresentative-staging trap), L34/42/50
(health checks & failover, verified in Path A), L36 (cells = natural blast-radius boundary +
the shared-component trap that sinks Path C).

## What it sets up next
Topic #52 — **Content moderation & abuse systems at scale**: turning from infrastructure to
the systems that run on top of it. Classify a firehose of user content (spam/fraud/harmful)
faster and cheaper than a human could read it, with a cheap-model-first funnel (L16/29
narrow-before-expensive), human-review queues + appeals, and the precision/recall dial (L16)
where BOTH errors are costly (miss real abuse, or wrongly silence a real user). Trade: safety
coverage vs false-positive harm & review cost. (Also still queued: feature stores/ML serving
#53, billing & metering #54, edge computing #55, data-privacy deletion & compliance #56 — the
spine stays full, no fresh topics needed this round.)
