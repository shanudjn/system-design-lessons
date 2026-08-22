# Learning record — Lesson 0066: Global Rate Limiting & Quota

**Date:** 2026-08-22
**File:** `lessons/0066-global-rate-limiting.html`
**Spine topic:** #66 — Global rate limiting & quota
**Trade named:** limit accuracy vs coordination latency & cost

## What this lesson covered
Takes L08's single-node token bucket and makes its limit hold across a **30-server, 3-region fleet** (40k req/s)
enforcing "**60 requests/minute per key**." The whole problem: on one machine the counter and enforcer were the same
thing; the moment the counter is shared across many independent enforcers, exactness costs coordination. The rescuing
idea: rate limiting **tolerates bounded approximation** (L21) — "close enough" is the correct spec, not a compromise —
which is what lets the hot path stay network-free.

Four-move spine:
- **Estimate** — price the work of a request (~5 ms) against the cost of coordinating to count it. A central counter
  checked per request adds a round trip: same-region ~0.5 ms (tolerable) but **cross-region ~70 ms** (a 14× tax on a
  5 ms request), and a global fleet always has someone paying the cross-region trip. Second wall: a viral key doing
  ~10k req/s is **one hot counter** — thousands of increments/sec on a single entry hashing can't spread (L3/48). The
  opposite extreme (never coordinate, static split 60÷30 = 2/min per server): a skewed key (client in EU → ~all
  traffic on 10 eu-west servers) caps at 10 × 2 = **20/min = 1/3 of budget** while ~40/min sits idle elsewhere; and
  the "fix" of full-limit-per-server leaks **30 × 60 = 1,800/min (30× too loose)**.
- **Model** — three designs on one axis (how often enforcers talk): central counter (exact, expensive) / static
  division (cheap, wrong under skew) / **local buckets + periodic reconciliation** (the winner). Each server keeps a
  local token bucket admitting with ~0 network cost; every `T`s (≈1 s) it reports usage to a **coordinator** that sums
  the true global count and reallocates **unused budget from idle servers to busy ones** — so a fully-skewed key still
  gets its full 60, the thing static division couldn't do. Coordination cost is **per-active-server-per-interval**
  (~6 msgs/s for a key on 6 servers), **flat in request rate** → cheap exactly for hot keys, lazy for cold keys.
  Safety = bounded approximation (L21). Window mechanics: fixed window can be gamed at the boundary (60 at :59 + 60 at
  :00 = 120 in ~2 s) → **sliding window / bucket refill** (L08). **Hard-quota fork**: rate limits forgive overshoot;
  a hard money/resource cap ("$500/day", "1,000 seats") does not → switch from count-and-reconcile (may overshoot) to
  **pre-authorized leases** — pull budget from the coordinator (sole issuer) BEFORE spending, never over-issues, at
  the cost of latency up front + trapped budget (L40).
- **Trace** — (A) well-behaved key (~1 req/s): served entirely from the local bucket, **ZERO coordination on the hot
  path**; syncing happens in the background off to the side. (B) abusive key A=10 req/s (10× over), caches stale up to
  T=1 s: **worst-case overshoot ≈ A × T = 10 × 1 = 10 requests** → ~70 through then hard-blocked (~17% over for one
  second); halve T→0.5 s → overshoot ≤5, at 2× the sync chatter; T→0 = exact but back to the central-counter tax.
  (C) partition (eu-west loses the coordinator in us-east) = **CAP (L11)** in a rate-limiter costume: **fail-open**
  (keep serving on stale state → leak up to ~N_regions × share) / **fail-closed** (reject → 429 good customers over
  your own outage) / **graceful default = degrade to a static per-region share** (bounded leak, stays available,
  resumes global optimization when healed).
- **First bottleneck** — structural: a global limit is **one logical counter every request contends on**; you can't
  make it exact + cheap + partition-proof at once (the same pick-two shape as last lesson's recall/latency/memory
  triangle). Master dial = reconciliation interval `T`. Second wall: consistent hashing spreads a million keys but
  can't split ONE hot key (L4) → bigger local share + sync less often for that key, and shard the **coordinator** by
  key (L3). Third wall: the window itself (fixed vs sliding, L08) and long-window quotas (more slack → sync lazily and
  still be accurate; but zero overshoot tolerance → leases).

Deepest point: a global limit is a distributed agreement about a single number, so L11's CAP truth applies — but
unlike a bank balance, rate limiting **doesn't need consistency**, so a bounded approximation isn't a compromise,
it's the spec, and that's what keeps the hot path free of the network.

## Reuses (how it threads the course)
L08 (the token bucket + window mechanics it distributes); L21 (approximate counting — "close enough" as the correct
spec); L11 (CAP — the partition choice under a shared counter); L40 (per-tenant quotas + the never-overshoot lease);
L3/4 (the hot key one counter can't shard away; sharding the coordinator by key); L42/27/49 (the fleet + front door
the limit sits in front of); L48 (hot-key echo).

## Interactive quiz
4 questions: (1) why a central counter per request fails for a global fleet (cross-region round-trip tax + hot key;
rate limiting only needs approximate), rejecting "Redis can't count" and "can't express per-key"; (2) static division
under skew throttles the key to 1/3 while idle budget is stranded (and full-per-server leaks 30×), rejecting "nothing
wrong / even" and "client exceeds limit"; (3) overshoot ≈ arrival-rate × T = 10, halving T halves it at 2× chatter,
rejecting "no overshoot, buckets cap at 60" and "unbounded"; (4) partition = CAP, safe default = degrade to static
per-region share, rejecting "any behavior is fine" and "reject-all is the only correct behavior."

## What it sets up next
Next in the queued spine: #67 **data lakes & the lakehouse (open table formats)** — beyond L29's warehouse: raw
Parquet files on object storage (L20) made queryable by an open table format (Iceberg/Delta/Hudi) giving ACID
snapshots, schema evolution (L24), time-travel, compaction, and storage/compute separation. Then #68 delivery
guarantees & the dead-letter lifecycle, #69 hot/warm/cold path & serving-vs-analytics split, #70 idempotent resumable
data APIs. The spine still has 4 queued topics (67-70), so this run did NOT need to add new ones — the course won't
run dry.
