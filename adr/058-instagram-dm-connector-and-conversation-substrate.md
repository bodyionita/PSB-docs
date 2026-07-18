# ADR-058: Instagram DM connector — export-first ingestion, conversation substrate, sessionized stance-gated distillation

**Status:** Accepted · 2026-07-18 (grilled decision-by-decision) · **Defines** the M9.5 milestone
in [08-implementation-plan.md](../08-implementation-plan.md) · **Resolves** the
[ADR-009](009-instagram-connector-deferred.md) deferral (export path chosen; API spike scoped and
gated) · **Builds on** [ADR-029](029-conversational-ingestion-stance-gate-review-queue.md) (stance
gate), [ADR-048](048-m6-chat-distiller-build-decisions.md) (the distiller loop + captures-row
materialization this generalizes), [ADR-042](042-reprocess-all-from-raw-and-data-survival.md)
(P10), [ADR-056](056-temporal-correctness-date-tokens.md) (anchoring),
[ADR-057](057-multimodal-media-ingestion-substrate.md) (media substrate) · **Extends**
[05-connectors.md](../05-connectors.md) to v3.

## Context

The user's Instagram export (2.7GB) holds 588 DM threads / ~159k messages spanning 2016–2026,
with 1,203 photos, 545 videos, 1,552 voice notes and 4,222 link/reel shares referenced by local
`uri` next to clean per-message JSON (`sender_name`, `timestamp_ms`, `content`, reactions, edit
markers). Sender attribution is fully structural — no vision guessing. Two verified data hazards:
Meta's classic **mojibake** (UTF-8 double-encoded through latin-1; must be repaired at parse) and
**deleted accounts** (`instagramuser_…`, real name lost — including the single biggest thread at
28k messages). The Graph API cannot serve personal DM history *as such*, but the user's account
**is a professional (business) account**, which reopens the API question for the *ongoing* path.
Grilled 2026-07-18.

## Decisions

**1. Export-first: the official data export is the historic spine.** No Meta approval, lossless
metadata, full 10-year history. The API is only ever the *freshness* path (§10), never the
backfill.

**2. Local prep tool (`second-brain/tools/instagram-export/` in the code monorepo).** A
versioned, tested CLI that runs against the unzipped export on the user's machine:
- **Parse + repair:** message JSON → normalized per-message records; mojibake repaired
  (latin-1→utf-8 round-trip) before anything else sees the text; ADR-041 diacritic folding then
  applies downstream as usual.
- **Opt-in triage manifest (CSV — user call):** the tool inventories threads (resolved names,
  slug, msg count, date range, per-media counts, one-sided flag) sorted by message count
  descending; **everything defaults to `skip`**; the user whitelists (`ingest: yes`) and may set
  `name_override` / extra aliases per thread (the deleted-account fix). Auto-tiers: threads with
  **<5 messages and one-sided** are pre-binned skip; **`message_requests/` and group threads
  (participants > 2) are excluded outright — not even inventoried** (user call: Instagram DMs
  only; groups matter only for future WhatsApp/Facebook, last stage). The manifest holds real
  names → **local + gitignored, never committed**; docs use placeholders (docs repo is public).
- **Local video processing (ADR-057 §2):** ffmpeg keyframes + audio-track STT + vision on
  keyframes → summary text (+ optional thumbnail); only the summary crosses the wire.
- **Resumability (binding requirement, user call):** a local SQLite progress ledger tracks every
  unit (thread → message batch → media item) through states; every API call retries with backoff
  honoring `Retry-After`; **media derivations are cached locally** so a re-run never re-pays a
  call; re-running the tool is always safe and cheap.
- **Upload:** batched, authenticated, **idempotent** — the server upserts by
  `(source, thread_id, message_id)`, so re-uploading a half-done thread never duplicates. Photos
  + voice audio upload raw (kept); videos upload as summary text only.

