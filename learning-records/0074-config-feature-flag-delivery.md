# Lesson 0074 — Config & Feature-Flag Delivery: Flip a Switch on 40,000 Servers in Seconds

**File:** `lessons/0074-config-feature-flag-delivery.html`
**Spine topic:** #74 (config / feature-flag delivery at scale)
**Date:** 2026-08-30

## Worked example
One flag, `checkout_v2`, deployed-but-off, released to **1%** of users, across a **40,000-server** fleet (L27). Goal: ramp it **1% → 5% → 25% → 100%** watching error rates, and kill it instantly (→ 0%) if it misbehaves — all in *seconds*, no deploy, no restart. The value lives in a **config plane** whose whole job is to deliver changes to the fleet fast and safely. The data is tiny; the hardness is that changing it changes the running behavior of 40,000 machines at once.

## What it covered
- **Estimate — naïve polling is a load-vs-freshness trap.**
  - Config changes are rare (~10/day = 1.16×10⁻⁴/s). Short-poll every 1 s = **40,000 req/s** and up to 1 s stale; 100 ms freshness = **400,000 req/s**. Fleet does 40,000×86,400 = **3.456B** polls/day for 10 changes ≈ **346M pointless polls/change**.
  - Payload: 100 KB blob × 40k = **4 GB** per round (32 Gbps at 1/s). ETag/`304` unchanged check = 200 B × 40k = **8 MB/s** (500× cut); delta on change = 100 B × 40k = **4 MB** once.
  - **Long-poll (hanging GET):** ask once, plane HOLDS open until change-or-30s-timeout. Idle reconnects = 40,000/30 ≈ **1,333 req/s** — **~300×** less than 100 ms short-poll AND near-instant. Removes the "when to ask" guesswork.
  - The tempting disaster = read the config plane per request → puts it on the hot path of all 40k req/s + a 40,000-way dependency (L34). Delivery ≠ evaluation.
- **Model — the config plane, four machines.**
  - **Versioned store** — monotonic version bumps every change (v1200→v1201), immutable versions → exact rollback (L25/38); read-optimised tiny DB with an outsized blast radius.
  - **Delivery (latency-vs-statefulness dial)** — short-poll (simple, stale, fleet/P load) / **long-poll** (default: near-push, client-initiated so firewall-friendly, nearly-stateless plane, fleet/timeout load) / **push-streaming** (L26 socket, lowest latency but 40k stateful conns + reconnect/slow-consumer issues). Always send a **delta**.
  - **Local cache + fail-static** — RAM (instant reads) + disk (survive restart); background loop updates it. A flag read is a µs map lookup, never a network hop. **Fail-static** (L34 AP): plane unreachable or value fails validation → serve last-known-good, never block/blank/crash; baked default snapshot for boot-during-outage.
  - **Bucketing / targeting** — `bucket = hash(uid) mod 10000; on = bucket < pct*100` (25% → <2500). Deterministic (same user, same answer, every server), stateless, **flicker-free** (raising % only ADDS users, monotone). L4 stable-hash instinct; ramp = L31 canary for *behavior*.
- **Trace.** (A) 1%→25%: validate → write v1201 → held long-polls wake → delta → atomic in-memory swap → next request buckets 25% ON, ~2 s, no deploy. (B) kill-switch 25%→0%: same path in the "off" direction, ~2 s vs L31 redeploy = minutes (~100× faster — only a VALUE moved). (C) plane outage: long-poll fails → serve last-good cache, backoff+jitter retry (L7); boot-during-outage reads baked default. Fail-static = plane is a channel, never a hot-path dep.
- **First bottleneck — the flip is NEVER atomic.**
  1. **Mixed-version window** — ~2 s of propagation, some servers v1200 / some v1201 (eventual consistency L6/11); a user's two requests hit both → flicker / half-applied workflow. Can't make it atomic → make it SAFE: **sticky per-session eval** (pin the decision) + **backward/forward-compatible features** (L24/31 expand-contract — a config push IS a rolling deploy of behavior, old+new must coexist).
  2. **Bad value spreads as fast as good** — fat-finger 100%, unparseable value crashing 40k on load, faster than a human reacts. → schema-validate before accept, client fail-static drops garbage, **canary the config change itself** to 1% of servers watching error rate/p99 (L17/31). A config push deserves a deploy's guardrails.
  3. **Correlated SPOF** — 40k depend on one service; one bad/empty config fails ALL at once (worse than any single-machine failure, L7/34). → fail-static (unreachable = non-event), **CP source-of-truth** (consensus L10/62 → one agreed version) + **AP cached/CDN delivery** (L14/48 — config is identical bytes, fan out from a cache tier, never all 40k on the primary). Write CP / delivery AP.
  4. **Publish thundering herd** — a change wakes all 40k held long-polls at once (L2/26). → jitter the notify/reconnect, serve the delta from a CDN/cache tier (perfect cache hit), relay-tree fan-out (plane → ~100 relays → ~400 servers each) so no node holds 40k conns (L15/26 shape).
- **Deepest point.** A config plane is a tiny **extreme database** whose writes change every other system's behavior in seconds → "release" (L31) becomes its own distributed system with all prior problems (versioning L25, eventual consistency L6/11, blast radius L31, fail-static L34, herd L2/26, stateless hash L4) compressed into seconds. The engineering isn't the store or the delivery — it's respecting that **the speed you built is the risk you took on**, so every deploy guardrail (validation, canary, instant revert to an immutable version, audit log) has to be rebuilt around the value.

## Numbers used (all checked, `python3`)
- Polls: 40,000×86,400 = **3,456,000,000/day**; /10 = **345.6M per change**. Short-poll 1 s = **40,000 req/s**; 100 ms = **400,000 req/s**.
- Long-poll: 40,000/30 = **1,333.3 req/s**; vs 100 ms short-poll ratio 400,000/1,333 = **300×**.
- Payload: 100 KB×40,000 = **4 GB** (×8 = 32 Gbit); ETag 200 B×40,000 = **8 MB/s**; delta 100 B×40,000 = **4 MB**.
- Bucketing: 25% of 10,000 = cutoff **2,500**.

## Threads reused
L31 (deploy vs release, feature flags, canary — this is the delivery mechanism + canary applied to config), L26 (long-poll/push, held connections, reconnect storm), L27 (the 40k-server fleet), L4 (consistent hashing — stable-hash bucketing), L6/11 (eventual consistency, CAP — the mixed-version window; write CP / delivery AP), L34 (service discovery — fail-static, AP delivery vs CP truth, shared-dep health trap), L14/48 (CDN & caching — identical bytes, fan out from a cache tier), L25/38 (ledger/event sourcing — immutable versions, exact rollback), L2/15 (thundering herd & fan-out), L17 (observability — watch error rate/p99 to gate a canary), L24 (expand-contract — old+new coexist).

## Sets up next
- **Lesson 75 — Append-only audit logs & tamper evidence:** an immutable, verifiable "who did what when" (including who flipped which flag) — hash-chaining each entry to the last (a mini-blockchain), Merkle proofs a single entry is present & unaltered, write-once storage. Trade: verifiability & compliance vs write cost and the impossibility of edits.
- Spine was down to one queued topic, so this run ADDED a new batch (76–80: health checks & graceful degradation; blob/media processing pipelines; multi-level & write-through vs write-back caching; sharding strategies & resharding live; serialization formats cost) so the course never runs dry.
