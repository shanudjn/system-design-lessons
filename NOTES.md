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
35. Logical clocks & causal ordering — order events across machines with NO trustworthy
    global clock (L23's skew, L22's clock-fragile lease): **Lamport** timestamps give a
    total order that respects causality (happened-before) but can't tell concurrent from
    causal; **vector clocks** (L23 recap) can, at O(N) size; **hybrid logical clocks** bolt
    physical time on so timestamps are both causal AND ~wall-clock. Trade: ordering
    precision vs metadata size & coordination.
36. Cell-based architecture — the ultimate blast-radius tool (L07 bulkhead, L28, L31 all
    point here): partition the WHOLE stack (LB + app + data) into independent **cells**,
    route each user to one cell, so a bad deploy, poison input, or hot tenant takes down
    1/N of users, not all of them. Cell sizing, routing/placement, and the shuffle-sharding
    refinement. Trade: blast-radius isolation vs cross-cell coordination & overhead.

### Advanced topics (next batch — queued so the course never runs dry)
37. Read/write splitting & CQRS — the L33 pipeline naturally produces separate **read models**;
    formalize it: one write side optimized for correctness/consistency (L06/25) feeds, via CDC/
    events (L33), many purpose-built read sides (search L16, cache L02, denormalized views L15/29)
    — commands vs queries as two shapes of the same data. When the split earns its keep vs when it's
    accidental complexity. Trade: read/write optimization vs the eventual-consistency gap between them.
38. Event sourcing — take L33/L25 to its limit: store the **stream of events** as the source of
    truth and DERIVE current state by folding them (the ledger L25 generalized), so history is free,
    replay (L09/33) rebuilds any view, and audit is total — paid for in snapshotting to avoid
    re-folding millions of events, schema-evolving old events, and the "how do I query current state"
    problem (→ CQRS L37). Trade: perfect auditability/replayability vs query complexity & storage.
39. Tiered storage & data lifecycle — the L20/29/33 data piles grow forever; not all bytes are worth
    the same. Move data hot→warm→cold→archive (RAM/SSD/HDD/object-store/glacier) by age &
    access frequency (L02 skew again), with TTL/retention policies and the retrieval-latency cliff
    of cold tiers. Compaction, tombstones (L20), and the cost-vs-latency dial per tier. Trade:
    storage cost vs retrieval latency & operational policy.
40. Multi-tenancy & noisy neighbors — one platform, many customers sharing a fleet (L27) and DBs
    (L03): isolate them so one tenant's spike/abuse can't starve the rest — per-tenant rate limits
    (L08), quotas, fair queuing, and the shared-vs-silo-vs-pool spectrum (row-level → schema →
    database → cell L36 per tenant). Tenant routing, the hot-tenant problem (L03/19), and
    shuffle-sharding (L36) as the blast-radius refinement. Trade: density/cost efficiency vs
    isolation & fairness.
41. Graph & relationship systems — "who are my friends' friends?" over a billion-edge social graph:
    why a relational JOIN explodes at depth (L12 N+1 across hops), adjacency lists vs a native graph
    store, partitioning a graph without cutting every edge (the hard part — supernodes/celebrities
    recur, L15), and BFS/traversal at scale. Trade: traversal speed vs partitionability of connected data.

## Lesson format conventions
- Four reusable "moves" framing introduced in Lesson 01: estimate → model →
  trace the paths → find the first bottleneck. Reuse this spine across lessons.
- Every design choice should NAME its trade-off explicitly (that's the skill).
