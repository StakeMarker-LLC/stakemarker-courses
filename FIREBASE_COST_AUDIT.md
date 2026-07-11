# Firebase Blaze Cost Audit — Data Egress / Ingress / Writes

**Date:** 2026-07-11 · **Scope:** `stakemarker-llc/stakemarker-courses` (read-only; no data files were modified)

> **Update (same day):** Owner clarified the architecture: the site is served
> from **Cloudflare** (not GitHub Pages / Firebase Hosting), iOS/Android cache
> all HTML/CSS until out of date, and Firebase handles **only auth and
> round syncing**. That confirms distribution of this dataset costs Firebase
> $0 — Findings 3/4 below are Cloudflare-bandwidth and client-bandwidth
> concerns only, not Blaze line items. Finding 1 (no-op version churn) still
> matters: it defeats the mobile HTML/CSS/data cache and forces spurious
> re-downloads. The remaining Blaze cost surface is **Firestore/RTDB
> operations for live rounds + sync and Auth**, which live in the app repo
> (out of this session's scope) — see the companion notes in the session for
> the live-round audit checklist.

This repo holds the derived golf-course dataset consumed by the StakeMarker app.
The app / functions repo was not in session scope, so client-side consumption is
inferred from what this repo exposes; each finding lists what to verify there.
Prices cited are published US/nam5 Firebase rates and may drift.

## Measured facts

| Metric | Value |
|---|---|
| `courses.json` raw | 2,202,985 B (2.10 MiB) |
| `courses.json` gzip -9 | 578,795 B (3.8× smaller) |
| Records | 9,459 (all with complete 18-hole `pars`/`hcps`) |
| Avg record (minified) | ~232 B |
| `courses-meta.json` | 63 B |
| Columnar re-encoding | 1,607,144 B raw / 458,699 B gzip (−21 % over the wire) |
| Per-state shards (51) | median 23 KB, max 181 KB raw |
| `source` values | `opengolf` × 9,454, `stakemarker` × 5 |
| Sync commits | authored by `github-actions[bot]`; **no workflow files live in this repo** — the sync runs in an external repo |
| Last sync | 2026-06-01 — 40 days before this audit |

Byte share of the minified payload by field:
`hcps` 23.2 % · `pars` 19.3 % · `id` 18.9 % · `name` 15.3 % · `source` 8.6 % · `city` 8.3 % · `state` 5.6 %.

## Findings

### 1. Confirmed: no-op version churn forces spurious full re-downloads

Commit `fd56feb` (2026-06-01) bumped the `version` field in `courses-meta.json`
(2026-05-26 → 2026-06-01) **without changing `courses.json` at all**. Any client
or seeding job that keys its cache on `version` treated this as new data:

- every client re-downloads the full 2.2 MB payload for zero change;
- if a job re-seeds a Firestore mirror on version change, that is up to 9,459
  billable writes per no-op bump, plus snapshot-listener traffic on connected
  clients.

**Fix (in the external sync workflow):** bump `version` only when `courses.json`
bytes actually change — or better, make `version` a content hash of
`courses.json` so cache-busting is intrinsically correct. This is the one
defect in this repo's pipeline that directly multiplies egress.

### 2. Sync has been silent for 40 days — verify intent

Cadence was: 4 runs on 2026-05-23 (initial incremental build 383 → 1,537),
then 05-25, 05-26, 06-01 — then nothing through 2026-07-11. Either the
scheduled workflow was disabled/failing (a staleness problem, and the README's
"rebuilt incrementally" claim is currently false in practice), or it was
changed to skip no-op commits (cost-positive). Check the workflow run history
in the repo that hosts the sync job.

### 3. Distribution channel is the dominant cost lever — keep it off Firebase

This dataset being a public GitHub repo is itself the biggest cost optimization
available: `raw.githubusercontent.com` or jsDelivr (`cdn.jsdelivr.net/gh/...`)
serve it with ETag/compression at **$0 Firebase cost**. Comparison per
distribution channel, using measured sizes:

| Channel | Cost profile |
|---|---|
| GitHub raw / jsDelivr | $0. jsDelivr adds brotli + immutable version-pinned URLs. |
| Firebase Hosting | $0.15/GB after 10 GB/mo free. Auto-gzip → ~579 KB/download ⇒ ~$0.09 per 1,000 downloads. Acceptable, not free. |
| Cloud Storage | ~$0.12/GB egress and **no automatic gzip** — a naive upload serves 2.2 MB/download (~$0.26 per 1,000). Avoid, or store pre-gzipped with `Content-Encoding: gzip`. |
| Cloud Function serving the file | Worst: invocations + compute + $0.12/GB outbound. Never serve a static 2 MB file through a function. |
| Firestore, one doc per course | Worst read pattern: a full-collection fetch is 9,459 reads ≈ $0.0057 per client refresh ($0.06/100k, nam5). 1,000 clients refreshing daily ≈ **$170/mo**, vs $0 on the GitHub CDN. The 50k/day free reads absorb only ~5 such refreshes. |

**Verify in the app repo:** where clients actually fetch course data from. If
anything other than the GitHub CDN path, migrating the bulk dataset fetch there
is the single highest-value change. Client flow should be: fetch the 63 B
`courses-meta.json` first (or use `If-None-Match`), download `courses.json`
only on version change.

### 4. Payload reductions (secondary lever)

Ranked by benefit vs. risk; these cut client bandwidth everywhere and egress
cost wherever a Firebase channel is still in the path:

1. **Ship gzip/brotli end to end** — 2.20 MB → 579 KB (gzip). Free if served
   via jsDelivr/Hosting; on Cloud Storage it must be done explicitly.
2. **Drop the `source` field** from the published file (−189 KB raw, 8.6 %).
   It is `opengolf` for 9,454 of 9,459 records; the 5 `stakemarker` ids can
   be listed in `courses-meta.json` if provenance must survive.
3. **Columnar layout** (`{ids:[…], names:[…], pars:[[…]], …}`): 1.61 MB raw /
   459 KB gzip, −21 % over the wire vs. today. Requires a client-side decode
   shim — bundle it with a format-version field.
4. **Per-state sharding** (51 files, median 23 KB): >90 % egress cut per client
   if users only need nearby courses. Only worth the complexity if a *billed*
   channel remains in the path; pointless on the free GitHub CDN.
5. **Do not** shorten or regenerate `id` values (18.9 % of payload). They are
   almost certainly referenced by user rounds in Firestore; churning them
   breaks references and forces full re-downloads — the savings are not worth it.

### 5. Ingest side is effectively free — keep it that way

Sync commits are authored by `github-actions[bot]`, so ingestion (fetching
OpenGolfAPI, diffing, pushing here) runs on GitHub Actions, not Firebase.
Inbound traffic to Firebase is free regardless. **Anti-recommendation:** do
not move this sync into Cloud Functions / Cloud Scheduler — it would add
invocations, compute, and outbound-networking charges for zero benefit. The
incremental `updated_at`-based re-fetch described in the README is the right
design; Finding 2 is about whether it is still running, not its shape.

### 6. If a Firestore mirror of this dataset exists

A blind full re-seed is 9,459 writes ≈ $0.017 (trivial), but writes to
unchanged documents still bill and still generate listener/sync traffic for
connected clients. If the app mirrors courses into Firestore at all:

- write only actually-changed documents (per-doc content hash comparison);
- prefer no mirror: serve the bulk file from the GitHub CDN and keep Firestore
  for user data (rounds, profiles), where per-document access is the point;
- if per-course lookup inside Firestore is genuinely needed, a Firestore
  **data bundle** served via CDN gives query semantics without per-client reads.

### 7. Cloud Functions invocation costs (added on request)

Billing meters that apply to Firebase Functions (2nd gen runs on Cloud Run;
published US pricing, free tier per month in parentheses):

| Meter | Rate | Free tier |
|---|---|---|
| Invocations / requests | $0.40 per million | 2M |
| Compute — CPU | ~$0.000024 per vCPU-second | 180k vCPU-s |
| Compute — memory | ~$0.0000025 per GiB-second | 360k GiB-s |
| Outbound networking | $0.12/GB | 5 GB |
| `minInstances` idle time | billed continuously; ≈ $6–12/mo per always-warm instance | none |
| Cloud Scheduler | $0.10 per job/month | 3 jobs |
| Cloud Logging | $0.50/GiB ingested | 50 GiB |
| Artifact Registry (deploy images) | ~$0.10/GB-month | 0.5 GB |

**Scale check:** invocations alone are effectively free at golf-app scale.
A per-score-write trigger for a foursome is ~150 invocations/round; even
100 rounds/day ≈ 450k invocations/mo — inside the free tier, and ~$0.18/mo
beyond it. If the Functions line item on the bill is non-trivial, it is a
*pattern* problem, not organic load. The patterns that actually move it:

1. **`minInstances` > 0** — a single always-warm instance costs more per
   month than millions of invocations. Verify every function that sets it.
2. **Cascading / recursive triggers** — an `onDocumentWritten` handler that
   writes back to a document matching its own (or another function's)
   trigger path. This is the classic runaway-bill bug; it also multiplies
   Firestore writes and every client listener's reads.
3. **Retry storms** — event triggers deployed with retries enabled will loop
   indefinitely on a permanently-failing event (bad data, thrown exception).
4. **Fan-out writes inside triggers** — a leaderboard recompute that writes N
   docs per score turns 1 write into N+1 writes plus N×listeners reads.
5. **Slow/overweight functions** — compute is billed in vCPU/GiB-seconds, so
   a 2 s, 1 GiB handler costs ~20× a 200 ms, 256 MiB one. Also check the
   function region matches the Firestore region (cross-region = egress).
6. **Callable functions used as a read proxy** — routing reads through a
   function adds invocation + compute on top of the same Firestore read the
   client could do directly under security rules.
7. **Per-invocation logging** — chatty `console.log` at high volume can
   exceed the 50 GiB logging free tier quietly.

**Audit checklist for the app/functions repo:** enumerate every deployed
function and for each record: trigger type & path, retry setting,
`minInstances`/`memory`/`timeout`, what it writes (and whether any write can
re-enter a trigger path), external calls, and log volume per invocation.
Cross-reference against the Usage dashboard's invocation counts.

### 8. Correction: live rounds run on Realtime Database, not Firestore

Owner confirmed the app syncs via **RTDB**. RTDB bills only two meters —
**storage ($5/GB-month)** and **download ($5/GB, including SSL/protocol
overhead)** — and on Blaze there is **no free RTDB allowance**: billing
starts at the first byte. There are no per-operation charges, so the
Firestore read/write math in §3 and §6 does not apply to round sync.
What this changes:

- **Writes are free.** High-frequency writes (presence, GPS pings) cost
  nothing directly — their cost is the download they trigger on every
  *other* device listening to that path, plus per-message protocol overhead.
- **Listener scope is the entire cost model.** Attaching a listener
  downloads the whole node once, then streams deltas for any change under
  it. The dominant RTDB anti-pattern is listening high in the tree
  (`/games`, `/rounds`, `/users`) so every device downloads the initial
  subtree and then receives every other user's changes. Listeners must sit
  on the narrowest node that matters (`/rounds/{roundId}`, ideally split
  hot child paths like `/rounds/{id}/scores`).
- **Reconnect churn matters.** Phones on a golf course drop connections
  constantly; each reconnect re-establishes listeners against the listened
  node. Keep listened nodes small and enable SDK disk persistence so
  unchanged data isn't re-shipped. Audit any `keepSynced(true)` — it holds
  broad paths in permanent sync.
- **Fat round nodes multiply everything.** If the round node embeds the
  course scorecard, player profiles, or chat alongside live scores, every
  initial attach and many deltas ship those bytes too. Course/static data
  belongs on Cloudflare (already free), never in RTDB.
- **Storage is likely negligible** (rounds are small JSON) *unless* GPS
  tracks or history accumulate unbounded — at $5/GB-month forever, archive
  or prune old rounds if they grow.

**Ballpark:** a foursome's live round with narrow listeners is roughly
0.5–2 MB total wire traffic across all four devices ⇒ $0.0025–0.01 per
round; 100 rounds/day ≈ **$7.50–30/month**. A download line item well above
that scale indicates a broad listener, a fat node, or static data being
served out of RTDB.

**Diagnostic:** run `firebase database:profile` against production for a few
minutes during live play — it breaks down downloaded bytes by path and
immediately identifies which node is the spender. RTDB-triggered Cloud
Functions should live in the RTDB instance's region (classic instances:
`us-central1`) to avoid cross-region egress.

## Priority summary

| # | Action | Where | Impact |
|---|---|---|---|
| 1 | Confirm clients fetch `courses.json` from GitHub raw/jsDelivr, not a Firebase surface | app repo | Largest recurring egress/read cost, up to ~$170/mo per 1k daily full-refresh clients if on Firestore |
| 2 | Stop no-op `version` bumps (hash-based version) | sync workflow repo | Eliminates spurious 2.2 MB re-downloads fleet-wide |
| 3 | Meta-first / ETag conditional fetch on clients | app repo | Caps steady-state client traffic at 63 B per check |
| 4 | Drop `source`, consider columnar format | sync workflow repo | −8.6 % raw now; −21 % wire with columnar |
| 5 | Verify sync schedule is alive | sync workflow repo | Correctness (staleness), not cost |
| 6 | Keep sync on GitHub Actions; avoid Functions/Storage serving | both | Prevents new billable meters |
