# Learning record — Lesson 0015: Feed fan-out

**Date:** 2026-07-02
**File:** `lessons/0015-feed-fanout.html`
**Spine topic:** #15 — Notification/feed fan-out (push vs pull, fan-out-on-write vs read)

## The worked example
A social app home feed: when Ada taps Post, her update must reach every
follower's feed; when Ben opens the app, his feed is built from the ~200
accounts he follows. One choice: do the work at write time (push into every
inbox) or read time (gather from every followee). Traced with the four moves.

## What the lesson covered
- **Estimate — which side to precompute.** 100M DAU, follow ~200, post 2×/day,
  read 5×/day → 200M posts/day, 500M reads/day, read:write ≈ 2.5:1 (read-heavy →
  precompute the read = push).
  - *Push total work:* 200M × 200 = 40 B inbox appends/day (~463k/s, tiny appends);
    reads = 500M/day, each one O(1) inbox lookup.
  - *Pull total work:* writes 200M/day (~2.3k/s, free); reads 500M × 200 = 100 B
    fan-in fetches/day (~1.16M/s scatter-gather + merge/sort).
  - Push = ~40 B heavy ops vs pull ~100 B, and a fast read → push wins for the
    average user. Inbox stores ids (post+author ≈16 B) not bodies: 800 × 16 ×
    100M ≈ 1.25 TB.
- **Model — inbox + async fan-out.** Post path: synchronous durable INSERT →
  return "Posted" → enqueue fan-out job → worker does N idempotent appends
  (Lesson 05/09 queue, Lesson 13 idempotency). Trade = eventual consistency on
  feeds (fine for social); read-your-writes special-cased (show author their own
  post from the posts table).
- **Trace paths.** Push read = 1 inbox lookup + batched hydrate of ~50 ids (2
  round trips). Pull read = 200-way IN-query + merge/sort (the 100 B monster).
  Post path breaks for a **celebrity**: 1,000 accounts × 5 posts × 5M followers =
  25 B writes/day = ~60% of all fan-out from 0.0025% of posts, arriving as spikes,
  ~80% wasted on inactive followers.
- **Next bottleneck — the hybrid.** Route by follower count: push below a
  THRESHOLD, pull above it; read merges pushed inbox + celebrity fetches (cached
  once per celebrity, shared by all fans → hot key as a feature). Threshold =
  the dial between read fan-in cost (too low) and write amplification (too high);
  real systems ~tens-of-thousands–1M, can decide per-read. Follow-on walls:
  ranked feeds (precompute candidate set, rank at read), deletes (tombstone +
  filter-at-read), inactive-user tax (fan out to active only, backfill lazily),
  new-follow cold start (pull to backfill).

## Trade-offs named
- Push vs pull = write-time work vs read-time work (precompute vs on-demand,
  same dial as CDN L14 and covering index L12).
- Async fan-out = fast post vs eventual consistency on feeds.
- Push = read speed bought with (unbounded, at the tail) write amplification.
- Celebrity threshold = read fan-in vs write amplification.
- Ranked feed = precompute membership vs compute order on demand.
- Deletes = fan-out-the-delete (storage) vs filter-at-read (read-time work).

## Builds on
L02 (cache, hot keys, thundering herd), L03 (time-sortable ids), L05/L09
(queues + workers, async), L06 (read-your-writes, eventual consistency),
L13 (idempotent retries), L14 (precompute near the reader).

## Sets up next
- #16 Search systems — TF-IDF/BM25 ranking; the "precompute the candidate set,
  compute the order on demand" split reappears.
- #17 Distributed tracing — when a feed read fans in to many services, find the
  slow (p99) hop.
