# Learning record — Lesson 0055: Edge Computing & Compute-at-the-Edge

**Topic:** Pushing not just cached bytes but *code* to the edge — running logic
milliseconds from the user, and the cold-start and consistency walls that decide
whether it helps.
**Worked example:** A global app on 300 edge PoPs (L14/L50) fronting a single
Virginia origin at 40k req/s. Move auth, routing, personalization, and filtering
onto the PoPs; keep the authoritative database at the origin.

## What this lesson covered
- **Estimate — the prize and the trap.**
  - Round trip won: Virginia↔Sydney ~15,500 km ÷ ~200,000 km/s (⅔ c) = 77.5 ms
    one way → ~155 ms floor, ~200 ms in practice (L14). Edge-answered request
    ~15 ms vs ~215 ms home → the ~200 ms gap is both the prize and the cold-start
    budget.
  - Cold-start budget: **isolate ~5 ms « 200 ms saved → win even cold; container
    ~250 ms > 200 ms saved → a cold request is SLOWER than calling the origin.**
    Same code, opposite outcome, decided by the isolation mechanism.
  - Why functions go cold at the edge (counter-intuitive core): origin runs one
    function at 40k/s → permanently warm. Split across 300 PoPs = 133/s each;
    split across 1,000 tenant functions = **0.13 req/s = 1 request per ~7.5 s**.
    With a 5 s warm window the function is evicted between requests → cold is the
    COMMON case. Being near the user (300 places) divides traffic 300 ways, which
    is exactly what makes each place cold. Locality vs warmth.
- **Model.**
  - Edge function = per-request, tight CPU (tens of ms), small memory, no durable
    local disk — designed to be started cheaply and thrown away constantly.
  - **Isolate vs container** decides feasibility: isolate ~5 ms / ~3 MB, thousands
    per shared process (weaker isolation, leans on the language sandbox); container
    ~250 ms / ~150 MB, ~50× fewer per node and every cold request blows the budget.
    Edge compute is only affordable because isolates make cold ~5 ms and resident
    ~3 MB.
  - **One rule for state — edge runs the DECISION, origin owns the TRUTH.** Three
    flavors chosen by *how stale the data may be*: (1) no shared state (JWT verify
    w/ cached key, signed cookie, URL rewrite); (2) eventually-consistent edge KV
    (flags/catalog, L23/L48, seconds stale OK); (3) forward to origin for anything
    authoritative + strongly consistent (order/money/balance/inventory, L11/23/25/32).
  - Four properties: near-zero cold start (isolates), bounded execution (small/short),
    local-or-eventual state, graceful fallback (forward home).
- **Trace.** (A) fully-edge-answered request ~15 ms, origin untouched (~14× faster);
  (B) cold start WINS on isolate (~15 ms) / LOSES on container (~260 ms > 215 ms)
  from identical code — cold start is the design center, not a corner case;
  (C) "Buy" — edge does TLS/auth/bot-filter/rate-limit near the user then FORWARDS
  the authoritative write home (contrast: catalog read → edge KV ~1 ms; live
  balance → forward, because staleness = wrong money).
- **First bottleneck — state (consistency can't spread).** Cold start is *solved*
  by isolates; the wall is that a strongly-consistent DB can't exist in 300 places
  (L11/L23) — replicating inventory to 300 PoPs re-imports global coordination on
  the hot path (L23 ~90 ms+/hop) ×300 or an oversell bug. The dial is "how stale
  can this be?" (same as L02 TTL / L14 freshness / L23 local-writes-vs-global-order).
- **Second wall — operational reach.** Code now runs in 300 places → 300-PoP blast
  radius → canary + health-gated promotion (L31/L36), cross-PoP observability (L17),
  and thin-edge discipline to avoid the **thick edge** (a distributed monolith
  copied 300 times, coupled to origin schemas). Cold-start long tail → keep hot
  functions warm (L27 warm-pool shrunk). Edge KV writes are eventually consistent
  (not a lock — L22 lives at the origin).

## Trade named
Latency & offload vs consistency & operational reach. Latency is cheap to spread
across 300 places; consistency is not. Move the logic outward, keep the source of
truth home.

## Reuses
L14 push-to-edge + speed-of-light round trip, L50 steer-to-nearest-PoP, L11/L23
eventual consistency & no-strong-consistency-across-many-replicas, L48 cache/KV
replication, L27 utilization/warm-pool dial, L13/25/32 exactly-once authoritative
writes, L31/L36 canary + blast-radius control, L17 cross-PoP observability, L08
edge rate-limiting.

## Sets up next
**Lesson 0056 — Data privacy, deletion & the right to be forgotten:** when a user
says "delete me," the data lives in an immutable event log (L38), backups, caches
(L48) and edge KVs (this lesson), warehouses (L29), and derived read models (L37).
Trace one deletion across that sprawl; crypto-shredding (L30) vs true deletion;
audit trails. Trade: compliance & privacy vs immutability & derived-data sprawl.
