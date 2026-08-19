# Learning record — Lesson 0063: Building a Distributed Key-Value Store (capstone)

**Date:** 2026-08-19
**File:** `lessons/0063-distributed-kv-store.html`
**Spine topic:** #63 — Building a distributed key-value store (capstone)
**Trade named:** tunable consistency vs availability & operational complexity

## What this lesson covered
The capstone: stop building primitives one at a time and ASSEMBLE the course into one real system — a
**Dynamo-style key-value store**. Worked example: a **100-node, 3-datacenter, 1 TB shopping-cart store**
that must stay writable while a datacenter is on fire and never silently lose a cart-add. The "always
writable" business rule is the seed — it forces the **AP** corner (L11), hence a leaderless quorum store,
NOT a single-leader Raft-per-key store.

Four-move spine:
- **Estimate** — fleet sized by **replication-amplified** load: N=3 → 3 TB stored → ~30 GB/node; writes
  ×N=3 → ~3k/node, reads ×R=2 → ~10k/node (100 nodes comfortable). The quorum table: one inequality
  `W+R>N` slides the whole system between consistency and availability. N=3 combos: 1/1 (fast, stale,
  2-down-tolerant), **2/2** (read-your-writes, 1-down, the balanced default), 3/1 (fragile writes), 1/3
  (fragile reads). Quorum = "wait for the W-th fastest, not all N" = L62's majority tail trick.
- **Model** — six primitives as THREE PLANES. Placement: consistent-hashing ring + virtual nodes → a
  **preference list** = next N=3 DISTINCT physical nodes/racks (L03/04), a pure function so any node
  coordinates with zero lookups. Replication/consistency: quorum W/R + **sloppy quorum** + **hinted
  handoff** (stay writable through failure) + **version vectors** (L23/35, tell concurrent from
  happened-after). Membership/repair: **gossip** (L43, no-SPOF AP registry, O(log N)) + **Merkle
  anti-entropy** (L43, repair only differing keys, not a 30 GB copy). Under it an **LSM engine** (L47);
  beside it a small **Raft core** (L62) owning the ring/token map (L20/30 metadata/data split).
- **Trace** — (A) healthy W=2 write returns on the 2nd-fastest replica, never waits for the slow/dead 3rd;
  (B) read finds two CONCURRENT versions (version vectors: neither ≥ other) → keep BOTH siblings → merge by
  cart UNION (a business rule) + **read repair** the stale replica; NOT silent LWW (would drop an item);
  (C) DC partition → store keeps writing on both sides (available), two truths briefly exist (not
  consistent), reconcile on heal via hinted handoff + anti-entropy + merge = CAP's A-over-C paid in exactly
  the machinery Move 2 built.
- **First bottleneck** — in a capstone the wall is the **SEAMS** where two correct primitives pull apart:
  (1) sloppy quorum buys availability by VOIDING the `W+R>N` overlap → the whole reconciliation stack
  (vectors/siblings/read-repair/anti-entropy) exists to clean up the divergence you deliberately allowed;
  (2) a HOT KEY still bakes its N replicas while the fleet idles — consistent hashing fixes rebalancing,
  not skew (L04's explicit warning) → cache (L02) / key-split (L21) / extra read replicas, not a better
  hash; (3) the ring must be CP even though the data is AP — a gossiped ring → nodes disagree on ownership
  → write and read touch DISJOINT replica sets → silent loss, so topology lives in Raft (L62) while
  liveness stays in gossip; (4) the top-level AP/CP fork (leaderless quorum vs Raft-group-per-shard) is
  decided ENTIRELY by "what does this data cost when briefly wrong?" — mergeable cart → AP, un-mergeable
  balance → CP (L11/25) — the SAME six primitives assemble into either machine.

Deepest point of the whole course: a distributed database is not a monolithic invention — it is these
primitives composed. Once you see a system as "consistent hashing for placement, quorums for the
consistency dial, gossip for membership, anti-entropy for repair, LSM for the disk, consensus for the one
agreed map," you can read Dynamo / Cassandra / Bigtable / CockroachDB / Riak as different SETTINGS of the
same knobs, and design a new one by choosing those settings deliberately. The one question that sets every
knob is the one we started from in Lesson 1: what does this data cost the business when it's briefly wrong?

## Reuses (how it threads the course)
L03/04 consistent-hashing ring + virtual nodes (placement) and its "solves rebalancing not skew / hot key"
warning; L06/11 quorum reads/writes, the `W+R>N` overlap that dials consistency, and CAP's A-vs-C fork;
L23/35 version vectors (concurrent vs happened-after); L43 gossip membership + Merkle anti-entropy; L47 LSM
storage engine; L62 consensus core (holds the ring) + "majority not all" tail trick; L20/30 metadata/data
split; L02 caching + L21 sharded counter (hot-key patches); L11/25 the CP scalar-balance contrast; L17 tail
latency.

## Interactive quiz
4 questions: (1) which R gives read-your-writes after a W=2 write → R=2 (W+R=4>3 overlap), why "all N
already have it" and R=1 are wrong; (2) two concurrent cross-partition cart adds → version vectors detect
concurrent, keep siblings, merge by union (not LWW, not reject); (3) hot key + "add nodes so consistent
hashing spreads it" → no, hashing fixes rebalancing not skew, patch with cache/key-split, why vnodes and
raising N globally are wrong; (4) why the ring is Raft-CP not gossip-AP → disagreement on ownership →
disjoint replica sets → silent loss, why "Raft is faster" and "gossip can't carry structured data" are
wrong.

## What it sets up next
Capstone complete — primitives assembled into a working store. Remaining queued spine: #64 **stream
processing & windowing** (stateful tumbling/sliding/session windows, watermarks L17/29, checkpointing for
exactly-once state, keyed state & rescaling — beyond L29's lambda/kappa sketch), #65 vector databases &
semantic search (ANN over embeddings, HNSW/IVF, hybrid retrieval), #66 global rate limiting & quota
(L08's limiter coordinated across regions, CAP tension of a shared limit). Next author picks #64 as the
lowest-numbered unmarked topic. Spine still has 3 queued topics, so no new topics added this run.