**3. Server conversation substrate (source-generic).** New tables `connector_threads` /
`connector_messages` / `connector_media` (02-data-model), source-tagged (`instagram` now;
`whatsapp`/`slack` later reuse the same substrate). Messages are **connector raw**: they are the
durable server-side source both re-sessionization and re-distillation replay from. A future
source = a new producer (fetcher or import tool) into the same tables — 05's "one module +
config" promise, upgraded to the conversation shape.

**4. Sessionization is deterministic code, server-side (rule 12 spirit — no LLM segmentation).**
A thread's messages sort chronologically and split where the gap between consecutive messages
exceeds **6h** (config knob; re-runnable over stored messages since raw survives). Empirical
basis (measured on the three biggest named threads): p50 session ≈ 5–12 messages, p95 ≈ 44–99,
all-time worst ≈ 551 msgs ≈ 9k tokens — **every session in 10 years fits one LLM call with 10×
headroom**, so there is **no summary-chaining, ever** (the user's "no fake derived data"
requirement satisfied structurally). Pathology guard only: a session exceeding a token cap
(config, ~20k est.) hard-splits at its **largest internal gap**, recursively — each part a plain
session; no derived summary is ever fed back as pseudo-raw.

**5. Transcript rendering contract.** One session renders as a compact, LLM-legible transcript:
session header (date span, weekday, participants), `[HH:MM] Name: text` lines, `— Mar 16 —`
dividers on day change, reactions attached to the message they react to, media as inline
placeholders (`<photo: …description…>`, `<voice: …transcript…>`, `<video: …summary…>`,
`<shared reel from @owner: caption>`), the ADR-057 §5 attribution contract binding throughout.
Sender attribution is structural from JSON — never inferred. The user's own account name maps to
a first-person marker via the manifest so the distiller always knows which voice is "me".

**6. Distillation = the M6 loop, generalized (not forked, not inlined into the organizer).**
`ChatDistillerService`'s chat-specific seams (`ChatDistillStore`, message source) widen to a
**source-agnostic session source**; the loop is unchanged: per-session distill →
multi-candidate extraction → **stance gate (ADR-029, semantics untouched)** → auto-endorsed
candidates **materialize `captures` rows (`source=instagram`, `source_ref` = session)** → the
organizer (single writer, invariant 7); stance-unclear → review queue. The captures-row
materialization is exactly ADR-048's P10 trick: `reprocess-all` replays distilled memories with
zero connector-specific machinery, while **re-distillation** (from stored sessions) remains
separately possible. **Anchoring (forced by ADR-056):** materialized captures carry `anchor_at` =
the session's own time, so "tomorrow" said in 2019 resolves against 2019, never ingestion day.
**Rejected:** sessions-as-captures-directly (puts stance-gating inside the atomic-capture-shaped
organizer; conflates roles 05 separates) and an IG-specific distiller (duplicated prompt logic).

**7. Review-queue protection at backfill scale (three knobs, semantics unchanged).** A historic
campaign is thousands of sessions; even 10% stance-unclear would bury the queue. During backfill
runs: **(a)** stance-gate judgment unchanged; **(b)** a **salience floor** for *filing* review
items (config, default `medium`+ — lower-salience unclear candidates are logged skips);
**(c)** a **per-run review cap** (config ~200) — past it, further unclear items log-skip while
distillation continues. Skips are never silent (P8: run summary reports floor/cap counts) and
never loss (stored sessions ⇒ any skip is recoverable by re-distill). Auto-endorsed memories are
uncapped — they cost no attention.

