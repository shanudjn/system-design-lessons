# Learning record — Lesson 0033: Change Data Capture & the Outbox

**File:** `lessons/0033-cdc-outbox.html`
**Topic (curriculum spine #33):** Change data capture & the outbox pattern
**Trade named:** freshness/consistency vs pipeline complexity

## The one system
An `orders` service (L12/24/25) doing **5,000 order writes/sec** must save each order AND
propagate it to four other systems that don't share its database: **search** (L16), **cache**
(L02), **warehouse** (L29), **notify / next saga step** (L26/32). The obvious "write the row,
then publish `OrderPlaced` to the broker (L09)" is a **dual write** — two writes to two
independent systems with no shared transaction (L32's wall in miniature). This lesson is how to
get changes out reliably: make the announcement a byproduct of the one commit, then ship it from
a durable log.

## What the lesson covers (four moves)
- **Estimate — how often the dual write lies.** Steady drip: a 99.9% publish success rate loses
  `0.001 × 5,000 = 5 events/sec = 432,000/day` orders that commit to the DB but are never
  announced (not searchable, not counted, no confirmation — silent, compounding). Burst: one
  30-second broker blip orphans `30 × 5,000 = 150,000` at once (write-first → orders with no
  event; publish-first → 150k phantom announcements for orders that then fail to save). No retry
  closes the crash-in-the-gap window — the commit and the publish are two separate facts.
- **Model — outbox vs CDC (shared move: one commit + propagate from a log).**
  **Transactional outbox** = insert the event row into an `outbox` table IN THE SAME transaction
  as the order (L13/25) → both commit or neither; a separate **relay** reads unpublished rows and
  publishes them out-of-band, **at-least-once** (L09). **CDC** = skip the table, tail the DB's own
  **commit log** (WAL/binlog — durable by definition, how the DB guarantees the D in ACID); a
  connector (Debezium) emits an event per committed row change, no app code. Trade: outbox =
  **clean designed domain events + any DB**, at the cost of a table + relay + cleanup; CDC = **no
  app change + lowest latency**, at the cost of RAW row events coupling consumers to your physical
  schema + privileged log access. **Combine:** CDC-tail the outbox table → clean events, no polling.
- **Trace — clean / crash-after-commit / relay-republish.** Path A: one atomic commit, relay
  publishes, consumers act. Path B (the dual write's death, survived): crash right after COMMIT →
  the outbox row is still there → relay publishes it later → event is **LATE, never LOST** (dual
  write recorded the publish-intent only in the next line of code; the outbox records it as a
  committed row). Path C: relay crashes after publishing but before marking `published=true` →
  republishes → **duplicate** → safe because consumers are **idempotent** (L13, keyed on the same
  `event_id` the commit minted) → exactly-once EFFECT, not exactly-once on the wire (L09).
- **First bottleneck — everything now flows through one relay.** (1) Relay throughput/lag: 500-row
  batches every 100 ms = 5,000/s keeps pace; falling behind = **consumer lag** (L09) = every
  downstream's staleness. (2) Polling load on the primary → CDC-tail the outbox instead. (3)
  Outbox grows `5,000 × 86,400 × 200 B ≈ 86 GB/day` unless published rows are deleted /
  partitioned-and-dropped (L20/24) — a conveyor belt, not an archive. (4) Ordering: sharding the
  relay for scale needs **partition-by-key** (L09) so one order's events stay ordered. Bonus: the
  same durable ordered log you propagate from is the one you **replay** to rebuild a new consumer
  from scratch (L09/29 kappa). Four traps: retrying the dual write, non-idempotent consumer,
  un-cleaned outbox, CDC leaking your schema (L18/24).

## Threads reused
- **L32** — the closing "commit locally AND emit an event" = the dual write this lesson fixes.
- **L09** — the log, at-least-once delivery, partition-by-key ordering, consumer lag, replay.
- **L13** — idempotency keys / idempotent consumers → exactly-once effect on at-least-once delivery.
- **L25** — the atomic commit (write two things or neither); the ledger's append-only shape.
- **L02/L16/L29/L26** — the downstream consumers (cache, search, warehouse, notify).
- **L20/L24** — compaction/partition-and-drop for the ever-growing outbox; schema-safe events.
- **L07** — why a synchronous retry loop couples your commit to broker health (cascade).

## What it sets up next
**Lesson 0034 — Service discovery & health checking.** The relay had to "publish to the broker"
and consumers had to "call search" — but in L27's elastic fleet, instances boot and die every
minute, so which address do you connect to? A **registry** (register on boot, expire via TTL
heartbeat — L26's connection registry generalized), **health checks** (liveness vs readiness / the
drain signal, L31), and where the lookup lives (DNS vs client-side vs service-mesh sidecar). Trade
shifts to **routing freshness vs lookup cost & staleness.**

Spine status: #33 done; #34–36 (service discovery & health checking, logical clocks & causal
ordering, cell-based architecture) remain queued, and a fresh batch #37–41 (read/write splitting &
CQRS, event sourcing, tiered storage & data lifecycle, multi-tenancy & noisy neighbors, graph &
relationship systems) was added this lesson so the course stays well ahead of running dry.
