# Learning Record — Lesson 0053: Feature Stores & ML Serving Infrastructure

**Date:** 2026-08-09
**File:** `lessons/0053-feature-stores-ml-serving.html`
**Curriculum topic:** #53 (Advanced batch — "systems that run on top of the infrastructure," second turn)
**Trade named:** feature freshness vs serving latency & training-serving consistency

## The worked example
One **fraud model** scoring each payment in a ~50 ms budget. The model doesn't read the raw
transaction; it reads ~200 **features** — precomputed signals about the entity: `card_txn_count_1m`,
`user_avg_amount_30d`, `merchant_fraud_rate_7d`, `amount_vs_user_avg`. The lesson's question is not
"is the model good?" (assume yes) but "how do we hand it ~200 correct, current values in a few ms,
10k/s, computed the SAME way as in training?" — i.e. the **feature store**. First lesson to design
the ML-serving plumbing that L52's classifier, L16's ranker, and recommenders all silently depend on.

## The four moves
1. **Estimate — request-time computation is impossible on both axes; precompute + look up.**
   Naive: 10,000 txns/s × 200 features = **2,000,000** aggregation queries/s = **400×** a DB's safe
   ~5,000 q/s (melts checkout), AND one request running 200 sequential ~5 ms aggregations = **~1,000 ms**
   = **20×** the 50 ms budget. No faster DB fixes a 400× and a 20× wall together. Flip it: precompute
   latest-value-per-entity → 100M users × 200 × 8 B = **160 GB** in a KV cluster, serving = one batched
   ~2 ms lookup. (L15 precompute-the-read, aimed at model inputs.)
2. **Model — one definition, two stores, two correctness rules.**
   - **Two stores, opposite jobs:** serving wants one entity/latest/2 ms and never cares about history →
     **online store** (KV cache, L02/48). Training wants the full time-series of every value over years →
     **offline store** (columnar warehouse, L29). "Fast latest lookup" and "complete history scan" can't
     be one engine (L29's serving-vs-analytics-shapes).
   - **One feature definition:** both online-latest and offline-history produced by the SAME transformation
     code. The instant training's SQL and serving's stream aggregation differ (window edges, rounding, a
     one-sided bug) they diverge → the anti-drift mechanism.
   - **Two rules = the whole difficulty:** (a) **train-serve consistency** — serve the number you trained
     on; break it = **train-serve skew** (small silent shift across 200 inputs, no error, no alert). (b)
     **point-in-time correctness** — build each training row from values AS OF that past instant, never now;
     break it = **leakage** (join old labels to CURRENT features → future info, incl. this txn's own later
     chargeback, leaks in). This is WHY offline keeps history not just latest — a latest-only store literally
     can't answer "what was this on day 60," so it can't train honestly. A feature store = cache + shared
     definition + time-versioned history; the last two additions are the entire reason it exists (vs L48 cache).
3. **Trace** — (A) serving read: entity keys already on the request → ONE batched online-store read → ~200
   latest values in ~2 ms → model → approve at ~25 ms; store never computes/scans on the hot path. (B)
   freshness PER FEATURE: `user_avg_amount_30d` moves ~1%/txn over ~100 txns → 6 h batch staleness harmless →
   cheap BATCH; `card_txn_count_1m` goes 0→50 in a 60 s card-testing attack (the fraud signal) → a batch
   finishes the attack before updating → STREAMING off the txn event log (L09/33), ~1-2 s (L29 kappa). (C)
   training build: WRONG = join to current values → merchant_fraud_rate_7d "now" includes post-t fraud →
   offline AUC 0.99, production ≈ old rules (future doesn't exist at decision time); RIGHT = point-in-time
   ("as-of") join over offline history, fetch each feature as of just-before-t → no leak. Paths B & C are one
   thing: offline records the history of everything the pipeline wrote; the training join reads it back.
4. **First bottleneck — the served number and the trained number must be identical.**
   (1) Train-serve skew SURVIVES a shared definition because the two STORES drift: streaming writes online at
   12:00:03, batch materializes offline at a slightly different window edge → agree almost always, disagree
   exactly at the fast-moving moments that matter most. Sharing the definition removed the CODE divergence,
   not the STORE divergence. (2) Honest fix = **feature logging**: at serving time write down the exact
   feature vector the model got; weeks later join the delayed label to the LOGGED features → skew impossible
   (one number, recorded once — L13 decide-once-record-it) and point-in-time correctness for free (the logged
   vector IS the state at the request instant). Secondary walls: online store = hot dependency at 10k/s (shard
   L04, replicate, hot-entity coalesce L03/48); freshness lag never zero (even streaming has ~1-2 s, L06/26).
   Deepest point: a model in production is only as good as the promise that the numbers scoring it are the
   numbers it learned from — a systems problem, not a modeling one.

## Four traps
1. Computing features at request time — 400× throughput + 20× latency at once; precompute, fetch-not-compute
   on the request path (L15).
2. Training on current feature values — leakage; the model reads the answer (0.99 offline) and flatlines live;
   keep time-versioned history + point-in-time join, or train on logged served values.
3. Two code paths for one feature — train-serve skew, silent; one shared definition + log & train on served
   values to close the store-to-store gap.
4. One freshness for everything — streaming a 30-day avg is overkill; batching a 1-minute count blinds the
   model to a live attack; choose freshness per feature (L29 lambda/kappa, L39 tier-by-need).

## Reuse / callbacks
L15 (precompute-the-read = the online store), L29 (serving vs analytics opposite shapes + lambda/kappa = two
stores + batch/streaming), L02/48 (cache shape + hot-key care = online store), L39 (tier-by-need = freshness
per feature), L06/26 (replication/propagation lag = freshness never zero), L13 (decide-once-record-it =
feature logging kills skew at the root), L03/04 (shard/hot-key on the online store), L09/33 (event log the
streaming features consume). Contrast with L48: a plain cache solves latency only; a feature store adds the
shared definition + time-versioned history, which are the whole point.

## What it sets up next
Topic #54 — **Billing & metering systems**: turn a usage firehose into invoices correct to the cent — no event
double-counted or dropped. Idempotent event ingestion (L13, a retried meter event mustn't bill twice),
aggregation into usage counters (L21 sketches vs exact), pricing/rating rules, and reconciliation against the
ledger (L25) to catch drift before overcharging. Trade: billing accuracy vs metering cost & latency. (Also
still queued: edge computing #55, data-privacy deletion & compliance #56 — spine stays full, no fresh topics
added this round.)
