# Lesson 0036 — Cell-Based Architecture

**Published:** 2026-07-23
**File:** `lessons/0036-cell-based-architecture.html`
**Spine topic:** #36 (advanced batch)
**Trade named:** blast-radius isolation vs cross-cell coordination & overhead

## Worked example
A **multi-tenant SaaS** on Lesson 27's fleet: **286 servers**, **40,000 req/s**, **8,000,000
tenants**, today a single shared stack (one LB → one app pool → one shared data tier). The
opening incident: one tenant fires a pathological query (Lesson 12 selectivity trap) that
saturates the shared data tier, so **every** tenant times out — a 100% blast radius. Cut the
whole stack into 8 independent **cells** and the same query is confined to its tenant's cell:
only 1/8 of tenants are hit.

## The four moves
1. **Estimate** — blast radius is the justifying number. Monolith: `8,000,000 tenants × 5 min
   = 40,000,000 tenant-minutes`, **100%**, for *every* single-stack failure. 8 cells of 1M
   tenants (`40k/8 = 5k` req/s, `286/8 ≈ 36` servers each): the same incident = `1M × 5 min =
   5M tenant-minutes` = **1/8 = 12.5%**, 8× less. General law: `N` cells ⇒ blast radius `1/N`
   (want ≤5% ⇒ N ≥ 20). Stronger than L7/L28/L31 because the wall is PHYSICAL — no shared
   queue/pool/row — so it stops *all* failure modes, not one anticipated kind.
2. **Model** — a **cell** = complete self-sufficient stack (own LB + app + data), one hard
   rule: **no request may cross a cell boundary** (or a cross-cell dependency rebuilds the
   monolith). The one un-cellable part = the **router** (`tenant → cell`): must be dumb (cached
   lookup, no per-request DB/logic), HA + replicated, rarely changed — it's the only shared
   fate, a router bug is the one bug with 100% blast radius. Placement: fixed `hash % N`
   (stateless, but L3 resharding pain → L4 consistent hashing) vs **lookup table** (flexible:
   place whales, migrate hot tenants L24, pin by geo L19 — at the cost of a consistent table).
   Cell **sizing FLOOR**: each cell re-pays a minimum footprint (3-node quorum L10 + app + LB +
   headroom L27 ≈ 8 nodes). N=8 → 36/cell (12.5%); N=32 → ~9/cell (3.1%); N=64 → need `64×8 =
   512` nodes ≈ 2× the fleet (1.6%). Near the floor, halving blast radius ~doubles fixed cost.
3. **Trace** — (A) bad deploy: roll cell-by-cell = L31's canary made physical (deploy unit ==
   blast unit; a v2 bug hits one cell ≈ 5k req/s, gated on error-rate/p99 L17, 7/8 untouched
   vs the monolith's `40k × 300 s = 12M` requests). (B) hot tenant / poison query: monolith =
   all 8M down; cells = only that tenant's cell → **but all 1,000,000 innocent cell-mates go
   down too** = the wall (cells isolate the *system* from a bad cell, not cell-mates from each
   other).
4. **First bottleneck → shuffle sharding** — give each tenant a random SUBSET of k cells; a
   noisy neighbour fully knocks out another tenant only if their whole subsets coincide → rides
   `C(N,k)`. 8 cells, k=2: `C(8,2)=28` → fully co-victim prob `1/28 ≈ 3.6%` (vs plain cells'
   `1/8 = 12.5%`, a 3.5× shrink; partial-overlap tenants keep a healthy cell and retry there
   L7/L34). Scale: `C(26,5)=65,780`, `C(100,5)≈75,287,520` → 1-in-75-million (the AWS trick).
   Shines on the stateless/retriable request plane; hard for stateful data (would need the
   tenant's rows in all k cells) → data plane keeps a home cell + explicit migration.

## Traps covered
1. A hidden shared component (shared DB/cache/global lock L22/config service) = **correlated
   failure**, blast radius back to 100% — audit the request path for anything outside the cells.
2. A fat/fragile router (business logic or per-request DB calls) → the bottleneck + SPOF you
   built cells to escape; keep it dumb, HA, deployed separately and most carefully.
3. Cells sized wrong: too big = no real isolation; too small = re-paid floor × N + router-table
   bloat + N× operational toil. Size to the largest failure you can absorb.
4. A GLOBAL operation that skips the cell boundary (fleet-wide config push, shared migration
   L24, deploy-all-at-once) = 100% blast radius no matter how many cells — roll *everything*
   (code, config, data) cell by cell.

## Builds on
L07 (bulkhead — same instinct, one layer; cell generalizes it), L28/L31 (bounded queues,
canary — isolation + the deploy unit the cell becomes), L27 (fleet sizing → cell sizing per
cell), L10 (quorum = the per-cell floor), L03/L04 (sharding + consistent-hashing routing/
placement), L24 (moving a hot tenant between cells), L34 (retry-to-another-instance = how a
shuffle-shard client survives a degraded cell), L17 (per-cell error-rate/p99 gate), L12 (the
poison-query selectivity trap).

## Sets up next
**Lesson 0037 — Read/write splitting & CQRS** (spine #37): L33's CDC/outbox pipeline and L15's
precomputed feeds already produced separate **read models** beside the write model — formalize
it. One write side optimized for correctness/consistency (L06/25) feeds, via events (L33), many
purpose-built read sides (search L16, cache L02, denormalized views L15/29); commands vs queries
as two shapes of the same data with independent scaling. When the split earns its keep vs when
it's accidental complexity. Trade: read/write optimization vs the eventual-consistency gap
between the two sides.
