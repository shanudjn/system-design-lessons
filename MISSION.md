# Mission: Learn system design through worked examples

## Why
Build a principal-engineer mental model of how real systems are designed —
not trivia, but the load-bearing *why* behind each decision. Learn it the way
it sticks: one concrete worked example at a time, traced end to end.

## Success looks like
- Given a system ("design a URL shortener", "a news feed", "a rate limiter"),
  I can reason from requirements → data model → APIs → scaling → failure modes.
- I can name the trade-off behind each choice (consistency vs. availability,
  read vs. write optimization, push vs. pull) and when I'd pick the other side.
- I can spot the bottleneck in a design and say what breaks first under load.

## How to teach (carried over from the learner's other tracks)
- **Worked examples first.** Every lesson takes ONE concrete system and traces
  it. Concrete example before the abstract rule.
- **Hard ideas, simple words.** Difficulty comes from the concept, never the
  prose. Short sentences. Define every term before leaning on it.
- **Dark mode, self-contained HTML lessons** with a tiny interactive quiz.
- One orientation picture up top. End each lesson by inviting follow-ups.

## Curriculum spine (planned — revise as we go)
1. Core vocabulary via one example: design a URL shortener (data model, hashing,
   read/write path, caching, the first bottleneck).
2. Scaling reads: caching strategies + CDNs (worked: a hot-key read path).
3. Scaling writes: sharding + partitioning (worked: an ID generator).
4. Async work: queues + workers (worked: send-a-notification fan-out).
5. Consistency & replication (worked: a like-counter under concurrency).
6. Designing for failure: timeouts, retries, idempotency, backpressure.

## Status
- Track scaffolded 2026-06-21. No lessons shipped yet — first lesson TBD once
  the learner confirms where to start.
