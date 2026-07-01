# Learning record — Lesson 0014: CDN design

**Date:** 2026-07-01
**File:** `lessons/0014-cdn-design.html`
**Spine topic:** #14 — CDN design (edge caching, cache keys, invalidation, origin shielding)

## The worked example
One 200 KB product photo stored at an origin in Virginia; one shopper in
Sydney ~15,500 km away; 10 billion asset requests/day. Traced end to end with
the four moves.

## What the lesson covered
- **Estimate — the two problems.**
  - *Latency floor:* light in fiber ~200,000 km/s → 15,500 km ≈ 78 ms one way,
    ~200 ms real RTT; a cold HTTPS fetch is ~4 RTTs (TCP 1 + TLS1.2 2 + request
    1) ≈ 800 ms. An edge at ~5 ms RTT → ~20 ms. A faster origin CPU can't beat
    distance.
  - *Bandwidth/bill:* 10e9 × 200 KB = 2 PB/day = ~185 Gbps; at $0.08/GB that's
    $160k/day all-origin. 95% hit ratio → origin egress 5% = $8k/day (20× cut),
    origin bandwidth ~9 Gbps.
- **Model — cache key + TTL.** Key = host+path+query; a per-click tracking param
  in the key mints one entry per request → hit ratio → 0 (fix: strip/whitelist).
  Precision-vs-hit-rate dial; `Vary` for encodings. TTL = freshness vs offload;
  hit ratio ≈ 1 − 1/R where R = requests per TTL window → hot 99.9% / cold ~0%
  (Zipf skew, L02).
- **Trace paths.** Hit (~20 ms, origin untouched, ~95%); miss (one-time regional
  tax); miss storm (synchronized expiry → 300 PoPs stampede origin = L02
  thundering herd × #PoPs) collapsed 300→1 by an **origin shield** (tiered cache)
  + request coalescing (single-flight).
- **Next bottleneck — invalidation.** Purge (direct, but eventually consistent
  across ~300 PoPs + stampede-prone) vs versioned immutable URLs (content-hash
  filename, cache a year, nothing to invalidate — default; purge as escape hatch
  for the un-renameable HTML + takedowns). Plus the cold long tail and
  personalized/dynamic content: cache the shared shell, personalize the slice.

## Trade-offs named
- A second copy of the truth: speed bought with staleness.
- Cache key: precision (correct) vs breadth (high hit ratio).
- TTL: freshness vs origin offload.
- Shield: an extra hop on a miss vs an origin that survives the herd.
- Purge (universal, eventually consistent) vs versioned URLs (instant, only for
  renameable files).

## Builds on
L02 (cache, speed-of-light, thundering herd, single-flight, Zipf skew),
L05 (spike-absorbing layer in front), L13 ("do the work once, near the request").

## Sets up next
- #15 Notification / feed fan-out — push-on-write vs pull-on-read (the same
  "before the request or during it?" trade, now for timelines).
- #16 Search systems — TF-IDF / BM25 ranking; the cache-key idea returns as
  "which queries share a cached result set?"
