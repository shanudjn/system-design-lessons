# Learning record — Lesson 0019: Geo & Proximity

**Date:** 2026-07-06
**File:** `lessons/0019-geo-proximity.html`
**Spine topic:** #19 — Geo / proximity systems (now ✅)

## The one system
"Find the drivers within 2 km of me, nearest first" — a rider's search over 1M
moving cars, answered in a few ms. A point + a radius + a haystack of moving needles.

## What the lesson covered
- **Estimate — the two obvious designs fail.**
  - Check every driver: 1M haversine calls/search × 50k searches/sec = **5×10¹⁰**
    dist-calls/sec. Wrong shape (O(N) per search).
  - Index one coordinate: 2 km ≈ 0.018° lat (1° ≈ 111 km); a lat-band is a 4 km-tall
    strip wrapping the planet → returns everyone at your latitude worldwide, then a
    linear longitude filter. **Two 1-D indexes don't intersect into a 2-D box.**
  - Trade: dimensionality — a B-tree is a line, a map is a plane.
- **Model — fold 2-D into 1-D with a space-filling curve.**
  - **Geohash:** repeatedly halve lng/lat, interleave the bits, group 5 → base-32
    char. Worked walk: lng −122.4 → bits 00101, lat 37.8 → 10110, interleave →
    `01001 10110` → idx 9 = `9`, idx 22 = `q` → "9q…". Nearby points share a prefix,
    so `WHERE geohash LIKE '9q8yy%'` (a B-tree prefix range, L12) answers 2-D "near?".
    Precision: 5 chars ≈ 4.9 km cell (matches a 2 km radius).
  - **Boundary problem:** a car 100 m away across a cell edge has a different prefix;
    the circle spills into neighbours. Fix = query the **3×3** neighbour cells, gather
    candidates, run **exact haversine**, keep ≤ 2 km.
  - **Worked cut:** 1M drivers / ~10,400 cells ≈ 96/cell → 9 cells ≈ 900 candidates →
    exact filter 900 not 1M = **~1,100×**.
  - Cell size = **recall vs work** (pick ≈ the radius so a 3×3 box holds the circle).
  - **Quadtree:** subdivide only where dense → balanced points-per-leaf; adaptive but
    a mutable in-memory tree. Trade: statelessness vs adaptivity.
  - **S2:** sphere → cube faces → **Hilbert curve** (never jumps, unlike geohash's
    **Z-order**) → 64-bit cell ids, more uniform cells. Trade: simplicity vs locality.
- **Trace — write & read paths.** Write: 250k pings/sec (1M/4 s), each an O(1) cell
  update, hot & in-memory. Read: encode → 3×3 → ~900 candidates → exact haversine →
  sort → top N. Coarse cheap filter at write/index, exact expensive filter at read
  (the L12/L14/L16 "narrow cheap, refine expensive" funnel).
- **Next bottleneck — the hot cell.** Stadium empties → 50k in one cell (~500×
  average) = a **hot shard** (L03 hot key / L08 hot rate-limit key / L09 hot
  partition, now on a map). Fix: adaptive precision (finer cells downtown, i.e. a
  quadtree), cap/sample the returned set, or replicate the hot cell for reads.
  Trade: uniform grid vs load that follows the crowd.

## Reuses / callbacks
- L18 cursor (1-D sort key that can't answer 2-D "nearest") — the direct handoff.
- L12 B-tree seek (what a geohash prefix rides on).
- L14/L16/L12 "narrow cheap vs refine expensive" funnel (grid → exact distance).
- L03/L08/L09 hot-shard/hotspot wall (now a hot cell).
- L06/L08 "approximate to stay scalable" (cap/sample the candidate set).

## What it sets up next
- **#20 Object / blob storage** — storing 100 PB (e.g. dashcam video): chunking,
  erasure coding vs replication, metadata service, the small-file problem.
- **#21 Analytics & counting at scale** — HyperLogLog / Count-Min / Bloom: "how many
  unique riders searched each cell today?", trading exactness for memory (the same
  approximate-to-scale move the hot cell forced here).

Spine still has #20–#21 queued, so no new advanced topics added this run.
