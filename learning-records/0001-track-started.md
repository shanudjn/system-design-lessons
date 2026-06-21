# System Design track started; Lesson 01 (URL shortener) shipped

On 2026-06-21 the learner chose to learn **system design via worked examples**
(picked it over continuing the VidoFlip compiler lesson). New track folder
`system-design/` scaffolded; a `learning/` SessionStart hook now asks which
track to study at the start of every session (see global memory
[[learning-hub-session-start]]).

**Lesson 01 — URL shortener** introduced the reusable four-move spine:
estimate → data model → trace the read/write paths → find the first bottleneck.
Key load-bearing ideas planted, to build on later:
- The read:write imbalance (100:1) drives the whole design.
- Borrow uniqueness from the DB counter; Base62-encode it (no collisions).
- The mapping is **immutable**, which is *why* a cache is almost free here —
  no invalidation problem. This is the hook for Lesson 02 (cache eviction,
  CDNs, stampede) and for later consistency lessons (where the value DOES
  change and invalidation returns).

**No evidence of learner baseline yet** — pitched at "programming-literate,
estimation taught from scratch." Watch the quiz answers and follow-ups; if they
say "too easy," raise density (mirror the VidoFlip "raise the baseline" move).
