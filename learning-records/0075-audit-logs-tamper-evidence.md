# Lesson 0075 — Append-Only Audit Logs & Tamper Evidence: Prove Nobody Rewrote History

**File:** `lessons/0075-audit-logs-tamper-evidence.html`
**Spine topic:** #75 (append-only audit logs & tamper evidence)
**Date:** 2026-08-31

## Worked example
One platform's audit log of privileged actions — every refund (L25), permission grant, flag flip (L74), and data deletion (L56) — recorded as "who did what, when, to what." 50M audited actions/day, ~500 B each, kept 7 years. The adversary is a **rogue insider** (or an attacker with stolen admin access) who does something they shouldn't, then tries to edit or delete the log entry that would expose them. Goal: make the record **tamper-evident** — not necessarily un-editable, but with any edit **impossible to hide** — and able to *prove* two things to a hostile auditor: "show me everything actor X did" and "prove this one entry is genuine and unaltered."

## What it covered
- **Estimate — scale is not the enemy; trust is.**
  - Rate 50M/86,400 = **579 writes/s**; storage 25 GB/day → 9.1 TB/yr → **~64 TB / 7 yr**. Trivial — a single indexed table could hold it.
  - The real cost = the **cost of forgery**: in a plain mutable table, erasing a fraudulent refund = **one silent statement** (DELETE/UPDATE), ms, no trace, available to exactly the insider you're trying to catch. Design goal = raise forgery cost from "one silent statement" to "rewrite the whole tail AND alter every externally-published fingerprint" (the second half being impossible for the attacker).
  - Locking table permissions is real defense-in-depth but is *prevention*; tamper-evidence is the layer that still works *after* prevention is bypassed (disk/backup/DBA access).
- **Model — four layers, each raising the price of a lie.**
  - **Append-only / WORM (L20)** — no UPDATE/DELETE grant; immutable-retention object storage. Makes editing against policy — a *policy, not a proof* (a compromised storage layer isn't bound by it).
  - **Hash chain** — `H_n = SHA-256(entry_n ‖ H_{n-1})`. Changing entry k changes H_k → breaks H_{k+1}…tip. So a **single silent edit is impossible**; it forces a full-tail rewrite. Honest limit: hashing is fast (millions/s) and public, so an attacker with write access **can** recompute a clean tail — **the chain ALONE is insufficient**. What it buys alone: anyone who *remembers a later hash* catches the edit.
  - **Merkle tree** — leaves = entry hashes, parents = hash of children, one **root**. **Inclusion proof** = the ~log₂(N) sibling hashes leaf→root → prove one entry without the other N−1. **Consistency proof** (O(log N)) = new root is a pure extension of the old → append-only proven, not trusted.
  - **External anchor** — periodically publish the tip/root somewhere out of the attacker's control (separate security domain / notary / partner / public chain). **The true root of trust**: a rewritten tail won't match an already-published anchor, and the attacker can't change the anchor. Cadence = a freshness dial (L2/14/74): more often → smaller un-pinned window.
  - Signatures (L30) complement, don't replace: they prove authorship/content of one entry but not order or completeness. Chain = order; anchor = anti-rewrite; signatures = authorship. Compose all three.
- **Trace.** (A) append: build entry → H_n = h(e|H_{n-1}) → append → update Merkle root → periodically anchor; note step 2 needs the prev tip → **writes serialize**. (B) full verify: re-walk genesis→tip recomputing each hash, then compare tip to last anchor (in practice verify only since last anchor + consistency proof). (C) prove one entry: return entry + 37-hash (~1.2 KB) inclusion proof; verifier recomputes root, compares to anchored root. (D) attack — delete (seq gap + H4 breaks), edit (H3→tip break), full-tail rewrite (internally consistent but ≠ the anchored tip). Only tampering *newer than the last anchor* escapes.
- **First bottleneck — the chain is strictly sequential.** Each append needs the prev hash → can't parallelize like a sharded table. Fine at 579/s; chokes a firehose (L74's 100k/s). Fixes: **batch into blocks** (Merkle tree per block, chain the block roots — group-commit; entry not provable until block closes, L29 batch window) or **shard into independent chains** (per region/service, each own anchor — L3/9 throughput, but **lose a single global total order** → L35 logical clocks to merge). Trade: verifiable global order vs write throughput.
- **Walls.**
  1. **Tamper-evident ≠ tamper-proof, and can't see un-logged actions.** Detects, doesn't prevent; an action that *bypassed the logger* leaves no gap. → write the entry **atomically with the action** at a **choke point** (L33 outbox), **monotonic seq #** so a delete shows a gap, and someone must **actually verify** (an unwatched log is a log with extra steps).
  2. **The anchor is the true root of trust → must be genuinely out of reach.** Anchoring into the same DB moves trust, doesn't remove it. Needs a separate security domain / external witness. Trade: independence vs operational coupling & latency.
  3. **Immutability vs right-to-be-forgotten (L56).** Append-only says never delete; GDPR says a user can demand deletion. → **crypto-shredding** (L30): store PII encrypted per-subject, "delete" = destroy the key → entry & chain stay byte-intact and still verify; content is unrecoverable ciphertext. Trade: immutability vs erasability, reconciled by "present but meaningless."
- **Deepest point.** An audit log **inverts** the course's usual goal — instead of fast/elastic/changeable, it deliberately makes one part **impossible to change and proves it**. The value isn't storage (a table does that) but **evidence** a hostile auditor verifies with math. Built almost entirely from pieces already owned (hashing, Merkle, outbox L33, envelope encryption L30, partition L3/9, freshness dial L2/14/74) — the new idea is only the *goal*: verifiable history. Trade: verifiability & compliance vs write cost and the impossibility of edits.

## Numbers used (all checked, `python3`)
- Rate: 50,000,000 / 86,400 = **578.7 → 579 writes/s**.
- Storage: 50M × 500 B = **25 GB/day**; ×365 = **9.125 TB/yr**; ×7 = **63.875 ≈ 64 TB/7yr**.
- Entries/7yr: 50M × 365 × 7 = **127,750,000,000 ≈ 1.28×10¹¹**.
- Tree depth: ⌈log₂(1.28×10¹¹)⌉ = ⌈36.89⌉ = **37 levels** (2³⁷ = 137.4B > 128B).
- Inclusion proof: 37 × 32 B (SHA-256) = **1,184 B ≈ 1.2 KB**.
- Anchor cadence example: every minute = **1,440 anchors/day**.

## Threads reused
L74 (config/flag delivery — records who flipped which flag), L25/38 (ledgers/event sourcing — immutable append-only records), L30 (hashes, digital signatures, envelope encryption, crypto-shredding), L56 (GDPR deletion — the right-to-be-forgotten collision), L33 (CDC/outbox — write the entry atomically with the action at a choke point), L20 (object storage — WORM/immutable retention), L3/9 (partition-for-throughput — shard the chain, lose global order), L35 (logical clocks — recover cross-shard order), L29 (batch window — block batching latency), L2/14/74 (freshness-vs-cost dial — the anchor cadence).

## Sets up next
- **Lesson 76 — Health checks & graceful degradation:** the difference between "up" and "useful." Shallow vs deep health checks, why a check probing a *shared* dependency can fail the whole fleet at once (L34), load-shedding tiers and feature degradation (drop recommendations to keep checkout alive, L28), and the fail-open vs fail-closed choice per dependency. Trade: availability of the core vs completeness of the experience.
- Spine still has topics 76–80 queued (health checks; blob/media transcoding pipelines; multi-level write-through vs write-back caching; sharding strategies & resharding live; serialization formats cost), so no new topics were added this run.
