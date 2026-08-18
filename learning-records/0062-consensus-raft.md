# Learning record — Lesson 0062: Consensus Internals (Raft)

**Date:** 2026-08-18
**File:** `lessons/0062-consensus-raft.html`
**Spine topic:** #62 — Consensus internals (Raft / Paxos)
**Trade named:** correctness & understandability vs the latency of agreement

## What this lesson covered
Opening the black box every earlier lesson leaned on ("store it in a consensus cluster"). Worked
example: **five nodes holding a tiny replicated config store** (the lock/leader/registry truth from
L10/22/34/43) that must agree on ONE ordered log of commands despite crashes and an unreliable
network, so every copy stays identical — a **replicated state machine** (same commands + same order
= same state; agree on the LOG and you've agreed on everything). Designed with **Raft**.

Four-move spine:
- **Estimate** — why a **majority** (⌊N/2⌋+1 = 3 of 5): any two majorities of 5 OVERLAP in ≥1 node
  (3+3 > 5, L11 pigeonhole) → two leaders per term impossible AND a committed entry impossible to
  lose. Fault tolerance N=2f+1 → f=2; even N buys nothing (6 also tolerates only 2, L10) → odd
  clusters. Commit cost = **one fsync + one majority round trip** (~2 ms intra-DC, ~30 ms
  cross-region, the L11/23 tax with a number). Ceiling ~1,000 commits/s per leader (fsync-bound),
  lifted ~100× by **group-commit batching** (L47/33 — amortizes the flush, never the RTT).
- **Model** — Raft's three pieces: (1) **leader election** — terms (a logical clock L35 + fencing
  epoch L10/22), one vote per term, majority wins, at most one leader/term by overlap; (2) **log
  replication** — leader appends (idx,term,cmd) → AppendEntries → **committed** when a majority
  stores it durably → apply + ack + propagate commitIndex; log-matching (prev-entry check) repairs
  stragglers so logs converge to identical prefixes; (3) **safety** — the election restriction
  (refuse to vote for a less-up-to-date log) chains with overlap so a new leader ALWAYS holds every
  committed entry (**Leader Completeness**).
- **Trace** — (A) clean ~2 ms commit that never waits for the two slowest nodes (majority, not all);
  (B) leader crash → election in ~150–300 ms, **randomized** timeout breaks the split-vote symmetry
  that would otherwise loop forever (L10 flap / L26 herd); (C) the crux — a **committed** entry
  (majority, client acked) survives any leader change (overlap carries it), while an **uncommitted**
  entry (minority, client saw a TIMEOUT = unknown, L13) may be safely OVERWRITTEN because the client
  retries with an idempotency key. Committed vs uncommitted IS the exact boundary of what a
  distributed system may promise.
- **First bottleneck** — agreement is a **toll paid per decision** (a durable majority). Attack the
  cost-per-entry, not the toll: batch + pipeline. When one leader still isn't enough, **shard into
  many Raft groups** (multi-Raft, L03; capstone L63) — throughput scales with #groups but you LOSE
  the single global order (cross-shard → 2PC/saga L32). Consensus is the most expensive storage you
  own → put ONLY agreement-critical metadata inside, keep bulk data out (L20/30 split). A majority
  must be alive or the cluster STOPS (CP, L11 — the opposite of L34's AP routing pick); Raft/Paxos
  handle crash-stop, not Byzantine (liars need BFT).

Deepest point: consensus is the one place in a distributed system where "it probably worked" is
banned. Everywhere else we embraced approximation (L02 stale cache, L06 eventual, L09 best-effort,
L21 sketches) because it was cheaper and good enough; consensus is what you build when good-enough
isn't — when two nodes disagreeing about who holds the lock corrupts everything downstream. You pay
a round trip + a flush on a majority per decision to buy a fact every node agrees on forever, which
is why you route the system's small load-bearing truths through this small, slow, expensive core.

Paxos noted as the original (same guarantees, famously harder to follow); Raft trades nothing in
correctness for teachability.

## Reuses (how it threads the course)
L11 quorum overlap (the pigeonhole that makes two leaders impossible and committed entries
unlosable — the entire safety argument); L35 logical clocks (the term); L10/22 fencing epoch,
leader election, flap; L13 idempotency + timeout=unknown (why discarding an uncommitted entry is
safe); L26 jitter / L10 flap (randomized election timeout); L47/33 group-commit batching; L20/30
metadata/data split (keep bulk data out); L34 the opposite AP pick for routing; L03 sharding
(multi-Raft); L32 2PC/saga (cross-shard consequence).

## Interactive quiz
4 questions: (1) does a committed/acked entry survive a leader change → yes, by majority overlap +
election restriction (Leader Completeness), and why N-of-N is the wrong "fix"; (2) is overwriting an
uncommitted minority entry a bug → no, client saw a timeout and retries idempotently (L13), and why
acking-early is worse; (3) 5→6 nodes "to be safer" → same fault tolerance (2), wider quorum, wasted
cost (even N buys nothing); (4) single leader maxed out → shard into multi-Raft (trades the global
order), why ack-before-replication and multi-leader are both wrong.

## What it sets up next
#63 — **Building a distributed key-value store (capstone):** wire consistent hashing (L03/04) +
quorum reads/writes (L06/11) + gossip & Merkle anti-entropy (L43) + LSM storage (L47) + this
lesson's consensus core (for metadata) into one Dynamo/Bigtable-style system, and watch where the
seams pull against each other. Remaining queued spine: #64 stream processing & windowing, #65 vector
databases & semantic search, #66 global rate limiting & quota. Next author picks #63 as the
lowest-numbered unmarked topic.
