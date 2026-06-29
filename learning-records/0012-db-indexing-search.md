# Learning record — Lesson 0012: Indexing & Search

**Date:** 2026-06-29
**File:** `lessons/0012-db-indexing-search.html`
**Title:** Indexing & Search: How a Database Finds One Row in 100 Million

## What this lesson covered
Took ONE concrete table — `orders`, 100M rows × 200 bytes — and asked it three
questions (point lookup, customer's orders, full-text "blue running shoes").
Followed the four-move spine:

- **Estimate** — no-index baseline: 100M × 200 B = **20 GB**, scanned at
  500 MB/s = **40 s** to return 1 row (we touched 100M for nothing). With a
  B-tree: ~4 page reads × 0.1 ms = **~0.4 ms** → a **~100,000×** speed-up.
  First trade-off: an index is read speed bought with write cost + storage.
- **Model** — built the B-tree from arithmetic: page = 8 KB, entry ≈ 16 B →
  fan-out **~500**; `500³ = 125M > 100M` ⇒ **3 levels**; `500⁴ = 62.5B` ⇒ only
  4 levels for a billion rows. Depth = `log₅₀₀(rows)`, grows very slowly.
  Balanced (page splits) + sorted/linked leaves (ranges + ORDER BY). Index-type
  table: **B-tree** (=, ranges, sort — the default) vs **hash** (= only, kills
  order — L03's hash-vs-range trade *inside* a table) vs **inverted** (text).
- **Trace the paths** — (A) PK point lookup ~0.4 ms; (B) **secondary index =
  two hops** (bookmark lookup: leaf holds a pointer, not the row) → fixed by a
  **covering index** (`INDEX(customer_id, amount)` → index-only; storage for
  latency, L02 shape); (C) **range query** where ordered leaves shine and hash
  can't; (D) full-text — B-tree matches a prefix but not `%shoes%`, so build an
  **inverted index**: tokenize → normalize/stem → postings lists → **intersect
  (AND)** the sorted lists; turns a 10M-row scan into a small merge.
- **Next bottleneck (reversed)** — too many / wrong indexes hurt. **Write
  amplification**: PK + 5 secondaries ⇒ one INSERT = **6 physical writes** (~6×).
  **Selectivity trap**: break-even is `matched × 0.1 ms < 40 s` ⇒ **~400k rows =
  0.4%** of the table; `status='shipped'` matching 50M ⇒ 50M random reads ≈
  **5,000 s, 125× slower** than a scan, so the planner ignores the index — pure
  write tax. Rule: index high-cardinality columns your queries filter/sort on.

## Trade-offs named
- Index = cheaper reads **vs** more expensive writes + storage (the master trade)
- B-tree **vs** hash: order (ranges/sort) **vs** one-read equality only
- Covering index: bigger index **vs** skipping the heap second hop
- Inverted index: rich/fast text search **vs** build + update cost + staleness
  (lives in a separate system trailing the DB)
- Write amplification: each index speeds its reads **vs** taxes every write
- Selectivity: an index pays off only in proportion to how much it skips

## Callbacks to earlier lessons
L02 cache "storage for latency" (covering index, cached top B-tree levels);
L03 hash-vs-range ordering trade (now inside one table) + the 40k writes/sec
write wall (write amplification); L05/L07 at-least-once retries replaying writes
(the idempotency cliffhanger).

## What it sets up next
- **Idempotency:** the search index trailing the DB and L05/L07 retries both
  replay writes — next is making a replayed/duplicated write a safe no-op
  (dedup keys, exactly-once *effects* not delivery).
- **CDN design / feed fan-out:** pre-computing reads at the edge — the same
  "do the work before the query" instinct the inverted index used.
