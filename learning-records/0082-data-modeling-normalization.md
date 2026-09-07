# Lesson 0082 — Data Modeling: Normalization vs Denormalization: One Order-and-Customer Schema, Laid Out Both Ways

**File:** `lessons/0082-data-modeling-normalization.html`
**Curriculum spine:** topic #82 (advanced batch) — now ✅
**Date:** 2026-09-07

## The one worked system
The orders platform from L79: **10 billion orders, 100 million customers**, each
order holding a few **line items** (product, qty, price). The hot path = the
**order-history page** ("my 50 most recent orders, each item's product name,
category, price"), read ~**500,000/s**; orders placed at ~**40,000 writes/s** (L79).
The one decision under the microscope: the *shape* of the table — split the facts
into many tables (normalized) or copy them together (denormalized).

## What it covered (the four moves)
- **Estimate** — the read: normalized order-history is a **4-way join**, dominated
  by 150 nested-loop product seeks @ 0.4 ms (L12) ≈ **61 ms**; denormalized
  pre-joined = one index range scan ≈ **0.5 ms** → **~120×** faster (the join
  precomputed at write time, L15). The **read:write ratio** is the deciding bet:
  ~12:1 reads:order-writes, but ~**50,000,000:1** reads : product-renames → paying
  the rare write to cheapen the constant read is decisive. Storage: a name stored
  once = 2 GB vs copied per order = 200 GB (**100×**) — but bytes are cheap; the
  real bill is that copies can **drift**.
- **Model** — normalization = **functional dependencies** obeyed: `customer_id →
  email`, `product_id → name,price`; **3NF** = every non-key column depends on "the
  key, the whole key, and nothing but the key," i.e. each fact lives in the table
  whose key it depends on, pointed to by a **foreign key**. Copying a fact into many
  rows creates the **update anomaly** (rename = 1 row normalized vs ~2,000,000
  denormalized, + an inconsistency window + partial-failure corruption; cousins =
  insert/delete anomaly, "a fact with no home"). Same choice in doc stores = **embed
  vs reference** (embed bounded/owned/read-together; reference shared/unbounded/
  independently-updated) + the unbounded-array/doc-size wall.
- **Trace** — (A) read: denorm ~0.5 ms vs norm ~61 ms, paid 500k/s → denorm wins
  120×. (B) place order: 4 small inserts (norm) vs 3 copy-carrying inserts (denorm)
  — a wash, both touch few rows; denorm write pain shows up on CHANGE, not INSERT.
  (C) rename product 7 (in 2M rows): 1 atomic row (norm) vs ~2,000,000-row rewrite
  ~200 s with a two-names-at-once window and crash-corruption risk (denorm) → norm
  wins 2,000,000×. The two layouts are **photographic negatives** — each cheap
  exactly where the other is dear.
- **First bottleneck** — the **update-anomaly tax**, dissolved by refusing the
  global choice: keep a **NORMALIZED source of truth** (writes/renames = 1 row) and
  **DERIVE** a denormalized read model kept in sync **async** (materialized view /
  **CQRS** L37 / **CDC-outbox** L33). The copy is derived + rebuildable → drift
  becomes **lag** (L06 eventual consistency), not corruption. Walls:
  over-normalization = a join tax (8-way join, L12 selectivity trap); join-less
  stores (Cassandra/Dynamo) model the table around the QUERY and denormalize up
  front, owning consistency (L13); when the read:write bet turns → migrate (L24/79).

## Key idea to carry
A **normalized fact is the truth; a denormalized fact is a cache of it.** So
denormalization is safe exactly when the copy is *derived and rebuildable*, and
dangerous exactly when the copy is the *only* copy. "Normalize vs denormalize"
reframes to: *where does the truth live, and what is merely a fast copy?* Store
truth normalized (one home per fact), serve reads from derived denormalized copies,
keep the copies honest with L15/L37/L33 machinery under eventual consistency.

## Reuses
L12 (B-tree seek a join pays per row; selectivity trap of over-normalization), L15
(precompute-the-read = the read model), L18 (cursor index on the history page), L37
(CQRS — write model vs read model), L33 (CDC/outbox keeps the derived copy in sync),
L06 (eventual consistency = what the copy's lag is), L13 (idempotent fan-out when
denormalizing at write time), L24/79 (schema migration when the bet turns).

## Trade named
Write correctness & storage vs read latency.

## Sets up next
**Lesson 83 — Full-text & relevance pipelines end-to-end:** beyond L12/16 —
analyzers/tokenizers, index-build vs query-time split, near-real-time indexing
(L16/29 freshness), typo tolerance & synonyms, faceted search over shards (L79).
Trade: index richness & freshness vs build cost & query latency.

## Note on the spine
Spine still has topics #83–85 queued (full-text/relevance pipelines, graph/
recommendation traversal, multi-region data placement & residency). Three remain —
not yet at the end, so no new topics added this run. Add a fresh batch when the
queue is down to the last one or two.
