# Lesson 0024 — Schema Changes & Migrations at Scale

**File:** `lessons/0024-schema-migrations.html`
**Spine topic:** #24 (Schema changes & migrations at scale)
**Built directly on:** Lesson 23's teaser — we kept two continents' *data* in
agreement; next change its *shape*.

## The one system
A payments service's `orders` table: 100M rows, 50k reads/sec, 5k writes/sec.
Split the ambiguous `total` (int cents, assumed USD) into `amount_cents` +
`currency`, live, with zero downtime.

## What it covered (four moves)
- **Estimate** — why the naive path is an outage:
  - Table-rewriting `ALTER` on 20 GB @ 100 MB/s → **~200 s** lock; at 5k writes/sec
    that queues **~1,000,000** writes (L07 cascade). Plus the code/schema gap: a fleet
    deploy is gradual, so the instant the column renames, every old server's
    `SELECT total` errors — **a schema deploy and a code deploy are never simultaneous**.
  - Backfilling 100M rows @ 10k rows/sec (deliberately gentle) = **~2.8 h** window where
    old rows have the value and older ones don't.
- **Model** — the **expand/contract** dance (parallel change), 5 reversible steps:
  1. EXPAND: add nullable `amount_cents`, `currency` (backward-compat, sub-second lock).
  2. DUAL-WRITE: app writes both shapes (code deploy).
  3. BACKFILL: batched background job fills old rows (~2.8 h).
  4. CUTOVER: readers switch to new columns (plain code deploy — instant, reversible).
  5. CONTRACT: stop dual-write → bake → `DROP COLUMN total`.
  Invariant protected: at every step old code AND new code both work.
- **Trace** — read+write at 4 stages (A: added-but-unused, B: dual-write+backfill,
  C: post-cutover with old col warm, D: contract). Pinned the two ordering rules:
  - **Dual-write ON before backfill** (else a gap-write leaves new col stale and the
    NULL-only backfill skips it).
  - **Cut readers over only after verifying zero-NULL**; keep old col warm as instant
    rollback (cutover is a code deploy, not a schema change).
- **First bottleneck** — the backfill vs the live table: one 100M-row `UPDATE` locks
  the table (L22) and rolls back all-or-nothing → **batch** into 20k committed chunks
  (LIMIT 5k), **throttle on replica lag** (L06) so stale reads don't spread, page by
  **cursor not offset** (L18) so batch 20,000 doesn't re-scan 100M rows. DROP is the
  one irreversible step → bake first.

## Trade-off named
**Migration safety vs deploy velocity.** The fastest change is one ALTER + one deploy
(locks a hot table, opens a code/schema gap). Expand/contract spends velocity — extra
column, dual-write, throttled backfill, verify-before-flip, bake-before-drop — to buy
zero downtime and a working rollback at every step. Skip the dance on a table nobody
depends on; on a 100M-row hot table the dance *is* the design.

## Recaps leaned on
L06 replication lag & read-your-writes (backfill throttle), L07 cascading timeouts
(locked-table failure mode), L12 write amplification (dual-write cost), L18 cursor vs
offset (batch paging), L22 lock (what a long ALTER holds).

## Sets up next
**Lesson 25 — Payments & ledgers.** We made the `orders` table safe to *reshape*; next
make the *money* in it trustworthy — double-entry immutable append (never UPDATE),
exactly-once money movement (L13 idempotency, now load-bearing for dollars), and
audit-able reconciliation. Trade shifts to **correctness/auditability vs throughput**.

## Housekeeping
Spine had only #25, #26 left → added advanced topics #27–#31 (autoscaling & capacity
planning, backpressure end-to-end, data warehousing/OLAP, secrets & encryption,
progressive rollout) so the course never runs dry.
