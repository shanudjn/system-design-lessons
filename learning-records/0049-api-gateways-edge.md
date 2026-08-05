# Learning Record — Lesson 0049: API Gateways & the Edge

**Date:** 2026-08-05
**File:** `lessons/0049-api-gateways-edge.html`
**Curriculum topic:** #49 (Advanced batch — "the course never runs dry" queue)
**Trade named:** centralized cross-cutting concerns vs a fragile shared choke point

## The worked example
A mobile app's home screen over **40,000 req/s** (Lesson 27's fleet) to **30
backend microservices**, each owned by a different team. Every request needs the
same cross-cutting work done first (authN, rate limit, TLS, routing), and the
home screen alone needs data from **six** services at once. The question: where
does that shared work live — in the 30 services (30 copies) or in one **API
gateway** front door (one copy, one choke point)?

## The four moves
1. **Estimate — what the front door buys, each a cost flipped over.**
   (a) Cross-cutting security written **once** vs 30 codebases: a key rotation /
   tighter limit = 30 deploys, 30 chances to drift, one forgotten service = the
   hole → one consistent place. (b) Six-call home screen's ~3 dependent hops
   collapsed from `3 × 100 ms = 300 ms` (mobile RTT) to `100 + 3 × 2 = 106 ms`
   by moving composition into a **BFF** (~2.8× cut, 6 phone connections → 1).
   (c) `20% × 40,000 = 8,000 req/s` of junk rejected at the door for a token
   check (L28 admission control — the cheapest request is the one you reject).
2. **Model — the seven-job pipeline + the identity trick.** TLS-terminate (L30)
   → authenticate → rate-limit (L08) → route (L34/42) → aggregate (L15) → observe
   (L17) → translate. The trick that keeps services simple: authenticate the
   external token **once**, forward a **signed identity header** over **mTLS**;
   services **authorize** (what may this identity do?) but don't
   **re-authenticate** (who is this?) — authN centralized, authZ local. Two lines:
   **north-south** (gateway) vs **east-west** (service mesh/sidecars, L34), and one
   god-gateway vs **per-client BFFs** (mobile/web/partner).
3. **Trace** — clean request (signed header, service authorizes); BFF aggregation
   (one /home → fan out → compose); edge rejection (401/429, shield, no backend
   touched); and the **partial failure** that is aggregation's bill — composing N
   services couples N fates, so one slow `recommend-svc` sinks the whole screen
   unless a **per-call timeout + graceful degrade** returns the five that answered
   (L07).
4. **First bottleneck — the front door itself, failing two ways.**
   (a) The **fattest SPOF** you own: 100% of traffic crosses it → its death is a
   total outage though every backend is healthy (L07/26/42) → fix by making it a
   **stateless disposable herd** behind L42's LB/anycast, N+1 headroom (L27),
   drain-on-deploy (L31/34), token-auth = no session to pin (L26). (b) A **deploy
   bottleneck**: logic accretes → the 1990s **ESB** reborn, a shared codebase with
   100% blast radius every team queues behind (L36) → fix by keeping it **thin**
   (cross-cutting only; business logic + aggregation pushed out to BFFs and
   services; routes as per-team **declarative config** so adding a service isn't a
   gateway code change).

## Four traps
1. The god gateway / ESB (business logic accretes → fat, fragile, deploy bottleneck, L36).
2. Treating the gateway as "infra that just works" (it's the SPOF — run several, health-check, drain).
3. Re-authenticating at every hop OR trusting the internal call blindly (the gateway-bypass / confused-deputy hole → mTLS + signed header + zero-trust internally).
4. Aggregating without per-call timeouts/fallbacks (inherit the worst latency & availability of all six).

## Reuse / callbacks
L07 (partial failure, timeout, SPOF, graceful degrade), L08 (token-bucket rate
limit at the edge), L15 (fan-out → BFF aggregation), L17 (tracing minted at the
gateway), L26 (disposable stateless front door), L28 (admission control /
goodput), L30 (TLS/mTLS, identity, terminate-once), L34 (service discovery + the
east-west mesh line), L36 (shared-component blast radius = deploy bottleneck),
L42 (load balancing in front of and behind the gateway).

## What it sets up next
Topic #50 — **Global traffic management (GeoDNS / anycast / failover)**: this
lesson put a front door in front of one fleet; the next routes a user to the
nearest *healthy* region (L23 multi-region, L42 load balancing) via geo/latency
DNS or anycast, with health-based failover, a DNS TTL that bounds how fast you can
drain a dead region, and the split-brain risk of two regions both thinking they're
primary (L10/11). Trade: proximity & availability vs routing staleness & complexity.
