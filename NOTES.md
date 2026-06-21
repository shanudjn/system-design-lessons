# Teaching Notes — System Design track

## Learner
- Wants to learn **system design via worked examples** (asked 2026-06-21).
- Cross-track preference (see global memory): **hard ideas, simple words** —
  difficulty from the concept, never the prose.

## How to teach (conventions carried from the VidoFlip + junior-to-senior tracks)
- **Worked examples first.** Each lesson takes ONE concrete system and traces
  it end to end. Concrete example BEFORE the abstract rule.
- **Dark mode, self-contained HTML.** Palette: bg `#0e0e16`, card `#16161f`,
  ink `#e8e8f2`, accent `#a594ff`. Embedded CSS + tiny JS quiz with feedback.
- One orientation picture up top. Short declarative sentences. Define/earn every
  term before leaning on it.
- Print the `open` command at the top of each lesson. End every lesson by
  inviting follow-up questions.
- Keep `reference/glossary.html` aligned with the terms used.

## Calibration (starting assumptions — confirm as they answer)
- General system design (not VidoFlip-specific). Interview-flavoured but the
  goal is a real mental model, not memorized answers.
- Unknown baseline. Lesson 01 (URL shortener) pitched at: assumes programming
  literacy, teaches estimation + caching reasoning from scratch. Adjust density
  up if they say "too easy" (mirror the VidoFlip track's "raise the baseline").

## Curriculum spine (bottom-up, revise as we go)
1. ✅ **URL shortener** — estimate, data model, counter+Base62, read/write
   paths, 301vs302, first bottleneck → cache + the 4 properties that justify it.
   (Lesson 0001)
2. Scaling reads — cache eviction (LRU/TTL), CDNs, cache-miss storm / stampede.
3. Scaling writes — sharding, partitioning, distributed ID generation.
4. Async work — queues + workers (notification fan-out).
5. Consistency & replication — a like-counter under concurrency.
6. Designing for failure — timeouts, retries, idempotency, backpressure.

## Lesson format conventions
- Four reusable "moves" framing introduced in Lesson 01: estimate → model →
  trace the paths → find the first bottleneck. Reuse this spine across lessons.
- Every design choice should NAME its trade-off explicitly (that's the skill).
