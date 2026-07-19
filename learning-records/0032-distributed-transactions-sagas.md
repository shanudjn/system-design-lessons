# Learning record — Lesson 0032: Distributed Transactions & Sagas

**File:** `lessons/0032-distributed-transactions-sagas.html`
**Topic (curriculum spine #32):** Distributed transactions & sagas
**Trade named:** atomicity vs availability across service boundaries

## The one system
A travel platform booking **2,000 trips/sec**. One "trip" = four steps across four
independent, DB-per-service boundaries: reserve **flight ($400)**, **hotel ($600)**,
**car ($150)**, **charge card ($1,150)** — required to be **all-or-nothing**. Core premise:
a database transaction (ACID) is a **single-database** feature enforced by one lock manager +
one log; across four databases there is no shared lock manager or log, so atomicity must be
**built**, not borrowed. The lesson is two ways to build it and what each gives up.

## What the lesson covers (four moves)
- **Estimate — the cost of a cross-service lock (= 2PC).** A 2PC participant locks at *prepare*
  and can't release until the coordinator's *commit* arrives — a full extra round trip away.
  2PC window ~50 ms vs a ~5 ms local commit → contended rows held **10×** longer (a hot flight:
  ~200 → ~20 bookings/sec, L22 serial gate). Commit availability = the **product** of all
  participants: `0.999⁴ ≈ 99.6%` ≈ **35 h/yr** unbookable even when each service is "three nines"
  (three participants already `0.999³ ≈ 99.7%`). Darker failure: coordinator crashes after all
  vote YES but before "commit" → participants **in-doubt**, frozen holding locks until recovery
  (~5 min TTR = hot rows frozen 5 min from one crash). CP = consistency by blocking (L11).
- **Model — 2PC vs saga.** 2PC (coordinator; prepare = lock+vote, commit = all-or-abort) gives
  true cross-service atomicity + isolation but pays the 10× tax + availability product + in-doubt
  freeze; right only inside a small, trusted, low-contention boundary (XA on 2 DBs you own).
  **Saga** = ordered list of **local** transactions each paired with a **compensating
  transaction**; each step commits immediately (no lock held across the network), on failure
  walk backward compensating. Ordered by undo-difficulty: pre-**pivot** steps compensatable
  (cancel a HELD hold), post-pivot retriable → the **charge is the pivot, placed last**, so you
  never un-charge. Driver: **orchestration** (central state machine — clear/debuggable, holds
  saga state but NO participant locks, so its crash freezes nobody) vs **choreography** (L09
  events, no central piece but flow smeared across services, cycle risk). Trade for the driver:
  central clarity vs distributed coupling.
- **Trace — happy, sold-out car, declined card.** Path A: four standalone local commits, each
  lock freed in ~5 ms (vs 2PC holding all four ~50 ms) — that IS the availability/throughput win.
  Path B (car sold out): compensate hotel + flight backward (release holds); charge never made
  (it's step 4). Path C (card declined / why charge is last): pre-pivot failure cancels holds;
  had charge been FIRST, a later failure forces a real refund (visible debit + refund), not a
  clean undo → ordering by undo-difficulty is a core design decision.
- **First bottleneck — a saga has no isolation.** Immediate local commits mean half-done state
  is VISIBLE (the "I" of ACID is gone). Anomaly 1: **dirty read** — saga Y sees saga X's
  soon-to-be-cancelled hold as SOLD and turns a customer away. Anomaly 2: **lost update** — two
  sagas both grab the last seat, double-book (L06, no single DB could stop it). Fix is NOT to
  re-add the cross-service lock (that's what we fled) but to make intermediate state honest:
  **semantic lock** (mark HELD/PENDING not SOLD), commutative updates (L06/L21), revalidate at
  the pivot (L06 CAS). Steps + compensations must be **idempotent** (L13; messages are
  at-least-once, L09) and **retriable** (a compensation that fails is retried till it succeeds).
  Four traps: (1) 2PC across hot/autonomous services; (2) compensation-as-rollback (business
  undo, refund≠un-charge, can't un-send email → un-undoable steps go after the pivot); (3)
  non-idempotent steps double-charge; (4) ignoring the visible isolation window. The saga makes
  the trip *eventually* atomic (L11 AP; a compensation is the reconciliation AP requires).

## Threads reused
- **L31** — shipped one service safely → now many services commit together (this lesson's framing).
- **L11** — CAP: 2PC = CP (block for consistency), saga = AP (available, reconcile after).
- **L10** — coordinator failure; 2PC's in-doubt freeze is the darker version (holds everyone's locks).
- **L22** — serial-gate lock throughput (the 10× hot-row tax).
- **L06** — lost update + compare-and-set (the double-book anomaly + revalidate-at-pivot).
- **L09** — at-least-once event delivery (why steps/compensations must be idempotent) + choreography.
- **L13** — idempotency keys for safe retries of steps and compensations.
- **L25** — compensating transactions as the "append a reversal, never delete" shape.

## What it sets up next
**Lesson 0033 — Change data capture & the outbox pattern.** The saga steps had to "commit
locally AND emit an event" = a **dual write** (two systems, either can fail → event with no row,
or row with no event). Fix: make the event a byproduct of the ONE atomic commit — a
**transactional outbox** written in the same local transaction (L13/L25), shipped by **tailing
the DB commit log (CDC)**, the same trick that feeds search (L16), cache (L02), and warehouse
(L29). Trade shifts to **freshness/consistency vs pipeline complexity.**

Spine status: #32 done; #33–36 (CDC/outbox, service discovery & health checking, logical clocks
& causal ordering, cell-based architecture) remain queued, so the course does not yet need a
fresh batch.
