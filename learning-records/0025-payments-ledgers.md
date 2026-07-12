# Lesson 0025 — Payments & Ledgers

**File:** `lessons/0025-payments-ledgers.html`
**Spine topic:** #25 (Payments & ledgers)
**Built directly on:** Lesson 24's teaser — we made the `orders` table safe to
*reshape*; next make the *money* in it trustworthy.

## The one system
A digital wallet, ~10M transfers/day. One concrete op traced all lesson: **Alice
pays Bob $50.00 (5000 cents)** — so it can never be lost, doubled, or un-audited.
All money in **integer cents**, never floats.

## What it covered (four moves)
- **Estimate**
  - Growth: 2 entries/transfer → 20M entries/day → **~7.3B rows/yr ≈ 730 GB** (cheap;
    correctness is the problem, not storage).
  - Cost of error: 2% retries × 10M/day × $50 = **$10M/day** wrongly moved — in
    payments that's a forensic audit of wrong *balances*, not a refundable overcharge.
- **Model** — the **double-entry ledger**, three tables:
  - `accounts` (users + **system accounts** like `bank`, `fees`),
    `transactions` (one per request, carries `idem_key` + status, **UNIQUE(idem_key)**),
    `entries` (append-only, immutable, ≥2 per txn).
  - Rule 1 atomicity: all entries of a txn commit together or none.
    Rule 2 balance: entries of one txn **sum to zero** → whole ledger always totals 0
    (money conserved; outside world = `bank` system account, top-up = `bank −X, user +X`).
  - Balance is **derived** `SUM(amount)`, not stored. Fee example: 3 entries still sum 0.
- **Trace** — three paths:
  - **A happy transfer:** two entries in **one atomic DB commit** (partial commit =
    $50 vanishes, ledger ≠ 0 forever). Optional `balance ≥ 0` guard inside same commit
    (L23 un-mergeable invariant).
  - **B duplicate retry:** same `idem_key` → **UNIQUE violation before any entry
    written** → return original result. Exactly-once *effect* (L13), not delivery (L09).
  - **C refund:** append a **compensating transaction** (reverse the signs), never
    delete → history stays complete → "sum every entry = 0" reconciliation stays valid.
- **First bottleneck** — hot account balance = `SUM` of 100M entries → 1.6 GB index
  scan @ 500 MB/s ≈ **3.2 s/read**. Fix: **snapshot** running balance
  (`snapshot + SUM(since)`, 1,000 rows not 100M → **100,000× cut**) + **shard the
  account** into `bank#0..7` (L21 mergeable counter) for write throughput. Both kept
  strictly as **derived** accelerators — a corrupted snapshot is a stale cache you
  recompute from entries; the rejected mutable `balance` column was the *only* copy of
  the truth.

## Trade-off named
**Correctness & auditability vs throughput.** The mutable balance column is the fastest
design and loses on all three requirements. The ledger spends throughput/storage (two
immutable entries, derived balances, nothing edited) to buy rebuildable truth, safe
retries, and a provable audit trail — then buys throughput back with snapshots/sharding
that never get to *be* the truth.

## Recaps leaned on
L06 lost update & hot row (why not a mutable balance), L07 timeout = partial failure
(why retries happen), L09 no exactly-once on the wire (→ exactly-once effect), L12
derived index/snapshot shape, L13 idempotency key (now for dollars), L21 mergeable
sharded counter (hot account), L23 un-mergeable invariant (`balance ≥ 0`).

## Sets up next
**Lesson 26 — Real-time delivery (WebSocket/push at scale).** Ledger settled the money;
now tell millions of live phones the instant their balance changed — millions of open
connections, presence, sticky routing, and the thundering reconnect (L02/L07). Trade
shifts to **statefulness vs elastic scale**.

## Housekeeping
Spine already had #26 plus advanced batch #27–#31 queued (from L24), so no new topics
needed this run. Marked #25 ✅.
