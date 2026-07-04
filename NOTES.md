# Teaching Notes — System Design track

## Learner
- Wants to learn **system design via worked examples** (asked 2026-06-21).
- Cross-track preference (see global memory): **hard ideas, simple words** —
  difficulty from the concept, never the prose.

## How to teach (conventions carried from the VidoFlip + junior-to-senior tracks)
- **Worked examples first.** Each lesson takes ONE concrete system and traces
  it end to end. Concrete example BEFORE the abstract rule.
- **Dark mode, self-contained HTML.** Palette: bg `#0e0e16`, card `#16161f`,
  ink `#e8e8f2`, accent `#a594ff`. Embedded CSS + tiny JS quiz with feedback.
- One orientation picture up top. Short declarative sentences. Define/earn every
  term before leaning on it.
- Print the `open` command at the top of each lesson. End every lesson by
  inviting follow-up questions.
- Keep `reference/glossary.html` aligned with the terms used.

## Calibration (starting assumptions — confirm as they answer)
- General system design (not VidoFlip-specific). Interview-flavoured but the
  goal is a real mental model, not memorized answers.
- Unknown baseline. Lesson 01 (URL shortener) pitched at: assumes programming
  literacy, teaches estimation + caching reasoning from scratch. Adjust density
  up if they say "too easy" (mirror the VidoFlip track's "raise the baseline").

## Curriculum spine (bottom-up, revise as we go)
1. ✅ **URL shortener** — estimate, data model, counter+Base62, read/write
   paths, 301vs302, first bottleneck → cache + the 4 properties that justify it.
   (Lesson 0001)
2. ✅ **Scaling reads** — cache size vs hit rate (skew), LRU/LFU/TTL eviction,
   read-through/cache-aside, cache stampede + single-flight/jitter/SWR, CDNs &
   the speed-of-light bottleneck. (Lesson 0002)
3. ✅ **Scaling writes** — the single counter's throughput ceiling vs the
   single disk's storage ceiling (two separate walls), distributed ID
   generation (ID blocks/ticket server vs Snowflake bit-budget vs UUID), hash
   vs range sharding + hotspots/scatter-gather, deterministic routing, and why
   `% N` resharding moves ~94% of keys → consistent hashing. (Lesson 0003)
4. ✅ **Consistent hashing** — keys and nodes on one `2³²` ring, clockwise
   ownership so adding a node splits only one arc (~1/N moves, not ~94%);
   virtual nodes for even load (~1/√V) and failure that spreads instead of
   dumping on one neighbor; hot keys + heavy rebalance named as what it does
   NOT solve. (Lesson 0004)
5. ✅ **Async work** — the 2M-link bulk API can't run inline (~2.8 h vs ~60 s
   timeout); producer→queue→workers, visibility timeout so a crashed worker's
   message reappears, at-least-once delivery → duplicates fixed by idempotency
   keys (not "exactly-once"), poison messages → retry limit + DLQ, fan-out with
   per-key ordering via partitions, and consumer lag/backpressure as the next
   wall. (Lesson 0005)
6. ✅ **Consistency & replication** — a like-counter under concurrency: "+1"
   is a read-modify-write whose gap races into lost updates (fix: atomic
   increment / compare-and-set); replication scales reads but adds lag →
   read-your-writes broken; strong vs eventual; quorums (W+R>N guarantees
   overlap); next wall = still-hot single row → sharded/approximate counter
   (trade exact read for scalable write). (Lesson 0006)
7. ✅ **Designing for failure** — one timed-out call traced end to end: a
   timeout is a partial failure (work may have happened), so set it from
   p99/p99.9 not a lazy 1 s default (which collapses the thread pool 100×); a
   blind retry duplicates → idempotency key makes retries safe; backoff + jitter
   + retry budget stop the retry storm / metastable failure; circuit breaker
   fails fast on a dead dependency; next wall = unbounded queue → backpressure +
   load shedding (honest 429). (Lesson 0007)

8. ✅ **Rate limiting** — the free-tier API key (60/min): size the limit as a
   sustained rate + burst against the downstream ceiling; the fixed-window
   boundary bug (120 across the seam, promised 60) → sliding window; token
   bucket (capacity=burst, refill=average; burst once then clamp to r) vs leaky
   bucket (smooth drip, burst latency); next wall = the 10× per-server counter
   leak → centralized atomic counter (lost-update race) → shard/approximate
   (accuracy vs scalability). (Lesson 0008)

