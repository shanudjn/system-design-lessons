# Lesson 0084 — Graph & Recommendation Traversal at Scale: One "People You May Know" Request Across a Billion-Edge Graph

**File:** `lessons/0084-graph-traversal-pymk.html`
**Curriculum spine:** topic #84 (advanced batch, added after L83) — now ✅
**Date:** 2026-09-09

## The one worked system
The social graph from L41: **1 billion users**, average **200 friends** each
(mutual friendship → 200 billion directed edges, ~2 TB, sharded across ~100
nodes). One request is traced end to end: **People You May Know (PYMK) for one
user, "Alice"** — find her friends-of-friends, rank them by shared (mutual)
friends, return the top 50. A tiny toy graph (Alice → Bob, Carol, Dave; their
friends → Eve, Frank, Grace, Heidi) carries the concrete walk on the page.

## What it covered (the four moves)
- **Estimate** — graph size: 1e9 × 200 × 8 B = 1.6 TB edges (~2 TB with
  overhead) → ~20 GB/node across 100 shards, RAM-resident. The **fan-out
  explosion** at degree 200: hop1=200, hop2=40,000, hop3=8,000,000,
  hop4=1.6 billion (> the whole user base) → why PYMK stops at **2 hops**
  (strong signal + tractable). The **mutual-friend score is free**: the hop-2
  hits are a multiset, and each id's multiplicity = number of 2-hop paths =
  number of mutual friends, so dedup-histogram and scoring are one pass.
  Asymmetry: traverse-at-read ≈ 200 scattered reads/request → ~1M adjacency
  reads/s at peak (fresh, heavy) vs precompute nightly (1e9 × 40k = 4e13
  touches ≈ 7 min at 1e11/s, then one KV lookup — L15 precompute-the-read).
- **Model** — graph stored as **adjacency lists** sharded by `hash(user_id)`;
  a friend's list lives on a different shard, so a 2-hop walk **scatters across
  ~200 shards** (L19/L21 scatter-gather; latency = slowest shard). Can't keep
  friends co-located: balanced graph partitioning (min-cut) is **NP-hard** and a
  small-world graph resists any clean cut → **locality vs balance** trade. PYMK
  framed as candidate-generation (cheap 2-hop walk) → ranking (mutual friends +
  recency + shared org + ML re-rank on survivors) → business filters — the
  L16/L53 two-phase funnel.
- **Trace** — (A) traverse-at-read: read adj(Alice) → scatter to friends'
  shards → gather ~40k hits → histogram (Eve:3, Frank:2, Grace:1, Heidi:1) →
  rank → top 50 (fresh, ~200 cross-shard reads). (B) precompute: nightly batch
  writes `pymk:{u}` → read = one lookup ~1 ms (cheap, stale to last run).
  (C) **supernode**: a celebrity friend with 10M edges makes one hop read 10M
  ids → hot shard (L79) AND carries ~zero signal (FoF-via-celebrity means
  nothing) → the two problems align, so **skip/cap high-degree vertices** fixes
  cost and noise together.
- **First bottleneck** — the fan-out concentrated in supernodes. Two moves:
  **precompute** bounds how *often* you pay the walk (L15); **degree-capping**
  bounds how *big* any single hop gets. Walls around it: (1) freshness —
  precompute is stale → **offline base + online patch** (expand only the edges
  new since the batch; L29/L53 lambda split); (2) cross-shard traversal tax →
  **replicate hot/high-degree vertices**, cache hot lists (storage-for-latency,
  L02/L48); (3) candidate generation lives on an **offline↔online spectrum**
  (L53), the graph walk being one candidate source feeding a broader ranker.

## Trades named
- **Traversal freshness vs precompute cost** (the spine trade).
- Locality vs balance in graph partitioning (co-locate friends vs even shards).
- Recall vs cost in the candidate funnel (more hops / keep supernodes = more
  true matches but more work + noise).
- Degree-capping: a possibly-missed hub connection vs a hard ceiling on cost.

## Deepest point
A graph query's cost is set by the **degree of the vertices it touches**, not
the number of hops on the whiteboard. "Friends of friends" is one line of
English and a multiplicative bomb (×200 per hop, ×10M for a celebrity). Every
graph technique here is the same instinct: **bound the fan-out and pre-shape
it** — fewest hops with signal, precompute, degree-cap, replicate hubs, patch
the stale base with a cheap online delta.

## Reuse / callbacks
L41 (graph model), L15 (precompute-the-read), L16 (candidate→rank funnel),
L53 (offline vs online candidate/feature generation, lambda), L79 (sharding +
scatter-gather + hot-key/supernode), L04 (hashing keys→nodes), L18/L21
(scatter-gather, mergeable counts, top-K/straggler traps), L02/L48 (cache /
replicate hot data), L29 (batch-vs-stream freshness / lambda).

## What it sets up next
**Lesson 0085 — Multi-region data placement & residency.** We've treated every
store (the graph included) as if a row can live anywhere. Next: where a row is
legally and physically *allowed* to live — geo-partitioning by user home region
(L23/L79), data-residency/GDPR constraints (L56), follow-the-sun latency
(L14/L23), and the cross-region join tax. Trade: local latency & compliance vs
global query simplicity.
