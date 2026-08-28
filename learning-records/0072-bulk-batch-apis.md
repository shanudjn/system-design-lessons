# Lesson 0072 — Bulk & Batch APIs: Collapsing 200 Round Trips Into Two

**File:** `lessons/0072-bulk-batch-apis.html`
**Spine topic:** #72 (makes concrete L18's throwaway "fetch each item separately and you re-create the N+1")
**Date:** 2026-08-28

## Worked example
A mobile home feed shows **200 posts**; each post carries an `author_id` (not a denormalized name — L12/24), and the screen must render each author's **name + avatar**. The naïve client loops `GET /users/{author_id}` → **1 feed call + 200 author calls = 201 round trips**. Goal: turn `1 + N` into a small constant without giving up the freshest name.

## What it covered
- **Estimate — round trips, not work, are the killer.**
  - *Network:* mobile RTT ~100 ms. Sequential 201×100 = **20.1 s**; 6-way parallel ⌈200/6⌉=34 +1 feed = 35 rounds = **3.5 s**; HTTP/2 all-in-flight → transport ~1–2 RTT BUT server runs 201 handlers ≈ **700 ms** CPU + 200 pool slots. The floor is count × RTT, independent of the trivial ~3 ms/call work → no transport trick saves it.
  - *Payload:* full user ~2 KB, need ~40 B (name+avatar). 200 full = **400 KB** vs 8 KB needed = **50×** over-fetch.
  - *Target:* `POST /users:batchGet {ids, fields}` → one `SELECT ... WHERE id IN (...)` (~60 distinct after dedup, ~2 ms, 8 KB) → **2 round trips ~200 ms**, ~100× fewer trips.
- **Model — three collapses.**
  - **Batch endpoint / multiget** — client bundles + DEDUPS ids (200 posts → ~60 distinct authors), one set query, `1+N`→`1+1`; response is a map to re-join; costs API surface + needs cap + partial-failure.
  - **Request coalescing / DataLoader** — same collapse invisible to per-item code / GraphQL resolvers: collect a tick's ids, dedup, one batched query, per-request cache. The fix GraphQL's per-item resolvers REQUIRE.
  - **Field selection** — `?fields=` / GraphQL query names exact fields → kills over-fetch AND under-fetch with one shape. Batching(round trips) ⟂ field selection(payload) → want BOTH (a batch of 200 *full* users = 1 trip but 400 KB).
- **Trace.** (A) N+1 unfolds: 201 trips, ~140 duplicate author fetches, ~700 ms wasted CPU. (B) batch erases it: 2 trips, one IN-query, deleted user comes back as `null` (per-item, not a batch failure). (C) GraphQL `{feed{author{name}}}` = 1 client call but the per-post `author` resolver fires 200× (N+1 moved server-side, invisible) until a DataLoader coalesces the loads into one query.
- **First bottleneck — a batch is a new unit of failure, latency & load.**
  1. **Unbounded batch = self-inflicted DoS** (100k ids → slow mega IN-query + `100k×2KB` = **~200 MB** response + OOM) → size cap (100–1,000) + paginate the batch (L18/28); cap size is the dial.
  2. **One status/latency for the whole batch** sinks 199 good rows on one bad id + waits for the SLOWEST item → per-item results (L32) + tail latency (L17).
  3. **Batching the wire ≠ batching the work** — a per-id server loop just relocated the N+1 (often serialized/slower) → one `WHERE id IN` or per-shard **scatter-gather** (L3/19, local compute + central union).
  4. **Batching burns the cache** — 200 individual GETs are edge/client-cacheable (L2/14, popular author served to all), one POST batch is not → cache-friendly `GET ?ids=` (sorted) or client-side coalescing.
- **Deepest point.** The N+1 is a **boundary-crossing** bug: each fetch is cheap, crossing the network boundary 200× is not (fixed RTT floor dwarfs the work). Cure = move the fan-out to the cheap side (one IN / scatter-gather inside the DC) and cross the expensive client↔server boundary ONCE. Batching / DataLoader / GraphQL+loader = three spellings of that idea.

## Numbers used (all checked, `python3`)
- 201 × 100 ms = 20,100 ms ≈ 20 s (sequential); ⌈200/6⌉=34, +1 = 35 rounds × 100 ms = 3.5 s (6-way parallel).
- Server overhead 200 × 3.5 ms = 700 ms.
- Payload: 200 × 40 B = 8 KB; 200 × 2 KB = 400 KB → 50× over-fetch.
- Batch screen: feed + batch = 2 trips ≈ 200 ms; 201/2 ≈ 100× round-trip reduction.
- DoS: 100,000 × 2 KB = 200,000 KB = **~200 MB** (≈0.2 GB) response.

## Threads reused
L18 (paged feed, GraphQL resolver N+1, field selection), L17 (tail latency + N+1 of serial feature-store calls), L12 (B-tree IN-query seeks), L3/4 (sharding — why a batch across a sharded key is a scatter-gather), L19/21 (scatter-gather + merge, local compute/central union), L2/14 (edge & client caching the batch gives up), L28 (unbounded batch = memory/DoS), L32 (partial failure — some items succeed, some fail).

## Sets up next
- **Lesson 73 — Deduplication & entity resolution at scale:** "are these two records the same customer?" across 500M rows; N² pairwise (≈1.25×10¹⁷) is impossible → **blocking keys** shrink the candidate set, **similarity scoring** ranks, **MinHash/LSH** (L65 neighbours) finds near-dupes without N², **survivorship** decides the merge winner. Trade: match recall vs precision & compute cost.
- Then Lesson 74 — config & feature-flag delivery at scale.
- Spine still has 73–75 queued, so no new topics added this run.
