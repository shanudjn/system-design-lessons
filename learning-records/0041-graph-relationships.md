# Learning record — Lesson 0041: Graph & Relationship Systems

**File:** `lessons/0041-graph-relationships.html`
**Curriculum spine item:** #41 (first unmarked → now ✅)
**Trade named:** traversal speed vs the partitionability of connected data.

## The one worked system
A social graph: **1 billion nodes**, avg **200 friends** each → **200 billion directed edges** (~3.2 TB). The single hot query: **friend-of-friend (FoF)** — walk 2 hops and return the distinct not-yet-friends (the pool behind "People You May Know").

## Four moves
- **Estimate — why the relational JOIN explodes at depth.** A self-join does `d^k` index seeks and re-pays a `log N` (~38-comparison) B-tree descent on each. Hop 1 = 200 rows / 1 seek (fine). Hop 2 = 40,000 rows / 200 seeks ≈ 20 ms (Lesson 12's N+1). Hop 3 = 8M rows / 40,000 seeks ≈ 4 s. Hop 4 ≈ 1.6B rows / 8M seeks ≈ 13 min. Hop 5 ≈ whole graph ("six degrees"). Cost is geometric in depth: ×200 per hop.
- **Model — the two fixes.** (1) **Adjacency list**: store a node's neighbors together so "get my friends" is one read, not a join. (2) **Index-free adjacency**: a node points directly at its neighbors' records, so one hop is `O(1)` and — the key win — its cost is *independent of graph size N*. Both still visit `d^k` nodes (the answer's own size); what's removed is the per-graph-size `log N` tax. Trade: a graph store gives up the relational model's generality (weak at set/aggregate queries) for one thing done superbly.
- **Trace — three paths.** (A) On one machine, the 40,000-node FoF is a sub-ms in-memory pointer-chase. (B) Shard 3.2 TB by `hash(user_id)` across 100 machines → `P(same shard) = 1/100` → **~99% of edges cut** → every hop is a ~0.5 ms network round trip, bounded by the slowest shard (Lesson 27 tail). (C) A **supernode** (celebrity, 100M degree) fans one edge to 100M nodes and is a hot shard — Lesson 15 fan-out + Lesson 3 hot shard as graph structure. The average degree (200) is a lie (Lesson 2 skew).
- **First bottleneck — you can't cut connected data cleanly.** An edge *is* a dependency; balanced graph partitioning is NP-hard. Buy the best imperfect partition: **replicate** the hot subgraph (Facebook TAO), **partition by community** not by hash (~99% → ~30% cut), **vertex-cut** the supernodes, and **cap depth + precompute** (mutual-friend counts) because `d^k` is unavoidable past ~3 hops.

## Four traps
1. Serving deep traversals with relational JOINs (`d^k` seeks × `log N` each).
2. Sharding a graph like a table (hash cuts ~99% of edges → network storm; more shards = worse).
3. Trusting the average degree (hides the 100M-degree celebrity; degree is Zipf).
4. Traversing arbitrarily deep (frontier grows 200×/hop → whole graph in ~5 hops).

## Prior lessons reused
L2 (degree skew), L3 (sharding + hot shard = supernode), L6 (replication lag returns when you replicate the graph), L12 (N+1 / index selectivity), L15 (fan-out + celebrity special-casing), L27 (fan-out tail latency), L33 (change stream keeps graph & relational stores in sync).

## Sets up next
**Lesson 0042 — Load balancing algorithms & layers** (spine #42): L4 vs L7 balancing, the algorithm ladder (round-robin → least-connections → EWMA/least-latency → power-of-two-choices → consistent-hash stickiness, L4), why round-robin starves behind one slow backend, health checks + connection draining (L31/34), the LB as a SPOF (L7/26). Trade: even load distribution vs stickiness & simplicity.
