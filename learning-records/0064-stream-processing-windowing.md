# Learning record — Lesson 0064: Stream Processing & Windowing

**Date:** 2026-08-20
**File:** `lessons/0064-stream-processing-windowing.html`
**Spine topic:** #64 — Stream processing & windowing
**Trade named:** latency & correctness of windowed results vs state size & recovery cost

## What this lesson covered
The streaming answer to L29's stale batch warehouse: keep a **continuously-updating** aggregate correct over a
never-ending stream. Worked example: a **live sales leaderboard** — "revenue in the last minute, per product,
refreshed every second" — over a **100,000-events/sec purchase stream**, end-to-end lag under a few seconds.
Framing: a stream processor is a **database query turned inside out** — it stores the standing query and keeps
the answer materialized in **state**, fighting three enemies at once (input never ends, arrives out of order,
the machine holding the answer can die).

Four-move spine:
- **Estimate** — the shape-defining fact: state scales with **KEYS × WINDOWS** (~1M active products × 24 B ≈
  24 MB running sums), NOT with the 20 MB/s (100k × 200 B) firehose, because an aggregate collapses many events
  into one number per key (L21). This flips the cost model: the expensive/fragile thing is now the **state**, not
  throughput. Re-running batch every second re-scans a rolling 60 s = 60× work (L29 cost ∝ history × frequency);
  streaming reads each event ONCE, cost ∝ **new** data.
- **Model** — three window shapes: **tumbling** (fixed non-overlapping, 1 update/event, the default), **sliding**
  (60 s len / 10 s step → 6 overlapping windows → 6× state & updates, responsiveness vs state trade), **session**
  (gap-defined, per-entity, unbounded-state hazard on a never-idle entity → cap max length). The idea it all turns
  on: bucket by **EVENT TIME** (stamped at source, fixed forever, replayable) not **PROCESSING TIME** (arrival →
  wrong bucket + non-deterministic). Event time creates "when is a window done?" → a **WATERMARK** =
  `max_event_seen − allowed_lateness` asserts "seen everything ≤ t"; when it passes a window's end the window
  **fires**. Allowed-lateness = the exact **latency-vs-completeness dial** (L17 tail / L29 late data). State lives
  in RAM + a local **LSM store** (L47); **CHECKPOINT** snapshots {all state + source offsets} together → on
  recovery restore state AND rewind the log (L09) to the matching offset → each replayed event folds in ONCE =
  **exactly-once STATE**, explicitly NOT exactly-once delivery (L09/13; sink still needs idempotent upsert by
  (window,key) or transactional/outbox sink, L13/33).
- **Trace** — (A) a $40 sale at event_time 10:00:59 arriving 10:01:04 lands in [10:00,10:01) because a 5 s
  watermark held it open, then fires when max_seen hits 10:01:05; processing-time bucketing would have mis-placed
  it into 10:01. (B) same sale = 1 update tumbling vs 6 updates sliding (the multiplier made concrete). (C) worker
  crashes 7 s after checkpoint C5 (offset 5,000,000, P-7 sum $3,200) with live sum $4,120 in RAM → restore C5,
  rewind to its offset, replay ≤1 interval (10 s × 100k ≤ 1M events) → sum climbs back to exactly $4,120, no
  double-count.
- **First bottleneck** — the **STATE itself**, not throughput. (1) Rescaling 20→40 workers must RELOCATE keyed
  state; `hash(key) % N` remaps ~94% of keys (L03) → ~94% of state ships → minutes of downtime → fix = fixed
  **KEY GROUPS** (e.g. 1024, consistent hashing L03/04 applied to state; key→group mapping never changes) so a
  resize moves only ~1/N. Cost of rescaling a stateful job ∝ state size, opposite of a stateless tier. (2)
  Checkpoint interval = two-sided dial: often → short replay but steady overhead; rarely → cheap steady state but
  long replay (L24 backfill batch / L02 TTL shape); incremental checkpoints (L47 changed LSM segments) cut it. (3)
  **Unbounded state** — never-idle session, no-window global aggregate, or a **stalled watermark** (one dead source
  partition freezes max_seen so nothing fires) → OOM; every window needs a reason it closes (fire / gap / TTL,
  L20/39 lifecycle).

Deepest point: windows tame the endlessness, event-time + watermarks tame the disorder, checkpoints + key groups
tame the fragility of state. Get those three right and "a correct, live, always-fresh aggregate over a firehose"
becomes a set of dials pointed at your latency/correctness budget.

## Reuses (how it threads the course)
L29 lambda/kappa **speed layer** (this IS that layer) + watermark for late data; L09 partitioned log, offsets,
at-least-once & no exactly-once delivery; L13 idempotent sink; L17 tail / late arrivals; L33 transactional/outbox
sink; L47 LSM state backend (+ incremental checkpoints); L21 aggregate collapses events into a compact running
summary; L03/04 consistent hashing (as key groups for cheap rescaling); L24 backfill-batch size / L02 cache TTL
(the interval dial); L20/39 state lifecycle so it doesn't grow forever; L05 async-work firehose shape.

## Interactive quiz
4 questions: (1) 5 s-late event → [10:00,10:01) by event time + watermark, why processing-time and "always drop
late" are wrong; (2) tumbling→sliding cost = 6× overlap multiplier, why "loses event time" and "can't checkpoint"
are wrong; (3) after crash recovery does the DASHBOARD stay correct → not without an idempotent/transactional sink
(exactly-once state ≠ delivery), why "automatic" and "shorter interval fixes it" are wrong; (4) rescaling stall
from `%N` remapping ~94% of state → fix with key groups, why "unavoidable" and "power-of-two count" are wrong.

## What it sets up next
Next in the queued spine: #65 **vector databases & semantic search** (embeddings + approximate nearest-neighbor
HNSW/IVF, recall vs latency, hybrid keyword+vector, re-indexing as the model changes — where L16's keyword index
can't reach), then #66 global rate limiting & quota (L08 coordinated across regions, CAP tension of a shared limit).
Next author picks #65 as the lowest-numbered unmarked topic. The spine was down to 2 queued topics, so this run
ADDED 4 new advanced topics (#67 data lakes/lakehouse & open table formats, #68 delivery guarantees & the
dead-letter lifecycle, #69 hot/warm/cold path & serving-vs-analytics split, #70 idempotent resumable data APIs) so
the course won't run dry.
