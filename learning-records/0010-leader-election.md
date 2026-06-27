# Learning record — Lesson 0010: Leader Election & Coordination

**Date:** 2026-06-27
**File:** `lessons/0010-leader-election.html`
**Title:** Leader Election & Coordination: One Leader, No Split-Brain

## What this lesson covered
Took ONE concrete cluster — five identical nodes that must run a monthly
billing cron job *exactly once* — and traced how they stay single-headed.
Followed the four-move spine:

- **Estimate** — sized the failover clock: 1 s heartbeat, 3 missed beats = 3 s
  timeout, ~3.5 s of leaderless dead air (3 s detect + 0.3 s random election
  delay + 0.2 s vote round-trip). Named the detection-speed-vs-false-positive
  dial: too short a timeout turns a benign GC pause into a needless election →
  **leadership flapping** (the rebalance storm from Lesson 09, restated).
- **Model** — terms/epochs, one-vote-per-node-per-term, and the majority rule
  (quorum = ⌊N/2⌋+1 = 3 of 5).
- **Trace the paths** — (A) clean failover with the higher term winning and the
  rebooted old leader stepping down; (B) network partition where a majority makes
  two leaders *mathematically* impossible (two groups of 3 in a set of 5 must
  overlap by pigeonhole, and the shared node can't vote twice) — quantified the
  avoided damage ($1M of duplicate charges on 200k customers × $5); (C) the
  zombie/slow leader that unfreezes stale and is shut out by a **fencing token**
  (resource rejects any write with epoch < its high-water mark).
- **Next bottleneck** — sizing the cluster: odd N, because even N gives the same
  fault tolerance (N − quorum) as the odd below it at higher cost/quorum (table
  for N=3..7). The wall behind it: the minority side gives up availability to
  keep safety — the CAP choice.

## Trade-offs named
- timeout: fast failover **vs** false positives (flapping)
- majority quorum: safety (one leader) **vs** minority availability
- fencing: simple heartbeats **vs** epoch threaded through every write
- cluster size: odd N optimal; even N = wasted node + higher quorum

## Callbacks to earlier lessons
Lesson 09 rebalance/coordination cliffhanger; Lesson 07 p99-sizing reflex for the
timeout; Lessons 05 & 07 idempotency keys (fencing token as a sibling idea).

## What it sets up next
- **Lesson 11 — CAP in practice:** the partition we survived stated as a law;
  what choosing C vs A actually does to real reads and writes.
- Idempotency revisited: fencing token connected back to idempotency keys.
