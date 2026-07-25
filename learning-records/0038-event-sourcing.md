# Learning record — Lesson 0038: Event Sourcing

**Topic:** Event sourcing — when the log of changes is the source of truth.
**File:** `lessons/0038-event-sourcing.html`
**Curriculum spine #:** 38 (first of the "next batch" advanced topics; 39–41 still queued).

## The one worked system
Lesson 25's **digital wallet**, re-built event-sourced. Instead of a mutable
`balance` row (or ledger entries + a derived balance), store the ordered,
append-only stream of **events** as the sole truth and compute state by folding.

## What it covered (the four moves)
- **Estimate — two storage walls.**
  - Log growth: 5,000 events/s × 300 B = 1.5 MB/s = ~130 GB/day ≈ **47 TB/yr**,
    and it *never shrinks* (immutable append). Vs a state DB's ~flat 100 GB.
  - Fold cost grows with one aggregate's history: hot account at **100M events**,
    fold @ ~5M/s = **20 s per read**. (State DB answered in 0.4 ms.)
- **Model — four parts.** Immutable past-tense **events** (corrected by appending
  a compensating event, never editing); the **aggregate** as fold + invariant
  boundary; the **fold/apply** (`state = events.reduce(apply, empty)`, walked
  $0→$100→$70→$120→$100); the **guarded append** (expected-version = L06
  compare-and-set → enforces `balance ≥ 0` with no lock; two concurrent
  withdrawals can't both land at v6).
- **Trace.** Command = fold-to-decide → append-if-version-matches. Query = read a
  **projection** (L37), because you can't SELECT over a log → event sourcing
  *needs* CQRS. Replay = build any new read model, or time-travel to a past date,
  free because every fact was kept (L09/29/33 kappa).
- **First bottleneck — folding a long-lived aggregate.** Fix = **snapshot** =
  Lesson 25's snapshot generalized to "a cached fold": snapshot every 1,000
  events → fold ≤ 1,000 ≈ 0.2 ms = **~100,000× cut**. Safe because the snapshot
  is derived and rebuildable, never the truth.

## Trade-offs named
- Storing events vs state: **total history (audit, time-travel, replay) vs
  unbounded storage + no direct current-state query.**
- Snapshot: **derivation speed vs a cache that can go stale** (rebuild from log).
- Aggregate size: **invariant coverage vs fold cost & write contention** (big =
  L06/L22 serial gate; cross-aggregate consistency is a saga L32).
- Events forever-immutable: **honest audit vs "can't fix, only append + upcast."**

## Key distinction reinforced
Event sourcing ≠ CQRS. ES decides **what the truth is** (the log); CQRS (L37)
decides **how you read it** (projected shapes). Independent patterns, but they
pair naturally — an event-sourced write side emits exactly the stream CQRS's
projections consume. Also the inversion vs L33: outbox = state is truth, events
derived; event sourcing = events are truth, state derived ("outbox becomes the DB").

## Four traps
1. Mutate/delete an event (rewrites history, corrupts every fold/projection).
2. Fold the whole log per read (O(history) → 20 s) instead of snapshot + project.
3. Aggregate drawn too big (one version counter serializes all writes, L06/L22).
4. Events coupled to current code (won't deserialize after refactor → version + upcast).

## What it sets up next
**Lesson 0039 — Tiered storage & data lifecycle.** The event log (this lesson),
object pile (L20), and warehouse (L29) all grow forever, but old data is read far
less often (L02 skew). Move data hot→warm→cold→archive by age/access frequency,
with TTL/retention and the cold-tier retrieval-latency cliff. Trade shifts to
**storage cost vs retrieval latency & operational policy.**
