# Learning record — Lesson 0056: Data Deletion & the Right to Be Forgotten

**Topic:** Actually erasing one user across a sprawling, partly-immutable system —
the bill for fifty-five lessons of making data durable and copied.
**Worked example:** Erase user U-8842 ("Alice") from a 200M-user platform whose
data lives in a primary DB (L12/24), an append-only event log (L38), read models
(L37/L33), caches (L48), edge KV at 300 PoPs (L55), a warehouse (L29), a search
index (L16), 35 nightly backups, and third-party exports (L45).

## What this lesson covered
- **Estimate — the copy count and why the row is 1%.**
  - Alice lives in ~15 systems + 35 daily backups → the primary-DB row is ~1-2%
    of the copies; a third of the rest are immutable.
  - Rewriting immutable stores per user is arithmetic-impossible: a 10 TB backup
    can't be surgically edited → restore-edit-reseal = 10 TB × 35 backups =
    350 TB per user; × 100k deletes/month = **~35 EB/month**. Not "expensive" —
    impossible. Deletion must be O(1), not O(data).
  - Crypto-shred deletes a 32-byte DEK; all 200M keys = **6.4 GB** (fits a fast
    key store). One key-delete turns Alice's ciphertext to noise everywhere at
    once — O(exabytes hunt) → O(32 bytes).
- **Model — the four-part kit.**
  1. **Data map / personal-data registry** — you can't delete what you can't find;
     an untracked copy is an undeletable copy. Sprawl grows the map.
  2. **True deletion vs crypto-shredding**, chosen by one dial "is this store
     rewritable?": true DELETE for mutable (DB row, read models re-derived, search,
     cache); crypto-shred (destroy per-user DEK, L30) for immutable (event log L38,
     backups, WORM). Catch: crypto-shred only counts if EVERY copy of the key is
     gone — it moves the problem from "reach every copy of the data" to "reach every
     copy of the key" (6.4 GB vs PB → far smaller, controllable surface).
  3. **Suppression tombstone** — hash(id)+timestamp+status, no PII — every
     re-materialization path (restore, event/webhook replay, import) checks it, so
     deletion is a defended STATE, not a one-shot event. Paradox: to prove someone
     is forgotten you must remember that you forgot them. Doubles as audit proof.
  4. **Idempotent orchestrator** — saga/durable job (L13/L32/L46) fanning out to
     every registered store with retries until each acks.
  - Four properties: completeness, irreversibility, durability-of-deletion,
    provability.
- **Trace.** (A) primary DB + read models = mutable DELETE, the easy ~1-2%;
  (B) append-only event log — CANNOT delete event #4,412 (breaks hash chain + all
  downstream replays); crypto-shred instead → event survives, PII payload becomes
  noise, one key-delete erases all ~5,000 of her events; only works because data
  was BORN per-user encrypted; (C) 35 sealed backups (crypto-shred + age-out),
  warehouse 3.65 PB day-partitioned (crypto-shred or scheduled partition rewrite
  L46), edge KV 300 PoPs (propagate delete / TTL), third parties (FORWARD the
  request — you control your reach, only request theirs).
- **First bottleneck — structural (two faces).** Reach wall (derived-data sprawl:
  # deletes = # copies; a missed registry entry leaks forever) and immutability
  wall (event log/backups/WORM built un-editable on purpose). Resolution: separate
  the FACT from the PAYLOAD — keep the immutable event, encrypt the personal payload
  per-user, key-destruction removes the person while honest history survives.
  Privacy-by-deletion → **privacy-by-encryption**. Two more walls: key management
  is the new crown jewel (a surviving key copy = not deleted); erasure is not
  absolute — legal holds / retention (financial records 7 yr, L25) force partial
  erasure, deleting all that isn't legally pinned and logging why the rest stays.

## Trade named
Compliance & privacy vs immutability & derived-data sprawl. Delete the meaning,
not the record; keep the proof, not the person.

## Reuses
L38 immutable event log (the store you can't rewrite), L30 key hierarchy /
encrypt-at-rest (now load-bearing for privacy), L37/L33 read models & CDC (re-derive
without the user), L48 caches + L55 edge KV (copies that age out), L29 warehouse
(day-partitioned sprawl), L45 webhooks (data exported to third parties),
L13 idempotency + L32/L46 sagas & durable jobs (retriable fan-out), L25 ledger
retention (legal hold making erasure partial), L16 search index.

## Sets up next
**Lesson 0057 — Bulk data pipelines & backfills:** the one-off log-reprocessing
this lesson did for one user generalizes to reprocessing history for BILLIONS of
rows (schema change L24, new warehouse model L29). Trace one backfill: rate-limit
against live production (L28), checkpoint for crash-resume, idempotent re-runs (L13)
so retried batches don't double-count. Trade: throughput vs impact on the live system.
