# Learning record — Lesson 0034: Service Discovery & Health Checking

**File:** `lessons/0034-service-discovery.html`
**Topic (curriculum spine #34):** Service discovery & health checking
**Trade named:** routing freshness vs lookup cost & staleness

## The one system
One **caller** (the order service) trying to reach one **callee** (the search service —
**200 instances**, **40,000 req/s** → **200 req/s each**) across L27's autoscaling,
L31-deployed fleet. The unexamined assumption of every prior lesson ("a caller calls the
search service") breaks here: in an elastic fleet the callee isn't at *an* address, it's at
200 addresses that change every few seconds. So the real question is not "what's search's
address?" but "who is alive and **ready right now**?" — and the answer keeps changing.

## What the lesson covers (four moves)
- **Estimate — how fast a hardcoded list rots and what it costs.** Churn: a rolling deploy
  (L31) replacing all 200 instances over 20 min = `200/1200s ≈ one address change every 6 s`,
  before autoscaling and crashes are even added. Cost of one stale entry: a caller refreshing
  every 60 s that holds a just-crashed address keeps sending it `200 req/s × 60 s = 12,000`
  black-holed requests before the next refresh — each a timeout (L07). Shrink the window to a
  9 s registry TTL → `200 × 9 = 1,800`; add active ejection on connection-refused → ~0. Stale
  in BOTH directions: dead addresses linger; new healthy capacity is invisible until redeploy.
- **Model — the registry + the two health questions.** Registry = `service name → {address, ttl,
  ready}` set. Three verbs: **register on boot**, **heartbeat every 3 s to renew a 9 s TTL lease**,
  **deregister on graceful exit**; a crash just stops heartbeating → the entry **expires** (death
  detected in ≤ 3 missed beats — L10's failover / L22's lease / L26's connection registry, all the
  same lease pattern). TTL is a two-sided dial: too long → dead lingers; too short (≤1 missed beat)
  → a GC pause / network blip false-evicts a live instance → **flap** (L10). The outage-preventing
  distinction: **liveness** ("is the process alive?" → fail = *restart it*) vs **readiness**
  ("should it take traffic now?" → fail = *remove from pool, do NOT restart*; covers warming-up and
  L31 draining). Three lookup homes: **DNS** (universal, zero client code; stale caches, no
  health/load, clients ignore TTLs), **client-side** (freshest + smart LB; discovery logic in every
  language), **sidecar/mesh** (fresh + language-agnostic + uniform; a proxy per instance + a latency
  hop).
- **Trace — boot / crash / graceful shutdown.** A: instance boots cold → fails readiness while
  warming → registers + heartbeats only once READY → appears in the pool → takes its 200 req/s
  (capacity appears when it can SERVE, not when it BOOTS). B: instance crashes → callers with cached
  lists black-hole some requests → each fails fast → caller **retries another instance** (L07
  backoff+jitter, safe because indexing is idempotent, L13) → at ≤9 s the TTL expires → new lookups
  never see it. C: graceful shutdown (L31) → **deregister + fail readiness FIRST** → drain in-flight
  → then exit → zero requests cut off. B vs C differ only in *ordering*: a crash dies before leaving
  the pool; a graceful shutdown leaves the pool before dying. Readiness is the healthy instance's
  off-ramp.
- **First bottleneck — the registry is everyone's single dependency.** (1) Must **fail-static**:
  callers cache the last known good list and keep routing if the registry is unreachable — a
  deliberate **AP** choice (L11), the OPPOSITE of L10's CP election, because a stale route is cheap
  to be wrong about (retry another) where a stale leader is split-brain. (2) The registry itself must
  survive failure → a replicated **consensus** cluster (etcd/Consul/ZK, Raft/Paxos — L10 is the layer
  underneath). (3) Heartbeat load scales platform-wide, not per service: `10,000 instances / 3 s ≈
  3,333 renews/s` → push work to gossip / local agents (L26's off-node truth). (4) Propagation lag
  never fully closes → bound it (short cache TTL) and survive it (L07 retry). Four traps: conflate
  liveness/readiness (crash-loop a warming box, or traffic to a cold one); a health check probing a
  **shared** dependency (one DB blip fails ALL instances → registry ejects the whole fleet =
  correlated total outage, L07/26); a strongly-consistent registry on the hot path (a partition
  blocks all routing — L11's CP cost in the worst place); a mistuned TTL.

## Threads reused
- **L27** — the elastic autoscaling fleet that makes membership churn every few seconds.
- **L31** — rolling deploy (the churn) + draining; readiness as the graceful off-ramp.
- **L26 / L10 / L22** — heartbeat-renewed TTL lease; the consensus/coordination store underneath.
- **L07** — retry/backoff to another instance covers the detection window; correlated-failure trap.
- **L11** — CAP: discovery chooses AP/fail-static where leader election chose CP.
- **L02 / L14** — the freshness-vs-cost cache dial reappears as DNS staleness & propagation lag.
- **L13** — idempotent retries make routing-around-a-death safe.

## What it sets up next
**Lesson 0035 — Logical clocks & causal ordering.** The TTL lease, L22's lease, and L23's
last-write-wins all trusted a wall clock, but clocks on different machines disagree (L23 skew), so
"which event happened first?" has no trustworthy global answer. Build ordering with no synchronized
clock: **Lamport** timestamps (total order respecting causality, can't tell concurrent from causal),
**vector clocks** (can, at O(N) size — L23 recap), **hybrid logical clocks** (physical time bolted on
so a timestamp is both causal and ~wall-clock). Trade shifts to **ordering precision vs metadata size
& coordination.**

Spine status: #34 done; #35–36 (logical clocks & causal ordering, cell-based architecture) and the
#37–41 batch (read/write splitting & CQRS, event sourcing, tiered storage & data lifecycle,
multi-tenancy & noisy neighbors, graph & relationship systems) remain queued — the course stays well
ahead of running dry.
