# Learning Record — Lesson 0040: Multi-Tenancy & Noisy Neighbors

**Date:** 2026-07-27
**File:** `lessons/0040-multi-tenancy.html`
**Curriculum spine:** topic #40 (first of the "many customers on one system" arc)

## The concrete system
A B2B SaaS (project tracker) with **50,000 tenants** sharing one app fleet
(Lesson 27, 40k req/s) and one Postgres cluster whose app-side **connection
pool holds 300 connections**. Every row is tagged `WHERE tenant_id = ?`. The
whole point: sharing makes each tenant cost pennies — and sharing is also the
one thing that lets one tenant hurt all the others.

## What the lesson covers (the four moves)
- **Estimate — the blast radius of one greedy tenant.** Normal headroom is
  Little's Law (L13): `300 conns ÷ 5 ms = 60,000 q/s` vs 40k demand. One
  tenant's bulk export = **200 queries × 2 s each** grabs 200 of 300
  connections → everyone else drops to `100 ÷ 0.005 = 20,000 q/s` < 40k demand
  → unbounded queue (L28) past the knee (L27). **100% blast radius from
  0.002% of the tenants, no per-request rule broken.**
- **Model — the isolation spectrum + three fairness controls.** Spectrum:
  shared table → schema/tenant → DB/tenant → **cell/tenant** (L36); density↑
  trades against isolation↓. Three controls on a live shared pot: per-tenant
  **rate limit** (L08 token bucket keyed on `tenant_id`, 50k buckets not one
  global), per-tenant concurrency **bulkhead** (L07, caps in-flight *slots* —
  the direct fix for the 2-second-hold drain), and **fair queuing** (a line
  per tenant, not one FIFO). Static cap vs work-conserving/weighted fair
  queuing (reclaims idle capacity but must preempt a borrower instantly).
- **Trace** — normal request (cheap, lets go); spike w/o isolation (100%
  outage); same spike with a 30-conn bulkhead (`270 ÷ 0.005 = 54,000 q/s` →
  zero impact, the greed paid by the greedy); a genuine **whale** needing
  5k req/s routed out to its own DB/cell (bigger room, not a tighter wall).
- **First bottleneck — a limit counts requests, not work.** A tenant inside
  every count-based cap still sends 10 un-indexed scans at ~40 s each (L12
  selectivity trap) and monopolizes CPU. Fix: **meter the scarce resource
  itself** (CPU-seconds/rows/bytes, L08 generalized) + per-query guardrails
  (statement timeout, row caps, forced LIMIT). Then: **shuffle-sharding**
  (L36, `C(100,2)=4,950` → ~0.02% full co-victim vs 100%), route hot tenants
  out (L03/19), and wall the async tier + cache too (L05/09/02) — not just the
  front door.

## Trade-off named
Density & cost efficiency vs isolation & fairness. (Sub-trades: static cap vs
work-conserving fairness; measurable-but-wrong request count vs
accurate-but-costly cost metering.)

## Reuse / callbacks
L02 skew (Zipf tenant usage), L03 sharding + hot-shard + routing, L07
bulkhead/429, L08 token bucket now keyed per tenant, L12 selectivity trap,
L13 Little's Law, L27 utilization knee, L28 bounded queue/goodput, L36 cells
+ shuffle-sharding.

## Sets up next
Lesson 0041 — **Graph & relationship systems**: friends-of-friends over a
billion-edge graph, why a relational JOIN explodes at depth (L12 N+1 across
hops), adjacency lists vs native graph stores, partitioning a graph without
cutting every edge (supernodes recur, L15), and BFS at scale. Also refilled
the spine with a fresh batch (topics 42–46: load balancing, gossip/anti-entropy,
time-series storage, webhooks/outbound delivery, distributed job scheduling).
