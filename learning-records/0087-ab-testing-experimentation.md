# Lesson 0087 — A/B Testing & Experimentation Platforms

**File:** `lessons/0087-ab-testing-experimentation.html`
**Curriculum spine:** topic 87 (advanced batch, queued after L86).
**Worked example:** the L86 marketplace (5M DAU, one "Products you may like" strip). One change = **reranker_v2** (L86's new re-rank model). One metric = **strip CTR** (fraction of exposed users who click a card), baseline 10%. Randomization unit = the user. Question: prove v2 helped before it reaches all 5M users.

## What it covered
- **The core idea:** an A/B test is a machine for separating a **real effect** from **noise**. A before/after comparison confounds the change with everything else that changed that week; only a **parallel control group** cancels external noise and earns causal attribution.
- **Estimate — size & duration:**
  - Sample size per arm ≈ `16·p(1−p)/δ²`, where 16 = (z₉₅+z₈₀)²·2 = (1.96+0.84)²·2 = 15.68.
  - p=0.10 → p(1−p)=0.09; 2% relative lift (δ=0.002) → **360,000/arm = 720,000 total**; 4% lift (δ=0.004) → 90,000/arm. `n ∝ 1/δ²` → halving the detectable effect costs 4× users (1% lift ≈ 2.9M).
  - At 5%+5% ramp (250k treatment/day) the floor is met in ~1.5 days, but **run ≥1 week** for weekday/weekend representativeness + novelty washout. Trade: rigor vs velocity.
- **Model — the platform:**
  - **Deterministic hash bucketing** (L03/04): `bucket = hash("reranker_v2:"+user_id) % 10000`; 0–499 control, 500–999 treatment, rest untouched. Stateless, consistent, comparable groups; salted with exp name so a user gets an independent bucket per experiment.
  - **Exposure logging** (L64 stream): log the user only when the strip actually renders; analyze **EXPOSED** not **ASSIGNED** (triggered analysis); both arms must log identically or bias/SRM appears.
  - **Layered assignment:** same layer = mutually exclusive (interacting changes); different layer = orthogonal via independent salt (concurrent, non-confounding). Enables hundreds of simultaneous experiments.
- **Trace three paths:** (A) one user assigned statelessly, same bucket on any server forever; (B) clean 7-day readout with the three gates — SRM ✓ → lift +0.40pp (CI [+0.34,+0.46], p<0.001) → guardrails ✓ → ship; (C) broken split 1,750,000 vs 1,762,000 = 6.4σ (χ²≈41, p≈2e-11) → SRM → invalidate & rerun.
- **First bottleneck + walls:** trust rests on a valid split → **SRM is the tripwire** checked before any metric (analysis order: SRM → lift → guardrails). Walls: **peeking** (stop at first p<0.05 → real α 5%→>20%→~100%; fix = pre-register fixed horizon or sequential/always-valid tests); **metric-gaming** → guardrail metrics veto a primary win; **interference/SUTVA** (treatment eats shared inventory, cannibalizes control) → cluster/geo/switchback randomization (L23/34), fewer units → bigger MDE.
- **Four traps:** before/after instead of A/B; peeking; trusting a metric with no guardrails; reading results without the SRM check.
- **Interactive quiz (4 Q):** why a before/after CTR climb isn't evidence (no control); a "0.3%" split imbalance = SRM at 6.4σ; the peeking trap; a CTR win that fails guardrails → ship nothing.

## Reuses / threads pulled through
L86 (the re-ranker being tested), L03/04 (hash-to-bucket, stateless consistent routing — now to a variant), L64 (high-volume exposure event stream), L27/28 (latency knee / goodput cliff a guardrail catches), L23/34 (shared invariant / shared dependency — here interference breaking the independent-split assumption).

## What it sets up next
**Lesson 88 — Notification systems end-to-end:** the experiment said "ship it," so deliver the result — one "your order shipped" across email/SMS/push/in-app: preference & dedup (L73), per-channel retries + DLQ (L09/68), rate-limiting a user's inbox (L08), template rendering, quiet-hours/batching digest. Trade: reach & timeliness vs user annoyance & cost.

Spine still has topics 88–90 queued, so no new topics were added this run.
