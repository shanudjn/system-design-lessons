# Learning record — Lesson 0013: Idempotency

**Topic:** Making retried and replayed writes safe — exactly-once *effect* on
at-least-once *delivery*.

**Worked example:** A payments API `POST /charge $50` at 10M charges/day. A
timed-out request is retried (Lesson 07) and the card is charged twice — $50 →
$100. The lesson builds the idempotency key + dedup store that makes the second
attempt a safe no-op.

## What it covered (four-move spine)
- **Estimate:** the bug in dollars — 10M/day × 2% duplicate rate = 200,000
  double-charges/day × $50 = **$10M/day** unprotected. Little's Law (L = λ·W =
  116/s × 0.8 s ≈ **93** charges in flight at once) shows retries routinely race
  a still-running original. Named the core impossibility: **exactly-once
  delivery over a network is impossible**, so you buy an exactly-once *effect*
  on an at-least-once *wire*.
- **Model:** client picks a unique **idempotency key** (UUID) and reuses it on
  every retry; server keeps a durable **dedup store** (`key → status, saved
  response`), checked *before* hitting the card network. Sized it: ~1 KB/row ×
  10M/day ≈ 10 GB/day at a 24 h TTL.
- **Trace the paths:** (A) first attempt — claim key, charge, save COMPLETED;
  (B) retry after success — replay saved response, no second charge; (C)
  concurrent retry — the **check-then-act gap** (L06 lost-update race) closed by
  an **atomic claim** (unique-constraint INSERT → one winner, loser gets 409);
  (D) crash between charging and recording — fixed by committing effect + record
  in **one transaction**, or pushing the key **downstream** so the provider
  dedupes.
- **Next bottleneck:** the **TTL trap** (too short → a late queue redelivery or
  human re-submit double-charges; too long → storage bloat — size to the slowest
  realistic retry) and **natural idempotency** (absolute `PUT`/delete/
  conditional writes need no key; relative deltas and creates do).

## Trade-offs named
- Exactly-once effect = at-least-once delivery + idempotent processing (can't fix
  it on the wire).
- Dedup store = correctness bought with a hot-path lookup + a store you must
  never lose.
- Atomic claim closes check-then-act, exactly like L06's atomic increment / L10's
  single leader.
- Effect + record must commit together; when you can't share a transaction, every
  downstream hop must dedupe.
- TTL: short risks correctness, long costs storage.
- Natural idempotency is free but partial (absolute writes only); the key is
  universal but costs a store + lookup + TTL.

## Callbacks used
L05/L07/L09 (retries & queues replay writes), L06 (lost-update race → atomic
claim), L02 (durable-store shape, storage-for-safety), L10 ("exactly one").

## Sets up next
- **Lesson 14 — CDN design:** edge caching, cache keys, invalidation ("do the
  work before the query").
- **Lesson 15 — feed/notification fan-out:** at-least-once delivery to millions,
  so every delivered effect must be idempotent (this lesson's machinery, reused).
Added topics 16 (search systems) and 17 (distributed tracing & observability) to
the spine so the course never runs dry.