### Advanced topics (queued so the course never runs dry)
9. ✅ **Message queues** — one 100k-events/sec click stream through a log:
   estimate (20 MB/s, 12 TB/7d, 20 partitions = parallelism vs order); log vs
   mailbox (events kept, per-group committed offset, the work→commit danger gap);
   three delivery semantics (at-most-once loses / at-least-once duplicates /
   "exactly-once" = at-least-once + idempotency key or transactional offset+output,
   no exactly-once on the wire); partition-by-key for per-key order → hot partition;
   next walls = consumer lag (the number to alarm on), poison message blocking a
   partition → DLQ, rebalance storm → coordination. (Lesson 0009)
10. ✅ **Leader election & coordination** — five nodes must run a billing job
    exactly once: heartbeat (1 s) + timeout (3 missed = 3 s) sizes a ~3.5 s
    failover, and too-short a timeout makes leadership flap (the rebalance storm);
    terms + one-vote-per-term + majority quorum (⌊N/2⌋+1 = 3 of 5); split-brain
    under partition is impossible because two majorities of 5 must overlap and the
    shared node can't vote twice → minority steps down (safety over availability);
    the zombie/slow leader fenced by an epoch (fencing token) the resource rejects
    if lower; odd N because even buys no extra fault tolerance (N−quorum); leans on
    a coordination service (ZK/etcd/Consul, Raft/Paxos). (Lesson 0010)
11. ✅ **CAP in practice** — one account balance ($100), three replicas, a cut
    trans-atlantic cable: partition tolerance isn't optional (the network imposes
    it), so CAP collapses to "choose C or A, and only while partitioned"; quorum
    `W+R>N` overlap (pigeonhole, recap of L06/L10) makes reads linearizable and
    the split forces the dilemma onto the *minority* side; CP refuses a non-quorum
    write (errors, but the double-spend is impossible — L10's "minority steps down"
    for data) vs AP answers both sides (divergence → a $60k last-write-wins
    reconciliation bill; right for a mergeable cart, wrong for a scalar balance);
    next wall = the choice CAP hides → **PACELC**, the everyday ~90 ms
    latency-vs-consistency tax, and "AP is only as good as your ability to
    reconcile" → idempotency. (Lesson 0011)
12. ✅ **DB indexing & search** — one 100M-row orders table, three queries:
    estimate the no-index scan (20 GB ÷ 500 MB/s = 40 s for 1 row) vs ~0.4 ms
    indexed; build the B-tree from page size (8 KB ÷ 16 B ≈ 500 fan-out → 500³ =
    125M > 100M so 3 levels, depth = log₅₀₀ rows grows slowly); B-tree keeps order
    (ranges/sort) vs hash (= only) — L03's hash-vs-range trade inside one table;
    secondary index = a second hop (bookmark lookup) → covering index (storage for
    latency, L02 shape); full-text "blue running shoes" → inverted index (tokenize/
    stem → postings lists → intersect, not a 10M scan); next walls = write
    amplification (5 indexes → 6× writes) and the selectivity trap (status matches
    50M → index 125× slower than scan, planner ignores it). Break-even ≈ 0.4% of
    table. (Lesson 0012)
13. ✅ **Idempotency** — one `POST /charge $50` retried after a timeout turns
    $50 into $100; price it (10M/day × 2% dup = $10M/day) and use Little's Law
    (L=λW≈93) to show retries race a live original; the fix = client-chosen
    idempotency key + durable dedup store (`key→status,response`) checked before
    the card network; trace happy path, retry-after-success (replay saved
    response), the concurrent-retry check-then-act race (closed by an atomic
    unique-key claim, L06's race again), and the crash-in-the-middle (commit
    effect+record in one transaction, or push the key downstream); exactly-once
    *effect* on at-least-once *delivery* (no exactly-once on the wire); next
    walls = the TTL trap (forget too soon → late redelivery double-charges; too
    long → bloat) and natural idempotency (absolute PUT/delete/conditional needs
    no key). (Lesson 0013)
