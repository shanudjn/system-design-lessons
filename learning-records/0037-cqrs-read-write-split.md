# Learning record — Lesson 0037: Read/Write Splitting & CQRS

**File:** `lessons/0037-cqrs-read-write-split.html`
**Spine topic:** #37 (first of the "next batch" advanced list)
**Date authored:** 2026-07-24

## What this lesson covers

Formalizes the "second copy of the data beside the first" instinct that Lessons
2 (cache), 15 (precomputed feed), 16 (search index), 29 (warehouse), and 33
(outbox/CDC stream) all reached for without naming it. **CQRS** — Command Query
Responsibility Segregation — keeps one **write model** (the truth) and many
**read models** (derived, purpose-built), connected by a **projection** off the
change stream.

Worked example: the **orders** service (100M orders, 5,000 writes/s, 50,000
reads/s → 10:1).

### The four moves
1. **Estimate** — the driver is SHAPE, not just volume. The detail page is a
   6-way join ≈ six L12 index seeks ≈ `2.4 ms` = **48 cores** at 20k/s; a
   denormalized read-model document does it in one `0.4 ms` lookup = **8 cores**
   (**6×** cut). But the denormalization that makes the read cheap (product name
   copied into every order doc) is what makes a write dangerous (a name change
   fans out to every doc, L15/24 → L06 lost-update). Normalized wins writes,
   denormalized wins reads, no one table wins both. And there are four reads —
   detail (doc), search (inverted index L16), dashboard (columnar L29), status
   (cache L02) — each wanting a different shape.
2. **Model** — write model (normalized, ACID, correctness-first, L06/25) + read
   models (one per read shape) + the **projection** consuming L33's durable
   outbox/CDC stream **idempotently** (L13). Distinguished from a **read replica**
   (L06): a replica is a byte-for-byte copy → scales VOLUME with the SAME shape,
   can never become an index or warehouse. CQRS changes the SHAPE.
3. **Trace** — a command (commit order + outbox event in ONE transaction, ack
   from the write model, project async → each read model catches up after its own
   delay) and four queries (each on its own store; a dashboard scan can't starve
   checkout — L29/L36 isolation). Then Path C: read-your-writes exposes the gap.
4. **First bottleneck** — the **eventual-consistency gap**: the write is acked
   before any read model reflects it (tens–hundreds of ms, seconds under L06/L09
   lag), so read-your-writes looks like data loss. Fix selectively: serve the
   user's own fresh data from the write side / return it in the command result /
   wait-for-version (monotonic reads) / accept lag where harmless. NOT by making
   projections synchronous (re-couples, can't span stores atomically — L32).

### The judgment call (per spine)
The same gap marks where CQRS is **accidental complexity**: if reads share the
write's shape and fit one DB, you've bought two models + a pipeline + a gap for
no read benefit. The ladder: one model → read replica (volume) → materialized
view/cache (one derived shape) → full CQRS (divergent shapes). Climb only as far
as the reads force you.

### Four traps
1. Dual-writing to each store instead of projecting (L33's trap → diverge on crash).
2. Non-idempotent projection (double-apply on at-least-once redelivery, L13).
3. Trusting a read model as the truth (stale → L06/32 lost update; validate
   invariants on the write model, rebuild read models by replaying the log L09/29).
4. Hiding the eventual-consistency gap from the UX.

**Trade named:** read/write optimization vs the eventual-consistency gap between
the two sides.

## Reuse / callbacks
L2 (cache = first read model), L6 (replicas, lost updates, normalized writes),
L9 (at-least-once, consumer lag, replay), L13 (idempotent projections), L15
(precompute the read, read-your-writes trick), L16 (inverted index read model),
L25/32 (invariants live on the write model), L29 (columnar read model, "normal
form is a bet"), L33 (outbox/CDC as the projection stream), L36 (read/write
stores that can't starve each other).

## What it sets up next
**Lesson 0038 — Event sourcing.** Today the write model stored current *state*
and emitted events as a side effect. Event sourcing inverts that: the **stream of
events IS the truth**, and current state is a fold over it (L25's ledger
generalized). History is free, any read model is a replay away (this lesson's
"rebuild by replay"), audit is total — paid for in snapshotting, event-schema
evolution, and the "query current state" problem that CQRS answers. CQRS and
event sourcing pair naturally: read models become projections of the event log.
Trade shifts to: perfect auditability/replayability vs query complexity & storage.
