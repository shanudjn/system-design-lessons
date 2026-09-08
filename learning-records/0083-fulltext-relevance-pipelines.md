# Lesson 0083 — Full-Text & Relevance Pipelines End to End: One Query and One Document, Traced Through the Whole Machine

**File:** `lessons/0083-fulltext-relevance-pipelines.html`
**Curriculum spine:** topic #83 (advanced batch) — now ✅
**Date:** 2026-09-08

## The one worked system
The product catalog from L12/L16: **10 million products**, searched at **5,000
queries/second**, results expected in **<100 ms**. Two concrete things are traced
end to end: **one document** being indexed (`Men's Running Shoes — Blue Runner
(Waterproof)`, category *Shoes*) and **one query** being run (`runing shose` — two
typos). The lesson assembles the machine L12 (inverted index) and L16 (BM25 ranking)
left in pieces: everything between raw text in a DB and a ranked, faceted page.

## What it covered (the four moves)
- **Estimate** — index size: 10M docs × ~50 distinct terms = 500M postings × ~8 B =
  4 GB raw → ~1–1.5 GB delta-encoded/compressed (L80/29), RAM-resident on a few
  nodes; vocabulary ~1–2M terms. The **build-vs-query asymmetry**: indexing a doc is
  ~0.2 ms CPU → full 10M rebuild ~2,000 s single-core ≈ 1 min on 32 cores (heavy,
  once — L15 precompute), while a query analyzes 2 words, walks 2 postings lists,
  scores survivors, keeps top-10 in <20 ms (light, per request). Naive typo match =
  1.5M dictionary comparisons/term → ~15B/s at load → impossible without a
  **Levenshtein automaton / FST** walk over the term dictionary.
- **Model** — the **ANALYZER** stage by stage: char filter (strip punctuation/
  accents) → tokenizer (split) → lowercase → possessive/stopword → **stemming**
  (running→run, shoes→shoe) → synonyms → final terms `men, run, shoe, blue, runner,
  waterproof`. The **IRON SYMMETRY RULE**: index-time and query-time analysis must be
  identical, or a perfect match silently returns ZERO (the #1 "search is broken and I
  can't tell why" bug — the doc is stored under `run`, an un-stemmed query looks up
  `running`, miss). Recall-broad-then-rank funnel (L16): retrieve ~600k OR-candidates
  → BM25 score → top-K heap → optional expensive re-rank on ~1,000. Typos & synonyms
  = deliberate **recall-widening term expansions**; synonym placement (index-time =
  fast query/fat index/reindex-to-change vs query-time = lean/flexible/per-query cost)
  is the L15/82 precompute-vs-flex trade.
- **Trace** — (A) index a doc: catalog write (source of truth, L82) → CDC/outbox
  (L33) → analyzer → append to a NEW segment → visible at next refresh (~1 s,
  near-real-time). (B) clean query "running shoes" → [run, shoe] → postings → BM25 →
  facets → ~15 ms. (C) typo "runing shose" → analyze → [runing, shose] → Levenshtein
  repair → [run, shoe] → **converges onto Path B** (a repaired query == a clean one;
  the repaired words still pass the SAME stemmer).
- **First bottleneck** — you **can't update an inverted index in place**: postings are
  sorted + delta-compressed, so splicing one doc_id = re-encode/rewrite the packed
  list, ×~50 terms/doc, fighting concurrent reads = O(rewrite). Fix = the **LSM shape**
  (L47): new/changed docs → small IMMUTABLE **segments**; **REFRESH** exposes them
  (the freshness event; interval = freshness-vs-query-speed dial); **SEARCH** queries
  all segments and merges (L21/79); deletes = tombstone bitset filtered at read
  (L15/20); background **MERGE** combines small→big, dropping tombstones (bounds fan-out
  & file count).

## Walls beyond the first
- **Sharded facets** (L79 scatter-gather): counts merge by ADDITION (L21 mergeable),
  but top-N facet *rankings* hit the **top-K-from-shards / deep-pagination trap** (L18)
  — a category #21 on every shard could be #1 globally but truncated before the merge.
  Fix: over-fetch per shard or a second round (accuracy vs query cost).
- **Analyzer change = full reindex**: old docs carry old terms (`running`), new docs
  new terms (`run`) → symmetry breaks for half the corpus → rebuild everything under
  the new analyzer and **atomically swap** (L31 blue-green, L24 migration).
- **The index is stale by construction**: a derived read model (L82) fed by CDC/outbox
  (L33); pipeline lag = search staleness (L06). Freshness is bought, never free.

## Key idea to carry
**Text isn't searchable — TERMS are, and the analyzer is the machine that turns one
into the other, the same way on both sides.** The search index is a disposable, lossy,
pre-shaped copy of the text (stems can't rebuild the prose), so everything hard about
search is the cost of keeping that copy rich, fresh, and aligned with the terms a query
will actually ask for. Reframed: index = derived read model (L82) built by precompute
(L15), fed by a change pipeline (L33), searched by scatter-gather (L79), rebuilt by
migrate-and-swap (L24/31).

## Reuses
L12 (inverted index/postings this lesson feeds), L16 (retrieve-broad-then-rank,
IDF/BM25, two-phase funnel), L15 (precompute-the-read = heavy-index/light-query), L47
(LSM segments+merge), L79 (sharded index + scatter-gather), L21 (mergeable facet counts,
accuracy-vs-cost), L18 (top-K-from-shards / deep-pagination trap facets inherit), L33
(CDC/outbox syncs the derived index), L82 (index as derived, rebuildable read model —
not the source of truth), L24/31 (reindex-and-swap on analyzer change), L06 (lag =
staleness), L80/29 (posting compression).

## Trade named
Index richness & freshness vs build cost & query latency.

## Sets up next
**Lesson 84 — Graph & recommendation traversal at scale:** "people you may know" over
a billion-edge graph (L41) — BFS fan-out explosion, precompute vs traverse-at-read
(L15), edge sharding & the supernode/hot-key problem (L79), offline vs online candidate
generation (L53). Trade: traversal freshness vs precompute cost.

## Note on the spine
Before this run the queue held #83–85 (three). Completing #83 left only #84 and #85 —
the "last one or two" threshold flagged in the L0082 record — so this run ADDED a fresh
batch of 5 advanced topics (#86 recommendation/candidate-ranking, #87 A/B testing &
experimentation, #88 notification systems end-to-end, #89 data quality & pipeline
observability, #90 compaction/GC/space reclamation). Queue now holds #84–90 (seven).
Add the next batch when it's down to the last one or two again.
