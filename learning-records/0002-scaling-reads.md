# Lesson 02 — Scaling Reads: Eviction, CDNs, and the Stampede (shipped 2026-06-21)

Built directly on Lesson 01's URL shortener. Took the "solved" cache and
stress-tested it through the same four moves (estimate → model → trace → next
bottleneck), exposing three cracks:

1. **Capacity / eviction.** Worked the numbers: ~200 bytes/entry, 12 GB usable
   → only ~60M of 6B links (1%) fit in RAM. Saved by **skew** — the hot 1%
   absorbs ~96% of clicks, so a 1% cache yields a 96% hit rate and drops DB
   load from 3,800/s to ~152/s (25×). Eviction rules: **LRU** as the main rule
   (keeps the hot set; adapts fast to bursty virality), **TTL** only as a
   memory/recycle backstop since the mapping is immutable. Contrasted **LFU**
   (remembers longer, forgets fads slower).
2. **Cache stampede / thundering herd.** Traced read-through (cache-aside) and
   then the dangerous path: a hot key (2,000/s) expires; during a 20 ms DB read,
   40 identical queries pile up; synchronized TTLs make it tens of thousands at
   once. Key insight planted: **a cache hides load, it doesn't remove it — so
   ask "what happens the instant the cache is empty?"** Fixes with named
   trade-offs: single-flight/request coalescing, TTL jitter, stale-while-revalidate.
3. **Distance.** Next bottleneck is the speed of light (~100 ms Tokyo↔Virginia).
   **CDN/edge/PoP** brings it to ~5 ms — but caching the redirect at the edge
   re-raises the **301-vs-302 analytics trade-off** from Lesson 01, one layer out.

## Sets up next
- **Lesson 03 — Scaling writes:** sharding, partitioning, distributed ID
  generation (one counter stops being enough).
- The cache-**invalidation** problem has now been explicitly dodged twice
  (immutability). It comes due when values can change → consistency & replication.
- Added advanced topics 7–12 to NOTES spine (consistent hashing, rate limiting,
  message queues, leader election, CAP, indexing/search) so the course never runs dry.

## Learner baseline
Still no signal on difficulty. Lesson 02 raised density a notch (more math,
more named trade-offs). Watch quiz/follow-ups; adjust if "too easy"/"too hard".
