# Lesson 0023 — Multi-Region Active-Active

**File:** `lessons/0023-multi-region-active-active.html`
**Spine topic:** #23 (Multi-region / active-active)
**Built directly on:** Lesson 22's teaser — put writers on different continents with no lock to serialize them.

## The one system
A global e-commerce shopping cart served live from two regions — `us-east` (Virginia)
and `eu-central` (Frankfurt), both accepting local writes (active-active).

## What it covered (four moves)
- **Estimate** — the cross-region latency tax and the availability upside:
  - Virginia↔Frankfurt ≈ 6,400 km; light in fiber ≈ 204,000 km/s → 31 ms one-way,
    ~64 ms straight-line RTT, ~90 ms measured (L02/L14 speed-of-light floor).
  - Synchronous cross-region write ≈ 91 ms vs ~1 ms local → **~90× tax** per write.
  - Two independent 99.9% regions in active-active: down only if both down =
    10⁻⁶ → **six nines, ~32 s/yr** vs 8.76 h/yr single region (L07 redundancy).
- **Model** — write locally + replicate async (L11 AP: available, tolerate divergence);
  two conflict-resolution families:
  - **LWW** — one timestamp compare, silently DROPS the loser, trusts a skewable
    wall clock (L22 clock skew).
  - **CRDT** — merge is commutative/associative/idempotent → converges with zero
    coordination. Set (union), PN-counter (sum per-region sub-counts), register (LWW).
  - **Version vector** — tells *concurrent* from *after* without wall clocks (L06/L10).
- **Trace** — four paths:
  1. add/add → clean UNION, nothing lost.
  2. add/remove same item → genuine conflict; resolution is a business POLICY
     (add-wins for a cart), CRDT only guarantees replicas pick the *same* one.
  3. LWW under 50 ms clock skew → the newer write is silently dropped.
  4. counter: PN-counter keeps both concurrent +1s (→ 3); a register loses one
     (L06 lost update, now across an ocean).
- **First bottleneck** — the un-mergeable **global invariant**: $100 balance withdrawn
  $80 (VA) + $70 (FR) concurrently CONVERGES to −$50. The CRDT agreed perfectly and
  still broke `balance ≥ 0`, because merging preserves **convergence, not invariants**
  (L11's scalar-balance dilemma, named precisely). Fixes: **single-home** the key
  (L03/04/10/22), **synchronous consensus** (L10/11 CP, blocks on partition), or
  **escrow** (split the budget per region — L06/08 shard-the-counter on an invariant).

## Trade-off named
**Local writes vs global order.** Sort every piece of state into "mergeable"
(CRDT, write anywhere, coordinate never) or "globally ordered" (balance/invariant —
pay to coordinate), and keep the expensive bucket as small as possible.

## Recaps leaned on
L02/L14 speed-of-light, L06 lost update + versions, L07 redundancy, L09 replication
log, L10 leader/epoch, L11 CAP/AP-vs-CP + scalar balance, L15/L20 tombstones,
L21 mergeable sketches (CRDTs in disguise), L22 clock skew + single-writer escape hatch.

## Sets up next
**Lesson 24 — Schema changes & migrations at scale.** We kept two continents' *data*
in agreement; next change its *shape* — add/rename a column on a live 100M-row table
with zero downtime via the expand/contract dance (dual-write → backfill → cutover →
drop). Trade shifts to **migration safety vs deploy velocity**.
