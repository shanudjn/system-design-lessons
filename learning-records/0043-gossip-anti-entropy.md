# Learning record — Lesson 0043: Gossip & Anti-Entropy

**Date:** 2026-07-30
**File:** `lessons/0043-gossip-anti-entropy.html`
**Curriculum spine item:** #43 (first item of the "Gossip & anti-entropy" advanced batch)
**Worked example:** a 1,000-node coordinator-free key-value store (Dynamo/Cassandra-style) agreeing on two things at once — membership (~50 KB roster) and replicated data (100 GB per replica).

## What this lesson covered

Followed the four-move spine (estimate → model → trace → first bottleneck):

- **Estimate** — All-to-all heartbeats are O(N²) ≈ 1,000,000 msg/s at N=1,000 and quadruple when the cluster doubles; a central registry (L34) or consensus cluster (L10) just re-introduces the SPOF/bottleneck a peer cluster is trying to remove. Gossip contacts only fan-out=3 random peers/round → 3,000 msg/round, ~333× cheaper, linear in N. Epidemic growth walk 1→4→16→64→256→1024 = log₄(1000) ≈ 5 rounds to reach everyone; honest note that the tail drags to O(log N), ~5–8s at a 1s interval.
- **Model** — two jobs, two mechanisms because membership is tiny/churny and data is huge/stable:
  - **SWIM** for membership: O(1) failure detection (ping one random peer, then k=3 indirect ping-reqs before believing a death), suspicion + incarnation-number refutation so a GC-paused node isn't wrongly evicted, dissemination piggybacked on the ping/ack traffic.
  - **Merkle-tree anti-entropy** for data: root hash proves 100 GB identical in 32 B; a mismatch is chased down only the divergent branch (~17 levels, ~1.1 KB of hashes to locate, ~1 MB bucket to ship) → cost scales with divergence, not dataset size (~100M× less than a naive full-range diff).
  - Two repair triggers: read-repair (hot keys, on the read path) + background Merkle sweep (cold keys) + hinted handoff (transient outages).
- **Trace** — (A) a real crash converging cluster-wide in seconds via indirect ping → suspect → dead; (B) a 2s GC pause where the node refutes with a higher incarnation number and is never evicted (the anti-flap safety valve); (C) key K stale on one replica after a partition, repaired both on-read (version compare, L06/L35) and by Merkle sweep.
- **First bottleneck** — gossip is eventually consistent (L11 AP): a disagreement window always exists. Shrink it with faster/wider gossip and more sweeps, but never to zero, because zero is exactly the coordination gossip exists to avoid. Decision line drawn crisply: gossip (AP) for "roughly who/what, cheap, no SPOF, scales" vs consensus (L10 CP) for exact/instant answers (leader, commit, lock) vs central registry (L34, SPOF at scale).

Arithmetic double-checked: N² ≈ 1M; 3,000 msg/round; log₄(1000) ≈ 5; Merkle over 100 GB / ~100k leaf buckets → depth ≈ 17, ~34 hashes × 32 B ≈ 1.1 KB to locate, ~1 MB bucket shipped, ~100M× / ~100,000× reductions.

## Reuses / threads pulled forward
L06 replication & version compare, L10 consensus (the CP contrast), L11 CAP/AP, L27 utilization knee, L29 narrow-the-bytes-before-computing, L34 registry/readiness/TTL-flap, L35 vector clocks, L42 routing.

## What it sets up next
**Lesson 0044 — Time-series & metrics storage** (spine #44): store the observability firehose (L17) at millions of points/sec — append-heavy writes, downsampling/rollups (raw → 1m → 1h) and retention tiers (L39), delta-of-delta + columnar compression (L29), the cardinality explosion (L17's ids-on-traces-not-labels), query-by-time-range. Trade: resolution & retention vs storage cost.

Spine still has #44, #45 (webhooks/outbound delivery), #46 (distributed job scheduling) queued — course will not run dry.
