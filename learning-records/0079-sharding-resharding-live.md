# Lesson 0079 — Sharding & Resharding Live: Splitting a Hot Shard Without Dropping a Write

**File:** `lessons/0079-sharding-resharding-live.html`
**Curriculum spine:** topic #79 (advanced batch) — now ✅
**Date:** 2026-09-04

## The one worked system
The e-commerce **orders** table (continuing L78, reusing L12/25/29) grown to **10
billion rows / ~10 TB** taking **40,000 writes/s** at peak — past any single node
(comfortable ceilings ~2 TB and ~5,000 writes/s). Split it across shards, route
every read/write by a **shard key**, and later split a hot shard while it serves
live traffic. Adversary = **skew** (traffic is never even).

## What it covered (the four moves)
- **Estimate** — two SEPARATE walls (L3): storage `10 TB ÷ 2 TB = 5`, throughput
  `40k ÷ 5k = 8`; take the **max (8)** + headroom → **16 shards** (625 GB &
  2,500 w/s each). Why `hash % N` is a trap: adding a shard remaps where
  `hash%16 == hash%17` (~1/17), so **~94% of rows move** to grow ~6% → consistent
  hashing (L4) moves ~1/N instead.
- **Model** — three strategies as one question (key→shard): **range** (locality /
  sequential-key hotspot), **hash** (even load / no locality + %N reshard pain),
  **directory** (flexible placement / lookup hop + hot dependency). Shard key
  chosen against **cardinality + even distribution + dominant query** at once —
  the tension IS the headline trade. Shard key = a **one-way door** (changing it
  moves 100% of data).
- **Trace** — (A) point read BY shard key → one shard, 16× capacity at 1×
  latency; (B) no-shard-key query → **scatter-gather** to all 16, latency =
  slowest (`1−0.99^16 ≈ 15%` chance of a slow shard, L17 tail), 16× load → fix by
  sharding on the filtered field / a secondary copy (L12/33) / warehouse (L29);
  (C) **split a hot shard live** = L24's expand → dual-write → backfill → verify →
  cutover → contract on a whole key range.
- **First bottleneck** — resharding without dropping a write: **dual-write ON
  before backfill** (else in-flight writes lost on the new shard) + **verify
  caught-up before cutover** (keep old shard warm = instant rollback). Walls:
  a single hot KEY no split can cool (sharding balances RANGES, not one key →
  decompose/replicate/cache, L6/21/78); cross-shard transactions lose atomicity
  (→ co-locate by shard key, else saga L32); shard key is a one-way door
  (→ choose deliberately, soften with directory/secondary indexes, L33).

## Key idea to carry
Sharding doesn't make one big database; it makes **many small independent ones**
and asks you to pretend they're one — and the pretense holds ONLY for queries
that name the shard key. The shard key silently divides queries into fast/one-
shard vs slow/all-shards and transactions into atomic vs distributed, nearly
permanently. A hot RANGE is a sharding problem (split it); a hot KEY is an
application problem (decompose it).

## Reuses
L3 (two ceilings, hotspot, ~94% move), L4 (consistent hashing), L24 (online
migration + the two ordering rules), L19 (scatter-gather + hot cell), L17 (tail
amplification across fan-out), L6/21 (hot-row wall, decomposing a hot key), L32
(cross-shard transactions as sagas), L33 (secondary copies via outbox/CDC), L29
(send analytics scatter-gathers to the warehouse).

## Trade named
Even load & locality vs query flexibility.

## Sets up next
**Lesson 80 — Serialization formats & the wire:** we keep shipping bytes between
shards (Path B merges 16 partials), services, and clients (L33). What format are
those bytes in? JSON vs Protobuf/Avro/Thrift, schema evolution (L18/24), the
CPU + bytes bill at 40k req/s, zero-copy/columnar for analytics (L29). Trade:
human-readability & flexibility vs size, speed & schema discipline.

## Note on the spine
Reached the last previously-queued topic (#80 was the tail). Added five new
advanced topics (#81–85: backups/PITR, normalization vs denormalization,
full-text/relevance pipelines, graph/recommendation traversal, multi-region data
placement & residency) so the course never runs dry.
