# Learning record — Lesson 0039: Tiered Storage & Data Lifecycle

**Topic:** Tiered storage & data lifecycle — when the ever-growing log outgrows fast disk.
**File:** `lessons/0039-tiered-storage-lifecycle.html`
**Curriculum spine #:** 39 (40–41 still queued; course not yet out of runway).

## The one worked system
Lesson 38's **event log**: 5,000 events/s × 300 B, immutable, appended forever,
kept **7 years** for compliance (~330 TB). The question isn't *how* to store it
(append to a log) but **where each record should live as it ages**, because a
record written today and one written 3 years ago are the same size but worth
wildly different amounts to keep on fast disk (L02 read skew).

## What it covered (the four moves)
- **Estimate — the bill, all-hot vs tiered.**
  - Volume: 130 GB/day × 365 × 7 ≈ **330 TB**.
  - Skew (L02): ~90% of reads touch the last 7 days = ~1 TB = **0.3%** of the pile.
  - All-hot SSD @ $0.10/GB-mo = **~$33,000/mo** (~$396k/yr).
  - Tiered by age across the ladder → HOT 1 TB ($100) + WARM 10.8 TB ($216) +
    COLD 35.8 TB ($143) + ARCHIVE 285 TB ($285) = **~$744/mo = ~44× cut**.
    Biggest byte-slice (archive, 86%) = smallest bill-slice (38%).
- **Model — three parts.** The **tier ladder** (HOT SSD $0.10/~1ms → WARM HDD
  $0.02/~10ms → COLD object $0.004/~100ms-1s+fee → ARCHIVE glacier $0.001/
  minutes-hours; each step ~5× cheaper, ~10× slower). **Temperature** (access
  frequency) decides the rung — age is only a *proxy*, faithful for a write-once
  log. The **lifecycle policy** = a background, throttled sweep (L24) that demotes
  by age and **expires** at 7 yrs, keeping **tiering** (cost move, still
  retrievable) strictly separate from **expiration** (compliance move, gone), and
  the small **index HOT** (L20 metadata/data-plane split) while only payloads cool.
- **Trace.** Write always lands hot; hot read ~2 ms (the 90%); cold/archive read
  issues a restore job → **202 "retrieval in progress"** → minutes + per-GB fee;
  the sweep moves **copy-new-then-free-old** (L20 commit-metadata-last) and deletes
  at year 7.
- **First bottleneck — the retrieval-latency cliff.** Cold→archive isn't 10×
  slower, it's a **cliff**: ms → minutes/hours (spun-down disk / tape staging).
  Naive "keep everything warm" forfeits the 44×. Real fix: tier to each slice's
  **retrieval SLA + temperature** — a sub-1s support slice stops at COLD not
  ARCHIVE (4× more for a thin slice), and the always-hot index means existence
  lookups never cliff.

## Trade-offs named
- The ladder itself: **price vs retrieval latency**, tier by tier (each step
  ~5× cheaper AND ~10× slower — no free lunch).
- Lower tier's storage saving vs its **per-read retrieval tax** — cheap-to-store,
  expensive-to-read is a *bet* you rarely read it; only demote data that honors it.
- Lifecycle encodes two legal limits: **retention floor** (must keep) vs privacy
  **ceiling** (must delete) — cost on one side, law on the other.

## Four traps
1. Tier by **age** when age ≠ temperature → retrieval fees on still-read data can
   exceed staying hot (tier by access frequency; keep a promotion path).
2. Confuse **tiering with expiration** (delete required data, or keep personal data
   past its lawful max) — same sweep, different masters, keep separate + audited.
3. Put the **index on a cold tier** too → every existence check cliffs (keep the
   small metadata plane hot, L20).
4. Archive a **billion tiny objects** individually (per-request/min-size fees) →
   pack into big segments first (L20 small-file); and never free the old copy
   before the new one + index are committed.

## What it sets up next
**Lesson 0040 — Multi-tenancy & noisy neighbors.** Turn from one tenant's data
over time to many tenants sharing one system at once: per-tenant rate limits (L08),
quotas, fair queuing, the shared→silo→pool spectrum (row → schema → DB → cell L36
per tenant), tenant routing, the hot-tenant problem (L03/19), and shuffle-sharding
(L36). Trade shifts to **density & cost efficiency vs isolation & fairness.**
