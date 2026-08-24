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
47. ✅ **Storage engines: LSM-trees vs B-trees** — Lesson 44's metrics firehose (1,000,000 writes/s × 16 B
    = only 16 MB/s of DATA, but randomly interleaved keys) killed the B-tree: sorted-in-place means each 16 B
    insert is a read-modify-write of a whole 8 KB page (8,192÷16 = **512×** write amp), scattered keys defeat
    page-batching → needs 16 MB/s × 512 = 8 GB/s ≈ **16×** the disk's sequential bandwidth (or ~1M random
    writes/s vs a disk's few-hundred-to-tens-of-thousands IOPS = 20–10,000× short); no same-class disk closes
    it. NAMED trade sorted-in-place vs log-structured: "always sorted on disk" and "never random-write" can't
    both hold. Fix = **LSM-tree** (RocksDB/Cassandra/LevelDB): append to sequential **WAL** + RAM **memtable**
    (16 MB/s = 3.2% of disk), **flush** full memtable as one immutable sorted **SSTable** (64 MB fills every
    4 s, flushes in 0.13 s), background **compaction** merge-sorts files down dropping shadowed values +
    **tombstones**. The **amplification triangle** (can't min all three): write amp (~10–30× leveled, all
    SEQUENTIAL vs B-tree's random 512×), read amp (probe memtable + many SSTables), space amp (dead versions/
    tombstones until compacted); leveled (low read/space, high write) vs size-tiered (reverse). Trace: write
    (WAL+memtable, ack, never a random page); point read (newest→oldest, would probe ~7 files → per-SSTable
    **Bloom filter** L21 "definitely not here", 10 bits/key → 0.6185^10 ≈ **0.8%** FP → ~1 file, no false
    negatives); range read (Bloom CAN'T help — it's point-membership, not range → merge overlapping SSTables,
    the LSM's honest weakness, B-tree's home turf L12/18); delete = write a tombstone (space amp up until
    compaction). First wall = compaction is a SECOND write workload on the same disk → sustainable ingest
    ceiling = bandwidth ÷ write-amp ≈ 500 ÷ 20 = **25 MB/s** (not the WAL's 500); cross it → SSTables pile up
    → read/space amp climb → **write stall / compaction debt** (L27 knee, L28 backpressure inside one DB).
    NAMED trade foreground latency vs background work (one disk split between serve-now and clean-up).
    Scoreboard: B-tree = read/range-heavy OLTP; LSM = write-heavy logs/metrics/ledgers/KV. Four traps: B-tree
    default under write-heavy load; assuming LSM made writes free; Bloom for range scans; ignoring tombstones/
    space amp. Reuses L02 buffer-the-expensive/herd, L12 B-tree/page/disk numbers, L15/20 delete-by-marker,
    L21 Bloom filter, L25/38/44 write-heavy workloads, L27/28 knee+backpressure. Trade: write throughput vs
    read amplification & space. (Lesson 0047)
48. ✅ **Distributed caching & invalidation at scale** — a product catalog (10M products × 2 KB = 20 GB),
    50k reads/s over a DB that safely serves ~5k/s (L27 knee), changing ~500 writes/s across L27's 286 app
    servers: L02/L14 cached in front of ONE thing and the job was HITS; at fleet scale a cache is a COPY and
    the hard word is COHERENCE — the instant the source of truth (L06/25) moves, every copy is a small lie.
    Estimate why ONE shared tier beats a private cache per server: 286 private caches = up to 286 stale copies
    to chase per write (+ 5.72 TB to hold the catalog 286×), a shared tier holds ONE copy per key (any server
    routes it to the same node, consistent hashing L04) → 20 GB once + one DELETE to invalidate; and it must
    run at 99% hit (DB can't serve 50k/s without it: 1-5k/50k=90% offload minimum, zero headroom → aim 99%,
    misses=500/s). Model the WRITE (where caching gets hard): cache-aside (write DB, invalidate — honest
    default, source-of-truth first so a crash leaves a stale CACHE not a stale DB) vs write-through (always
    fresh, cache write on every DB write's critical path) vs write-behind (fast ack from RAM, flush DB later,
    L47 memtable across the cache/DB boundary, data-loss risk if cache dies pre-flush); + the sharper fork
    DELETE (idempotent+commutative → two writers can't leave a wrong value, but a guaranteed MISS after each
    write) vs versioned OVERWRITE (no miss, but two writers race to a lost update IN the cache L06 unless a
    version stamp / compare-and-set gates it). Trace hit (99%, ~0.2 ms, DB untouched), miss (populate + TTL
    backstop = L14 dial), clean write (DB first THEN invalidate — invalidate-first lets a reader repopulate the
    not-yet-updated DB), and the STALE-REPOPULATION RACE (reader misses→reads DB v1→stalls; writer updates
    v2+deletes; reader wakes+SETs v1 → cache stuck stale till TTL — the stale write comes from a READER not a
    writer, so delete-vs-overwrite can't fix it; bound by TTL / version-CAS on populate / delete-after-delay).
    First wall = adding a per-server NEAR CACHE (L1, 20 MB/server) to beat the network hop brings N copies BACK:
    can't synchronously delete a copy you can't address → invalidation becomes a best-effort pub/sub BROADCAST
    (L09, 500×286=143k drops/s, cheap) whose ONLY hard guarantee is a short TTL (a disconnected/lagging/
    restarting subscriber misses the message → stale till expiry; TTL is the contract, not decoration); and
    deleting one HOT key (10k reads/s → L=R×T=10k×0.010=100 simultaneous misses, L13, landing on ONE hot shard
    L03) reignites L02's THUNDERING HERD → single-flight/request-coalescing (100 DB reads→1), serve-stale/SWR
    (L02, expiry never a miss, only where seconds-stale ok), or versioned overwrite (hot key never goes absent).
    Four traps: invalidate-before-write-DB (reader fills the gap with the old value); overwrite w/o a version
    (lost update in the cache); trust the fan-out is reliable (it's best-effort → keep the TTL short); delete a
    hot key & expect the DB to absorb the herd. Reuses L02 cache-aside/TTL/eviction/herd/single-flight/SWR,
    L03 hot shard, L04 consistent-hash routing, L06 lost update + CAS, L09 pub/sub, L13 Little's Law, L14
    freshness-vs-speed + TTL-vs-purge, L27 fleet/knee/99%-offload, L47 memtable-then-flush. Trade: freshness vs
    cache hit rate & load. (Lesson 0048)
49. ✅ **API gateways & the edge** — a mobile home screen over 40k req/s to 30 microservices, each owned by a
    different team: a pile of cross-cutting work (authN, rate limit, TLS, routing, aggregation, observability)
    belongs to EVERY service and to NONE → give it one home, the **API gateway** (a reverse proxy at the edge
    all north-south traffic crosses). Estimate the win, each a cost flipped: security written ONCE vs 30
    codebases (a key rotation / tighter limit = 30 deploys, 30 chances to drift, one forgotten service = the
    hole → 1 consistent place); a 6-call home screen's 3 dependent hops collapsed 3×100ms=300ms →
    100+3×2=**106 ms** by moving composition into a **BFF** (~2.8× cut, 6 phone connections → 1, L15 fan-out);
    20%×40k = **8,000 req/s** of junk 401/429'd at the door for a token check (L28 admission control/goodput —
    the cheapest request is the one you reject). Model the seven-job pipeline (TLS-terminate L30 → authenticate
    → rate-limit L08 → route L34/42 → aggregate → observe L17 → translate) + the identity trick that keeps
    services simple: authenticate the external token ONCE, forward a **signed identity header** over **mTLS**,
    services AUTHORIZE (what may this identity do?) but don't re-AUTHENTICATE (who is this?) — authN centralized,
    authZ local. Two lines keep the gateway from swallowing the system: **north-south** (gateway, untrusted
    outside) vs **east-west** (service MESH/sidecars L34, inside the trust boundary), and one god-gateway vs
    **per-client BFFs** (mobile/web/partner, each team-owned, aggregation lives here). Trace clean request
    (signed header, service authorizes) / BFF aggregation / edge rejection (shield, no backend touched) / the
    partial failure that is aggregation's BILL (compose N → couple N fates → one slow recommend-svc sinks the
    whole screen unless per-call **timeout + graceful degrade** returns the 5 that answered, L07). First wall =
    the front door itself, failing TWO ways: the **fattest SPOF** you own (100% of traffic crosses it →
    death = total outage though every backend is healthy, L07/26/42 → fix by making it a STATELESS disposable
    HERD behind L42's LB/anycast, N+1 headroom L27, drain-on-deploy L31/34, token-auth = no session to pin
    L26), and a **deploy bottleneck** (logic accretes → the 1990s **ESB** reborn, a shared codebase with 100%
    blast radius every team queues behind, L36 → fix by keeping it THIN: cross-cutting concerns ONLY, business
    logic + aggregation pushed OUT to BFFs and services, routes as per-team DECLARATIVE config so adding a
    service isn't a gateway code change). Four traps: the god gateway (ESB); treating it as infra that "just
    works" (it's the SPOF — run several, health-check, drain); re-authenticating at every hop OR trusting the
    internal call blindly (the gateway-bypass/confused-deputy hole → mTLS + signed header + zero-trust);
    aggregating without per-call timeouts/fallbacks (inherit the worst of all six). Reuses L07 partial-failure/
    timeout/SPOF, L08 token bucket, L15 fan-out, L17 tracing, L26 disposable stateless front door, L28 admission
    control/goodput, L30 TLS/mTLS/identity, L34 discovery + east-west mesh, L36 shared-component blast radius,
    L42 load balancing. Trade: centralized cross-cutting concerns vs a fragile shared choke point. (Lesson 0049)
50. ✅ **Global traffic management (GeoDNS / anycast / failover)** — a service live in THREE regions
    (Virginia, Frankfurt, Singapore) with a Sydney shopper: before any L42 load balancer, the device must be
    handed an IP, so which region's address does the name resolve to, near+healthy, right now? A fixed IP fails
    twice — proximity (Sydney→Singapore ~63 ms floor vs →Virginia ~160 ms; a cold HTTPS load is ~3 RTTs →
    3×97 ≈ ~290 ms dragged across the Pacific, L14 speed-of-light at continent scale) and availability (that
    region dies → 100% outage, wasting L23's spare regions the front door can't steer to). Model the TWO places
    you can steer: DNS-based (GeoDNS/latency: authoritative server returns a different IP by resolver/ECS
    location; rich control — geo/latency/weight/health — but every answer is CACHED for its TTL at resolver/OS/
    browser, so a failover propagates only as caches expire) vs anycast (same IP announced via BGP from many
    regions, network delivers each packet to the nearest; failover = withdraw the announcement, reconverges in
    SECONDS, no DNS cache to wait out — but coarse "nearest by topology" not latency/weight, and a mid-session
    re-route can RESET live TCP → default for stateless/QUIC, careful for long TCP). Both gated by a health check
    = L34 readiness lifted to the REGION (detection ≈ interval × threshold). Trace: steady state (Sydney→SG
    ~95 ms, cache 60 s); GeoDNS region death (failover bounded below by detection 30 s + TTL 60 s ≈ 90 s cached
    tail; new users fine sooner); anycast death (reconverge in seconds, cost = TCP resets); FALSE failover =
    partition makes a live region look dead from a single-vantage checker → needless reroute, or if it PROMOTES
    a writer → split-brain (L10/11), two primaries, divergence. First wall = DNS caching bounds drain speed: TTL
    is a request not a command (browsers pin, resolvers round up/ignore), lower TTL 60→20 s shrinks the window
    (~90→~50 s) but triples DNS QPS (667→2,000/s @ 40k clients) and past a point buys nothing → don't fight DNS,
    LAYER anycast for the fast path + health-gated GeoDNS for rich selection + L42/L34 inside each region. Four
    traps: fixed un-steered address over multi-region; treating DNS failover as instant; health-checking a region
    from one place (can't tell dead from unreachable = CAP at the routing layer, L11); letting routing (AP, stale
    route = a retry) decide write-promotion (CP, needs consensus + fencing, L10/23). Reuses L10 consensus/split-
    brain/fencing, L11 CAP dead-vs-unreachable, L14 speed-of-light, L23 multi-region spares, L31 weighting/canary,
    L34 readiness/health/flap, L42 load balancing (region-above, server-within). Trade: proximity & availability
    vs routing staleness & complexity. (Lesson 0050)
51. ✅ **Chaos engineering & fault injection** — a checkout flow (payment/inventory/recommendation svcs,
    L07/25/36/49/50) whose fifty lessons of DEFENSES are all untested CLAIMS: a defense you've never
    triggered is a hypothesis, not a fact. Estimate the odds one is silently broken: ~50 resilience
    mechanisms each ~10% wrong → `0.90^50 ≈ 0.5%` all-work → ~99.5% at least one broken, found at 3 AM at
    100% blast unless you look first; a controlled 1%/30-s experiment (~1,500 reqs) vs a live 100%/30-min
    outage (~9,000,000) = ~6,000× less damage. Model the experiment as the SCIENTIFIC METHOD (not a
    wrecking ball): a **steady-state hypothesis** in OUTPUT terms (checkout success ≥99.9%, p99<300ms, not
    CPU), a single injected **fault** (kill/latency/exhaustion/partition/region-drop/clock-skew — LATENCY
    the most revealing, because slow holds a thread/slot open where dead fails fast = L07's reason), a tiny
    **blast radius** + control group (L31 canary / L36 cell as the natural boundary), and an AUTOMATIC
    **abort/halt condition** (L31 rollback gate pointed at a fault). Trace: (A) kill 1 payment-svc instance
    → L34/42 eject+reroute + L07 retry + L13 idem key → HOLDS (confidence earned, widen); (B) +5s latency
    on nice-to-have recommendation-svc → EXPECT L07/49 timeout+degrade, ACTUAL no timeout set → BFF blocks
    → thread-pool freeze (L07/13) → checkout craters to 82% on 1% in 28s = the payoff, the 6,000× the
    estimate promised, fix the timeout & re-run → HOLDS; (C) partition inventory-svc but the abort+monitoring
    run THROUGH the partitioned path → dashboard stale-green, abort never arrives → controlled experiment
    becomes an incident (L36 shared-component trap in the TOOLING); (D) run it all in staging → passes
    everything, proves nothing (no real load/skew/hot-keys/cross-region latency = L31 unrepresentative-canary
    trap). First wall = the SAFETY OF THE EXPERIMENT ITSELF: value only unlocks in production (staging lies)
    but production = real customers, so the practice lives on (1) small SHRINKABLE-to-zero blast radius,
    (2) a safety plane (monitor+abort) that does NOT share fate with the target (L36 applied to tooling —
    partition the net, your abort can't cross it), (3) detection FASTER than harm (L17 observability, L27
    knee), and (4) a **blameless** culture — load-bearing, not soft: punish a failed experiment and no one
    runs the one that finds the expensive bug. Four traps: no steady-state hypothesis (vandalism); unbounded
    blast radius (experiment = outage); safety plane sharing fate with target; proving it only in staging.
    Reuses L07 timeouts/breakers/bulkhead/retry (the defenses under test, slow-vs-dead), L10/11 partition &
    split-brain (the fault), L13 idempotency (safe retry under fault), L17 observability (the abort's signal),
    L27 knee (fault at peak cascades where staging can't), L31 canary+auto-rollback (blast radius+abort),
    L34/42/50 health-check/failover (verified in Path A), L36 cells (blast boundary + shared-component trap).
    Trade: confidence in resilience vs the risk of the experiment itself. (Lesson 0051)

### Advanced topics (next batch — queued so the course never runs dry)
52. ✅ **Content moderation & abuse systems at scale** — one content stream, 500M posts/day (~5,787/s): why
    human-review-everything is a two-orders-of-magnitude non-starter (10 s/post × 2,160/mod-day → 231k
    moderators ≈ $9.3B/yr, still slow) → a **cheap-model-first funnel** (L16 two-phase / L29 narrow-before-
    expensive, aimed at CLASSIFICATION): blocklist (exact+**fuzzy/perceptual hash**, ~µs) → cheap classifier
    (~0.5 ms/all 500M, ~3 cores) → big multimodal (~50 ms/uncertain 4% = 20M, ~12 GPUs) → human (~10 s/the
    0.4% = 2M, ~926 mods, **250×** fewer, $37M/yr). The **precision/recall dial** where BOTH errors cost
    (FP = silence an innocent = censorship; FN = miss abuse = harm): one threshold can't win — at 1% bad
    (5M/day) aggressive = 80% recall / **40% precision** (6M innocents removed/day), conservative = 95%
    precision / **38% recall** (miss 62%) → **two thresholds** (auto-remove ≥0.95 @ ~99% precision, auto-allow
    ≤0.20, escalate the UNCERTAIN band; band-WIDTH = safety-vs-cost dial, band-POSITION = **per category**:
    favor precision for spam, recall+hash+report for imminent-harm/illegal). Trace: near-free clean post (94%,
    publish-immediately not block-on-review), hash-caught spam (cheapest = the repeated/known), ambiguous news
    photo escalated → human verdict = the answer AND next model's training label, and a **false positive**
    rescued by an **appeal** (straight to a human; reinstated-rate = a quality metric; FN caught by user
    REPORT [reactive], FP by appeal). First wall = **the human review queue** (bounded resource, L28 with
    PEOPLE as the bound): can't be FIFO (a 10k-views/min post waits 60 min → reaches 600k) → order by expected
    harm = **severity × reach**, triage the low-risk tail to auto-decide (SLA). Second wall = **adversarial**
    (unlike L16's static corpus): attackers mutate (V1agra, 2px crop dodges exact hash, coded language) →
    recall silently decays 95%→70% with NO alert → **fuzzy hashing** + continuous **feedback loop** (human
    labels + reports + appeal reversals → retrain). Deepest wall = every dial is a choice between two harms on
    different people that engineering makes tunable/measurable/per-category but never removes (+ protect the
    moderators absorbing the worst 0.4%). Four traps: one global threshold; FIFO queue; classifier-as-"done";
    auto-remove with no appeal/reason. Reuses L02/48 hot-lookup cache (blocklists), L16 two-phase + precision/
    recall, L28 backpressure/bounded-queue, L29 narrow-before-scan, L13 record-once decision. Trade: safety
    coverage vs false-positive harm & review cost. (Lesson 0052)
53. ✅ **Feature stores & ML serving infrastructure** — one fraud scorer approving a payment in ~50 ms needs
    ~200 **features** (computed signals: `card_txn_count_1m`, `user_avg_amount_30d`, `merchant_fraud_rate_7d`).
    Estimate kills request-time computation on BOTH axes: 10k txns/s × 200 = 2M aggregations/s = 400× a DB's
    safe 5k q/s (melts checkout) AND 200 sequential 5 ms aggs = ~1,000 ms = 20× the 50 ms budget; precomputed,
    the latest values are 100M users × 200 × 8 B = 160 GB served as one ~2 ms KV lookup (L15 precompute-the-read
    for model inputs). Model = TWO stores shaped for opposite jobs — **online store** (KV cache, latest-per-entity,
    fast, L02/48) for serving vs **offline store** (columnar warehouse, full time-series history, L29) for training
    — fed by ONE shared **feature definition** (the anti-drift mechanism), held to two rules: **train-serve
    consistency** (serve the number you trained on; break it = **train-serve skew**, silent accuracy loss) and
    **point-in-time correctness** (build each training row from values AS OF that past instant, never now; break it
    = **leakage**, the offline store keeps HISTORY precisely so an as-of join can't peek at the future — a latest-
    only store CAN'T train honestly). Trace: (A) serving read = one 2 ms batched KV lookup of ~200 precomputed
    values → model → approve, store never computes on the hot path; (B) freshness PER FEATURE = batch a 30-day avg
    (~1% shift/txn, 6 h stale harmless, cheap) vs stream `card_txn_count_1m` (0→50 in a 60 s card-testing attack =
    the fraud signal, a batch would finish the attack before updating → L29 kappa off the L09/33 event log, ~1-2 s);
    (C) training build = join old labels to CURRENT values → merchant_fraud_rate_7d "now" includes this txn's own
    later chargeback → offline AUC 0.99, production flatlines → point-in-time/as-of join over offline history fixes
    it. First wall = train-serve skew SURVIVES a shared definition because the two stores drift (streaming writes
    online at 12:00:03, batch materializes offline at a different window edge → disagree exactly at fast-moving
    moments that matter most) → honest fix = **feature logging**: write down the exact served feature vector, join
    the delayed label to it, so "trained on" and "served" are one recorded fact (L13 decide-once-record-it) — also
    gives point-in-time for free. Secondary walls = online store is a hot dependency (L03/04/48 shard/replicate/hot-
    key) + freshness lag never zero (L06/26 propagation window). Four traps: compute at request time; train on
    current values (leakage); two code paths per feature (skew); one freshness for all. Reuses L15 precompute-the-
    read, L29 serving-vs-analytics-shapes + lambda/kappa, L02/48 cache + hot-key, L39 tier-by-need (freshness per
    feature), L06/26 replication/propagation lag, L13 decide-once. Trade: feature freshness vs serving latency &
    training-serving consistency. (Lesson 0053)
54. ✅ **Billing & metering systems** — a cloud platform metering 50B events/day (~579k/s, 10 TB/day raw,
    3.65 PB/yr audit) turns a usage firehose into invoices correct to the cent on a $50M/mo book, where a
    quiet 0.5% error = $3M/yr mischarged (invisible on a dashboard, seven figures on a bill; metering error is
    asymmetric — undercount leaks revenue, overcount overbills → disputes). Four stages / four guards: **ingest**
    dedups an at-least-once firehose (L09) to counted-once via emitter-stamped **event_id** + atomic insert-if-
    absent (L13/L25/L06), durable-append-before-ACK against drops; **aggregate** into EXACT usage counters
    (L06 atomic add), NEVER a sketch — L21's approximate counters are disqualified for the billed number
    (keep a sketch ALONGSIDE only for the live "usage so far" dashboard: exact-slow bill vs approx-fast view,
    L29/53 two-shapes); **rate** tier-by-tier + **point-in-time** on event-time buckets (12.5M calls → $0+$4.50+
    $1.00=$5.50; a mid-cycle price change must rate each unit at the price in effect THEN, L53 PIT / L35 event-
    time); **reconcile** — every derived number (aggregate, invoice) is a sum of the durable raw log, so re-sum
    it independently and check against the invoice + the L25 ledger (billed-but-uncollected / collected-but-
    unbilled = drift classes) → HOLD a wrong invoice before it ships. Trace: one event→line item; a retried
    duplicate the event_id claim drops before double-charging (exactly-once EFFECT on at-least-once delivery,
    L13/09); month-end close held behind a **watermark** + grace (L29) so a straggler ingested next month bills
    to THIS cycle by event-time (L35), later stragglers → compensating line next invoice (L25/32). First wall =
    **the dedup memory**: "never counted twice" = remember every event_id for the whole retry horizon = 5.6 TB
    (7-day window × 800 GB/day of 16 B keys), the L13 **TTL trap** in dollars (forget too soon → late dup
    double-charges; keep forever → unbounded) → bound by the max retry horizon, and you may NOT shrink it with a
    sketch (approximate dedup = approximate bill). A **Bloom filter** over the FULL window cuts LOOKUPS the safe
    way — "definitely not seen" (no false negatives) → accept; "maybe" → exact check; its only error, a false
    positive, costs one extra lookup, never a wrong bill (L21 Bloom-as-guard) — but age a key out early and
    "not seen" becomes a lie → double-charge. Secondary walls = hot-account counter = EXACT sharded counter
    (L03/06/21/25, merged total still exact) + late events never fully stop (a cycle is closed-with-correction,
    never sealed, L25/32). Reuses L13 idempotency+TTL, L21 exact-vs-approx + Bloom guard, L25 ledger +
    compensating entries, L09 at-least-once, L29 watermark + batch-exact/stream-fast, L35 event-time, L53 PIT.
    Trade: billing accuracy vs metering cost & latency. (Lesson 0054)
55. ✅ **Edge computing & compute-at-the-edge** — push not just cached bytes (L14) but CODE to L50's 300 PoPs:
    one global app (Virginia origin, 40k req/s) runs auth/routing/personalization/filtering at the PoP nearest
    the user so a request that needs only local data is answered in ~15 ms instead of ~215 ms (the ~200 ms
    round trip skipped, L14 speed-of-light). The counter-intuitive trap: spreading 40k/s across 300 PoPs × 1,000
    functions gives each function ~0.13 req/s (1 per ~7.5 s) → functions keep going COLD, so cold start is the
    COMMON case and must stay under the round trip or the edge is SLOWER than the origin → **isolates** (~5 ms,
    ~3 MB, shared process) are non-negotiable where **containers** (~250 ms > 200 ms saved, ~150 MB) invert the
    win (weaker isolation traded for near-zero start). One rule for STATE: the edge runs the DECISION, the origin
    owns the TRUTH — local-only (JWT verify w/ cached key, signed cookie), eventually-consistent edge KV (flags/
    catalog, L23/L48, seconds stale OK), or FORWARD HOME for anything authoritative (order/money/balance, L11/23/
    25 — a strongly-consistent DB can't exist in 300 places without re-importing the round trip ×300). Trace:
    edge fully answers (~15 ms, origin untouched); cold start WINS on an isolate / LOSES on a container from the
    same code; a "Buy" the edge auths+filters near the user but forwards home. First wall = state (consistency
    can't spread) + operational reach (300-PoP blast radius → thin edge + canary, L31/36 + cross-PoP observability
    L17). Four traps: containers at the edge; authoritative state at the edge; the thick-edge distributed monolith;
    ignoring the blast radius. Trade: latency & offload vs consistency & operational reach. (Lesson 0055)
56. ✅ **Data privacy, deletion & compliance (GDPR/right-to-be-forgotten)** — erase user U-8842 ("Alice") from a
    200M-user platform: she lives in ~15 systems + 35 nightly backups, so the primary-DB row is ~1-2% and truly
    rewriting the immutable tier is ~35 EB/month of restore-and-reseal (impossible), while destroying one 32-byte
    per-user key erases every copy at once (6.4 GB of keys for 200M users). Model: a **data map** (can't delete
    what you can't find), **true deletion** where the store is rewritable vs **crypto-shredding** (destroy the
    per-user DEK, L30) where it's immutable (event log L38, backups, WORM) — the event survives, its PII payload
    becomes noise — plus a **suppression tombstone** (hash(id)+when, no PII) so a restore/replay can't resurrect
    her, and an idempotent orchestrator (L13/L32/L46) to fan out with retries. Trace: trivial mutable DELETE (the
    1%); the immutable log where you destroy the KEY not the log; the backup/warehouse (L29)/edge KV (L55)/third-
    party (L45) sprawl (crypto-shred + retention age-out + forwarded requests). First wall = structural: can't find
    (sprawl) / can't rewrite (immutability) → privacy-by-deletion becomes **privacy-by-encryption**; bounded by
    key management (new crown jewel) and legal holds (L25) that make erasure partial. Four traps: "DELETE the row"
    is deletion; rewriting immutable stores per user; forgetting resurrection; a surviving copy of the key. Trade:
    compliance & privacy vs immutability & derived-data sprawl. (Lesson 0056)

### Advanced topics (next batch — queued so the course never runs dry)
57. ✅ **Bulk data pipelines & backfills** — reprocessing history at scale: backfill a new `currency` column
    across 2B historical `orders` rows (L12/24/25) on a live DB (50k reads/s) with zero downtime. The naive
    single `UPDATE ... WHERE currency IS NULL` detonates THREE ways at once: (1) one all-or-nothing transaction
    holding locks+undo for 400 GB (200 B/row × 2B) → fail at 95% = 50 h rolled back to zero; (2) runs flat-out →
    saturates the disk/CPU serving live traffic → past L27 knee → L07 retry storm; (3) a ~55 h run (2B ÷ 10k
    rows/s) WILL be interrupted (deploy/failover/OOM) and has no resume. Fix = a four-part kit, each disarming
    one detonation: **chunk** into 5,000-row independently-committed batches (400k batches, ~1 MB undo each)
    paged by **cursor not offset** (L18: `WHERE id > :last_id LIMIT 5000` = O(limit) B-tree seek L12, constant
    at any depth; OFFSET re-reads+discards the prefix → O(N²), last batch scans ~2B rows); **checkpoint** the
    cursor after each commit (crash costs ≤1 batch = 0.5 s, not 55 h) — but commit+checkpoint is itself L33's
    dual-write, so put the cursor update IN THE SAME TXN as the batch OR make the batch idempotent; **idempotent
    re-runs** (L13) so the committed-but-uncheckpointed batch that WILL re-run is a no-op (`WHERE currency IS
    NULL` self-guards free; increments/inserts/ledger-appends L25 need a UNIQUE idempotency key); **throttle**
    (L08/28) adaptive on **replica lag** (L06 — backfill writes replicate; outrun the replica → lag climbs →
    read-your-writes breaks → back off; L27 controller pointed at lag, L28 backpressure on a batch job). Trace:
    (A) happy batch (seek→transform→small txn→checkpoint→sleep-to-rate), (B) crash at hour 50 (resume from
    checkpoint, in-flight batch redone or skipped — idempotency makes "did it commit?" a non-question), (C)
    spike → lag crosses 1 s → throttle cuts rate → recovers → ramps (guest yields to host). First wall =
    STRUCTURAL: backfill + production fight over ONE machine's disk/CPU/replication → speed capped by
    production's HEADROOM not worker count (parallelize by key-range L03 → 8 workers = 8× throughput but 8× load
    → spends headroom faster, doesn't escape); real escape = move the heavy history SCAN onto a replica/offline
    copy (L29 warehouse, built for full-history scans), write only results back throttled. Two more walls =
    backfill DURATION = the mixed-state window (partially-populated column, readers coalesce/fallback until
    cutover L24/L31) and the moving target (new rows mid-sweep → ascending-PK sweep handles it, else 2nd pass /
    dual-write bridge L24). Deepest point: a backfill is a SCHEDULING problem wearing a data problem's clothes —
    make the work interruptible/resumable/redo-safe/polite, treat its speed as a loan against spare capacity
    repaid the instant production needs it. Four traps: one giant statement; OFFSET pagination; no checkpoint /
    non-idempotent transform; no throttle / ignoring replica lag. Reuses L24 expand/contract-backfill-step +
    cutover + mixed-state, L18 cursor-not-offset, L12 B-tree seek + row/table sizes, L13 idempotency + L33
    atomic-commit/dual-write, L06 replica lag, L28 backpressure + L27 knee/control-loop, L08 rate limit, L03
    shard-by-key-range, L29 offline/warehouse copy, L07 retry storm, L25 ledger append, L46 durable resumable
    job. Trade: throughput vs impact on the live system. (Lesson 0057)
58. ✅ **Multi-cloud & vendor portability** — the orders platform (L12/24/25/57, ~500 TB, 40k req/s) all on
    Cloud A; leadership wants two clouds for resilience + price leverage. The naive "it's just containers, deploy
    to B + point DNS at both" is CORRECT about compute and SILENT about data — and data is the whole problem
    (compute is light/portable, data is heavy/stays put). Estimate the two walls: **data gravity** (seed 500 TB
    to B = 500,000 GB × $0.09/GB = **$45k** egress once, + at 10 Gbps=1.25 GB/s → 400,000 s ≈ **4.6 days**, chasing
    a moving target L57) + the **boundary tax** — every cross-provider byte is metered FOREVER (replicate the 5 MB/s
    write stream ≈ 13 TB/mo = **$1,170/mo**, but serve 20k reads/s×2 KB across the line = 104 TB/mo = **$9,300/mo
    ≈ $112k/yr** → the rule: keep compute next to its data; ingress free, egress $0.09/GB is the roach-motel L14).
    Model 3 layers by portability: **stateless compute** (same image runs anywhere → do it, the easy half),
    **stateful data** (heavy by gravity + sticky by proprietary managed-API SHAPE → anchors you, the whole cost),
    **control plane** (global DNS/traffic mgr L50 spans both clouds free; a strongly-consistent truth / leader
    CAN'T span them without re-paying L23's ~80-150 ms cross-boundary quorum + egress/message). Core dial =
    **managed leverage** (hosted DB/queue/serverless: they operate it, proprietary API = locked in) vs **LCD**
    (VMs/K8s/self-hosted Postgres/S3-compat: portable, but YOU run it again L10/34/47) — portability is insurance
    with a continuous premium, pay it where a claim is likely not reflexively. Trace: (A) stateless request just
    works (DNS steers, either cloud serves) until "reads/writes the data — where?"; (B) stateful fork — Option 1
    single-home data in A (B reaches across = per-read egress $112k/yr + latency, cheap to build/ruinous to run)
    vs Option 2 replicate (each cloud local copy → active/passive standby-promote OR active/active = L23 conflicts
    ACROSS providers, scalar invariant L25 still can't split the ocean); (C) Cloud A provider-wide outage — Option 1
    = B is a compute SHELL with no data = total outage anyway = **resilience theater**; Option 2 active/passive =
    DNS fails over → promote B's replica → survives (cost: replication bill + promotion window + L06/23 lag =
    data-loss window). First wall = **data gravity anchors the system**: compute was always portable, the cloud
    locked you in with the heavy/metered/proprietary DATA → (1) the "we'll move to negotiate" leverage is a BLUFF
    unless portability is paid continuously (and if it is, you've pre-paid most of the switching cost it was to
    save); (2) real escape = don't fight gravity, shrink what crosses the boundary to the thin replication DELTA
    (small) never the read traffic (huge) — L57 move-the-scan / L14 copy-near-the-user. Bounded further by strong
    consistency that can't span providers cheaply (→ active/passive over active/active, or single-home the
    invariant L23) + a DOUBLED multiplicative ops surface (2× IAM/networking/monitoring/on-call/bills L59). Honest
    answer for most = a warm DR copy, not symmetric active/active. Four traps: multi-cloud=deploy-it-twice (no data
    story); single-homed data behind a "resilient" 2nd cloud (theater); LCD-everything (throws away the managed
    leverage that IS the reason to be on a cloud); believing the leverage bluff. Reuses L14 egress/copy-near-user,
    L23 multi-region active-active/single-home-invariant, L50 global traffic mgr front door, L26 push-state-off-box
    (compute portable), L57 move-heavy-scan-off-primary, L10/34/47 cost of self-run stateful infra, L06/11/25
    consistency limits. Trade: portability & resilience vs complexity & cost. (Lesson 0058)
59. ✅ **Cost-aware architecture (FinOps)** — a $250k/mo cloud bill on the orders platform (L12/24/25/27/57/58,
    40k req/s, ~500 TB) and finance's one ask: "cut it 30% without breaking anything." Estimate = read the bill:
    it decomposes into COST DRIVERS on a Zipf curve (L02) — compute $88k/35%, storage $70k/28%, egress $60k/24%
    = ~87% in three drivers, "everything else" (queue/cache/KMS/LB/logs/DNS) $32k/13% = noise (20% off the tail =
    $6.4k, 20% off the drivers = $43.6k → the 30% goal only reachable through the drivers). Biggest hidden lever
    = UTILIZATION: 300 servers peak-sized (L27 knee) but avg load ~40% of peak → avg util ~40%×70% ≈ 28% → ~72%
    idle = ~$63k/mo of paid-for-nothing (reclaim MOST via autoscale-to-curve L27, NOT permanent shrink); egress
    hides CROSS-AZ chatter (3 AZs L36, $0.01/GB each way, ~$2.6-5.2k/mo on no dashboard). Model = cost as a
    DESIGNABLE quantity: (1) UNIT ECONOMICS = bill ÷ units ($250k/41.5B req = $6/M req, 0.06¢/order; cost-per-unit
    > revenue-per-unit → growth loses money, scale can't fix a broken unit economic); (2) ATTRIBUTION
    (showback/chargeback, L54 turned inward + L40 per-tenant) = you can't cut what you can't see, reveals top 3%
    tenants drive 40% cost + the free-tier tenant w/ NEGATIVE unit economics (10× req/order) invisible in the
    total; (3) three DRIVER-KNOBS each an earlier lesson — compute→utilization (L27 autoscale/right-size/reserved-
    for-baseline/spot-for-batch), storage→tiering (L39 hot→cold→archive), egress→locality (L14/58 cache-near-user/
    compute-next-to-data/kill-cross-AZ). Trace: (A) a read priced not timed — cache HIT ~$2e-8 vs MISS ~$6e-6 =
    ~300× (app CPU + DB read + cross-AZ + full egress) → cache saves DOLLARS not just latency, cost & perf aligned;
    (B) the L29 analytics query priced two ways — on the serving DB (evicts hot cache → Path A's 300× across normal
    traffic + bigger boxes) vs on a columnar warehouse over cold-tier bytes on SPOT (L39/57) = budget-buster vs
    rounding error, cost is a property of WHERE you run it; (C) the backfire — cut 300→200 boxes saves $29k/mo,
    runs fine at avg load, then a Black-Friday 2× spike (80k req/s, 200 boxes serve ~27k at the knee = 3× past it)
    → L27 cliff → L28 unbounded queue → L07 retry storm → ~3h checkout outage on a $50M/mo book (~$69k/hr) = $200k+
    → the deleted "waste" was the HEADROOM the spike needed. First wall = TWOFOLD: (1) cost is INVISIBLE until
    attributed (one $250k line can't tell the $44k cut from the $6k cut → tag/decompose/attribute/unit-economics
    FIRST, measure before turning anything off), then (2) the CHEAPEST DESIGN IS THE WRONG ONE — minimizing cost in
    isolation yields the worst system (no headroom = Path C, no replicas = L07 AZ-failure downtime, all-spot =
    reclaimed mid-checkout, all-cold = L39 retrieval cliff, no cache = Path A's 300×); cost is a CONSTRAINT to
    co-optimize within the SLOs, never a scalar to minimize — the idle that looks like waste on a spreadsheet is
    the spike buffer (L27) + failure margin (L07). Deepest point: the cloud turned cost into a RUNTIME variable
    (every placement/tier/headroom decision priced by the second) → cost is now a design-time property as real as
    latency, engineered on purpose — but the smallest bill and the best system are rarely the same design. Four
    traps: optimize the visible not the expensive (shave the 13% tail); cut headroom to hit a number (Path C);
    over-commit reservations (3-yr lock on a shifting baseline); ignore egress/cross-AZ (the byte that looks like an
    internal call). Reuses L27 fleet/knee/autoscale/headroom, L39 storage tiers, L14/58 egress/locality, L28/07
    spike+failure margin, L40/54 per-tenant accounting + metering (attribution/unit economics), L02 Zipf skew +
    cache hit-vs-miss, L29/57 heavy work on cheap/spot capacity. Trade: unit cost vs performance & headroom.
    (Lesson 0059)
60. ✅ **Rendering & delivery at scale (SSR/edge rendering)** — one product page onto a phone (40k req/s, ~100 ms
    to origin / ~15 ms to edge, 1.5 MB/s mobile): rendering is a PLACEMENT problem, not a front-end detail — the
    same HTML can be built at build time (static/SSG, cache it L14, ~0 cores, generic), on the origin (SSR, fresh +
    personal but 30 ms × 40k = ~150 render cores L59/27), at the edge (L55, ~15 ms near user, local data only), or on
    the phone (CSR, server ~0 but blank-then-waterfall ~770 ms to content + no SEO). Estimate: cost hides in serial
    round trips (dependent hops chain the ~100 ms L14 tax) and phone CPU (300 KB bundle = ~200 ms download BUT
    ~300 ms execute, and JS exec doesn't get faster with network). Two dials: WHERE built + how much must HYDRATE
    (server/static send finished HTML → fast FCP, but inert until the bundle downloads+executes to attach handlers →
    the FCP-to-TTI gap "looks ready, taps do nothing"; cost = bundle size → ship less = islands/partial hydration).
    STREAMING SSR flushes the shared shell now, streams the slow per-user slice (200 ms rec ML call L53) when ready /
    degrades to a placeholder (L28 don't-block-fast-on-slow, L49 graceful degrade). First wall = can't be BOTH fully
    cacheable AND fully personalized (personalization = unique response = CDN hit → 0 = L14's stray-param cache-killer
    reborn, ~150 cores back) → draw the PERSONALIZATION LINE: cache the shared shell (product/reviews/price, ~15 ms,
    ~0 cores), render only the per-user slice ("Hi Alice"/cart/recs) as a client island / streamed chunk / edge
    personalization; the dial = fraction of page that's per-user. Second wall = HTML cheap, interactivity (hydration)
    expensive → islands/server-components. Deepest: where you build HTML = same placement question as where a cache /
    computation / the truth lives, answered for the last hop to the screen. Four traps: personalize a cacheable page;
    hydrate the whole tree; pure CSR for content/SEO pages; slow slice blocks the fast shell. Reuses L14 CDN/TTL/shell-
    vs-slice/stray-param, L55 edge render/decision-vs-truth, L59 render priced per request, L27 knee/~150 cores, L28
    backpressure + L49 BFF graceful-degrade (streaming), L53 rec ML call, L02/48 cache one-copy-vs-N. Trade:
    time-to-first-byte & offload vs freshness & compute. (Lesson 0060)
61. ✅ **Bot detection & traffic authenticity** — the orders front door (40k req/s, L27/49/55/60) + its login
    endpoint: a request arrives as a bare HTTP call with no "I am human" stamp, and ~40% of traffic is automated
    (~16k req/s = a ~60-server render bill L59 before any abuse). The obvious defense fails: a per-IP rate limit
    (L08) catches the loud scraper (1 IP @ 2,000/s) but is BLIND to credential stuffing spread over 10,000 proxy
    IPs @ 1 attempt/min each (0.017 req/s/IP, under any per-IP limit) grinding 10M stolen pairs in ~16.6 h — the
    attack is only visible when you SUM the clients. Model = a FUNNEL (L14/16 narrow-cheap-perfect-expensive):
    [1] free passive **signals** (rate, IP reputation, TLS/header **fingerprint/JA3** — a UA claiming Chrome over
    a python-requests TLS handshake is a catchable lie), [2] active **challenges** (JS test / CAPTCHA / **proof-of-
    work** — 20 zero bits ≈ 2²⁰ ≈ 0.5s CPU, free for one login but 10M attempts × 0.5s ≈ 58 CPU-days: flips a free
    attack to priced-per-attempt), [3] two detection styles run TOGETHER — **rate-based** (catches the loud single
    client, blind to distributed) + **behavioral/aggregate** (catches the population: login-fail ratio spiking
    3%→47%, a shape no single IP shows). Trace the loud scraper (caught free at layer 1 → tarpit), the low-and-slow
    botnet (rate+fingerprint blind since it runs a REAL headless browser → only the endpoint-level fail-ratio
    behavioral signal + PoW/account step-up stop it), and the **graduated response** (score 0–1 → allow the ~99% /
    challenge the middle invisibly-first / block the confident) that keeps friction off humans. First wall = perfect
    separation is IMPOSSIBLE (a program can be built to look arbitrarily human → detection is a PROBABILITY, and the
    two errors trade off: a served bot = cheap+visible compute, a blocked human = expensive+INVISIBLE lost customer
    that never shows in your metrics → cranking sensitivity destroys the business it protects) → reframe from
    DETECTION to ECONOMICS: make each abusive attempt cost more than it earns (PoW/step-up/tarpit) while taxing real
    users ≈0, graduated response applying the cost only where suspicion is. Second wall = the adversary ADAPTS (L52):
    any single signal, once keyed on, gets mimicked and dies → **defense in depth** (many weak INDEPENDENT signals),
    mixing client-side (forgeable: fingerprint/JS/mouse) with server-side AGGREGATE (unforgeable: this account's fail
    rate across all IPs, this endpoint's population behavior) — a bot fakes its own request perfectly but can't fake
    the pattern that only emerges when the server sums thousands it has no view into; models retrained continuously
    (never "done"). Four traps: per-IP limiting alone (blind to distributed); max sensitivity (maximizes the costly
    invisible false positive); trusting one client-side signal (real browser forges it); CAPTCHA everyone (15–30%
    human abandonment > the bots' cost). Reuses L08 rate limiting + its distributed blind spot, L52 adaptive
    adversary → defense in depth, L55 edge (where the decision runs), L59/60 wasted render compute, L49 gateway
    admission control, L14/16 narrow-cheap-perfect-expensive funnel, L07/28 admission/reject-at-the-door. Trade:
    abuse reduction vs friction & false positives. (Lesson 0061)

### Advanced topics (next batch — queued so the course never runs dry)
62. ✅ **Consensus internals (Raft)** — open the black box used as "a consensus cluster" in L10/22/34/43: five
    nodes hold a tiny replicated config store (the lock/leader/registry truth) and must agree on ONE ordered log of
    commands despite crashes + an unreliable network, so every copy stays identical (**replicated state machine**:
    same commands + same order = same state; agree on the LOG and you've agreed on everything). Estimate the magic
    of **majority** (⌊N/2⌋+1 = 3 of 5): any two majorities of 5 OVERLAP in ≥1 node (3+3>5, L11 pigeonhole) → two
    leaders per term impossible AND a committed entry impossible to lose; fault tolerance N=2f+1 → f=2, and even N
    buys nothing (6 also tolerates 2, L10) so clusters are odd. Price a commit = **one fsync + one majority round
    trip** (~2 ms intra-DC, ~30 ms cross-region, L11/23 tax); ceiling ~1,000/s per leader (fsync-bound) lifted ~100×
    by **group-commit batching** (L47/33, amortizes the flush, never the RTT). Model Raft's three pieces: (1) leader
    election — **terms** (a logical clock L35 + fencing epoch L10/22, old-term msgs rejected), one vote/term,
    majority wins, at most one leader/term by overlap; (2) log replication — leader appends (idx,term,cmd) → sends
    **AppendEntries** → **committed** when a majority stores it DURABLY → apply + ack + propagate commitIndex;
    **log-matching** (prev-entry check) auto-repairs stragglers so logs converge to identical prefixes; (3) safety —
    the **election restriction** (refuse to vote for a less-up-to-date log) chains with overlap so a new leader ALWAYS
    holds every committed entry (**Leader Completeness**). Trace: (A) clean ~2 ms commit that never waits for the two
    slowest nodes (majority, not all); (B) leader crash → election in ~150–300 ms, **randomized** timeout breaks the
    split-vote symmetry that would loop forever (L10 flap / L26 herd); (C) the crux — a **committed** entry (majority,
    client acked) survives any leader change (overlap carries it), while an **uncommitted** entry (minority, client
    saw a TIMEOUT = unknown, L13) may be safely OVERWRITTEN because the client retries with an idempotency key (L13) →
    committed vs uncommitted IS the exact boundary of what a distributed system may promise. First bottleneck =
    agreement is a toll paid per decision (durable majority): attack cost-per-entry not the toll → batch + pipeline;
    when one leader still isn't enough, **shard into many Raft groups** (multi-Raft, L03, capstone L63) — throughput
    scales with #groups but you LOSE the single global order (cross-shard → 2PC/saga L32); consensus is your most
    expensive storage → put ONLY agreement-critical metadata in it, keep bulk data out (L20/30 metadata/data split);
    a majority must be alive or the cluster STOPS (CP, L11 — opposite of L34's AP routing pick), and Raft/Paxos handle
    crash-stop not Byzantine (liars need BFT). Deepest: consensus is the one place "it probably worked" is banned —
    everywhere else we embraced approximation (L02/06/09/21) because it was cheaper; here disagreement corrupts
    everything downstream, so you pay a round trip + flush per decision to buy a fact every node agrees on forever.
    Paxos = the original, same guarantees, famously harder to follow; Raft trades nothing in correctness for teachability.
    Four traps: ack on the leader's LOCAL write (lost acked write); a fixed election timeout (split-vote storm); bulk
    data in the cluster; expecting availability under partition (CP stops = working as designed). Reuses L11 quorum
    overlap, L35 logical clock (term), L10/22 fencing/leader election/flap, L13 idempotency + timeout=unknown, L26
    jitter, L47/33 group commit, L20/30 metadata split, L34 the opposite AP pick. Trade: correctness &
    understandability vs latency of agreement. (Lesson 0062)
63. ✅ **Building a distributed key-value store (capstone)** — assemble the course into ONE Dynamo-style store:
    a 100-node / 3-DC / 1 TB shopping-cart store that must stay writable while a DC is on fire and never lose a
    cart-add (the "always writable" rule → AP, L11 — the seed of every choice). Estimate the fleet by
    REPLICATION-AMPLIFIED load (N=3 → 3 TB, ~3k writes + ~10k reads/node) and the quorum table where ONE inequality
    `W+R>N` slides the whole system between consistency & availability (pick N=3,W=2,R=2 as the balanced default;
    W=1 when never-reject beats staleness). Model six primitives as THREE PLANES: placement = consistent-hashing
    ring + virtual nodes → a **preference list** of the next N=3 DISTINCT physical nodes/racks (L03/04, a pure
    function so any node coordinates with zero lookups); replication = quorum W/R + **sloppy quorum** + **hinted
    handoff** to stay writable through failure + **version vectors** (L23/35) to tell concurrent from ordered;
    membership = **gossip** (L43, no-SPOF AP registry, O(log N)) + **Merkle anti-entropy** (L43, repair only the
    differing keys, not a 30 GB copy); over an **LSM engine** (L47); beside a small **Raft core** (L62) that owns
    the ring/token map (L20/30 metadata/data split). Trace (A) healthy W=2 write that never waits for the slow 3rd
    replica (L62's "majority not all" tail trick); (B) read that finds two CONCURRENT versions (vectors: neither ≥
    other) → keep BOTH siblings → merge by cart UNION (a business rule) + **read repair** the stale replica, NOT
    silent LWW; (C) DC partition → keep writing on both sides (available), two truths briefly exist (not
    consistent), reconcile on heal via hinted handoff + anti-entropy + merge = CAP's A-over-C paid in exactly that
    machinery. First bottleneck = the SEAMS where primitives fight: (1) sloppy quorum buys availability by VOIDING
    the `W+R>N` overlap → the reconciliation stack exists solely to clean up the divergence you allowed; (2) a HOT
    KEY still bakes its N replicas — consistent hashing fixes rebalancing not skew (L04) → cache (L02) / key-split
    (L21) / extra read replicas, not a better hash; (3) the ring must be CP even though the data is AP (a gossiped
    ring → disjoint replica sets → silent loss); (4) the top-level AP/CP fork (leaderless quorum vs Raft-group-
    per-shard, L62) is decided ENTIRELY by "what does this data cost when briefly wrong?" — mergeable cart → AP,
    un-mergeable balance → CP (L11/25), same six primitives assemble into EITHER machine. Deepest point of the
    course: a distributed DB isn't a monolithic invention — it's these primitives composed; read Dynamo/Cassandra/
    Bigtable/Cockroach as different SETTINGS of the same knobs, and design a new one by choosing them deliberately.
    Four traps: gossip the ring (→ silent loss); LWW on a value that must merge; expect consistent hashing to fix a
    hot key; read `W+R>N` as unconditional (a sloppy quorum voids it — by design). Reuses L03/04 ring, L06/11 quorum
    overlap + CAP fork, L23/35 version vectors, L43 gossip/Merkle, L47 LSM, L62 consensus core, L02 cache/L21 shard
    (hot key), L17/62 tail trick, L20/30 metadata split. Trade: tunable consistency vs availability & operational
    complexity. (Lesson 0063)
64. ✅ **Stream processing & windowing** — a live sales leaderboard ("revenue in the last minute, per product,
    refreshed every second") over a 100k-events/sec purchase stream, the streaming answer to L29's stale batch
    warehouse. Estimate the shape-defining fact: state scales with KEYS × WINDOWS (~1M active products × 24 B ≈
    24 MB running sums), NOT with the 20 MB/s firehose — an aggregate collapses many events into one number per
    key (L21), flipping the cost model so the expensive/fragile thing is STATE, not throughput; re-running batch
    every second re-scans 60× (L29 cost ∝ history×frequency), streaming reads each event ONCE (cost ∝ new data).
    Model three window shapes (tumbling = 1 update/event; sliding 60s/10s = 6 overlapping windows = 6× state, the
    responsiveness-vs-state trade; session = gap-defined, unbounded-state hazard on a never-idle entity) and the
    idea it all turns on: bucket by EVENT TIME (fixed at source, replayable) not PROCESSING TIME (arrival →
    wrong + non-deterministic); a WATERMARK = max_event_seen − allowed_lateness asserts "seen everything ≤ t" and
    fires a window when it passes the window end — allowed-lateness is the latency-vs-completeness dial (L17 tail /
    L29 late data). State lives in RAM + a local LSM store (L47); CHECKPOINT snapshots {all state + source offsets}
    together so recovery restores state AND rewinds the log (L09) to the matching offset → each replayed event folds
    in ONCE = exactly-once STATE, NOT exactly-once delivery (L09/13 — sink still needs idempotent upsert by
    (window,key) or a transactional/outbox sink, L13/33). Trace: (A) a 5s-late event lands in its correct
    [10:00,10:01) window because a 5s watermark held it open; (B) same sale = 1 update tumbling vs 6 sliding;
    (C) worker crash → restore last checkpoint + rewind offset + replay ≤1 interval of events → sum climbs back
    exactly, no double-count. First bottleneck = the STATE itself: rescaling 20→40 workers must RELOCATE keyed
    state, and hash%N remaps ~94% (L03) → minutes of downtime → fixed KEY GROUPS (1024, consistent hashing L03/04
    on state) move only ~1/N; checkpoint interval = steady overhead vs replay time (L24/02 two-sided dial),
    incremental checkpoints (L47) cut it; unbounded state (never-idle session / no-window global / stalled
    watermark from a dead partition) → OOM, every window needs a reason it closes (fire/gap/TTL, L20/39). Four
    traps: processing-time bucketing; exactly-once state ≠ exactly-once delivery; %N rescaling; unbounded state.
    Reuses L29 lambda/kappa speed-layer, L09 log/offsets/replay, L13 idempotent sink, L17/29 watermark/tail, L33
    transactional sink, L47 LSM state backend, L21 aggregate-collapse, L03/04 consistent hashing (key groups),
    L24/02 interval dial, L20/39 lifecycle. Trade: latency & correctness of windowed results vs state size &
    recovery cost. (Lesson 0064)
65. ✅ **Vector databases & semantic search** — a semantic search over L48's 10M-product catalog that finds items by
    MEANING where L16's keyword inverted index can't (the query "comfortable shoes for marathon training" must match
    "cushioned long-distance running sneaker", zero shared words). Turn each product+query into a 768-dim vector
    (meaning as geometry, closeness = similarity); "find like this" = nearest-neighbor. Estimate the shock: NN has no
    key, so the EXACT answer reads the whole ~30 GB index (10M × 3 KB) every query → ~1 s at ~30 GB/s ≈ 1 QPS/box,
    ~1000× short → must approximate. Model: embeddings + cosine; why B-trees/k-d trees die past ~10-20 dims (curse of
    dimensionality — near≈far, nothing prunes, L12); two ANN engines — HNSW (navigable small-world graph, greedy walk
    ~log₂(10M)≈23 hops, visits ~2,000/10M = 0.02%, +15-30% graph memory, costly updates) and IVF (k-means into
    √N≈3,162 clusters, probe nearest nprobe=16 → ~50k = 0.5% scanned, L16-flavored) — plus product quantization (3,072 B
    → 96 B, 32× smaller, recall bought back by full-precision re-rank, L21 summary-vs-exact) and HYBRID keyword+vector
    fused by reciprocal rank (exact SKU/brand + semantic meaning, L16). Trace: query end-to-end ~17 ms (embed → ANN →
    re-rank → fuse → filter); HNSW 97% recall reading 6 MB not 30 GB (~5,000× less); the recall knob (efSearch/nprobe)
    curve bends viciously — 0.97→0.99 ~4× work, 0.99→1.0 = the whole 30 GB scan back. First bottleneck = the
    recall–latency–memory TRIANGLE (pick two, third pays) + churn rots an ANN graph → delta index + rebuild-and-swap
    (L31/47) + a model upgrade invalidates EVERY vector at once (v1/v2 coordinate systems not comparable, can't mix) →
    full re-embed of 10M + versioned rebuild + atomic cutover (L24/57), query model must match index model. Four traps:
    exact trees for high-D NN; chasing 99.9% recall; mismatched query/index model; vector-only for exact tokens.
    Reuses L16 (inverted index it complements + fusion), L12 (curse of dimensionality, skip-list shape HNSW borrows),
    L48/2 (10M catalog, index across a fleet), L47 (LSM compaction ≈ rebuild-and-swap), L31 (blue-green cutover),
    L24/57 (model versioning + backfill re-embed), L21 (quantization = summary-vs-exact). Trade: recall/quality vs
    query latency & index cost. (Lesson 0065)
66. ✅ **Global rate limiting & quota** — L08's single-node limiter coordinated across a 30-server, 3-region fleet
    enforcing "60/min per key." Estimate: a central counter per request adds a cross-region round trip (~70 ms ≫ the
    ~5 ms of work, a 14× tax) + one hot key (L3/48); static division throttles a skewed key to 1/3 (idle budget
    stranded) or, at full-limit-per-node, leaks 30× (30 × 60 = 1,800/min). Model: local token buckets + periodic
    reconciliation (busy nodes borrow idle nodes' unused budget) — safe because rate limiting tolerates bounded
    approximation (L21), "close enough" is the spec. Coordination cost is per-active-node-per-interval, flat in
    request rate (cheap exactly for hot keys). Trace: well-behaved key served with ZERO hot-path coordination; abusive
    key (A=10 req/s) overshoots ≤ A×T = 10 (halve T → halve overshoot, 2× chatter); a partition = CAP (L11) —
    fail-open leaks / fail-closed 429s good traffic / safe default = degrade to a static per-region share. First
    bottleneck = a shared limit is ONE logical counter every request contends on; can't be exact + cheap +
    partition-proof at once, T is the master dial; hot key one counter can't shard (L4) → bigger local share + shard
    coordinator by key (L3); fixed vs sliding window / bucket refill so the boundary can't be gamed (L08); hard
    money/resource quotas (L40) that can't overshoot flip to pre-authorized leases (coordinator = sole issuer, never
    over-issues, at the cost of trapped budget). Four traps: central counter per request; static division; overshoot
    as unknowable fuzz; ignoring the partition. Reuses L08 (token bucket/window it distributes), L21 (approximation as
    correct spec), L11 (CAP under a shared counter), L40 (per-tenant quotas + lease), L3/4 (hot key, shard the
    coordinator), L42/27/49 (the fleet + front door). Trade: limit accuracy vs coordination latency & cost. (Lesson 0066)
67. ✅ **Data lakes & the lakehouse (open table formats)** — L29's 11 TB / 11B-row / 3-year orders history stored as
    ~1.58M small Parquet files on object storage (L20), cheap & open to many engines but a *directory is NOT a table*.
    Estimate the "directory = table" lie: planning one query = LIST 1.58M keys (~1,580 paginated calls) + open every
    footer for stats ≈ 1.58M × 1 ms ≈ **26 min** before the scan; and any multi-file change (compact 1,440 × 7 MB files
    → 78 × 128 MB) is a **race** a reader catches half-done → silently wrong revenue. Fix = a thin **metadata layer**
    (catalog pointer → snapshot → manifest w/ per-file min/max stats) that names the table exactly & precomputes pruning
    → planning drops to a sub-second manifest scan (partition pruning + file skipping, L12/29), no LIST. THE idea: reduce
    an atomic change over MANY immutable files to a single **compare-and-swap of ONE tiny pointer** (L06/14 rename-don't-
    mutate/20 commit-last) → ACID atomicity, snapshot isolation (L32), time travel + instant rollback (L31/38), safe
    compaction (L20/29 small-file fix), schema evolution by column-ID (L24), optimistic-retry concurrent writers (L06),
    all corollaries of the one swap. Trace: read pruned sub-second on a pinned snapshot; write committed atomically or
    loser re-reads winner's snapshot & retries; bad batch rolled back by pointing catalog at older metadata (L56 caveat:
    time travel keeps deleted data → snapshot expiry is what finally deletes it + reclaims storage). First bottleneck =
    the hard part MOVED into the metadata: manifests bloat (1.58M entries → hundreds of MB) & need their own compaction +
    snapshot expiry; the commit is **one serialization point per table** (L66's shared counter → batch commits, L09/47);
    commit interval = L29's freshness-vs-cost dial reborn (pair w/ a speed layer L64 for sub-second). Four traps:
    directory-as-table; per-row commits; never compact/expire; expecting warehouse speed + zero management. Trade:
    openness & cheap shared storage vs query performance & metadata-management complexity. (Lesson 0067)
68. ✅ **Delivery guarantees & the dead-letter lifecycle** — one order event through a 4-hop pipeline (ingest →
    payment → fulfillment → notify, 2,000/s): end-to-end at-least-once vs effectively-once (L09/13/33). Estimate the
    two-generals gap (exactly-once *delivery* impossible; an ack can always be lost) and the duplicate cost (0.05%
    crash-before-ack → ~86,400 double-charges/day at one hop, ~345,600/day across four). Model = at-least-once + idempotent
    consumers, dedup living at EVERY hop keyed by a producer-stamped idempotency key carried through, effect+key in ONE
    atomic commit (L13/33 — else a crash silently loses or doubles). Trace: clean order; duplicate absorbed to a no-op;
    poison message head-of-line-blocking ~15,500 good orders (500/s × 31 s retry ladder) until quarantined to a DLQ;
    DLQ triage + replay (safe because idempotent). First bottleneck = certainty is STORED STATE: dedup store ~35 GB/day
    (691M keys) needs a bounded TTL window (L8/48/21); dedup lookup taxes the hot path & the un-acked backlog is
    backpressure (L27/28); effectively-once reaches only as far as effects can be made idempotent. Four traps:
    chasing exactly-once delivery; dedup not atomic with the effect; retrying a poison message forever; a DLQ with no
    tooling. Trade: delivery certainty & operability vs pipeline latency & storage of in-flight state. (Lesson 0068)
69. Hot/warm/cold path & the serving vs analytics split — one event feeding a low-latency serving store, a
    streaming speed layer (L64), and a batch warehouse (L29) at once, kept consistent enough; the read-path router
    that picks a path per query (L37 CQRS). Trade: freshness & query latency per path vs duplicated pipelines & cost.
70. Idempotent, resumable data APIs (uploads & long jobs) — resumable multipart uploads (L20), long-running job
    status endpoints (L05), and safely retryable mutations (L13/18) over flaky mobile networks: chunk + checksum +
    commit-last (L20/25), resume tokens, and progress polling. Trade: robustness on bad networks vs protocol &
    server-state complexity.

### Advanced topics (next batch — queued so the course never runs dry)
71. Quorum reads/writes & tunable consistency — the R + W > N dial from the inside (L06/11/63): pick R, W, N per call
    to slide between fast-but-stale and slow-but-fresh, read-repair and hinted handoff on the read path, why W=N kills
    availability. Trade: per-request consistency vs latency & availability.
72. Bulk & batch APIs (the N+1 and fan-out problem) — one screen needing 200 records: why 200 round trips (L18) is the
    silent killer, batch endpoints, request coalescing/dataloader, GraphQL-style field selection, and the over-fetch vs
    under-fetch tension. Trade: fewer round trips & payload control vs API & caching complexity.
73. Deduplication & entity resolution at scale — "are these two records the same customer?" across 500M rows: blocking
    keys to avoid N² comparisons, similarity scoring, MinHash/LSH (L65 neighbors), and the merge/survivorship decision.
    Trade: match recall vs precision & compute cost.
74. Config & feature-flag delivery at scale — push a config/flag change to 40k servers (L27) in seconds without a
    deploy: a versioned config plane, pull-with-long-poll vs push, staged rollout & instant kill-switch (L31/51),
    consistency of "everyone on the same flag." Trade: change speed & safety vs a new always-on dependency.
75. Append-only audit logs & tamper evidence — an immutable, verifiable record of "who did what when" (L38/56):
    hash-chaining each entry to the last (a mini-blockchain), Merkle proofs a single entry is present & unaltered, and
    write-once storage. Trade: verifiability & compliance vs write cost & the impossibility of edits.

## Lesson format conventions
- Four reusable "moves" framing introduced in Lesson 01: estimate → model →
  trace the paths → find the first bottleneck. Reuse this spine across lessons.
- Every design choice should NAME its trade-off explicitly (that's the skill).
