# Learning record — Lesson 0022: Distributed Locks & Leases

**File:** `lessons/0022-distributed-locks.html`
**Curriculum spine:** topic 22 (Distributed locks & leases) — now ✅
**Builds on:** L21's teaser (two workers both think they hold the lock), L10 (epoch/fencing + heartbeat-timeout sizing), L06 (compare-and-set race, replication lag), L07 (partial-failure mindset), L13 (idempotency as the escape hatch), L03/L04 (sharding/single-writer routing to design the lock away).

## What this lesson covered
One nightly billing job across five workers that must charge each customer **exactly once**, and every way a distributed lock lets two workers charge twice:

- **The hazard (traced up top):** lock TTL=15s. t=0 A acquires (token 41) & starts; t=6 A hits a 22s stop-the-world GC pause; t=15 lease expires (A frozen, can't renew); t=16 B acquires (token 42) & charges $50; t=28 A un-freezes, still *believes* it holds the lock, charges again → double-charge. The lock worked; A's *belief* outlived its lease.
- **Estimate — sizing the TTL:** heartbeat h=5s, TTL=3h=15s survives 2 missed renewals; dead holder → failover ≤ TTL = 15s. Shrink to h=1s/TTL=3s (L10's numbers) = faster recovery but a live holder pausing >3s is false-evicted = **flap**. Rule: TTL ≥ (worst pause to tolerate) + clock-skew margin — but no finite TTL survives an unbounded pause, so **TTL is a liveness knob, never safety**.
- **Model in 3 layers:** (A) atomic **set-if-absent** `SET k v NX PX 15000` → one winner; (B) **lease** = TTL + heartbeat renew so a dead holder can't block forever — but this *opens* the two-holder window; release must be **compare-and-delete** (L06 CAS) or you delete someone else's lock; (C) **fencing token** = monotonic counter handed out per grant (41,42,43…); the **resource** keeps `highest_token_seen` and rejects any write with a lower token → A's token-41 write is fenced out at the ledger, no clock needed. This is L10's epoch applied to *data writes*.
- **Trace 3 paths:** happy (acquire→renew→work→compare-delete), clean crash (hard crash → lease auto-expires in ≤15s → B safely re-runs, needs L13 idempotency), and the freeze (fencing turns the double-charge into a rejected write). Key reframe: you can't *prevent* the overlap (freezes are physics), only *neutralize* it (loser's writes refused).

## The first bottleneck (where it lands)
The lock itself is the most expensive tool: (1) **serialization ceiling** — one shared lock @ 5ms critical section = 1/0.005 = **200 ops/s** regardless of fleet size (10M charges = 13.9 h) → use fine-grained/per-key locks or shard the work (L03) so no shared lock; (2) **SPOF** — the lock lives in one authoritative place; a single Redis node failing over to a lagging replica (L06) can grant the same lock twice → wants consensus (etcd/ZK on Raft, L10), which costs a quorum round per grant; (3) **clock-skew-fragile** — any TTL/timestamp safety argument across machines is only as good as NTP/leap-second/VM-jump, which is exactly why fencing needs *no clock*, only "bigger than last time".

**Escape hatch (the real lesson):** a distributed lock is a **last resort**. Prefer **idempotency** (L13 — client key + dedup so duplicates are harmless) or **single-writer routing** (L04 consistent hashing / L10 leader-per-shard so concurrency is designed out). Distinguish an **efficiency lock** (double-run merely wasteful → a plain lease is fine) from a **correctness lock** (double-run corrupts → need fencing, or better, idempotency so the lock isn't load-bearing).

## Trade-offs named
Safety vs liveness (the core, dialed by TTL) · fast failover vs false-eviction/flap (short vs long TTL) · trust-the-writer vs trust-the-resource (TTL check vs fencing token) · prevention (impossible) vs neutralization (achievable) · coarse-safe-serial vs fine-grained-parallel locks · pessimistic locking vs optimistic idempotency.

## What it sets up next
**Lesson 23 — Multi-region / active-active:** we kept safety by forcing one writer at a time; now put writers on different continents and let both accept local writes for speed. No single lock to serialize them → conflict resolution: last-write-wins (and what it silently drops), CRDTs that merge cleanly (recap L06/L11), and the write-latency-vs-availability tax. The trade shifts from **safety vs liveness** to **local writes vs global order**.