14. ✅ **CDN design** — one 200 KB photo in Virginia, one shopper in Sydney
    15,500 km away: the speed-of-light tax (~200 ms RTT × 4 handshake/request
    round trips ≈ 800 ms cold) that no faster origin can beat, and the egress
    bill (2 PB/day × $0.08/GB = $160k/day) — both fixed by a nearby copy;
    model = edges keyed by host+path+query (a stray tracking param mints a key
    per request → hit ratio → 0; strip/whitelist), per-content TTL whose payoff
    rides on popularity (hit ratio ≈ 1−1/R, hot 99.9% / cold ~0%, Zipf recap of
    L02); trace the hit (~20 ms, origin untouched, ~95% offload → 20× bill cut),
    the miss (one-time regional tax), and the synchronized-expiry miss storm
    (L02 thundering herd × #PoPs) collapsed 300→1 by an origin shield + request
    coalescing; next wall = invalidation — purge (direct but eventually
    consistent + stampede-prone) vs versioned immutable URLs (rename, cache a
    year, nothing to invalidate; default for anything renameable, purge as the
    escape hatch), plus the cold long tail and un-cacheable personalized content
    (cache the shell, personalize the slice). Trade named throughout: freshness
    vs speed. (Lesson 0014)
15. ✅ **Notification/feed fan-out** — one post to every follower's home feed:
    a read-heavy app (100M DAU, follow ~200, post 2×/day, read 5×/day → 2.5:1)
    so precompute the read = push (fan-out on write, ~40 B cheap inbox appends/day,
    O(1) read) beats pull (fan-out on read, ~100 B fan-in fetches/day); model push
    as durable post + async fan-out worker (queue, idempotent appends → eventual
    consistency, read-your-writes special-cased); inbox stores ids not bodies (~1.25 TB);
    trace the cheap push read (1 lookup + batched hydrate) vs the 200-way pull gather;
    the celebrity breaks push (1,000 accounts × 5M followers = 25 B writes/day = 60%
    of fan-out from 0.0025% of posts, spiky + wasted on inactive followers) → HYBRID:
    push normal, pull celebrities (cached once per celeb), merge at read; follower
    THRESHOLD = the dial between read fan-in and write amplification; next walls =
    ranked feeds (precompute candidate set, rank at read), deletes (tombstone +
    filter-at-read), inactive-user tax, cold-start backfill. (Lesson 0015)
16. ✅ **Search systems (ranking)** — one query "red running shoes" over 10M
    products: L12's inverted index found ~600k OR-matches, but the shopper sees
    10 so the ORDER is the product; recall vs precision (AND=precise/misses,
    OR=broad/noisy → retrieve broad, rank to restore precision). Model the
    score: IDF=log(N/DF) makes rare "shoe"≈2.40 beat "red"≈1.30, "the"≈0.05
    (stopwords fall out free); TF-IDF=Σ tf×idf, but unbounded TF lets a
    keyword-stuffed title (12.48) crush a perfect match (5.62) → BM25 patches
    with saturating TF (tf·(k+1)/(tf+k), k=1.2: tf 1→4 only 1.00→1.69) +
    length normalization. Trace parse-must-mirror-index, retrieve→score→top-K
    heap (N log 10 not N log N), and text-ties broken by blended signals
    (popularity/recency); next walls = ML ranker at 1 ms/doc × 600k = 10 min
    → two-phase (cheap BM25 narrows to ~1k, expensive re-rank on those, the
    L12/L14 "narrow cheap, perfect expensive" funnel) and near-real-time
    indexing (freshness vs cost, L06 lag / L14 TTL again). (Lesson 0016)
17. ✅ **Distributed tracing & observability** — L16's search request (mean
    44 ms, p99 950 ms, every service's dashboard green): which of 20 hops ate
    the time? Estimate the firehose (50k req/s × 20 spans × 500 B = 43 TB/day →
    must sample); model the request as a tree of spans bound by one trace ID,
    reassembled from parent pointers, made distributed by trace-context
    propagation (traceparent header) whose weakest hop is a blind spot; the mean
    lies (99×35+950 → 44 ms hides the 950 ms tail) so alarm on p99; read the
    waterfall to expose an N+1 of 100 serial 8 ms feature-store calls (800 ms →
    one 8 ms batch = 6× cut); 1−0.99²⁰ ≈ 18% shows fan-out amplifies the tail;
    next walls = head-based sampling keeps the slow trace only 1-in-10,000 →
    tail-based sampling (buffer + centralize, keep slow/errored) and metric
    cardinality (ids go on traces, never metric labels). Three pillars: metrics
    say something's wrong, traces say where. (Lesson 0017)
18. API design & pagination — REST vs RPC vs GraphQL, offset vs cursor paging
    (why offset breaks under inserts), versioning, and the N+1 problem.
19. Geo / proximity systems — "find drivers near me": geohash vs quadtree vs
    S2 cells, the cell-boundary problem, and hot-cell load.
20. Object / blob storage — how to store 100 PB of files: chunking, erasure
    coding vs replication, metadata service, and the small-file problem.
21. Analytics & counting at scale — approximate structures (HyperLogLog for
    unique counts, Count-Min sketch, Bloom filters): trade exactness for memory.

## Lesson format conventions
- Four reusable "moves" framing introduced in Lesson 01: estimate → model →
  trace the paths → find the first bottleneck. Reuse this spine across lessons.
- Every design choice should NAME its trade-off explicitly (that's the skill).
