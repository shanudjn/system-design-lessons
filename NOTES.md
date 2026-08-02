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
18. ✅ **API design & pagination** — one `GET /feed` endpoint paged to a phone:
    estimate why you page (5,000×2 KB = 10 MB / ~5 s → a 40 KB page; size 20–50
    is the round-trips-vs-payload dial); offset counts positions in a shifting
    list → duplicates under inserts + O(offset) cost (page 50,000 reads ~1,000,020
    rows = 50,000× page 1) vs a cursor that names a stable place `(created_at, id)`,
    sits still under writes and seeks in O(limit) at any depth (L12 B-tree seek),
    needing a composite key so ties don't skip/repeat; REST (resources+verbs,
    cacheable, over/under-fetch) vs RPC (named actions) vs GraphQL (client picks
    fields → resolver reintroduces the N+1, 21→2 queries ≈10×, L17's batch fix);
    next wall = the published contract — additive changes safe, removals/renames
    break clients → versioning (URL vs header) + deprecation window. Trade:
    client convenience vs server cost & stability. (Lesson 0018)
19. ✅ **Geo / proximity systems** — "find drivers within 2 km" out of 1M moving
    cars: checking every driver is 5×10¹⁰ dist-calls/sec and a 1-D lat index
    returns a 4 km strip wrapping the planet (two 1-D indexes don't intersect into
    a 2-D box) → fold 2-D to 1-D with a space-filling curve; geohash interleaves
    lat/lng bits (point → 01001 10110 → "9q…") so nearby points share a prefix and
    a plain B-tree (L12) answers "who's near?"; the cell-boundary problem (a car
    100 m away in a different cell shares no prefix) → query the 3×3 neighbourhood
    + exact haversine on ~900 candidates (1,100× cut); cell size = recall vs work
    (≈ radius); quadtree adapts cell size to density (stateful vs geohash's
    stateless uniform grid); S2 = cube + Hilbert curve (better locality/uniformity
    than geohash's Z-order); next wall = the hot cell (stadium empties → 50k in one
    cell = a hot shard, L03/L08/L09 on a map) → adaptive precision / cap / replicate.
    (Lesson 0019)
20. ✅ **Object / blob storage** — one 100 PB pile of dashcam clips (avg 100 MB
    → 1 billion objects): a filesystem drowns in a billion-entry tree and 3×
    replication burns 300 PB to survive only 2 failures → an object store's flat
    `key→blob` namespace + a metadata plane (1 TB, consistent) split from the
    data plane (100 PB, durable — a 100,000× size gap justifies the split);
    chunk big objects (64 MB) for parallelism/repair vs metadata overhead;
    **erasure coding** RS(10,4) survives 4 losses on 140 PB (+40%) where matching
    it with copies needs 5× = 500 PB (~3.5× more) → saves 160 PB ≈ $38M/yr for
    MORE durability, paid in parity CPU + 10× repair amplification (rebuild 1
    fragment by reading k=10); replication still wins for hot/latency-critical
    (single-fetch, 1× repair) vs EC's read fan-out tail (L17). Trace the write
    (chunk→code→scatter 14 frags→commit metadata LAST, the L13 durable-commit)
    and read (fetch any 10, decode only if a data frag is missing); next wall =
    the **small-file problem** (4 KB objects → 25 trillion of them → 1 TB index
    balloons to 25 PB, 400 B fragment splinters, because costs are per-OBJECT not
    per-byte) → pack into container blobs + compact in-RAM index + tombstone/
    compaction deletes (L15). Trade named: durability per raw byte. (Lesson 0020)
21. ✅ **Analytics & counting at scale** — three dashboard questions over a
    billion-car / 25-trillion-object stream: exact answers scale with distinct
    items (distinct set 8 GB, per-car counters 16 GB, seen-keys set 400 TB) →
    swap for sketches on one hash-and-summarize engine. HyperLogLog counts
    cardinality from leading-zeros across m=2¹⁴ registers × 6 bits ≈ 12 KB at
    1.04/√m ≈ 0.81% error (~650,000× smaller); Bloom filter = membership with NO
    false negatives (m/n = −ln ε/(ln2)² ≈ 9.6 bits/key, k≈7, ~1.2 GB/1B keys) so
    "definitely not" skips the index lookup (L12/L20 guard); Count–Min = frequency
    in d×w ≈ 76 KB, min-of-d only ever OVER-estimates (bounded ε·N → great for
    heavy hitters, noisy in the tail). All three MERGE across shards (HLL max /
    CMS add / Bloom OR) → local compute, central union. Next walls = a sketch
    tells you HOW MANY, never WHICH (fix: CMS + min-heap for top-K), append-only
    (counting Bloom for deletes = 4× bits), and per-time-window sketches. Trade:
    a small chosen error for constant memory, accuracy placed where it matters.
    (Lesson 0021)

### Advanced topics (next batch — queued so the course never runs dry)
22. ✅ **Distributed locks & leases** — five workers, one nightly billing job that
    must charge each customer exactly once: a lock = atomic set-if-absent (SET NX PX),
    made a **lease** (TTL + heartbeat renew) so a dead holder can't block forever —
    but the lease OPENS the two-holder window. Size the TTL (heartbeat 5s → TTL=3h=15s
    → ≤15s failover; too-short TTL false-evicts a live-but-slow holder = flap, L10);
    no finite TTL survives an unbounded stop-the-world GC pause, so TTL is a LIVENESS
    knob, never safety. The "held lock but paused GC" hazard traced (A freezes 22s,
    lease expires, B acquires token 42 & charges, A wakes at t=28 & charges again =
    double-charge) → **fencing token** (monotonic counter, L10's epoch applied to DATA):
    the RESOURCE keeps highest_token_seen and rejects any write with a lower one, so
    A's token-41 write is fenced out — no clock needed. Release must be compare-and-delete
    (L06 CAS) or you delete someone else's lock. Next walls = the lock is a serial gate
    (1 shared lock @ 5ms = 200 ops/s = 13.9h for 10M), a SPOF wanting consensus (L10),
    and clock-skew-fragile → AVOID the lock: idempotency (L13, duplicates harmless) or
    single-writer routing (L03/L04/L10). Efficiency lock (double-run wasteful → lease ok)
    vs correctness lock (double-run corrupts → need fencing). Trade: safety vs liveness.
    (Lesson 0022)
23. ✅ **Multi-region / active-active** — one shopping cart served live from
    Virginia + Frankfurt: a synchronous cross-region write costs ~90 ms (6,400 km
    speed-of-light floor = 64 ms, L02/L14) vs ~1 ms local (90× tax) while a 2nd live
    region cuts downtime 8.76 h/yr → ~32 s/yr (six nines, L07); so write locally +
    replicate async (L11's AP: available, tolerate divergence). No single writer →
    conflicts you must resolve: LWW (one timestamp compare, silently DROPS the loser,
    trusts a skewable wall clock, L22) vs a CRDT (merge commutative/associative/
    idempotent → converges w/ zero coordination). Trace: two adds UNION cleanly;
    add-vs-remove same item = genuine conflict whose fix is a business POLICY
    (add-wins cart) not a fact; LWW under 50 ms clock skew drops the newer write;
    counter as PN-counter (per-region sub-counts, merge=sum) keeps both +1s → 3, where
    a register loses one (L06 lost update across an ocean); version vector tells
    concurrent from after w/o clocks. First wall = the un-mergeable global INVARIANT:
    $100 balance withdrawn $80 (VA) + $70 (FR) CONVERGES to −$50 — CRDT agreed
    perfectly, invariant broke (convergence ≠ invariant preservation, L11's scalar
    balance) → single-home the key (L03/04/10/22, remote writes pay 90 ms) / sync
    consensus (L10/11 CP, blocks on partition) / escrow (split the budget per region,
    L06/08 shard-the-counter on an invariant). Trade: local writes vs global order.
    (Lesson 0023)
24. ✅ **Schema changes & migrations at scale** — split the `orders` table's
    ambiguous `total` (int cents, assumed USD) into `amount_cents` + `currency` on a
    live 100M-row table (50k reads/s, 5k writes/s) with zero downtime. The naive
    `ALTER ... RENAME` is a double trap: a table-rewriting ALTER on 20 GB @ 100 MB/s
    locks ~200 s → queues ~1,000,000 writes (L07 cascade), AND a fleet code deploy is
    gradual so the instant the column renames, every old server's `SELECT total`
    errors — a schema deploy and a code deploy are NEVER simultaneous, and that gap is
    the outage. Fix = **expand/contract** (parallel change): (1) expand schema with
    backward-compatible nullable columns (sub-second metadata lock), (2) **dual-write**
    both shapes, (3) **backfill** old rows in background (~2.8 h @ 10k rows/s), (4)
    **cutover** readers with a plain code deploy (instant, reversible), (5) **contract**
    — stop dual-write, bake, DROP old column. Two ordering rules make it safe: dual-write
    ON *before* backfill (else a gap-write leaves the new col stale and backfill skips
    NULL-only rows), and cut readers over only *after* verifying zero-NULL, keeping the
    old col warm as instant rollback. First wall = the backfill vs live table: one
    100M-row UPDATE locks + rolls back all-or-nothing → batch into 20k committed chunks
    (LIMIT 5k), throttle on **replica lag** (L06), page by **cursor not offset** (L18).
    DROP is the one irreversible step → bake first. Trade: migration safety vs deploy
    velocity. (Lesson 0024)
25. ✅ **Payments & ledgers** — a digital wallet moving $50 from Alice to Bob so it
    can never be lost, doubled, or un-audited. Reject the mutable `balance` column
    (L06 lost update, no history, single corruptible copy of the truth) for the
    **double-entry ledger**: immutable append-only **entries** grouped into
    **transactions** that **sum to zero**, balances *derived* as `SUM(amount)` not
    stored, money entering/leaving via a `bank` **system account** so the whole ledger
    always totals 0 (money conserved). Estimate: 2 entries/transfer → ~7.3B rows/yr ≈
    730 GB (cheap); one duplicate class = 2% × 10M/day × $50 = **$10M/day** wrongly
    moved (an audit, not a hotfix). Trace: two entries in **one atomic commit** (else
    $50 vanishes, books never balance again, guards `balance ≥ 0` = L23 invariant);
    duplicate retry stopped by **UNIQUE(idem_key)** before any entry writes (L13
    exactly-once *effect*, no exactly-once on the wire L09); refund = **compensating
    transaction** appended, never a delete → keeps the "sum all entries = 0"
    reconciliation valid. First wall = hot account balance = `SUM` of 100M entries
    (1.6 GB index scan ≈ **3.2 s**) → **snapshot** running balance (`snapshot + SUM(since)`,
    100,000× cut) + **shard the account** (L21 mergeable counter) for write throughput,
    both strictly *derived* accelerators over an untouched ledger (a corrupted snapshot
    is a stale cache; the rejected mutable column was the only truth). Trade:
    correctness/auditability vs throughput. (Lesson 0025)
26. ✅ **Real-time delivery (WebSocket/push at scale)** — tell 10M phones "you got
    paid" the instant L25's ledger moves, in ~100 ms, without them asking. Estimate:
    10M idle sockets ≈ 200 GB across ~40 gateways @ 250k each (idle conns cheap), and
    polling every 5s = 2M req/s to carry only ~232 real events/s (10M transfers/day ×2
    notified) = ~8,600× waste AND still 5s stale → hold live connections. Model the
    split that makes a stateful layer scalable: a dumb DISPOSABLE **gateway** holds the
    raw **WebSocket** (an HTTP request UPGRADED into a persistent full-duplex socket),
    while the durable truth `user→gateway` lives OFF-gateway in a shared **connection
    registry** with a heartbeat-renewed TTL; **presence** = "is there a live entry?"
    for free. Trace: (A) Bob connects, LB picks any gateway, registry write is the only
    durable effect; (B) Alice pays Bob → dispatcher does registry lookup → forward to
    that one gateway (L09 pub-sub) → one frame down the open socket (~100 ms), fanned
    out to all his devices (L15); (C) Bob offline → NOT dropped, degrade to
    store-and-forward mobile push (APNs/FCM) + durable inbox (L15) — the live socket is
    a best-effort ACCELERATOR, never the system of record (L25's disposable-fast-path).
    First wall = correlated motion: gateway w/ 250k sockets dies → all 250k reconnect
    in ~1s (~6,410/s per survivor = 51× baseline, ~20 cores of TLS handshake, lockstep
    metastable storm, L02/L07 thundering herd) → **backoff + jitter** (L07, → ~1.7×
    baseline, breaks lockstep), **capacity headroom** (L07/L23 N−1 absorbs the dead
    box), **sticky-but-disposable** routing (socket sticky by nature, but any reconnect
    lands anywhere & re-announces). Secondary walls = slow-consumer backpressure (bound
    the per-conn send buffer, L07 unbounded queue) and the registry as a hot dependency
    (shard/replicate, L04/L06). Trade: statefulness vs elastic scale — can't make a
    connection stateless, but make it cheap to lose (truth off the gateway → a death
    costs a reconnect, not a message). (Lesson 0026)

### Advanced topics (next batch — queued so the course never runs dry)
27. ✅ **Autoscaling & capacity planning** — the API tier behind L26's gateways:
    40k req/s peak, 40 ms/req, 8 cores/server. Size the fleet TWICE and make them
    agree: Little's Law (L13) fixes the floor (λ·S = 40,000×0.04 = 1,600 busy cores =
    200 servers at an impossible 100%), and the utilization KNEE (M/M/1 response time
    T = S/(1−ρ): 40 ms → 133 ms @70% but 4,000 ms @99% — flat then vertical) sets the
    real target at 286 servers (the extra 43% ≈ $377k/yr buys distance from the cliff,
    not throughput). Model the autoscaler as a FEEDBACK LOOP (measure ρ → compare to
    ρ*=70% → desired = load/(C·ρ*) → act → cooldown), scale-out fast / scale-in slow,
    wide band + cooldown so it doesn't FLAP (L10's leadership flap in servers). Trace
    three loads on the one comparison that decides all — does load rise slower or faster
    than a server boots: gentle morning ramp (reactive works, the default), sudden 2×
    spike (reactive CAN'T catch it — fleet runs ρ=80k/57.2k=140% for ~3 min, backlog
    grows 22,800/s ≈ 4.1M requests, latency leaves the knee → timeouts → L07 retry
    storm/metastable), known 8:30 surge (beat only by PREDICTIVE/scheduled pre-scaling
    ahead of it). First wall = the ~3-min COLD-START lag = a control loop's DEAD TIME
    (minutes vs seconds-long spikes; you can't zero boot time) → attack the GAP not the
    boot: spike-buffer headroom (run 60% → 333 servers, absorbs +16% to 46,620/s
    instantly = one boot-window of slack), warm pools (pay idle, take traffic in
    seconds), predictive pre-scale (L Path C), faster warm-up (smaller images/snapshot);
    SCALE-TO-ZERO is the opposite extreme of the same dial (cost vs cold-start latency).
    Reuses L13 Little's Law, L07 metastable/retry storm past the knee, L10 flapping.
    Trade: cost vs headroom for spikes. (Lesson 0027)
28. ✅ **Backpressure & flow control end-to-end** — L27's unwinnable 2× spike
    (80k req/s offered, 40k servable for the ~3-min boot window): the naive "buffer the
    overflow" detonates twice — an unbounded queue grows 40k/s → ~144 GB in 180 s (×20 KB
    in-flight) → OOM crash, AND **goodput** collapses to ~0 while **throughput** stays a
    deceptive 40k/s (server pegged finishing requests whose clients already timed out at
    1 s, the goodput cliff). One fix: **bound every queue**, sized from the timeout
    (depth < timeout×rate = 40k → pick 20k → max wait 0.5 s, memory fixed 400 MB, goodput
    restored). A full bounded queue has two moves: **block** the producer (credit-based
    flow control = the "no" as a continuously-advertised number; TCP receive window /
    HTTP-2 window / Reactive Streams request(n)) when you CONTROL the producer, or **drop**
    (honest 429, L07) when you don't — backpressure propagates inward as blocking,
    terminates at the edge as shedding. Trace 3 responses by wasted-work: unbounded
    (crash + L07 retry-storm metastable), bounded-but-shed-deep (survives, but 40k/s of
    rejects already burned edge+API CPU = ~200 stolen cores, L27 knee math), bounded +
    **edge admission control** (downstream fullness → credit signal upstream → 429 the
    excess for ~free → goodput preserved). First wall = WHERE the "no" goes, three traps:
    an unbounded queue hiding at ANY hop reopens the OOM & swallows the signal (bound every
    hop); blocking a shared user-facing pool re-creates L07 thread-pool freeze (bulkhead +
    non-blocking edge); blind shedding wastes goodput (drop retries<first-tries, low<paid
    tier, stale head via FIFO→LIFO; NEVER health checks). Reuses L13 Little's Law (queue
    sizing), L07 timeout/429/retry-storm/metastable, L05/L09 consumer-lag & partition
    queues. Trade: throughput vs tail latency & stability. (Lesson 0028)
29. ✅ **Data warehousing & OLAP** — one analytics question ("revenue by region by day,
    last quarter") over 11B order rows / 50 cols / ~11 TB (L12/L25 orders): the row store
    drags whole 1 KB rows to read 2 of 50 columns → 900 GB → **30 min** for one tile (@ L12's
    500 MB/s), so flip the physical layout. **Columnar** reads only the columns asked for and
    compresses hard (a column = a billion similar values → dict/RLE/delta) — the funnel narrows
    the bytes BEFORE computing (L14/L16): **partition pruning** by day skips 1,005 of 1,095
    partitions (11 TB → 900 GB), **projection** reads 2 of 50 cols (→ 9 GB), compression ~3.6×
    (→ ~2.5 GB), scan @ 500 MB/s across 10 nodes → **~0.5 s** (360× vs the row scan, all from
    reading fewer bytes, not a faster disk). Model: a SEPARATE columnar **warehouse** (analytics
    & serving have opposite shapes AND opposite failure modes — a 900 GB scan evicts serving's
    hot cache & trips L27's knee → L07 retry storm; never share one DB), **partitioned** for
    pruning + **denormalized** into a **star schema** (fact + dimensions; precompute the join =
    L15's precompute-the-read; normalize for writes, denormalize for reads — normal form is a bet
    on write:read ratio), fed by **ETL/ELT** so the warehouse is a *copy*, hence *stale*. Trace
    the load (order queryable for ops in ms, enters analytics only at the next batch) and the scan
    (prune → project → scan → aggregate-and-**merge** partials = L21). First wall = staleness:
    "batch more often" loses (cost ∝ history × frequency; the 6.1 h full re-scan can't fit a 5-min
    window) → freshness needs a layer whose cost scales with NEW data → **lambda** (batch +
    speed/stream layers merged at read; pays in SAME logic maintained twice) vs **kappa** (one
    streaming pipeline, reprocess history by **replaying** the L09 log; pays in expensive replay).
    Three traps: point lookup on a scan engine (L12's selectivity trap REVERSED — send point reads
    to OLTP, aggregates to the warehouse), over-partition by minute → 1.58M small files → pruning
    dies (L20's small-file problem), late/out-of-order events break streaming windows → **watermark**
    holds windows open for stragglers (L17's tail/late-arrival). Trade: query freshness vs cost &
    complexity. (Lesson 0029)
30. ✅ **Secrets, keys & encryption at rest/in transit** — L29's warehouse concentrates 3
    years of payments (L25) + every customer record into one ~11 TB vault (~100k files).
    Naive one-key design fails twice: a leak exposes 100% of the vault, and rotation =
    decrypt+re-encrypt 22 TB @ 500 MB/s (L29) ≈ 12 h. Fix = **envelope encryption**: bulk
    data under per-file **DEKs** (AES-256-**GCM** = AEAD, confidentiality + tamper-detect),
    each DEK **wrapped** by a **KEK** that lives ONLY in a **KMS**/HSM and never touches a
    host → leaked DEK = 1 file (1/100,000), rotate KEK = re-wrap ~10 MB of DEKs, data never
    moves (~2,000,000× less touched). KMS = L10's coordination point in a security hat
    (never releases the KEK → compromised host is scoped/revocable/audited; must be HA).
    **TLS/mTLS** in transit, terminate everywhere not just the edge. Trace write (generate→
    encrypt→wrap→store envelope→wipe DEK) + read (fetch→unwrap→decrypt-or-refuse-on-tamper).
    First wall = KMS unwrap on EVERY read (40k/s = hot rate-limited SPOF, L26) → **cache the
    unwrapped DEK** for a bounded TTL (~6,000× cut); TTL = dial between KMS load and how long
    a stolen/un-revoked DEK stays live (L02 cache dial, but the cached thing is a key). Four
    traps: reuse a GCM nonce (catastrophic), encrypt w/o authenticating (bit-flip, L25),
    secrets in code/config/logs → secrets manager + short-lived creds → **secret-zero**
    bootstrap (bind to workload identity), terminate TLS too early (plaintext internal hops).
    Trade: security blast-radius vs operational friction. (Lesson 0030)
31. ✅ **Deployments & progressive rollout** — push a new version to L27's live fleet
    (286 servers, 40k req/s) without an outage. Big-bang = the all-at-once blast radius
    in release form: a bug meets 40,000×300 s = 12M requests (100% of users) before a
    ~5-min rollback; a 1% canary meets ~24,000 (500× cut) and never reaches the rest.
    Two dials — shrink the exposed slice or the time-to-undo: **rolling** (batches, cheap,
    slow rollback, capacity dip + mixed versions), **blue-green** (two full fleets, flip
    the LB → instant rollback, 2× transient fleet ~$143/h, but all-at-once at the flip),
    **canary + health-gated promotion** (1%→5%→25%→100% ramp gated on error-rate/p99 vs
    baseline, L17; smallest blast radius, costs metrics+bake time+automation), and the
    orthogonal **feature flag** = decouple **deploy** (code running) from **release**
    (behavior reaching users): dark-launch code off, flip on for 1% of users, rollback =
    a ms flag flip, no redeploy. Trace healthy ramp (each stage a cheap checkpoint,
    **drain**/graceful-shutdown before retiring v1, L26) vs poisoned (spike at 1% → gate
    fires → auto-rollback, 99% untouched). First wall = old+new run SIMULTANEOUSLY over
    one shared DB the LB flip can't reverse → rollback is only as fast as the slowest-to-
    reverse layer; safety demands backward/forward compatibility → expand→deploy→bake→
    contract (L24), additive-only contract (L18). Four traps: coupling a destructive
    migration to the deploy (code rolls back, schema doesn't), a thin/unrepresentative
    canary that proves nothing, dropping in-flight work instead of draining, feature flags
    as permanent 2^N test debt. Trade: release velocity vs blast radius. (Lesson 0031)

### Advanced topics (next batch — queued so the course never runs dry)
32. ✅ **Distributed transactions & sagas** — book a trip = reserve flight ($400) + hotel
    ($600) + car ($150) + charge ($1,150) across four DB-per-service boundaries, all-or-none;
    a transaction stops at the edge of its own DB so atomicity must be BUILT. Estimate the
    "hold a lock across all four" (= **2PC**): prepare→commit holds each row the whole ~50 ms
    window vs ~5 ms local = **10×** lock-hold on a hot flight (200→20 bookings/s, L22 serial
    gate); commit availability = the PRODUCT (0.999⁴ ≈ 99.6% ≈ 35 h/yr unbookable); coordinator
    crash after all vote YES but before "commit" = participants **in-doubt, frozen holding
    locks** (L10's coordinator, darker; CP = consistency by blocking, L11). The **saga** = AP
    answer: each step a LOCAL commit (no cross-service lock ever held), on failure walk backward
    running **compensating transactions** (L25) — ordered so pre-**pivot** steps are
    compensatable (cancel a HELD hold) and post-pivot are retriable, so the charge goes LAST
    and you never un-charge. Driven by an **orchestrator** (central, debuggable, holds saga
    state but NO participant locks) or **choreography** (L09 events, no central piece but flow
    smeared/cycle-risk). First wall = a saga has **no isolation** (I of ACID gone): committed-
    immediately half-done state is VISIBLE → dirty read (Y sees a soon-to-be-cancelled hold as
    SOLD) + lost update (two sagas both grab the last seat, L06) → fix NOT by re-adding the
    cross-service lock but by a **semantic lock** (mark HELD/PENDING not SOLD), commutative
    updates, revalidate-at-pivot (L06 CAS); steps + compensations must be idempotent (L13,
    at-least-once L09) and retriable. Four traps: 2PC across hot/autonomous services;
    compensation-as-rollback (it's a business undo, refund≠un-charge, can't un-send email →
    put un-undoable steps after the pivot); non-idempotent steps double-charge; ignoring the
    isolation window. Trade: atomicity vs availability across service boundaries. (Lesson 0032)
33. ✅ **Change data capture & the outbox pattern** — one `orders` service (L12/24/25,
    5,000 writes/s) must save the order AND tell search (L16), cache (L02), warehouse (L29),
    notify (L26/32); doing both is a **dual write** to two systems with no shared transaction
    (L32's wall in miniature) — write-then-publish loses the event on a crash (real order,
    nobody told → 432k silent losses/day at 99.9% publish; one 30 s broker blip orphans 150k),
    publish-then-write announces a ghost; no safe order, and no retry closes the crash-in-the-gap
    window. Fix = collapse two writes into one commit + propagate from a durable log: **transactional
    outbox** inserts an event row IN THE SAME TRANSACTION as the order (L13/25) → a separate
    **relay** publishes unpublished rows out-of-band, at-least-once (L09) → a crash makes the event
    LATE not LOST (the outbox row survives). **CDC** skips the table, tails the DB's own **commit
    log** (WAL/binlog — durable by definition) → every committed change emitted, no app change,
    lowest lag, but RAW row events couple consumers to your physical schema (vs outbox's clean
    designed domain events; combine by CDC-tailing the outbox). Trace clean / crash-after-commit
    (survived) / relay-republish (at-least-once → duplicate → **idempotent consumer** keyed on the
    same `event_id`, L13, exactly-once EFFECT not on-the-wire). First wall = everything flows through
    one **relay**: its lag = every downstream's staleness (L09 consumer lag), polling loads the
    primary (→ CDC-tail it), the outbox grows ~86 GB/day unless compacted (L20/24 partition-and-drop),
    ordering at scale needs partition-by-key (L09); bonus = the log you propagate from is the log you
    **replay** to rebuild a new consumer (L09/29 kappa). Four traps: retrying the dual write,
    non-idempotent consumer, un-cleaned outbox, CDC leaking your schema (L18/24). Trade:
    freshness/consistency vs pipeline complexity. (Lesson 0033)
34. ✅ **Service discovery & health checking** — one caller (order-svc) reaching one callee
    (search-svc, 200 instances, 40k req/s → 200 req/s each) across L27's elastic / L31-deployed
    fleet where the callee isn't at an address but at 200 that churn every few seconds (rolling
    deploy alone = ~1 change/6s). The naive config file is stale BOTH ways: a crash black-holes
    200×60=**12,000** requests in one 60s refresh window, AND new capacity is invisible till
    redeploy. Fix = live self-maintaining membership: a **registry** where instances **register
    on boot** + **heartbeat every 3s to renew a 9s TTL lease** (L10/22/26), so a crash is an entry
    that **expires** (death detected in ≤3 missed beats, no reply needed — L10's failover in the
    routing layer); TTL is a two-sided dial (too long→dead lingers, too short→a GC pause misses a
    beat→false-evict→**flap**, L10). The outage-preventing distinction: **liveness** ("alive?"→
    restart) vs **readiness** ("take traffic now?"→remove from pool, DON'T restart — the L31 drain
    off-ramp); swap them → crash-loop a warming box, or traffic to a cold one. Three lookup homes:
    DNS (cheap, stale caches, no health/load), client-side (freshest, LB logic in every language),
    sidecar/mesh (fresh+uniform, a proxy per instance). Trace boot (register only when READY →
    capacity appears when it can SERVE), crash (TTL expiry bounds the black hole + L7 retry to
    another instance papers the gap), graceful shutdown (deregister FIRST→drain→die, the ordering
    a crash gets wrong). First wall = the registry is everyone's single dependency → must
    **fail-static** (cache last-good list, choose **AP** — L11 — OPPOSITE of L10's CP election,
    because a stale route is cheap-to-be-wrong where a stale leader is split-brain), be a replicated
    **consensus** cluster (etcd/Consul/ZK, L10), shed heartbeat load that scales platform-wide
    (10k instances/3s ≈ 3,333 renews/s), and live with a propagation-lag window it bounds + retries
    past. Four traps: conflate liveness/readiness; a health check probing a SHARED dep (one DB blip
    fails ALL → registry ejects the whole fleet = correlated total outage, L7/26); a strongly-
    consistent registry on the hot path; a mistuned TTL. Trade: routing freshness vs lookup cost &
    staleness. (Lesson 0034)
35. ✅ **Logical clocks & causal ordering** — a group chat across two regions where a
    REPLY prints ABOVE the question it answers: FR's clock runs 50 ms behind VA, Bob replies
    20 ms after reading, so the reply's wall stamp lands 30 ms BEFORE the question → sorting
    by timestamp inverts causal order. Estimate the one inequality that causes it: skew (tens
    of ms; drift ~4.3 s/day unchecked, NTP pins to ~ms and can step BACKWARD) > gap between
    causally linked events (µs–ms) → inversion is routine, not rare, and can't be zeroed (L14
    speed-of-light residual). Fix = count CAUSALITY not seconds: define **happened-before (→)**
    from info flow (same-process order, send-before-receive, transitivity); **Lamport** (one
    counter, max+1 on receive) = O(1) TOTAL order that never sorts effect before cause, but
    arrow runs one way so it CAN'T detect concurrency (L(a)<L(b) ⇏ a→b); **vector clocks** (one
    counter per process, elementwise-max, L23's version vectors) DO detect concurrent vs causal
    at O(N). Trace the reply ([1,0,0]<[1,2,0] causal, both clocks fix it) vs Carol's unrelated
    msg ([0,0,1] neither ≤ → CONCURRENT, only vector refuses to invent an order → conflict
    resolver MERGES not drops, closes L06/L23 lost update). First wall = O(N) fine when N=servers,
    fatal when N=users (1M × 8 B = 8 MB metadata per keystroke, never shrinks) → **hybrid logical
    clock (HLC)**: pack (physical pt, counter c) into 64 bits → O(1), causal like Lamport, AND
    within clock-skew of real wall-clock (the honest repair for L22/23/34's trusted clocks; still
    total-order-only so no concurrency detection alone → bounded/pruned vectors or dotted version
    vectors for that; CockroachDB uses HLC). Four traps: wall-clock sort across machines; reading
    Lamport/HLC order as causality (fake order → L23 lost update); per-client vector at scale;
    thinking a logical clock removes the need for real time (L22 lease still needs physical
    expiry). Trade: ordering precision vs metadata size & coordination. (Lesson 0035)
36. ✅ **Cell-based architecture** — a multi-tenant SaaS on L27's fleet (286 servers, 40k
    req/s, 8M tenants) as one shared stack: a single poison query / bad deploy / hot tenant is
    EVERYONE's problem (8M × 5 min = 40M tenant-minutes, 100% blast radius, for every such
    incident). Cut the WHOLE stack (LB + app + data) into N independent **cells** that no
    request may leave → the same failure drops to **1/N** (8 cells = 12.5%, 8× less damage,
    7/8 physically unreachable from the fault) — stronger than L7 bulkhead / L28 queue / L31
    canary because the wall is PHYSICAL (no shared queue/pool/row), so it stops ALL failure
    modes, not one. Model the hard rule (no cross-cell calls or you rebuilt the monolith) + the
    one un-cellable part = the **router** (tenant→cell, must be dumb + HA + rarely-changed; a
    router bug is the one bug with 100% blast radius; fixed hash L03/04 vs lookup table for
    placement flexibility). Size against an overhead **FLOOR** (quorum L10 + headroom L27):
    near the floor, halving blast radius ~doubles fixed cost (64 cells = 64×8 = 512 nodes ≈ 2×
    the fleet). Trace a bad deploy (cell = L31 canary unit, deploy unit == blast unit) + a hot
    tenant (cells cap the SYSTEM at 1/8, but the 1M innocent cell-mates still share its fate =
    the wall). First wall → **shuffle sharding**: each tenant gets a random SUBSET of k cells,
    fully co-victim only if whole subsets coincide → rides C(N,k): C(8,2)=28 shrinks the group
    12.5%→3.6% (3.5×), C(100,5)≈75M → 1-in-75M (the AWS trick; shines on stateless/retriable
    request plane, hard for stateful data). Four traps: hidden shared component (DB/cache/global
    lock/config) = correlated failure back to 100%; fat/fragile router; cells sized wrong
    (too big = no isolation, too small = re-paid floor × N + toil); a GLOBAL op (config push,
    shared migration, deploy-all-at-once) that skips the cell boundary = 100% blast radius no
    matter how many cells. Reuses L07 bulkhead, L28/L31 isolation+canary, L27 sizing, L10
    quorum, L03/04 sharding+consistent-hashing routing, L24 tenant migration, L34 retry-to-
    another-instance. Trade: blast-radius isolation vs cross-cell coordination & overhead.
    (Lesson 0036)

### Advanced topics (next batch — queued so the course never runs dry)
37. ✅ **Read/write splitting & CQRS** — the orders service (L12/24/25/33, 5k writes/s, 50k reads/s,
    10:1) where one normalized table fails four reads at once. It's SHAPE not just volume: the detail
    page is a 6-way join ≈ 2.4 ms (six L12 seeks) = 48 cores @ 20k/s vs a denormalized read model's one
    0.4 ms lookup = 8 cores (6× cut) — but the denormalization that makes the read cheap is exactly what
    makes a write dangerous (a product-name change fans out to every doc, L15/24 → L06 lost-update), so
    normalized wins writes, denormalized wins reads, no one table wins both (search wants an inverted
    index L16, dashboards a columnar store L29, status a cache L02; "normal form is a bet on write:read
    ratio", L29). **CQRS** = one **write model** owns the truth (normalized, ACID, correctness-first
    L06/25) + many **read models** each shaped for one query (doc/search/columnar/cache) + a
    **projection** connecting them by consuming L33's durable outbox/CDC stream **idempotently** (L13,
    at-least-once L09). Distinct from a **read replica** (L06: byte-for-byte copy → scales VOLUME, SAME
    shape, can't become an index/warehouse); CQRS changes the SHAPE. Trace a command (commit order +
    outbox event in ONE txn, ack from write model, project async → each read model updated after its own
    delay) + four queries (each on its own store, a dashboard scan can't starve checkout — L29/L36
    isolation). First wall = the **eventual-consistency gap**: write acked before any read model
    reflects it (tens–hundreds ms, seconds under L06/L09 lag) → **read-your-writes** looks like data
    loss → fix selectively (serve the user's own fresh data from the write side / return it in the
    command / wait-for-version monotonic reads / accept lag where harmless), NOT by making projections
    synchronous (re-couples, can't span stores atomically L32). That gap also marks where CQRS is
    **accidental complexity**: reads sharing the write's shape + fitting one DB → two models + a pipeline
    + a gap for no benefit. The ladder: one model → read replica (volume) → materialized view/cache (one
    derived shape) → full CQRS (divergent shapes) — climb only as far as the reads force. Four traps:
    dual-writing to each store instead of projecting (L33's trap → diverge on crash), non-idempotent
    projection (double-apply on redelivery L13), trusting a read model as truth (stale → L06/32 lost
    update; validate invariants on the write model, rebuild read models by replay L09/29), hiding the
    gap from the UX. Trade: read/write optimization vs the eventual-consistency gap between them.
    (Lesson 0037)
38. ✅ **Event sourcing** — L25's digital wallet re-built event-sourced: store the **stream of events**
    as the sole source of truth and DERIVE current state by **folding** them (`state = events.reduce(
    apply, empty)`, L25's ledger generalized to every aggregate) — inverting L33/L37 where state was
    truth and events a side effect ("the outbox becomes the database"). Estimate two storage walls:
    the log grows ~130 GB/day ≈ **47 TB/yr and never shrinks** (vs a state DB's flat ~100 GB), and the
    fold cost grows with one aggregate's history (100M-event hot account = **20 s** to load per read).
    Model four parts: immutable past-tense **events** (corrected by APPENDING a compensating event L25/
    L32, never editing), the **aggregate** as fold-and-invariant boundary, the **fold/apply**, and the
    **guarded append** (expected-version = L06 compare-and-set enforces `balance≥0` with NO lock L22 —
    two concurrent withdrawals can't both land at v6). Trace command (fold-to-decide → append-if-version-
    matches), query (served from a PROJECTION L37, because you can't SELECT over a log → event sourcing
    NEEDS CQRS to answer "current state?"), replay (any new view or "balance last Tuesday" free, L09/29/33
    kappa + time-travel). First wall = folding a long-lived aggregate every command → **snapshot** = a
    cached fold (L25 generalized; snapshot every 1,000 → fold ≤1,000 ≈ 0.2 ms = **100,000× cut**, safe
    because derived/rebuildable, never the truth). Then: project for queries (CQRS), version+**upcast**
    events (schema is forever, L18/24 additive-only, L33 no schema coupling), unbounded log = price of
    total history. Four traps: mutate/delete an event (rewrites history, corrupts every fold/projection);
    fold whole log per read (O(history), 20 s); aggregate too big (L06 contention / L22 serial gate →
    cross-aggregate = saga L32); events coupled to current code (won't deserialize after refactor).
    Distinct from CQRS (L37): ES decides what the truth IS (the log), CQRS how you READ it (projected
    shapes) — independent but pair naturally. Trade: perfect auditability/replayability vs query
    complexity & unbounded storage. (Lesson 0038)
39. ✅ **Tiered storage & data lifecycle** — L38's event log (5k events/s × 300 B = ~130 GB/day,
    immutable, kept 7 yrs for compliance → ~330 TB) can't all live on fast disk; the L20/29/38 piles
    grow forever but reads are skewed to new data (L02): ~90% of reads hit the last 7 days (~1 TB =
    0.3% of the pile), so all-hot SSD @ $0.10/GB-mo = ~$33k/mo pays a fast price for data read ~1000×
    less. Fix = a **tier ladder** (HOT SSD $0.10/~1ms → WARM HDD $0.02/~10ms → COLD object $0.004/
    ~100ms-1s+fee → ARCHIVE glacier $0.001/MINUTES-HOURS; each step ~5× cheaper, ~10× slower) with
    each byte falling downhill as it **cools**; slicing 330 TB by age → ~$744/mo = **~44× cut** (biggest
    byte-slice = archive 86% = smallest bill-slice 38%). Model = the fixed ladder + **temperature**
    (access freq; age is only a PROXY — a reactivated 2-yr account is hot) + a background **lifecycle
    policy** (L24 throttled sweep) that demotes by age and at 7 yrs **expires**, keeping **tiering**
    (cost move, still retrievable) strictly separate from **expiration** (compliance move, gone), and
    the small **index HOT** (L20 metadata/data-plane split) while only payloads cool. Trace write
    (always lands hot), hot read (~2 ms, the 90%), cold/archive read (issue restore job → **202**, wait
    MINUTES + per-GB fee), lifecycle sweep (copy-new-then-free-old = L20 commit-metadata-last, delete
    at end). First wall = the **retrieval-latency cliff** (cold→archive isn't 10× slower, it's a cliff:
    ms → minutes/hours on tape) → fix NOT by flattening the ladder (forfeits the 44×) but by tiering to
    each slice's **retrieval SLA + temperature** (a sub-1s support slice stops at COLD not ARCHIVE, 4×
    more for a thin slice), keep the index hot so existence lookups never cliff. Four traps: tier by AGE
    when age≠temperature → retrieval fees on still-read data cost MORE than staying hot (tier by access
    freq); confuse tiering with expiration (delete required data / keep personal data past its legal
    ceiling — retention floor vs privacy ceiling); index on a cold tier (existence checks cliff);
    small-object retrieval tax (pack before archiving, L20). Reuses L02 skew, L20 object store/small-file/
    metadata-plane, L24 batched sweep, L29 warehouse, L38 immutable log. Trade: storage cost vs retrieval
    latency & operational policy. (Lesson 0039)
40. ✅ **Multi-tenancy & noisy neighbors** — a B2B SaaS, 50,000 tenants on one shared fleet (L27,
    40k req/s) + one Postgres cluster with a 300-connection pool: multi-tenancy = the cost model
    (pack many tenants → pennies each), its one risk = the word SHARED (every shared pot is a channel
    for one tenant to starve the rest = the **noisy neighbor**). Estimate the blast: normal load fits
    ~60k q/s headroom (Little's Law L13: 300 conns ÷ 5 ms), but one tenant's 200-query × 2-second-each
    export grabs 200 of 300 conns → others collapse to 100÷0.005 = 20k q/s < 40k demand → unbounded
    queue (L28) past the knee (L27) = **100% blast radius from 0.002% of tenants**, no rule broken.
    Model = the **isolation spectrum** (shared table WHERE tenant_id → schema/tenant → DB/tenant →
    **cell/tenant** L36; density↑ vs isolation↓) + three fairness controls on a live shared pot:
    per-tenant **rate limit** (L08 token bucket keyed on tenant_id, 50k buckets not one global — the
    global re-creates the shared pot + head-of-line blocking), per-tenant concurrency **bulkhead**
    (L07, cap in-flight SLOTS not arrivals — the direct fix for the 2-second-hold drain), and **fair
    queuing** (a line per tenant vs one FIFO). Static cap (simple, wastes idle capacity) vs
    work-conserving/weighted fair queuing (reclaims idle capacity but must PREEMPT a borrower instantly).
    Trace: normal request (cheap, shared, lets go), spike w/o isolation (100% outage), same spike w/
    30-conn bulkhead (270 left → 54k q/s → ZERO impact, greed paid by the greedy — E's export just runs
    slow), and a genuine **whale** (needs 5k req/s → don't tighten the wall, give a BIGGER ROOM: router
    moves it to own DB/cell L36, Path D). First wall = **a limit counts requests, not WORK**: a tenant
    inside every count cap sends 10 un-indexed scans at ~40 s each (L12 selectivity trap) → monopolizes
    CPU → fix = meter the SCARCE resource (CPU-seconds/rows/bytes, L08 generalized) + per-query
    guardrails (statement timeout, row caps, forced LIMIT); measurable-but-wrong (count) vs
    accurate-but-costly (cost, only knowable post-run → reactive). Three more walls = shared resource is
    still ONE blast surface → **shuffle-sharding** (L36: N partitions, k random per tenant, C(100,2)=4,950
    → full co-victim ~0.02% vs 100%); the **hot tenant** (Zipf L02/03/19, route the whale out); the
    noisy neighbor you can't rate-limit = BACKGROUND work + cache (per-tenant queues/quotas on the async
    tier L05/09 + per-tenant cache budgets L02 — wall every pot, not just the front door). Four traps:
    count-not-cost limit; one global limit not per-tenant; one isolation model forced on all sizes;
    wall the front door but leave async/cache open. Reuses L02 skew, L03 shard/hot-shard+routing, L07
    bulkhead/429, L08 token bucket per tenant, L12 selectivity trap, L13 Little's Law, L27 knee, L28
    bounded queue/goodput, L36 cells+shuffle-sharding. Trade: density & cost efficiency vs isolation &
    fairness. (Lesson 0040)
41. ✅ **Graph & relationship systems** — "who are my friends' friends?" over a billion-edge social graph:
    why a relational JOIN explodes at depth (d^k index seeks, L12 N+1 across hops), adjacency lists +
    index-free adjacency (per-hop cost O(1), independent of N) vs a native graph store, partitioning a
    graph without cutting every edge (the hard part — hash-shard cuts ~99% of edges; replicate hot
    subgraph/TAO, partition by community, vertex-cut the supernodes/celebrities that recur from L15/L3),
    and capping depth + precomputing because d^k is the answer's own size. Trade: traversal speed vs
    partitionability of connected data. (Lesson 0041)

### Advanced topics (next batch — queued so the course never runs dry)
42. ✅ **Load balancing algorithms & layers** — the front door to L27's fleet (286 servers, 40k req/s,
    ~140 req/s each): which backend serves each request? Estimate why the "fairest" rule breaks first —
    round-robin distributes request COUNTS evenly but is blind to backend STATE, so one server slowed 10×
    (400 ms/req → real capacity 8 cores/0.4s = 20 req/s) still gets fed its full 140 req/s → unbounded
    backlog (L28) that fails ~0.35% (1/286) of all traffic (counts turns, not trouble). Model two orthogonal
    choices: the LAYER (L4 = per-connection, cheap/protocol-blind but pins a multiplexed HTTP/2 or long-lived
    stream to ONE box chosen once; L7 = per-request, reads path/header/cookie, can retry L07, demuxes the
    stream — pays parse + TLS) and the ALGORITHM ladder, each rung buying back the info round-robin discarded:
    weighted-RR (unequal boxes), least-connections & least-latency/EWMA (REACT to load — a slow box's rising
    in-flight routes traffic away), **power-of-two-choices** (pick 2 at random → lighter; near-optimal balance
    with NO global scan and NO herd because random picks don't correlate — imbalance drops from growing like
    ln N/ln ln N to a near-constant ln ln N), **consistent-hash** for stickiness (L04 ring: same key→same
    backend, warm cache L02, only ~1/N remaps on churn vs %N's reshuffle L03) whose price is a HOT key pinning
    one box (L03/15/40) → bounded-load hashing (cap (1+ε)×avg, spill on the ring). Trace: healthy P2C request;
    a sick box HEALED in ms by load-awareness then ejected in seconds by a health check (L34 readiness);
    a deploy that drops ZERO requests via connection draining — deregister→drain→die (L31/34). First wall =
    the LB is the fattest SPOF (every request crosses it → its death = 100% outage, L07/26) → redundancy
    (anycast / VIP failover / DNS round-robin) but that DESTROYS the global load view (each LB sees only its
    own traffic → "global best" becomes a herd of balancers stampeding stale guesses, L02) → exactly why
    coordination-free RANDOMIZED algorithms (P2C) win at scale. Four traps: blind algorithm in front of
    servers that can slow (round-robin melts behind the slow box); a health check probing a SHARED dependency
    (one DB blip fails ALL → LB ejects the whole fleet = correlated total outage, L34); L4 in front of
    multiplexed/long-lived connections (pins hundreds of requests to one box → skew); treating the LB as
    infrastructure that "just works" (it's the SPOF; run several + drain on deploy). Reuses L02 locality &
    thundering herd, L03 %N reshuffle & hot shard, L04 consistent-hash ring, L07 partial failure/retry/SPOF,
    L26 stateful front door, L27 fleet sizing & knee, L28 unbounded queue, L34 readiness/health/drain.
    Trade: even load distribution vs stickiness & simplicity. (Lesson 0042)
43. ✅ **Gossip & anti-entropy** — a 1,000-node coordinator-free KV store agreeing on TWO things at once:
    membership (~50 KB roster, tiny/churny) + data (100 GB/replica, huge/rarely-changed). Estimate: all-to-all
    heartbeats = N² ≈ 1,000,000 msg/s (quadratic wall), a central registry/consensus just rebuilds the SPOF
    (L26/34/42) → GOSSIP: each node contacts fan-out=3 random peers/round (3,000 msg/round, ~333× less, LINEAR),
    epidemic spread reaches all 1,000 in ~log₄(1000) ≈ 5 rounds (exponential reach for linear cost, O(log N) to
    converge). Model by shape: membership via **SWIM** — O(1) detection (ping 1 random peer/period, then k=3
    INDIRECT ping-reqs before believing a death, rules out flaky link), **suspicion + incarnation refutation**
    (mark suspect not dead; a GC-paused node re-announces with higher incarnation# → overrides suspicion → no
    flap, L27 knee/L34 flap), dissemination PIGGYBACKED on pings; data via **Merkle-tree anti-entropy** — leaves
    hash key-buckets, root hashes 100 GB in 32 B → equal roots prove identical for free, mismatch chased down ONLY
    the divergent branch (~17 levels, ~1 KB hashes to LOCATE, ~1 MB bucket to SHIP) → cost scales with DIVERGENCE
    not dataset size (~100M× less than naive 100 GB diff, L29 narrow-before-compute). Two repair triggers:
    **read-repair** (on quorum-read disagreement, L06, write newest back → fixes HOT keys, misses cold) +
    background sweep (Merkle → fixes COLD keys) + hinted handoff (transient). Trace: real death converges in
    seconds; paused node refutes with higher incarnation → not evicted; stale replica repaired on-read AND by
    Merkle sweep (version compare L06/L35). First wall = **eventually consistent** (L11 AP): always a disagreement
    window; shrink with faster/wider gossip + more sweeps but NEVER to zero (zero = the coordination gossip avoids)
    → gossip = "roughly who's alive / roughly the data, cheap, no SPOF, scales" (AP) vs consensus (L10 CP) for
    exact/instant answers (leader, commit, lock — L22/25) vs central registry (L34, SPOF at scale). Four traps:
    gossip where you need consensus (2 nodes both hold the lock → double-run); whole-dataset diff not Merkle;
    hard timeout no suspicion → flap/churn storm; read-repair alone → cold keys never converge. Reuses L06
    replication/version, L10 consensus (the CP contrast), L11 AP, L27 knee, L29 narrow-before-compute, L34
    registry/readiness/TTL-flap, L35 vector clocks, L42 routing. Trade: convergence speed vs message overhead &
    staleness. (Lesson 0043)
44. ✅ **Time-series & metrics storage** — store L17's observability firehose: 10M active series × a 10-s
    scrape = 1M data points/s, kept for years. Estimate kills the naive row twice — inline labels ~216 MB/s
    ≈ 18.7 TB/day (same label string written 1M×/s) → SPLIT series from samples (labels stored once, sample =
    bare 16 B timestamp+value → 1.38 TB/day) → COMPRESS for the shape: delta-of-delta timestamps (regular
    interval → ~1 bit) + XOR values (slow-moving → ~1.3 B) crush 16 B → ~1.5 B/point (Gorilla ~1.37) → 130 GB/day
    ≈ 47 TB/yr (~11×). Model = series index (L12/16 inverted index, posting-list intersection picks which series
    a query reads) + samples in time-ordered CHUNKS that go immutable once past → compress hard, ROLL UP (raw 10s
    → 1m → 1h, ~90×/level, lossy+irreversible) + DROP whole partitions at retention (O(1), not a billion-row
    DELETE, L24/39). Trace a write (creating a series = the costly event; sampling an existing one ≈ a couple
    bytes+bits) and a range query (label index → time-partition prune L29 → decode only a few MB). First wall =
    cardinality, NOT volume: point rate is bounded (series ÷ interval) + compresses away, but series count is a
    PRODUCT of label cardinalities → one user_id label over 10M users explodes 750k series → 7.5 TRILLION (~750 TB
    index + unbounded RAM) while sample rate looks fine → keep identifiers OFF labels (L17 ids-on-traces; exemplars
    to jump), cap/allowlist labels at ingest, pre-aggregate. Four traps: high-cardinality label; metrics in a row
    store (forfeits the 11× compression, DELETE-not-drop); one resolution/retention for all; rollup before you
    know the questions (store min/max/sum/count, not just avg). Reuses L12/16 inverted index, L17 firehose+cardinality
    rule, L20 metadata/data split, L21 lossy-summary, L24 partition-drop, L29 columnar/pruning, L39 tiers/retention,
    L40 noisy-neighbor. Trade: resolution & retention vs storage cost. (Lesson 0044)
45. ✅ **Webhooks & outbound event delivery** — deliver "order.shipped" to 50,000 endpoints we neither
    control nor trust (5k events/s × ~2 subs = ~10k POSTs/s). The reversal (WE call THEM) breaks every rule:
    can't BLOCK (their downtime becomes ours) and can't DROP (they need every event). Estimate kills inline
    delivery: a dead endpoint hangs to a 30s timeout → Little's Law, 1% down = 100/s × 30s = 3,000 threads
    frozen on strangers' servers, never draining → your checkout pipeline dies from a merchant's expired cert
    → DECOUPLE via durable queue + separate worker fleet (L05/09). Model the four things a stranger forces:
    **at-least-once** (retry till 2xx, a silent drop is invisible to them), **idempotency key** (stable
    event_id in a header so their dedup makes your unavoidable duplicate a no-op, L13 outward), **HMAC
    signature** over timestamp+body with a per-sub shared secret (authenticity + integrity + replay-reject via
    the signed timestamp, constant-time compare — NOT a static bearer token, L30), **backoff + jitter** (1,2,
    4…s capped 1h, ~12 attempts in the first 68 min then hourly to 24h ≈ 35 attempts; jitter breaks the
    recovering-fleet thundering herd L07/26) ending in a **DLQ** + dashboard replay (never silent-drop, never
    infinite-hammer). Trace clean / failing-then-dead-lettered / the ambiguous timeout (request lost vs
    response lost — indistinguishable L07, which is WHY the idempotency key is non-negotiable). First wall =
    the shared delivery fleet: one dead endpoint at 500 deliveries/s × 30s = **15,000** in-flight slots (5×
    the healthy 3,000 baseline) = head-of-line blocking (L09) + noisy neighbor (L40) at once → **per-endpoint
    circuit breaker** (dead → ~0 slots, the top lever) + **per-endpoint concurrency bulkhead** (L07) +
    separate first-attempt/retry paths. Second wall = ordering: independent retries reorder events → don't
    promise order, make events **self-describing** (version/sequence + timestamp, receiver skips superseded,
    L35/37) rather than per-key partitions (which rebuild head-of-line on purpose). Four traps: inline
    delivery; no signature / static token; assume exactly-once or fresh id per retry; one shared pool with no
    per-endpoint isolation. Reuses L05/09 queue+DLQ+at-least-once, L07 timeout/backoff/breaker/bulkhead, L13
    idempotency+ambiguous timeout, L26 polling-waste+jitter, L30 HMAC, L33 outbox source, L35/37 reader-side
    ordering, L40 noisy neighbor. Trade: delivery reliability vs the burden pushed onto the receiver.
    (Lesson 0045)
46. ✅ **Distributed job scheduling** — 1,000,000 cron jobs across a fleet, each firing ONCE at the right
    time (~23 fires/s avg but spiky). One `cron` box fails 3 ways: SPOF (a midnight outage silently drops
    every midnight fire — no retrier, it's the thing that's down), throughput ceiling (200k × 0.5 ms = 100 s
    to dispatch the midnight batch serially → 2 min late on the happy path), and the herd (200k jobs written
    `0 0 * * *` come due in ONE tick = ~8,600× the average). Distributing buys "on time" (shard by hash(job_id),
    L03/04, per-shard lease failover L10/22) but REINTRODUCES the double-fire, so buy "once" back. Model =
    split **dispatcher** (decide what's due + enqueue, never run inline — a slow job must not stall the tick,
    L09 head-of-line) from **workers** (execute, L05/09); the **atomic claim** (conditional UPDATE = L06 CAS)
    so two racing dispatchers enqueue ONE run not two; and — because a GC pause always lets a paused owner
    overrun its 30 s lease (L22, lease = LIVENESS knob never safety) — an **idempotent job** keyed on
    (job_id, scheduled_time) so the unavoidable double-fire is a no-op (L13): exactly-once *attempts* impossible,
    exactly-once *effect* lives in the job (L45's shape on the clock), + fencing token (L10/22) for resources
    that need it. Trace clean fire / double-fire race (DB serializes the claim → 1 run) vs GC-pause double-fire
    (2 runs → idempotency saves it) / crash-then-recover (200k overdue jobs → per-JOB **coalesce** [1 digest]
    vs **backfill** [every missed billing hour], throttle+jitter the catch-up or it's the herd again). First
    wall = the herd is in the SCHEDULE not the scheduler (more dispatchers don't help) → **jitter** fire times
    (hash(job_id) mod 60 s → 200k over 60 s = ~3,333/s, trade timeliness precision for smooth load) + **clock
    skew** (L35: compare due-ness/lease-expiry against ONE authoritative clock not each node's wall clock;
    don't promise precision finer than NTP skew; leases expire on real elapsed time, logical clocks can't
    time 30 s). Four traps: one scheduler / inline execution; assuming a lease gives exactly-once; no
    missed-fire policy; the midnight herd + trusting local clocks. Reuses L03/04 shard+consistent-hash, L05/09
    queue+workers+head-of-line, L06 CAS, L07 SPOF/herd/backoff, L08 rate-limit catch-up, L10 election/failover,
    L13 idempotency+exactly-once-effect, L22 lease+fencing+GC-pause, L27/28 autoscale+bounded-queue, L35 clock
    skew/physical-vs-logical time. Trade: timeliness vs exactly-once execution. (Lesson 0046)

### Advanced topics (next batch — queued so the course never runs dry)
47. Storage engines: LSM-trees vs B-trees — L12 gave the B-tree for fast reads, but write-heavy systems
    (L25 ledger, L38 event log, L44 metrics) pay a random-write-per-change tax; a log-structured merge-tree
    turns writes into sequential appends + background compaction (L20/39), the read/write/space amplification
    triangle, and a per-SSTable Bloom filter (L21) to keep reads cheap. Trade: write throughput vs read
    amplification & space.
48. Distributed caching & invalidation at scale — many app servers share a cache tier (L02/L14); the hard
    part is coherence: write-through vs write-behind, invalidation fan-out, the stampede when a hot key is
    invalidated (L02 thundering herd), and TTL-vs-explicit-purge (L14). Trade: freshness vs cache hit rate
    & load.
49. API gateways & the edge — one front door (L42) that does auth, routing, request aggregation (the BFF),
    and enforces rate limits (L08) / quotas / mTLS (L30) so backends don't each re-implement them; the risk
    of a fat gateway becoming a SPOF (L07/26) and a deploy bottleneck (L31/36). Trade: centralized
    cross-cutting concerns vs a fragile shared choke point.
50. Global traffic management (GeoDNS / anycast / failover) — routing each user to the nearest HEALTHY region
    (L23 multi-region, L42 load balancing) via geo/latency DNS or anycast, health-based failover with a DNS
    TTL that bounds how fast you can drain a dead region, and the split-brain risk of two regions both
    thinking they're primary (L10/11). Trade: proximity & availability vs routing staleness & complexity.
51. Chaos engineering & fault injection — deliberately breaking things (kill a node, add latency, drop a
    region, partition the network) to VERIFY the failure designs actually hold (L07 timeouts/breakers, L34
    health checks, L36 cells, L46 failover); blast-radius control, steady-state hypotheses, and running it in
    production safely. Trade: confidence in resilience vs the risk of the experiment itself.

## Lesson format conventions
- Four reusable "moves" framing introduced in Lesson 01: estimate → model →
  trace the paths → find the first bottleneck. Reuse this spine across lessons.
- Every design choice should NAME its trade-off explicitly (that's the skill).
