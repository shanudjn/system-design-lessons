# Lesson 0080 — Serialization Formats & the Wire: The Same Message in JSON and Protobuf, Byte for Byte

**File:** `lessons/0080-serialization-formats.html`
**Curriculum spine:** topic #80 (advanced batch) — now ✅
**Date:** 2026-09-05

## The one worked system
One `OrderPlaced` event (order_id, customer_id, amount_cents, currency, status,
created_at) from the orders platform, shipped between services and consumers at
**40k req/s** (L33 outbox fan-out, L17's 20 hops). Written out **byte for byte**
in JSON and in Protobuf, then evolved over time. Adversary = **time** (the message
shape you ship today isn't next year's, and half the fleet runs old code when you
change it).

## What it covered (the four moves)
- **Estimate** — compact JSON = **127 bytes** (field NAMES on the wire = 55 B ≈
  43%, repeated every message; ints stored as decimal text). Protobuf = **28
  bytes** (1-byte field-NUMBER tags + varint ints; names live in the schema, sent
  0×) → **~4.5× smaller**. Varints verified: order_id 41 bits→6 B, customer_id 24
  bits→4 B, amount_cents 13 bits→2 B (`18 87 27`), created_at 31 bits→5 B. CPU ~3
  µs vs ~0.4 µs round-trip (**~7×**) → ~343 GB/day & ~2 cores across 20 hops at
  40k/s, plus lower p99 (fewer allocs/GC, L17).
- **Model** — four formats by WHERE the schema lives: JSON (in the message:
  readable/fat/unchecked), Protobuf & Thrift (shared `.proto`: compact/typed),
  Avro (a **schema registry** by id: tiny records). The **wire type** (3 bits/tag:
  varint / 64-bit / length-delimited / 32-bit) lets a reader skip an unknown
  field's exact bytes = forward-compat in the layout. "Schemaless" is a myth — the
  schema moves into every reader's code, un-versioned (L18 contract, unenforced).
- **Trace** — (A) ADD a field = new number → old readers skip, new default → safe
  everywhere (L18 additive). (B) REMOVE/RENAME/RETYPE: rename FREE in Protobuf
  (name not on wire, number is), BREAKING in JSON (name IS the key); **reusing a
  field number** reinterprets old bytes as a new type = silent corruption →
  `reserved` number+name, never reuse. (C) gradual rollout (L31) → half the fleet
  old → breaking change needs BOTH-directions compat → run as L24 expand →
  dual-populate → migrate → contract, on the wire.
- **First bottleneck** — the schema is a **distributed contract** across
  independently-deployed services: add-only, reserve removed numbers, migrate
  breaking changes; Avro's registry makes an incompatible schema a deploy-time
  error instead of a 3 a.m. runtime `undefined`. Walls: analytics wants
  **columnar/zero-copy** (Parquet/Arrow, L29) not row messages; binary costs
  **debuggability** (opaque bytes → log decoded copy, gRPC reflection); **the
  format follows the reader** (text for public/human, binary for known high-volume
  internal).

## Key idea to carry
A serialization format is a decision about **where the schema lives**, and that
sets **who must agree with whom before a message can change**. Put it in the
message (JSON) → no one coordinates, but you pay bytes every message and the
contract hides un-versioned until it breaks at runtime. Move it out
(Protobuf/Avro) → tiny fast wire, but a shared versioned artifact evolved in
lockstep-without-a-meeting — which is why field NUMBERS (not names) are the
contract and you may never reuse one. "Add-only" is the one always-safe move;
everything else is a migration.

## Reuses
L18 (published contract — additive safe, removes/renames break), L24 (expand/
contract, run on the wire), L31 (gradual rollout forces both-directions compat),
L17 (20 hops multiply marshalling CPU + tail), L29 (row vs columnar layout), L33
(outbox fan-out multiplies the bytes), L59 (cost lens on reclaimed cores).

## Trade named
Human-readability & flexibility vs size, speed & schema discipline.

## Sets up next
**Lesson 81 — Backups, restore & point-in-time recovery:** we've now shipped and
stored these bytes everywhere (L79 DB, L29 warehouse). What happens after a 2 a.m.
`DROP TABLE` or a corrupting deploy? Full vs incremental backups, the WAL/binlog
as a replay stream (L33/35), RPO vs RTO as two separate dials (L07/78), restoring
a 10 TB sharded DB (L79), and why an untested backup is not a backup. Trade:
recovery guarantees vs storage & operational cost.

## Note on the spine
Spine still has topics #81–85 queued (backups/PITR, normalization vs
denormalization, full-text/relevance pipelines, graph/recommendation traversal,
multi-region data placement). No new topics needed this run.
