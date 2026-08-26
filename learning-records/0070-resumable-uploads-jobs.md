# Lesson 0070 — Idempotent, Resumable Data APIs (Uploads & Long Jobs)

**File:** `lessons/0070-resumable-uploads-jobs.html`
**Spine topic:** #70 (the "read fan-out → surviving a bad network" arc, after L69)
**Date:** 2026-08-26

## Worked example
A phone on a flaky mobile network (disconnects ~every 3 min) does two hard things:
- **Upload** a **2 GB** dashcam video to object storage.
- **Long job**: "export my order history to CSV" that takes the server **~4 min**.

The naïve design (one big `POST` for the file, one blocking `POST` for the job) fails: the file never finishes (restarts from byte 0 on every drop) and the job times out → the client retries → the expensive export runs twice.

## What it covered
- **Estimate — the two failures as arithmetic.** Upload: 2048 MB ÷ 0.625 MB/s (5 Mbps) ≈ **3,277 s ≈ 55 min**; with drops ~every 180 s it crosses ~18 disconnect windows → P(survive) ≈ e^-18 ≈ 1-in-80M → effectively never. Job: ~4 min work vs ~60 s request timeout → guaranteed timeout → retry → duplicate work / double-charge (L13).
- **Pattern A — resumable chunked upload.** create session (upload_id = resume token, L18) → PUT 256 × 8 MB chunks, each **idempotent by index** (absolute PUT, L13) + **checksummed** (L20) → resume from **server-confirmed** received parts → **commit-last** makes the object visible only when all parts present + verified (L20/25). Drop wastes ≤1 chunk (~4 MB) vs ~1 GB → **256×** less waste; a 12.8 s chunk survives ~93% of the time → monotonic progress.
- **Pattern B — async job + status polling.** POST returns **202 Accepted** + client-chosen **idempotency key** (re-submit returns the SAME job, L13) → background worker (L5/9) → client **polls** `/jobs/{id}` with **exponential backoff** (L7) or gets **pushed** (webhook L45 / WebSocket L26) → download result via CDN (L14). Polling dial = frequency vs freshness vs load (L2 dial reborn).
- **Trace.** (A) upload survives a tunnel: `GET` session → resume at N+1, waste 1 chunk. (B) chunk 77 arrives but ACK lost → just re-PUT, no-op (never solve "did it land?"). (C) job survives client crash: re-submit key K-3 → server returns existing job J-5, not a 2nd 4-min run.
- **First bottleneck — resumable means the server holds in-flight STATE.** Orphaned abandoned sessions (~10% of 100k/day × ~1 GB ≈ **10 TB/day**) → **TTL** sweep (L8/13/48), two-sided dial (too short deletes a resumable upload, too long hoards orphans). State must live in a **shared store** (any server serves the resume, L26) and stay consistent with the bytes (record-received AFTER durable, L33 dual-write; re-verify at commit). Chunk-size trade (small = less waste + more overhead; big = fewer requests + more waste; 8 MB sweet spot). Job must itself resume across worker crashes (visibility timeout L5/9 + idempotent/checkpointed work, temp-write + atomic-rename = commit-last again).
- **Deepest point.** On an unreliable network you can't make a step succeed, so you make **repeating it free**: shrink the unit of work until a failure is cheap, give every unit a stable identity until a retry is safe. = L13 idempotency + L20/25 commit-last + L26 truth-off-connection + L18 cursor, assembled into one protocol.

## Numbers used (all checked)
- 5 Mbps = 0.625 MB/s; 2048 ÷ 0.625 ≈ 3,277 s ≈ 54.6 min.
- 3,277 ÷ 180 ≈ 18.2 disconnect windows; e^-18.2 ≈ 1.2e-8.
- 2048 ÷ 8 = **256 chunks**; 8 ÷ 0.625 = 12.8 s/chunk; e^-(12.8/180) ≈ 0.931.
- Waste per drop: naïve ~1024 MB vs chunked ~4 MB → ratio **256×**.
- Orphans: 100k uploads/day × 10% × 1 GB = **10 TB/day**.

## Threads reused
L13 (idempotency key; exactly-once effect; double-charge), L18 (cursor/resume token; API contract), L20 (chunking a blob; metadata plane; commit-last), L25 (atomic commit), L5 (producer→queue→worker; job status), L9 (durable log; visibility timeout → crashed-worker retry), L26 (truth off the disposable connection → shared state store), L33 (dual-write trap between bytes and the "received" record), L7 (timeouts, backoff, thread-pool exhaustion), L14 (serve result via CDN), L45 (webhook push vs polling), L2 (the cache dial → polling frequency dial), L8/48 (TTL).

## Sets up next
- **Lesson 71 — Quorum reads/writes & tunable consistency:** the `R + W > N` dial from the inside (L06/11/63) — pick R, W, N per call to slide between fast-but-stale and slow-but-fresh; read-repair and hinted handoff on the read path; why W=N kills availability. Trade: per-request consistency vs latency & availability.
- Then Lesson 72 — bulk & batch APIs (the N+1 / fan-out problem, L18).
- Spine still has 71–75 queued, so no new topics added this run.
