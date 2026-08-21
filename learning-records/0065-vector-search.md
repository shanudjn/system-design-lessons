# Learning record — Lesson 0065: Vector Databases & Semantic Search

**Date:** 2026-08-21
**File:** `lessons/0065-vector-search.html`
**Spine topic:** #65 — Vector databases & semantic search
**Trade named:** recall & quality vs query latency & index cost

## What this lesson covered
The first index in the course that answers "what is *similar*?" instead of "what is *equal*?" — the semantic
answer to L16's keyword blindness. Worked example: a **semantic product search** over L48's **10M-product
catalog** that must match "comfortable shoes for marathon training" to "cushioned long-distance running sneaker"
though they share almost no words. Framing: turn every product and query into a point in **768-dim space** where
closeness = similarity of meaning, then answer "find things like this" as "find the nearest points."

Four-move spine:
- **Estimate** — the shape-defining shock: nearest-neighbor has **no key to look up**, so the EXACT answer must
  measure distance to every vector = read the whole **~30 GB** index (10M × 768 × 4 B = 30.72 GB) per query. At
  ~30 GB/s memory bandwidth ≈ **1 s/query ≈ 1 QPS/box** — ~1000× short of "p99 < 20 ms, thousands QPS." No sharding
  closes a 1000× per-query cost cheaply → **must approximate** (stop looking at almost all vectors).
- **Model** — embeddings (a trained function meaning→vector; similar meanings map to nearby points) + **cosine**
  distance (angle). Why exact tree indexes fail: **curse of dimensionality** — past ~10-20 dims, near≈far, so a
  space-splitting tree (k-d) can't prune and degrades to brute force (L12). Two ANN engines: **HNSW** (navigable
  small-world graph, skip-list layers L12, greedy-hop toward q in ~log₂(10M)≈23 hops, visits ~2,000/10M = 0.02%;
  price = +15-30% graph memory + costly updates) and **IVF** (k-means into nlist=√N≈3,162 clusters ~3,163 each,
  probe nearest nprobe=16 → ~50k = 0.5% scanned, L16-flavored inverted file). Plus **product quantization** (chop
  768-d into 96 sub-vectors × 1-byte codes → 3,072 B → 96 B, **32×** smaller, 30 GB → <1 GB; lossy → recall bought
  back by a full-precision **re-rank** of the coarse top-200, L21 summary-vs-exact). Plus **hybrid** keyword+vector
  fused by **reciprocal rank fusion** (1/(rank+k) summed) — keyword catches exact SKU/brand, vector catches
  meaning, neither alone suffices.
- **Trace** — (A) query end-to-end ~17 ms: embed q with the SAME model → HNSW efSearch=200 visits ~2,000 → top-200
  candidates → (if quantized) re-rank at full precision → hybrid fuse with L16 BM25 → business filters → top-10;
  the zero-shared-words winner ranks #2, invisible to keyword. (B) HNSW vs brute force on the same q: ~2,000 vs 10M
  visited, 6 MB vs 30 GB read (~5,000× less), 97% recall vs 100%, ~3 ms vs ~1 s. (C) the recall knob: efSearch
  50→800 raises recall 0.90→0.992 but latency 1.2→13 ms, and the curve **bends viciously** near 1.0 (0.99→1.0 =
  the whole 30 GB scan back). Recall and latency are the same knob from two ends.
- **First bottleneck** — the **recall–latency–memory TRIANGLE**: pick any two, the third pays (more recall → visit
  more → more latency; less memory → quantize → less recall or more re-rank). The dangerous move = chasing 99.9%
  recall for neighbors no user notices, up the vicious end of the curve toward brute-force cost. Two operational
  walls: (2) **updates/freshness** — an ANN graph doesn't append cheaply; inserts rewire links, deletes leave
  tombstones, churn rots recall → delta index + periodic **rebuild-and-swap** (L31 blue-green, L47 LSM
  compaction). (3) **reindexing on model change** — an embedding is meaningful only relative to its model; v1 and
  v2 coordinates are **not comparable and can't be mixed in one index**, so a model upgrade = a full **re-embed of
  all 10M** + versioned rebuild + atomic cutover (L24/57 backfill), query embedded with v2 only after the switch.

Deepest point: embeddings turn meaning into geometry; ANN turns "compare to everything" into "walk to the
neighborhood"; recall is the dial for how approximate you'll be; and the model version is the hidden coordinate
system that invalidates everything at once when it changes.

## Reuses (how it threads the course)
L16 (the inverted index it complements + the hybrid fusion + IVF's inverted-file idea); L12 (curse of
dimensionality kills B-trees/k-d trees; skip-list shape HNSW borrows); L48/2 (the 10M catalog, caching the index
across a fleet); L47 (LSM compaction as the model for rebuild-and-swap, hot-delta + cold-base); L31 (blue-green
cutover for a fresh index); L24/57 (schema/model versioning + the backfill to re-embed everything); L21
(quantization = an aggregate trading exactness for compactness).

## Interactive quiz
4 questions: (1) why EXACT NN is impractical (no key → read all 30 GB → ~1 s → ANN fixes it), why "30 GB too big
to store" and "switch metric" are wrong; (2) why a k-d tree fails at 768 dims (curse of dimensionality, no
pruning), why "only Euclidean" and "can't update" are wrong; (3) 0.97→99.9% recall demand → the vicious
recall/latency curve, why "recall is free / build-time only" and "IVF always better" are wrong; (4) the v2 model
upgrade → can't mix incompatible coordinate systems, full re-embed required, why "same dimension count suffices"
and "pad/truncate to convert" are wrong.

## What it sets up next
Next in the queued spine: #66 **global rate limiting & quota** — L08's single-node limiter coordinated across
regions and a fleet (approximate counters L21, local buckets + reconciliation, per-tenant quotas L40, and the CAP
tension L11 of one shared limit under partition). Then #67 data lakes/lakehouse & open table formats, #68 delivery
guarantees & the dead-letter lifecycle, #69 hot/warm/cold path & serving-vs-analytics split, #70 idempotent
resumable data APIs. Next author picks #66 as the lowest-numbered unmarked topic. The spine still has 5 queued
topics (66-70), so this run did NOT need to add new ones — the course won't run dry.
