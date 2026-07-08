# Learning record — Lesson 0021: Analytics at Scale (approximate counting)

**File:** `lessons/0021-approximate-counting.html`
**Curriculum spine:** topic 21 (Analytics & counting at scale) — now ✅
**Builds on:** L20's teaser (count uniques across 25T objects), L06 (approximate/sharded counter), L08/L09 (heavy-key detection), L12/L20 (on-disk lookup a Bloom guards), L19 (approximate to stay scalable).

## What this lesson covered
Three live dashboard questions over a billion-car / 25-trillion-object upload stream, each answered in kilobytes instead of gigabytes:

- **The estimate that kills exact:** memory of the exact answer scales with the number of *distinct* items — distinct set ≈ 8 GB (1B × 8 B, ideal), per-car counters ≈ 16 GB, seen-keys set ≈ 400 TB. Unbounded, per window, per region.
- **HyperLogLog (cardinality):** leading-zeros coin-flip intuition (a 1-in-2ᵏ hash ⇒ ~2ᵏ distinct); m = 2¹⁴ = 16,384 registers × 6 bits = 12,288 B ≈ **12 KB**, standard error 1.04/√16,384 = **0.81%**; ~**650,000×** smaller than the 8 GB set.
- **Bloom filter (membership):** bit array + k hashes; "any bit 0 ⇒ definitely not added", "all 1 ⇒ probably added". **No false negatives.** m/n = −ln ε/(ln 2)² ≈ **9.6 bits/key** and k ≈ **7** for ε = 1%; ~**1.2 GB** for 1B keys (~40× vs a real set). "Definitely new" lets ingest skip the metadata index lookup; false positive = one wasted lookup.
- **Count–Min sketch (frequency):** d×w grid, increment one cell per row, read the **min** ⇒ only ever **over**-estimates; w = ⌈e/ε⌉ = 2,719, d = ⌈ln(1/δ)⌉ = 7 ⇒ ~19,000 cells ≈ **76 KB**; error bounded by ε·N ⇒ accurate for heavy hitters, noisy in the tail.
- **Mergeability:** HLL = max per register, CMS = add per cell, Bloom = bitwise OR ⇒ local compute, central union, no re-scan (map/reduce shape).
- **Trace:** one event `(car=42, key=abc123)` folded into all three fixed arrays without ever storing the item; O(k)/O(d) writes, constant-size reads.

## The first bottleneck (where it lands)
A sketch tells you **how many / how often**, never **which**. Fixes: CMS + a K-entry **min-heap** for the top-K heavy hitters; **counting Bloom** (4-bit counters, 4× memory) for deletes; one sketch **per time window** for "today" (merge adjacent windows). Deeper wall: the error budget (m, w, d, k) is fixed at build time and can't be sharpened afterward.

## Trade-offs named
Exactness vs memory · a percent of accuracy for constant memory (HLL) · one-sided error pointed at the cheap side (Bloom) · accurate-for-heavy vs noisy-in-tail (CMS) · write/read-cheap but append-only · removability vs 4× memory (counting Bloom).

## What it sets up next
**Lesson 22 — Distributed locks & leases:** two workers both think they hold the billing lock. Lease + fencing token (recap L10's epoch) vs naive lock, the GC-pause / clock-skew hazard, why a distributed lock is a last resort. The trade flips from accuracy-vs-memory to **safety vs liveness**.
