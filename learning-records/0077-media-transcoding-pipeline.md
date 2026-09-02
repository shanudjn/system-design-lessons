# Lesson 0077 — Media Transcoding Pipelines

**Published:** 2026-09-02
**File:** `lessons/0077-media-transcoding-pipeline.html`
**Spine topic:** #77 (was queued in the advanced batch)

## Worked example
One 10-minute 4K upload — a single ~3.3 GB H.264 file dropped into the object
store (L20) — turned into a full ABR ladder: 6 resolutions × 2 codecs
(H.264 + AV1), each cut into 100 keyframe-aligned 6-second segments. The opening
numbers: that fan-out is **1,200 transcode jobs from one file**, and encoded
monolithically the 4K rung alone is **~20 minutes** to first play — where
segmenting and publishing **segment 0** first gets there in **~8 seconds** for
the identical total work.

## What it covers
- **Estimate:** the fan-out (12 renditions × 100 segments = 1,200 jobs; 2
  uploads/s → 2,400 jobs/s → ~12,000 cores steady) and TTFP (20 min monolithic
  vs ~8 s segmented — same work, reordered).
- **The one idea:** stop treating the video as one file. The **segment** is the
  unit of parallelism, priority, resumability, and streaming, all at once
  (keyframe/GOP-aligned so each chunk decodes/encodes independently).
- **Model:** a **DAG** (probe → split[fan-out] → 1,200 transcodes → package[barrier]
  → publish); **priority lanes** (interactive / standard / background) so the
  waiting viewer beats the AV1 backfill (L9/28); the **storage/compute/TTFP
  triangle** resolved per-video by popularity — pre-encode hot, JIT the cold tail
  + cache (L48), tier the bytes (L39).
- **Trade named at each choice:** segment size (parallelism/latency vs
  compression/overhead), lanes (viewer latency vs batch throughput), pre-encode
  vs JIT (storage vs compute vs first-play), codec ladder (delivery bytes vs
  encode CPU, L14/59).
- **Trace:** upload live in ~8 s; a 10× launch burst (autoscale on **queue
  depth**, backpressure — defer, never drop); a cold video's first viewer served
  by JIT-and-cache; a worker dying mid-encode that re-runs one **idempotent**
  segment (deterministic path, L13) instead of a whole movie (resume, L70).
- **First bottleneck:** the fan-out floods a shared encoder fleet → lanes +
  autoscale + backpressure (L27/28).
- **Walls:** storing every variant wastes ~300 PB on a cold tail nobody watches
  (L2/39); the package barrier lets one **straggler** block a rendition →
  speculative execution on idempotent segments (L29/64/13); the codec ladder is a
  portfolio cost matched to the real audience (L14/59).
- **Deepest point:** transcoding is one big number paid in three currencies —
  disk, CPU, or the viewer's patience — and segmentation is what lets you choose
  which to spend, per video and per rung. Assembled from pieces already owned
  (L2/9/13/14/20/27/28/29/39/48/59/64/70).

## Trade named
Storage vs compute vs time-to-first-play.

## Sets up next
Lesson 0078 — **Multi-level & write-through vs write-back caching:** a cache
hierarchy (local in-process → Redis → CDN). On a write: write-through (safe,
slow) vs write-back (fast, loses data on crash) vs write-around, and cache
coherence across tiers (reusing L2/48 invalidation). Trade: read/write latency
vs durability & coherence. Remaining queued spine: 78–80 (multi-level caching,
resharding live, serialization formats) — add 3–5 new advanced topics when the
spine reaches its end.
