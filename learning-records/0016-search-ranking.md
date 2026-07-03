# Lesson 0016 — Search Ranking: which 10 of 40,000 matches come first

**Published:** 2026-07-03
**File:** `lessons/0016-search-ranking.html`
**Spine topic:** #16 Search systems — query parsing, ranking (TF-IDF/BM25), relevance vs recall.

> Note: the remote had already advanced to Lesson 0015 (Idempotency/CDN/Feed
> fan-out landed as 0013–0015) by the time this run pushed, so this run authored
> the genuine next lesson, **0016**, rather than a duplicate 0013.

## What it covers
Directly continues Lesson 12: the inverted index answered *which* documents
match; this lesson answers *which win* — ranking.

Four-move spine:
- **Estimate** — "red running shoes" OR-matches ~600,000 of 10M products; the
  shopper sees ~10, so 99.998% is never seen and the ORDER is the entire
  product. Names the master trade-off **recall vs precision**: AND = precise
  but misses near-matches, OR = broad but noisy → retrieve broadly, rank to
  restore precision at the top.
- **Model** — build the score. **IDF** = log(N/DF): idf("shoe")≈2.40 beats
  idf("red")≈1.30, idf("the")≈0.05 (stopwords fall out for free). **TF-IDF** =
  Σ tf×idf, but unbounded TF lets a keyword-stuffed title score 12.48 and crush
  a perfect "Red Running Shoes" at 5.62. **BM25** patches it: saturating TF
  tf·(k+1)/(tf+k), k=1.2 (tf 1→4 lifts factor only 1.00→1.69, ceiling 2.2) +
  length normalization.
- **Trace the paths** — query parsing must mirror index-time analysis exactly
  (stem/lowercase/stopwords) or matches vanish; retrieve→score→**top-K heap**
  (N log 10, not a full N log N sort); pure text score can't break quality ties
  → blend non-text signals (popularity, recency, availability).
- **Next bottleneck** — an ML ranker at ~1 ms/doc × 600k = 10 min, impossible
  vs a 200 ms budget → **two-phase**: cheap BM25 narrows to ~1,000, expensive
  re-rank on those (the L12 covering-index / L14 edge-origin "narrow cheap,
  perfect expensive" funnel). Plus **near-real-time indexing**: freshness vs
  indexing cost (L06 replication lag / L14 TTL wearing a search hat).

## Reuses
- L12 inverted index (retrieval) — this lesson is the ranking half.
- L06 replication lag + L14 TTL → reappear as index-freshness lag.
- The "narrow cheap, then perfect expensively" funnel from L12/L14.

## Sets up next
Lesson 0017 — **Distributed tracing & observability**: a search request now
fans across parser → retrieval → re-ranker → feature store; when it's slow,
which hop ate the 200 ms? Trace IDs, spans, sampling, the p99-vs-mean gap.
Also seeds #18 API design & pagination (paging a ranked, shifting result set).
