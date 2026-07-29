# Learning record — Lesson 0042: Load Balancing Algorithms & Layers

**File:** `lessons/0042-load-balancing.html`
**Curriculum spine item:** #42 (first unmarked → now ✅)
**Trade named:** even load distribution vs stickiness & simplicity.

## The one worked system
The front door to Lesson 27's API tier: **286 servers**, offered **40,000 req/s** at peak (~140 req/s each; per-server capacity `8 cores / 0.04 s = 200 req/s`, run at ~70%). A **load balancer** answers one question 40,000×/s — *"which backend?"* — while servers are dying, booting, and being deployed to underneath it.

## Four moves
- **Estimate — why the "fairest" rule breaks first.** Round-robin distributes request COUNTS evenly, which only equals even LOAD when backends are identical. One server slowed 10× (400 ms/req → real capacity `8/0.4 = 20 req/s`) still gets its full **140 req/s** → takes in 140/s, drains 20/s → backlog grows ~120/s without bound (Lesson 28). `1/286 ≈ 0.35%` of all traffic (~140 req/s) is routed into the tar pit and essentially all of it fails/times out (Lesson 7), and retries make it worse. Round-robin is blind to backend state — it counts turns, not trouble.
- **Model — two orthogonal choices.** (1) **Layer**: L4 (transport) forwards whole CONNECTIONS by IP:port, one decision per connection, cheap/protocol-blind — but pins a multiplexed HTTP/2 or long-lived stream (hundreds of requests) to ONE box. L7 (application) reads each HTTP REQUEST, one decision per request, can route by path/header/cookie and retry (Lesson 7), demuxes the stream — pays parse + TLS. (2) **Algorithm ladder**, each rung buying back info round-robin threw away: weighted-RR (unequal boxes) → least-connections & least-latency/EWMA (REACT to load: a slow box's in-flight climbs → routed away) → **power-of-two-choices** (pick 2 random, send lighter: near-optimal balance, no global scan, no herd; imbalance drops from growing like `ln N/ln ln N` to a near-constant `ln ln N`) → **consistent-hash** for stickiness (Lesson 4 ring: same key→same backend, warm cache Lesson 2, only ~1/N remaps on churn vs `%N`'s reshuffle Lesson 3), whose price is a HOT key pinning one box (Lessons 3/15/40) → bounded-load hashing (cap `(1+ε)×avg`, spill on the ring).
- **Trace — three paths.** (A) Healthy fleet, P2C: terminate TLS, read request, pick 2 random, send to the lighter — fleet stays near-even with no herd. (B) The sick box under a load-aware algorithm: its in-flight climbs (8→20→50→100) so least-connections / P2C stop choosing it — arrival falls from 140/s toward 20/s in ms; a health check then marks it unready and ejects it in seconds (Lesson 34), retries paper the gap (Lesson 7). (C) Mid-deploy: **deregister → drain in-flight (cap ~30 s) → die**, new version joins the pool only after passing readiness (Lesson 31/34) → zero dropped requests.
- **First bottleneck — the LB is the fattest SPOF.** Every request crosses it → its death is a 100% outage (Lessons 7/26), purer than any partial failure. Redundancy (anycast / VIP failover VRRP-keepalived / DNS round-robin) removes the SPOF but DESTROYS the global load view: each LB sees only its own traffic, so "global least-loaded" becomes a herd of balancers stampeding stale guesses onto the same "idle" box (Lesson 2). That's exactly why coordination-free RANDOMIZED algorithms (P2C) win at scale — independent random picks don't correlate into a herd.

## Four traps
1. Blind algorithm (round-robin) in front of servers that can slow down — feeds the dying box its full share.
2. Health check probing a SHARED dependency — one DB blip fails ALL backends → LB ejects the whole fleet = correlated total outage (Lesson 34).
3. L4 in front of multiplexed / long-lived connections — pins hundreds of requests to one box → skew, can't rebalance mid-stream. Use L7.
4. Treating the LB as infrastructure that "just works" — it's the SPOF; run several + choose coordination-free algorithms + drain on deploy.

## Prior lessons reused
L2 (cache locality + thundering herd), L3 (`%N` reshuffle + hot shard), L4 (consistent-hash ring), L7 (partial failure/retry/SPOF), L26 (stateful front door), L27 (fleet sizing + utilization knee), L28 (unbounded queue), L34 (readiness vs liveness, health checks, graceful shutdown/drain).

## Sets up next
**Lesson 0043 — Gossip & anti-entropy** (spine #43): keep 1,000 nodes agreeing on membership and 3 replicas agreeing on data with NO central coordinator (contrast Lesson 34's registry, Lesson 10's consensus). Epidemic/gossip dissemination, SWIM failure detection, Merkle trees to find cheaply *which* keys diverged (Dynamo-style repair of Lesson 6's replicas), read-repair vs background anti-entropy. Trade: convergence speed vs message overhead & staleness.
