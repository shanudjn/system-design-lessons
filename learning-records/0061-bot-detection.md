# Learning record — Lesson 0061: Bot Detection & Traffic Authenticity

**Date:** 2026-08-17
**File:** `lessons/0061-bot-detection.html`
**Spine topic:** #61 — Bot detection & traffic authenticity (the last queued spine topic)
**Trade named:** abuse reduction vs friction & false positives

## What this lesson covered
Telling humans from programs at the front door, framed as an **economics problem wearing a
detection costume**. Worked example: the orders platform front door (40k req/s, L27/49/55/60)
plus its login endpoint. A request arrives as a bare HTTP call with no "I am human" stamp.

Four-move spine:
- **Estimate** — ~40% of traffic is automated → ~16k bot req/s = a ~60-server render bill (L59)
  before any abuse. The obvious defense fails: credential stuffing with 10M stolen pairs spread
  over 10,000 proxy IPs at 1 attempt/min each (0.017 req/s/IP) stays under every per-IP limit yet
  grinds the whole list in ~16.6 h. And the governing asymmetry: a false negative (bot served) is
  cheap+visible compute; a false positive (human blocked) is expensive+INVISIBLE — a lost customer
  who never appears in success metrics (0.2% FP of 24k human req/s = 48 people/s ≈ 4.1M/day).
- **Model** — a FUNNEL (L14/16 narrow-cheap-perfect-expensive): [1] free passive **signals**
  (rate, IP reputation, TLS/header **fingerprint / JA3** — a UA/TLS mismatch is a catchable lie),
  [2] active **challenges** (JS test / CAPTCHA / **proof-of-work**: 20 zero bits ≈ 2²⁰ ≈ 0.5s CPU,
  free for one login but 10M × 0.5s ≈ 58 CPU-days — flips a free attack to priced-per-attempt),
  [3] **rate-based** detection (loud single client) + **behavioral/aggregate** (population shape:
  login-fail ratio 3%→47%) run together.
- **Trace** — (A) loud scraper caught free at layer 1 → tarpit; (B) low-and-slow botnet running a
  REAL headless browser slips past rate+fingerprint, exposed only by the endpoint-level behavioral
  signal, stopped by PoW + account step-up; (C) **graduated response** (score 0–1 → allow ~99% /
  challenge the middle invisibly-first / block the confident) keeps friction off humans.
- **First bottleneck** — twofold: (1) perfect separation is IMPOSSIBLE (a program can look
  arbitrarily human → detection is a probability, the two errors trade off) → reframe from
  DETECTION to ECONOMICS (make abuse cost more than it earns, tax real users ≈0); (2) the
  adversary ADAPTS (L52) → **defense in depth**, mixing forgeable client-side signals with
  unforgeable server-side AGGREGATE signals the attacker can't see; retrained continuously.

Deepest point: "bot detection" is not a wall you build once but a market you keep tilting — you
never label every request correctly, so the winning move is making abuse unprofitable, applied
precisely to the suspicious and never to the clearly-human.

## Reuses (how it threads the course)
L08 rate limiting + its distributed blind spot; L52 adaptive adversary → defense in depth; L55
edge (where the decision runs, before rendering); L59/60 wasted render compute for bots; L49
gateway admission control; L14/16 narrow-cheap-perfect-expensive funnel; L07/28 reject-at-the-door.

## Interactive quiz
4 questions: (1) low-and-slow distributed stuffing defeats per-IP rate limiting → behavioral/
aggregate detection; (2) the false-positive asymmetry — why max sensitivity is a trap; (3) why a
graduated response beats CAPTCHA-ing everyone; (4) the adaptive adversary (real headless browser
forges a single signal) → defense in depth + server-side aggregates.

## What it sets up next — course spine exhausted, new batch added
Lesson 61 was the **last queued spine topic**. Per the NOTES.md convention, this author appended a
new batch of 5 advanced topics so the course never runs dry:
- **62 — Consensus internals (Raft/Paxos):** open the black box used in L10/22/34/43 (replicated
  log, terms + majority vote, quorum commit, safety across partitions).
- **63 — Building a distributed KV store (capstone):** wire consistent hashing + quorums + gossip/
  Merkle + LSM into one Dynamo/Bigtable-style system.
- **64 — Stream processing & windowing:** stateful exactly-once, watermarks, checkpointing (past
  L29's lambda/kappa sketch).
- **65 — Vector databases & semantic search:** ANN over embeddings (HNSW/IVF), recall vs latency,
  where L16's keyword index can't reach.
- **66 — Global rate limiting & quota:** L08 coordinated across regions/fleet (sloppy counters,
  token sync, per-tenant global quotas, CAP tension).

Next author picks #62 (Consensus internals) as the lowest-numbered unmarked topic.
