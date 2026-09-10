# Lesson 0085 — Multi-Region Data Placement & Residency

**File:** `lessons/0085-data-placement-residency.html`
**Curriculum spine:** topic 85 (advanced batch added after L83).
**Worked example:** a global marketplace (300M users: EU 120M / US 130M / India 50M). One row = Alice, a German resident whose personal data may live only in the EU. One query = "revenue by product, worldwide, last month."

## What it covered
- **The new constraint:** placement stops being a pure performance choice. A privacy law can pin a row to a region, so location becomes a property of the row, not a free optimization.
- **Estimate — two forces in different units:**
  - Speed-of-light tax: Frankfurt↔Virginia ~6,500 km → ~32.5 ms one-way, ~65 ms RTT, ~90 ms with overhead (matches L23). ~90× a local ~1 ms read.
  - Compliance wall: up to 4% of global turnover (€20M on a €500M/yr company). A latency cost is payable; a violation is categorical. The wall reorders the design.
  - Data sizes: 600 GB of PII (pinned, split across 3 homes); a ~2.7 GB directory (id→region, no PII, replicated everywhere).
- **The core inversion:** stop moving data to the computation; move the computation to the data and let only tokens/aggregates cross the border.
- **Model:** `home_region` as a *legal* placement key (L79, chosen by law not hash); a globally-replicated fail-static **directory** (L34) to resolve a row's region without moving it; split data by **sensitivity** (PII home; catalog/tokens/aggregates travel) with **tokenization** (L30) as the bridge.
- **Trace three paths:** Alice home (~1 ms, compliant); Alice roaming to NYC (request follows her, data stays → ~90 ms tax, never a US copy); global join (naive = ~6 GB illegal cross-border move; legal = per-region local join → ship ~6 MB of (product, revenue) aggregates → merge, L21/L29 → ~1000× smaller AND compliant).
- **First bottleneck + walls:** the cross-region query over immovable rows → push compute down, merge anonymous results. Cross-region interaction → store per-endpoint linked by token. Permanent relocation → expand→cutover(flip directory)→contract, delete last (L24/L56). Directory → AP fail-static (L11/L34).
- **Four traps:** residency-as-latency-cost; pool-everything-into-one-warehouse; a CP/centralized directory on the hot path; copy-without-deleting-old on relocation.
- **Interactive quiz (4 Q):** caching EU PII in the US as a wall-not-cost; the illegal global join vs compute-then-merge; the roaming user (request follows, data doesn't); how to build the directory (AP fail-static, not CP-central, not stored-in-the-row).

## Reuses / threads pulled through
L23 (multi-region, ~90 ms cross-ocean, follow-the-sun), L79 (partition key), L56 (GDPR/deletion, irreversible delete), L14 (speed-of-light floor), L34 (globally-replicated fail-static directory), L21/L29 (scatter-gather, mergeable aggregates, compute-then-merge), L30 (tokenization/pseudonymization), L24 (expand→cutover→contract), L11 (AP vs CP).

## What it sets up next
**Lesson 86 — Recommendation & candidate-ranking systems:** "products you may like" as a pipeline — offline candidate generation (collaborative filtering / embeddings, L65/L53) → cheap retrieval → expensive re-rank (L16 two-phase funnel) → business filters (in-stock, dedup, diversity); cold-start, feedback loops, online vs batch features (L53). Trade: recommendation quality & freshness vs compute cost.

Spine still has topics 86–90 queued, so no new topics were added this run.
