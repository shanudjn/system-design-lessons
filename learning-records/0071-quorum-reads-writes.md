# Lesson 0071 — Quorum Reads & Writes: Tunable Consistency

**File:** `lessons/0071-quorum-reads-writes.html`
**Spine topic:** #71 (opens the `W + R > N` box used as a black box since L06/11/63)
**Date:** 2026-08-27

## Worked example
One key in a Dynamo-style KV store (L63), replicated to **N = 3** nodes {A, B, C}. A **coordinator** fans each request to all three and waits for a threshold: **W** acks on a write, **R** replies on a read. W and R are a **per-request dial** — the same store serves a throwaway "display name" read cheaply *and* a "read the plan before charging" read strictly.

## What it covered
- **Estimate — the two forces that make it a dial.**
  - *Availability* (each replica 99% up): W=1 → 1−0.01³ = six nines; W=2 → 0.99³+3·0.99²·0.01 = **99.97%** (0.03% fail ≈ **2.6 h/yr**, tolerates 1 down); W=3=N → 0.99³ = **97.03%** (**2.97%** fail ≈ **260 h/yr**, a **100×** worse rate — one hiccup blocks *every* write). Headline: **W=N makes you less available than a single machine.**
  - *Latency* (each replica 95% fast ≤5 ms, else ~200 ms): wait-for-1-of-3 fast 99.99%; wait-for-2 = 0.95³+3·0.95²·0.05 = **99.3%**; wait-for-3 = 0.95³ = **85.7%** → **14.3%** dragged to the tail (L17). Lower quorum = more available AND faster, but gives up overlap.
- **Model — the pigeonhole guarantee.** Write W=2 lands on {A,B}; read R=2 hits {B,C}; |{A,B}|+|{B,C}| = 4 > 3 → can't be disjoint → share B → B has the new value → read returns MAX(version). Break it: R=1 → W+R = 3 = N (not >) → the single replica read may be the one that missed the write → legal stale read. **The inequality is strict for a reason.**
- **The budget table (spend 4 across W,R for overlap on N=3):** `W=3,R=1` read-heavy (fast reads, fragile writes) · `W=1,R=3` write-heavy · `W=R=2` majority (`⌊N/2⌋+1`, tolerant of 1 death each side, sane default) · `W=1,R=1` below budget (fastest, eventual, staleness-OK keys like a view count).
- **Two healers.** **Read-repair** — a read that finds a stale replica writes the winner back to it (heals hot keys free; cold keys unhealed). **Hinted handoff** — a write to a down replica parks on a stand-in tagged "belongs to C"; C catches up on return (keeps writes available through failure, L11/34). Versioning (L35 vector clock) is the silent prerequisite for "which is newer / are these concurrent?"
- **Trace.** (A) quorum read catches a stale C (overlap forces it). (B) that read read-repairs C to the winner. (C) write with C down still succeeds on {A,B} + hint on D → handoff when C returns.
- **First bottleneck — overlap is NOT order.** Quorum resolves *staleness*, never *conflict*: two concurrent writes (X adds shoes → v5x on {A,B}, Y adds hat → v5y on {B,C}) both satisfy W=2 → siblings a version compare can't rank → must *separately* choose LWW (drops one, trusts skewable clock) / keep siblings + app merge (L23) / serialize the key via consensus (L62). **W=R=N is a trap** (worst availability + worst tail + STILL no total order). Honest strong = majority W=R=2; true linearizability needs L62 Raft. **Sloppy quorum** (hinted-handoff stand-ins count) softens strict overlap to "eventually" = L11's CAP choice (strict=C refuse write, sloppy=A take write + soft window).
- **Deepest point.** `W + R > N` is a **dial, not a switch**, because consistency, latency, and availability are one budget seen from three sides — you can't spend the same replica-wait on all three, so the good stores expose W/R *per call*.

## Numbers used (all checked, `python3`)
- Availability (p=0.99): ≥1/3 = 0.999999; ≥2/3 = 0.999702; all 3 = 0.970299.
- W=3 fail 2.97% → 0.0297×8760 = **260 h/yr**; W=2 fail 0.03% → **2.6 h/yr** → ~100× gap.
- Latency-fast (f=0.95): fastest-of-3 = 0.999875; ≥2/3 = 0.99275; all 3 = 0.857375 → 14.3% slow at R=3.
- Pigeonhole: W=2+R=2 = 4 > 3 overlap; W=2+R=1 = 3 = N no overlap.

## Threads reused
L6 (replication, lost updates, first W+R>N quorum, read-repair), L11 (CAP → strict-vs-sloppy quorum), L63 (the Dynamo-style KV store), L35 (vector clocks: newer-vs-older, concurrent-vs-causal), L23 (LWW vs CRDT/siblings for the concurrent-write conflict), L34 (fail-static → hinted handoff), L17 (tail latency → waiting for the slowest replica), L20 (durability from replication), L62 (consensus/Raft → the linearizability quorum alone can't give).

## Sets up next
- **Lesson 72 — Bulk & batch APIs (the N+1 / fan-out problem):** one screen needing 200 records fires 200 round trips (L18) — the silent latency killer; batch endpoints, request coalescing/dataloader, field selection, over-fetch vs under-fetch. Trade: fewer round trips & payload control vs API & caching complexity.
- Then Lesson 73 — deduplication & entity resolution at scale.
- Spine still has 72–75 queued, so no new topics added this run.
