# Learning record — Lesson 0030: Secrets, Keys & Encryption

**File:** `lessons/0030-secrets-encryption.html`
**Topic (curriculum spine #30):** Secrets, keys & encryption at rest / in transit
**Trade named:** security blast-radius vs operational friction

## The one system
Lesson 29's warehouse: **~11 TB** of order/payment history (L25) + customer records,
denormalized across **~100,000 column-partition files**. It must be **encrypted at rest**
(stolen disk/backup = useless ciphertext) and reached only **encrypted in transit** (wiretap
sees nothing). Two operational facts make it hard: keys must be **rotated** (schedule +
on-compromise), and every read must **decrypt** (so key material has to be fast + reachable
by the serving path without being reachable by an attacker). Why is one key the wrong answer?

## What the lesson covers (four moves)
- **Estimate — what one key costs, two axes.** Blast radius: one key over all 100k files →
  a leak exposes **100%** of the vault (SPOF); envelope's leaked DEK = **1 file** (1/100,000).
  Rotation time: one-key rotation = decrypt+re-encrypt every byte = **22 TB moved @ 500 MB/s
  (L29) ≈ 12.2 h** of downtime; envelope re-wraps the DEKs only (**~10 MB**, data ciphertext
  unchanged, never moves) → seconds, **~2,000,000× less data touched**. Trade: a key an
  attacker can't reach is one you can't use fast → use a safe key to protect fast keys.
- **Model — the key hierarchy + KMS as root of trust.** Three layers: **root key** (in HSM,
  never leaves) → **KEK** (master, wraps DEKs only) → **DEK** (per file, AES-256-**GCM** =
  AEAD → confidentiality + tamper-detect via the auth tag; ECB/no-auth are traps). **KMS**
  holds KEKs, exposes only `wrap`/`unwrap`, NEVER returns the KEK plaintext → a compromised
  app can only unwrap what it's authorized for, scoped/revocable/audited; KEK never on a host
  = nothing to scrape. KMS = **L10 coordination point** in a security hat (powerful → must be
  HA, access-controlled, audited). In transit: **TLS** (ideally **mTLS**, terminate
  *everywhere* not just the edge — early termination = plaintext internal hops).
- **Trace — write + read.** Write: generate random DEK → AES-256-GCM encrypt file →
  `KMS.wrap(DEK)` → store `[ciphertext|tag|nonce|wrapped_DEK|key-id]` → **wipe** plaintext
  DEK. Read: fetch → `KMS.unwrap` (a network round trip) → decrypt (tag mismatch → REFUSE) →
  wipe. Disk holds only ciphertext + a locked envelope; `key-id` is what makes rotation a
  metadata re-wrap. The read's step 2 unwrap is the whole point AND the whole problem.
- **First bottleneck — KMS on every read + four traps.** Unwrap-per-read = **40,000 KMS
  calls/s** (L27 rate) = rate-limited throttle + latency tax + L10/L26 SPOF (KMS down →
  nothing decrypts). Can't remove the KMS (that re-exposes the KEK) → don't call it per op:
  **cache the unwrapped DEK** in node memory for a bounded **TTL** (5 min, ~2,000 hot DEKs →
  2,000 unwraps / 300 s vs 12M → **~6,000×** cut). TTL = **blast-radius-in-time** dial: longer
  = fewer KMS calls but wider memory-scrape window + slower revocation (L02 cache dial, but
  the cached thing is a key). Traps: (1) roll-your-own crypto / **GCM nonce reuse** (leaks
  the auth key — catastrophic); (2) encrypt without authenticating (bit-flip corrupts data,
  L25); (3) secrets in code/config/logs → secrets manager + short-lived creds → **secret-zero**
  bootstrap (bind to workload/instance identity, no static secret); (4) terminate TLS too
  early (internal hops in plaintext).

## Threads reused
- **L25** — payment-data stakes; integrity (a bit-flip on a ledger amount) motivates AEAD.
- **L29** — the ~11 TB vault; 500 MB/s scan rate reused to price the 12 h rotation.
- **L10** — coordination-service properties (single authority, HA, audited) applied to the KMS.
- **L27** — 40,000 req/s serving rate = the KMS-per-read load.
- **L26** — hot shared dependency / SPOF shape (KMS down → nothing decrypts).
- **L02** — the cache-TTL dial (here performance vs blast-radius rather than freshness).

## What it sets up next
**Lesson 0031 — Deployments & progressive rollout.** Having spent this lesson shrinking the
*data* blast radius, next shrink the *release* blast radius: a big-bang deploy that breaks
hits 100% of users at once. Blue-green vs canary vs rolling, health-gated promotion, feature
flags decoupling deploy from release, instant rollback. Trade shifts to **release velocity
vs blast radius.**

Spine now has only #31 (deployments) queued. After #31 is written the course will need a
fresh batch of advanced topics appended (candidates: distributed transactions/sagas, service
mesh, chaos engineering, config management, tenancy/isolation, cost/capacity FinOps).
