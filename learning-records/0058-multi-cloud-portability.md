# Learning record — Lesson 0058: Multi-Cloud & Vendor Portability

**Topic:** Running one system across two cloud providers for resilience and price
leverage — and why the hard part is the data, not the servers. Generalizes L23's
multi-*region* (one provider, many places) into multi-*provider*, where the boundary
between clouds is metered and adds latency.
**Worked example:** The orders platform (L12/24/25/57), **~500 TB of data, 40k req/s,
all on Cloud A today**. Leadership wants it on Cloud A *and* Cloud B. The naive plan —
"it's containers, deploy the images to B and point DNS at both" — is correct about
compute and silent about data.

## What this lesson covered
- **Estimate — the two walls, both from "egress is metered, data is big."**
  - **Data gravity (move it once):** seed B with 500 TB = 500,000 GB × $0.09/GB =
    **$45,000** egress; at 10 Gbps = 1.25 GB/s → 500,000 ÷ 1.25 = 400,000 s ≈
    **4.6 days** of continuous streaming, chasing a moving target (L57).
  - **Boundary tax (keep it useful forever):** replicate the 5,000 w/s × 1 KB = 5 MB/s
    write stream ≈ 13 TB/mo = **~$1,170/mo**; but serving 20k reads/s × 2 KB across the
    boundary = 40 MB/s ≈ 104 TB/mo = **~$9,300/mo ≈ $112k/yr**. → The rule: **keep
    compute next to its data.** Ingress free, egress $0.09/GB — the L14 roach motel.
- **Model — three layers by portability + the lock-in dial.**
  1. **Stateless compute** — same container runs anywhere (state pushed off the box,
     L26). Portable, cheap to duplicate. The easy half; least of the work.
  2. **Stateful data** — the sticky core, resists moving two ways that stack: **weight**
     (data gravity) makes the bytes hard to move; **shape** (proprietary managed-service
     API) makes the code hard to move. This is the whole cost.
  3. **Control plane** — splits: global DNS/traffic mgr (L50) spans both clouds for free
     (it's above them); a strongly-consistent truth / leader election (L10/23) CAN'T
     span them without re-paying ~80–150 ms cross-boundary quorum + egress/message.
  - **Managed leverage vs LCD portability** (the real lock-in dial, set one service at a
    time): managed = provider operates it, proprietary API = locked in; LCD (VMs/K8s/
    self-hosted Postgres/S3-compat) = portable but YOU run it again (L10/34/47).
    Portability is insurance with a continuous premium — pay it where a claim is likely.
- **Trace.** (A) stateless request just works (DNS steers, either cloud serves) — until
  "reads/writes the data — where?". (B) stateful fork: **Option 1** single-home data in
  A, B reaches across = per-read egress ($112k/yr) + latency (cheap to build, ruinous to
  run); **Option 2** replicate so each cloud has a local copy → active/passive (standby
  promote) or active/active (L23 conflicts across providers; scalar invariant L25 still
  can't split the ocean). (C) Cloud A provider-wide outage: Option 1 → B is a compute
  SHELL with no data = total outage = **resilience theater**; Option 2 active/passive →
  DNS fails over → promote B's replica → survives (cost: replication bill + promotion
  window + L06/23 lag = data-loss window).
- **First bottleneck — data gravity anchors the whole system.** Compute was always
  portable; the cloud locked you in with the heavy/metered/proprietary DATA. (1) The
  "we'll move to negotiate price" leverage is a **bluff** unless portability is paid
  continuously — and if it is, you've pre-paid most of the switching cost the threat was
  meant to save. (2) Real escape: don't fight gravity, **shrink what crosses the
  boundary** to the thin replication delta (small), never the read traffic (huge) — L57
  move-the-scan / L14 copy-near-the-user. Bounded further by strong consistency that
  can't span providers cheaply (→ active/passive over active/active, or single-home the
  invariant L23) and by a doubled, multiplicative ops surface (L59). Honest answer for
  most teams: a warm DR copy, not symmetric active/active.

## Trade named
Portability & resilience vs complexity & cost. Two live data copies buy resilience and
leverage, paid in continuous egress + a second ops burden + cross-provider consistency;
simple single-homed data saves the money but quietly deletes the resilience.

## Reuses
L14 egress economics + copy-near-the-user, L23 multi-region active-active + single-home
the invariant (now across providers), L50 global traffic management (the spanning front
door), L26 push-state-off-the-disposable-box (compute portable), L57 move-the-heavy-scan
-off-the-primary (respect data gravity), L10/34/47 cost of running your own stateful
infra (what LCD buys back), L06/11/25 consistency limits (why strong consistency can't
span the boundary free).

## Sets up next
**Lesson 0059 — Cost-aware architecture (FinOps):** this lesson made every cross-boundary
byte a line item; next, make the bill itself the design constraint. The dominant cost
drivers (egress L14/58, storage tiers L39, compute utilization L27), attributing spend
per tenant/feature (L40/54) so you know *what* is expensive, and when the cheapest design
is the wrong one (the headroom you deleted to save money is the headroom the next spike
needed). Trade: unit cost vs performance & headroom.
