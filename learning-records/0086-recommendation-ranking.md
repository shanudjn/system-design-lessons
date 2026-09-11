# Lesson 0086 — Recommendation & Candidate-Ranking Systems

**File:** `lessons/0086-recommendation-ranking.html`
**Curriculum spine:** topic 86 (advanced batch added after L83).
**Worked example:** an online marketplace with a 10M-product catalog and 5M DAU. One user = Alice (a warm user: a month of tent-browsing, bought hiking boots, clicked a tent 3s ago). One request = render her "Products you may like" strip — turn 10M products into a ranked 10 in under ~100 ms.

## What it covered
- **The core split:** you can't run the good model on the whole catalog, so recommendation is a **funnel** — cheap-broad **candidate generation** (recall) then expensive-narrow **re-rank** (precision), L16's two-phase funnel applied to recommendations.
- **Estimate — the compute wall:**
  - Full scan: 10M × ~0.1 ms/score = 1,000 s ≈ **16.7 min per request**, ~10,000× over a ~100 ms budget. Faster servers don't close a 10,000× gap; the fix is fewer items to score.
  - Latency budget split → candidate count ~500 is *derived*: the most the re-ranker can score in its ~50 ms slice (500 × 0.1 ms). Funnel ratio 10M → 500 → 10.
  - Sizes: item-item table 10M × 100 × 12 B = ~12 GB; embeddings 10M × 128 × 4 B = ~5.12 GB; brute-force NN ~2.56B flops/query vs ANN ~1 ms (~1000×). Traffic ~174 req/s avg, ~520 peak → ~26 cores just for re-rank.
- **Model — the pipeline:**
  - Candidate generation: collaborative filtering (item-item, precomputed) + embedding ANN (L65) + heuristics (trending/recently-viewed/category), unioned + deduped for recall.
  - Re-rank: heavy model over ~500, features from a feature store (L53) — batch (long-term affinity, day-stale) + online (in-session intent, fresh).
  - Business filters: in-stock, dedup (don't re-rec what she bought), diversity, business rules.
- **Trace three paths:** (A) nightly offline batch build → publish versioned read-only artifacts (L24/L31 atomic swap); (B) Alice's warm request ~70 ms, offline base + online patch (the 3s-old click lifts related gear); (C) cold-start Dana (no history) → popularity + content embeddings + light exploration; item cold-start solved by content embeddings too.
- **First bottleneck + walls:** re-rank cost = candidates × cost-per-score → elastic multi-stage funnel (add a cheap pre-rank 500→100). Feedback loop / popularity bias (model trains on its own exposure) → exploration (bandit) + diversity + label debiasing. Training-serving skew → feature store parity (one definition, both sides; same symmetry rule as L83's analyzer). Candidate freshness → online patch over offline base (L84 lambda).
- **Four traps:** faster-model-fixes-full-scan (no — it's 10M × cost); trusting click logs as ground truth (exposure confounds); computing features differently offline/online; batch-only pipeline (misses seconds-fresh signal & new items).
- **Interactive quiz (4 Q):** why faster servers don't rescue the full scan; how a 3s-old click reaches the ranking (online feature patched onto batch base); cold-start fallback; the feedback loop / popularity bias and its fixes.

## Reuses / threads pulled through
L16 (two-phase recall→precision funnel), L65 (embeddings + ANN sublinear retrieval), L53 (feature store, batch vs online features), L84 (offline-base + online-patch lambda, from PYMK), L15 (precompute off the request path), L64/L29 (nightly batch, aggregate-then-merge), L83 (symmetry rule — train & serve must feature-ize identically), L24/L31 (versioned atomic swap of published artifacts).

## What it sets up next
**Lesson 87 — A/B testing & experimentation platforms:** we built a recommender and claimed it's "better" — how do you actually know? Deterministic bucketing by hash (L03/L04), the exposure-logging pipeline (L64 stream), sample-ratio-mismatch & peeking traps, guardrail metrics, overlapping experiments via layered assignment. Trade: statistical rigor & velocity vs blast radius.

Spine still has topics 87–90 queued, so no new topics were added this run.
