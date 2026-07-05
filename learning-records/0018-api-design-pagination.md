# Learning record — Lesson 0018: API Design & Pagination

**Date:** 2026-07-05
**File:** `lessons/0018-api-design-pagination.html`
**Spine topic:** #18 — API design & pagination (now ✅)

## The one system
`GET /feed` — the Lesson 15 home feed, paged to a mobile app that shows ~10 posts
at a time from a feed that grows every second.

## What the lesson covered
- **Estimate — why page at all.** Whole feed = 5,000 posts × 2 KB = 10 MB, ~5 s
  on cellular, mostly wasted. One page of 20 = 40 KB, ~instant. Page size 20–50
  is the **round-trips vs payload** dial.
- **Model — offset vs cursor.**
  - Offset (`LIMIT 20 OFFSET N`) counts *positions* in a shifting list → two bugs:
    (1) inserts above the window push rows down so page 2 re-serves page-1 posts
    (duplicates; deletes → skips); (2) `OFFSET 1,000,000` must read+discard 1M rows
    → page 50,000 does ~1,000,020 row-reads ≈ **50,000×** page 1. O(offset+limit).
  - Cursor (`WHERE (created_at, id) < cursor`) names a *stable place* → sits still
    under writes (new posts sort above it) and is a direct index **seek** → O(limit)
    at any depth (the L12 B-tree seek doing the work offset threw away).
  - **Composite key** needed: a bare timestamp cursor skips or repeats tied rows;
    append the unique `id` to make the sort key total.
  - Trade: random access (offset can jump to "page 7") vs stability & speed (cursor
    only goes next/prev).
- **Trace — REST vs RPC vs GraphQL + the N+1.** REST (resources+verbs, cacheable,
  idempotent GET, but one fixed shape → over/under-fetch); RPC/gRPC (named actions,
  flexible but non-uniform, uncacheable by default); GraphQL (client picks fields,
  kills over/under-fetch — but the resolver reintroduces the **N+1**: 1 page query +
  20 author lookups = 21 queries ≈ 42 ms → batch with one `IN` query = 2 queries ≈
  4 ms, **~10×**, the identical fix from L17's 100 serial feature-store calls).
  Trade: client flexibility vs server complexity.
- **Next bottleneck — the published contract.** Additive changes (new field) are
  backward compatible; rename/remove/re-type/default-change break field clients you
  can't force-update. Versioning (URL `/v2` vs `Accept` header) + deprecation window
  as the last resort. Trade: evolvability vs stability.

## Reuses / callbacks
- L17 N+1 (the batch fix, now designed into the API shape).
- L12 B-tree seek (what a cursor rides, what an offset wastes).
- L15 home feed (the paged resource).
- L04 ring / L09 committed offset ("name a stable place, don't count shifting positions").
- L14/L16 "narrow cheap vs wide expensive" as the page-size dial.

## What it sets up next
- **#19 Geo / proximity systems** — the cursor's 1-D sort key breaks when "next"
  means "nearest in 2-D": geohash vs quadtree vs S2 cells, the cell-boundary
  problem, hot-cell load.
- **#20 Object / blob storage** — chunking, erasure coding vs replication, metadata
  service, the small-file problem.

Spine still has #19–#21 queued, so no new advanced topics added this run.
