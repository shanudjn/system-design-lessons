# Lesson 0035 — Logical Clocks & Causal Ordering

**Published:** 2026-07-22
**File:** `lessons/0035-logical-clocks.html`
**Spine topic:** #35 (advanced batch)
**Trade named:** ordering precision vs metadata size & coordination

## Worked example
A **group chat** served from two regions (Lesson 23): Alice on Virginia (VA), Bob on
Frankfurt (FR). The symptom that opens the lesson: a **reply prints above the question it
answers**. FR's clock runs 50 ms behind VA; Bob replies 20 ms (real time) after reading
Alice's question, so FR stamps the reply `(T+20ms) − 50ms = T−30ms` — 30 ms *before* the
question. Clients sorting by wall-clock timestamp invert cause and effect.

## The four moves
1. **Estimate** — the one inequality: clock **skew** (tens of ms; quartz drift ~50 ppm ≈
   4.3 s/day unchecked, NTP pins only to ~ms and can step backward) vs the **gap** between
   causally linked events (µs–ms). Skew > gap ⇒ wall-clock order inverts causal order —
   routine, not rare — and can't be driven to zero (Lesson 14 speed-of-light residual).
2. **Model** — define **happened-before (→)** from information flow (same-process order,
   send-before-receive, transitivity); **concurrent (∥)** = neither direction holds.
   - **Lamport clock**: one counter, `local: L++`, `send: L++ attach`, `receive: L=max(L,m.L)+1`.
     `O(1)`, gives a **total order** (tiebreak by process id) that never sorts an effect
     before its cause. One-way arrow: `a→b ⇒ L(a)<L(b)` but **not** the converse → **cannot
     detect concurrency**.
   - **Vector clock**: one counter per process, elementwise-max on receive. Detects
     concurrent vs causal at **`O(N)`** (this is Lesson 23's version vector).
3. **Trace** — the reply: Lamport Q=1 < reply=3; vector Q=[1,0,0] < reply=[1,2,0] (causal),
   both restore order. Carol's unrelated message = [0,0,1]: neither ≤ Q=[1,0,0] → **concurrent**,
   and only the vector clock refuses to invent an order (so a conflict resolver MERGES rather
   than dropping a write — closes Lesson 6/23 lost update).
4. **First bottleneck** — vector clock `O(N)` is fine when N=servers, fatal when N=users:
   `1M × 8 B = 8 MB` of metadata to order one keystroke, and it never shrinks. →
   **Hybrid logical clock (HLC)**: pack `(physical pt, counter c)` into 64 bits → `O(1)`,
   causal like Lamport, AND within clock-skew of real wall-clock time (honest repair for the
   clocks L22/23/34 trusted). HLC is total-order-only, so concurrency detection still needs a
   bounded/pruned vector (dotted version vectors). CockroachDB uses HLC.

## Traps covered
1. Ordering by wall-clock timestamp across machines (the opening bug; L23 LWW).
2. Reading a Lamport/HLC total order as causality → fake order → L23 lost update.
3. A per-client vector clock at scale (the 8 MB-per-keystroke wall).
4. Thinking a logical clock removes the need for real time (L22 lease still needs physical expiry).

## Builds on
L23 (skew, version vectors), L22 (clock-fragile lease), L34 (TTL), L14 (speed-of-light
floor → skew can't reach zero), L6 (lost update — what a faked order silently causes),
L12 (latency numbers — how close related events are).

## Sets up next
**Lesson 0036 — Cell-based architecture** (spine #36): the blast-radius tool L7's bulkhead /
L28 / L31 all pointed at. Partition the whole stack (LB + app + data) into independent cells,
route each user to one cell, so a bad deploy / poison input / hot tenant takes down 1/N of
users, not all. Cell sizing, routing/placement, shuffle-sharding. Trade: blast-radius
isolation vs cross-cell coordination & overhead.