**8. The backfill op: reprocess-all shape + parallel Claude distill.** Admin-triggered from the
web Ops surface, confirm-gated, **single-flight**, background `agent_runs` row with live
progress, chronological walk, per-session state ⇒ **kill/crash resumes losslessly**; runs
whenever triggered (not confined to the 03:00–05:00 window). **Model: Claude (user call —
prefers its reasoning; subscription flat-cost).** Throughput via **parallel distill workers**
(config `backfill_concurrency`, default 4, max 8 — each a CLI subprocess; the provider has no
lock; RAM on the 4GB box is the practical cap) while **organize stays effectively serial
(max-inflight 1–2)**: two concurrent organizes minting the same new person → split entities (the
M3 defect class; invariant 7 holds). On subscription-quota exhaustion the runner **pauses with
backoff and resumes** — absorbed by the resumability design, not an error. A per-run **model
override** option exists for the distill/organize legs (default = normal `conspect` routing) —
Settings routing is never flipped for a campaign.

**9. Targeted re-runs (user call — never full-sweep).** Two status-scoped admin ops:
**re-derive** walks only `connector_media.status = unavailable` (or explicit ids); **re-distill**
walks only degraded sessions (had `unavailable` placeholders, or floor/cap-skipped candidates) or
an explicit list. Idempotent via session-scoped candidate dedup keys — re-distilling never
duplicates endorsed memories, only adds what newly-available content yields.

**10. Ongoing freshness: API spike (business account), gated; periodic re-export is the standing
fallback.** ADR-009's blocker (business account) is met — the user's personal account *is*
professional, so `/me/conversations` plausibly *is* their DM inbox, and app-role users are exempt
from App Review for self-use. Unproven, so a **time-boxed spike** (throwaway local script, ~half
a day) must demonstrate: own-DM read, **history depth** (documented ~20-most-recent-per-
conversation limits ⇒ gap risk between polls), sent-message visibility (echo events), media CDN
URL access, token lifetime. **Preference order: webhooks first** (real-time push to the VPS,
Meta-signed callbacks), frequent polling second. **Gate:** spike passes → a daily-connector task
(a thin fetcher into the §3 substrate — the architecture is already API-ready); spike fails →
**periodic manual re-export** is the recorded ongoing path (upsert + watermarks make each
re-export a cheap delta, not a re-ingest).

**11. Traceability + curation surfaces (in-milestone).** Every distilled memory is traceable:
node → capture → `source_ref` → session; a **session-transcript endpoint + web view** renders
the full conversation with photos inline / voice playable (ADR-057 §7). The web **Review tab
gains kind-filter chips with per-kind counts** (stance floods must not bury entity/vocab/dedup
items). A **manual entity-merge UI** ships (the M3 `POST /admin/entities/merge` endpoint finally
gets its surface): two searchable entity pickers (needs a small alphabetical browse endpoint),
side-by-side previews (stacked on mobile) reusing `NodePreview`, survivor choice, confirm-gated
apply, feed-visible, registered as a standing merge (ADR-042 caveat machinery reports it).

**12. Backlogged, explicitly.** Per-person context injection into the distiller (the user's own
counter-argument prevailed: full-session units + the nightly agents build the cross-conversation
picture; injecting a mid-construction profile during backfill would make early- and late-thread
distills non-deterministic peers — and re-distill (§9) makes adding injection later cheap and
safe) · screenshot-conversation ingestion pipeline (ADR-057 §6) · group conversations (future
WhatsApp/Facebook stage).

## Consequences

- ✅ Ten years of DM history become stance-gated, entity-resolved, correctly-dated memories with
  every memory one tap from its source conversation.
- ✅ The conversation substrate + generalized distiller make WhatsApp/Slack "a new producer +
  config", with sessionization/distillation/review machinery shared.
- ✅ Every heavy stage (media derivation, upload, distill, organize) is resumable and
  targeted-re-runnable; a killed campaign continues where it stopped.
- ⚠️ The distiller generalization touches M6's chat path — its tests must stay green; the chat
  distiller becomes the first consumer of the widened seams.
- ⚠️ Backfill wall-clock is bounded by the Claude subscription quota (parallelism drains the
  window faster; the runner waits it out). The user accepted days-scale campaigns.
- ⚠️ Deleted-account threads depend on manual `name_override` for identity; without it they
  ingest under a placeholder participant.
