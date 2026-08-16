# Learning record — Lesson 0060: Rendering & Delivery at Scale (SSR / Edge / Streaming)

**Date:** 2026-08-16
**File:** `lessons/0060-rendering-delivery.html`
**Spine topic:** #60 — Rendering & delivery at scale (SSR/edge rendering)
**Trade named:** time-to-first-byte & offload vs freshness & compute

## What this lesson covered
Getting HTML onto a user's screen framed as a **placement problem**, not a front-end
detail — the last hop of the whole course. Worked example: one product page on the orders
platform (40k req/s, phone on ~12 Mbps, ~100 ms to origin / ~15 ms to a nearby edge PoP).

Four-move spine:
- **Estimate** — timed a page load. The costs hide in (1) serial *dependent* round trips
  (~100 ms each, chained: HTML → bundle → data), and (2) the phone's CPU: a 300 KB bundle is
  ~200 ms to download but another ~300 ms to *execute* (~1 ms/KB), and JS execution doesn't
  speed up with the network. Server side: a per-request render is ~30 ms × 40k/s ≈ **150 cores**
  vs ~0 for a cached static page (Lesson 59's cache-hit-vs-miss, measured in render CPU).
- **Model** — two dials: **where** the HTML is built (build time / origin / edge / client) and
  **how much must hydrate**. Four strategies (SSG static, SSR origin, edge render L55, CSR
  client) + **streaming SSR** (flush shared shell now, stream slow per-user slice later) +
  **hydration** (server-rendered HTML is inert until the bundle downloads+executes → the
  FCP-to-TTI "looks ready, taps dead" gap; cost = bundle size → ship less via islands).
- **Trace** — same page rendered 3 ways timed (static ~42 ms to content/~0 cores/generic;
  SSR ~180 ms/~150 cores/personal; CSR ~770 ms blank-then-waterfall/~0 cores); the
  cache-vs-personalization collision (personalize → every response unique → CDN hit → 0 →
  L14 stray-param cache-killer reborn); streaming the 200 ms recommendation slice (L53) so it
  stops blocking the fast shell.
- **First bottleneck** — twofold: (1) can't be BOTH fully cacheable AND fully personalized →
  draw the **personalization line** (cache the shared shell, render only the per-user slice as
  an island / streamed chunk / edge personalization); (2) HTML is cheap to send, interactivity
  (hydration) is expensive → ship only the JS the islands need.

Deepest point: where you build the HTML is the *same* placement question as where a cache, a
computation, or the truth lives — answered for the last hop to the screen.

## Reuses (how it threads the course)
L14 CDN/TTL/shell-vs-slice/stray-param cache-killer; L55 edge rendering + decision-vs-truth;
L59 render priced per request (cache hit vs miss in cores); L27 knee / ~150 render cores;
L28 don't-block-fast-on-slow + L49 BFF graceful-degrade (streaming/placeholder); L53 rec ML
call (the slow slice); L02/48 caching one-copy-vs-N-renders.

## Interactive quiz
4 questions: (1) personalizing a cacheable page collapses the CDN hit ratio; (2) the FCP-to-TTI
hydration gap and shipping less JS; (3) why pure CSR hurts a content/SEO page; (4) streaming to
stop a slow slice blocking the fast shell.

## What it sets up next
**Lesson 61 — Bot detection & traffic authenticity** (the one topic left on the spine): every
one of those 40k req/s assumed a real human. Before the edge renders or rate-limits anything, it
must tell a browser from a script — benign crawlers vs hostile scrapers/credential-stuffers/
scalpers. Signals, challenges, rate-based vs behavioral detection, and the false-positive cost of
blocking a real user. Trade: abuse reduction vs friction & false positives. (Builds on L52
moderation/adversary, L55 edge, L59 wasted compute, L08 rate limiting.)

> Note: after Lesson 61 the spine's queued advanced topics are exhausted — the next author
> should add 3–5 new advanced topics per the NOTES.md convention so the course never runs dry.
