# Lesson 0073 — Deduplication & Entity Resolution: Are These Two Records the Same Person?

**File:** `lessons/0073-deduplication-entity-resolution.html`
**Spine topic:** #73 (dedup / entity resolution / record linkage at scale)
**Date:** 2026-08-29

## Worked example
**500M** customer records in one table, no shared id, grown by acquisitions + sloppy sign-up forms → the same human appears many times with small differences. Three followed rows: R1 `Jonathan Q Smith / jsmith@gmail.com / 555-0142 / 12 Oak Street`, R2 `Jon Smith / jsmith@gmail.com / (blank) / 12 Oak St`, R3 `J. Smith / jon.s@work.com / 555-0142 / 88 Pine Ave`. Goal: find every group that is one real person and collapse each into one clean **golden record**. No join key exists — we must *decide* sameness from fuzzy dirty fields.

## What it covered
- **Estimate — N² is a wall, blocking is the way through.**
  - All-pairs = N(N−1)/2 ≈ N²/2 = **1.25×10¹⁷** comparisons. At ~1 µs/comparison: **~3,960 years** on one core; **~4 years** even on 1,000 cores (10⁹/s) — impossible, and doubling N quadruples the work.
  - **Blocking:** compare only records sharing a cheap key (e.g. zip + last-name-prefix). Blocks avg **b=100** → comparisons = N·b/2 = **2.5×10¹⁰**, reduction = N/b = **5,000,000×** → ~**6.9 h** on one core, ~**25 s** on the cluster. The cut and the missed matches are the *same knob* (recall vs compute).
- **Model — four machines.**
  - **Blocking keys** — cheap deterministic fn equal records tend to share (zip+name-prefix, Soundex(last)+birth_year survives spelling, email domain, phone prefix); sort/hash by key, score only within a block (L3 shard-by-key).
  - **Weighted similarity score** — edit distance (Levenshtein) for short strings; **Jaccard** = |A∩B|/|A∪B| on token sets (worked: A={jon,smith,12,oak,street}, B={jonathan,smith,12,oak,st} → 3/7 = **0.43**); blend fields weighted by *discrimination* — rare shared value (exact email/phone) = strong evidence, common value (city, "John") = weak (L16 IDF instinct).
  - **MinHash + LSH** — MinHash: k-slot signature where P(two slots agree)=Jaccard, so k=100 estimates Jaccard to ±1/√k=**10%** (L21 sketch). LSH: split 100 slots into **b=20 bands × r=5 rows**, candidates if any band matches fully → P(candidate)=1−(1−sʳ)ᵇ, S-curve threshold ≈ (1/b)^(1/r) ≈ **0.55**; s=0.80→99.96% caught, s=0.55→0.64, s=0.30→**4.7%**. Blocks by *similarity* → typo can't hide a dupe. Tune b,r to move the knee (recall vs compute).
  - **Cluster + survivorship** — two thresholds → auto-match (≥0.85) / **review band** / no-match (≤0.45); matched pairs = graph edges → **connected components / union-find** = one cluster per person; **survivorship** picks winning field values (newest, most complete, most trusted source); keep source ids as **provenance**.
- **Trace.** (A) R1/R2: same block, exact email (strong) + close name + abbreviated same address → ~0.90 → auto-match → golden record. (B) R1/R3: shared RARE phone but different email AND address → ~0.22 yet genuinely ambiguous (spouse/roommate sharing a landline?) → **review band**, don't guess. (C) LSH: "Springfield" vs "Sprin*field" typo → different exact blocks (missed) but Jaccard~0.80 → 99.96% collide → candidate → matched.
- **First bottleneck — the two errors are ASYMMETRIC.**
  1. **False merge** (fuse two different people) = corruption, near-irreversible (A sees B's data → privacy incident L30/56), distinction destroyed. **False split** (one person left as two rows) = cheap reversible nuisance. → bias AWAY from merging: high auto-match threshold, wide review band, reversible merges w/ provenance. Precision > recall here.
  2. **Blocking recall** — single key silently caps recall (a dupe dirty in the key field is never compared, an invisible miss) → **multi-pass** blocking on different keys + LSH, **union** candidates (must be dirty in every key to escape).
  3. **Non-transitivity** — similarity isn't transitive (R1≈R2≈R3 ⇏ R1≈R3; Smith→Smyth→Smithe), so naïve connected-components **chains** near-matches into a giant blob (industrial-scale false merge) → dense clusters (near-cliques), size caps + quarantine, **cannot-link** (different verified IDs never merge).
  4. **Never done** — new records forever → **incremental** resolution (compute blocking keys, fetch only candidates in those blocks, attach to cluster — L19 "query the neighbourhood") + periodic full re-batch for drift; golden record = a *derived* view over immutable sources (L25/38), so merges stay reversible.
- **Deepest point.** Entity resolution is a **decision under uncertainty**, not a computation — no ground-truth join key exists; you *infer* an identity the data never stored from always-conflicting evidence. The engineering is an **error budget deliberately spent on the cheaper, undoable mistake** (leave a dupe) rather than the corrupting one (merge two people). Every mechanism pushes unavoidable errors to the survivable side of that line.

## Numbers used (all checked, `python3`)
- Pairwise: 500,000,000·(N−1)/2 ≈ **1.25×10¹⁷**. 1-core @1µs = 1.25×10¹¹ s ≈ 1,446,759 days ≈ **3,963 yr**. 1,000-core @10⁹/s = 1.25×10⁸ s ≈ 1,447 days ≈ **3.96 yr**.
- Blocking b=100: N·b/2 = **2.5×10¹⁰**; reduction N/b = **5,000,000×**; 1-core = 2.5×10¹⁰ µs ≈ **6.94 h**; 1,000-core = **25 s**.
- Jaccard: |{smith,12,oak}|/|{jon,jonathan,smith,12,oak,street,st}| = 3/7 = **0.4286**.
- LSH k=100, b=20, r=5: threshold (1/20)^(1/5) = **0.549**; P(candidate) = 1−(1−sʳ)ᵇ → s=0.80:**0.9996**, s=0.55:0.644, s=0.43:0.256, s=0.30:**0.0475**. MinHash error 1/√100 = **0.10**.

## Threads reused
L3/4 (sharding/consistent hashing — blocking key = shard of the comparison work), L21 (sketches — MinHash is a Jaccard sketch, small chosen error for constant memory), L16 (search ranking — IDF's "rare tokens carry more info" → field weights), L19 (geo/proximity — "query the neighbourhood, not the world" → incremental resolution), L65 (vector search — LSH / approximate-nearest-neighbour blocking), L25/38 (ledger/event sourcing — immutable sources, golden record as derived view → reversible merges), L30/56 (secrets/privacy — a false merge is a privacy incident).

## Sets up next
- **Lesson 74 — Config & feature-flag delivery at scale:** push a flag/config change to **40k servers** (L27) in seconds without a deploy — versioned config plane, pull-with-long-poll vs push, staged rollout + instant kill-switch (L31/51), making "everyone on the same flag" true across an always-partly-stale fleet. Trade: change speed & safety vs a new always-on dependency.
- Then Lesson 75 — append-only audit logs & tamper evidence (hash-chaining, Merkle proofs).
- Spine still has 74–75 queued, so no new topics added this run.
