# Learning Record — Lesson 0050: Global Traffic Management

**Date:** 2026-08-06
**File:** `lessons/0050-global-traffic-management.html`
**Curriculum topic:** #50 (Advanced batch — "the course never runs dry" queue)
**Trade named:** proximity & availability vs routing staleness & complexity

## The worked example
A service live in **three regions** (us-east/Virginia, eu-central/Frankfurt,
ap-southeast/Singapore) — each a full Lesson 49 stack (gateway + Lesson 42 fleet
+ data, Lesson 23) — with a shopper in **Sydney**. Before her request can reach
any regional load balancer, her device must resolve `shop.example.com` to an
**IP to connect to**. The whole lesson lives in that one step: which region's IP
does the name resolve to, for *this* user, *right now* — and how fast can that
answer change when a region dies?

## The four moves
1. **Estimate — proximity + availability, Lesson 23's two axes.**
   (a) Proximity: light in fiber ~200,000 km/s. Sydney→Singapore ~6,300 km →
   `2 × (6,300/200,000) ≈ 63 ms` floor (~95 ms real); Sydney→Virginia ~16,000 km
   → `2 × (16,000/200,000) = 160 ms` floor (~230 ms real); diff ~97 ms/RTT, and a
   cold HTTPS load ≈ 3 RTTs → `3 × 97 ≈ ~290 ms` saved by handing Sydney the near
   region. (b) Availability: a fixed IP anchored in one region = a 100% outage when
   that region dies — including users a healthy region could serve → the spare
   regions of Lesson 23 are only real if the front door can steer to them.
2. **Model — two places to steer + the health gate.**
   - **DNS-based (GeoDNS / latency):** authoritative server returns a different IP
     by the resolver's (or user's via **EDNS Client Subnet**) location. Rich control
     (geo/latency/weight/health) but every answer is **cached for its TTL** at
     resolver/OS/browser → a failover propagates only as caches expire.
   - **Anycast (BGP):** the *same* IP announced from many regions; the network
     delivers each packet to the nearest announcement. Failover = withdraw the
     announcement → reconverges in **seconds**, no DNS cache to wait out. But
     **coarse** (nearest by topology, not latency/weight) and a re-route can reset
     live TCP → default for stateless/QUIC, careful for long-lived TCP.
   - **Health check** gates both = Lesson 34's readiness lifted to the **region**
     (`detection ≈ interval × failure-threshold`; single-vantage = false-fail risk).
   - They compose: anycast the DNS servers / an edge for the fast path, GeoDNS logic
     for rich selection, Lesson 42/34 inside each region for server-level failure.
3. **Trace** — steady state (Sydney→SG ~95 ms, cache 60 s); GeoDNS region death
   (failover bounded below by `detection 30 s + TTL 60 s ≈ 90 s` cached tail; new
   users route to Frankfurt sooner); anycast death (BGP reconverges in seconds, cost
   = TCP resets); and the **false failover** — a partition makes a live Singapore
   look dead to a Virginia-based checker → needless reroute, or if failover also
   **promotes a writer** → **split-brain** (two primaries, divergence, Lessons 10/11).
4. **First bottleneck — DNS caching bounds how fast you can drain.**
   The TTL is a request, not a command (browsers pin, resolvers round tiny TTLs up
   or ignore them). Lower TTL 60→20 s shrinks the failover window (~90→~50 s) but
   triples DNS QPS (`40,000/60 = 667/s` → `40,000/20 = 2,000/s`) and past a point
   client caches ignore you. Fix: don't fight DNS — **layer anycast** for the
   seconds-fast path and reserve DNS changes for the coarse, rare region drain.

## Four traps
1. A fixed, un-steered address in front of multi-region (forfeits proximity AND
   availability; the redundancy is unreachable).
2. Treating DNS failover as instant (it's `detection + TTL + caches you don't
   control` — tens of seconds; use anycast for seconds).
3. Health-checking a region from one place (can't tell "dead" from "unreachable
   from me" = CAP at the routing layer, L11 → check from multiple vantage points,
   agree by quorum before draining).
4. Letting routing decide correctness (routing = **AP**, a stale route costs a
   retry; write-promotion = **CP**, needs consensus + fencing, L10/23 — conflate
   them and a routing hiccup becomes two primaries and lost data).

## Reuse / callbacks
L10 (consensus / split-brain / fencing — why promotion isn't a routing decision),
L11 (CAP: dead-vs-unreachable, the partition dilemma at the routing layer), L14
(speed-of-light tax at continent scale), L23 (multi-region active-active — the
spare regions this layer makes reachable), L31 (canary/weighting → weighted DNS
records), L34 (readiness & health checks lifted to the region; flap), L42 (load
balancing — region selection above, server selection within).

## What it sets up next
Topic #51 — **Chaos engineering & fault injection**: fifty lessons built failure
defenses (L07 timeouts/breakers, L34 health checks, L36 cells, L46/L50 failover),
but a defense you've never triggered is a hypothesis, not a fact. The next lesson
deliberately breaks things — kill a node, add latency, drop a region, partition
the network — to *verify* the designs hold, with blast-radius control and a
steady-state hypothesis so the experiment itself is safe. Trade: confidence in
resilience vs the risk of the experiment. (Also queued fresh: content moderation,
feature stores/ML serving, billing & metering, edge computing, data-privacy
deletion & compliance — so the spine never runs dry.)
