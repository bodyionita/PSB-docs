# Implementation Plan

**Version:** 2.4 · **Status:** Approved 2026-07-13 (2.4 = **M3 grilled to build-ready detail** —
chat pipeline (non-streaming + reveal, hybrid grounding, LLM query-condensation, cited-only `[n]`,
prompt-driven "not in notes", sessions) + the UI-editable per-group model routing / per-task effort
engine, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md); no migration. 2.3 = plan
split into a **task tracker** + a per-milestone [08-logs/](08-logs/); per-task implementation
narratives moved out. 2.2 = M2 grilled to build-ready detail —
[ADR-022](adr/022-embeddings-self-hosted-nomic.md)/[023](adr/023-semantic-relatedness-graph.md)/[024](adr/024-tag-vocabulary-reuse-and-consolidation.md);
M1 close folded into M2; 2.1 = M1 grilled to build-ready detail; v2 backlog additions)
**Rule:** ship in phases; every phase ends usable. A phase starts only when the previous
one's acceptance criteria pass. Code lives in `second-brain/` (monorepo, ADR-006).
**Process:** every session runs under [09-session-protocol.md](09-session-protocol.md)
(grill → record → pause; commit + push docs at pauses).

**How this file is organised.** Each milestone below carries its **scope · acceptance · build
decisions (the binding spec) · a task checklist** (`[x]` done / `[ ]` open, with the commit ref).
The verbose per-task **implementation narratives** — what was built, the independent review, the
end-to-end verification — live in **[08-logs/](08-logs/)**, one file per milestone
([m0](08-logs/m0.md) · [m1](08-logs/m1.md) · [m2](08-logs/m2.md)), linked from each task. This file
is the plan + status; the logs are the append-only record.

## M0 — Foundations

Monorepo skeleton (`server/`, `web/`, `deploy/`), CLAUDE.md from
[templates/CLAUDE.md](templates/CLAUDE.md). Server: config (pydantic-settings), asyncpg
pool, migration 001 (full schema from [02-data-model.md](02-data-model.md)), provider
registry with `openai` + `nebius` (OpenAI-compatible client) + `claude-max` (Agent SDK)
and the fallback chain, auth (login/session/logout + rate limit), `/health`.
Web: Vite+React+TS scaffold, design tokens/theme, auth screen, app shell + navigation
with the animation foundation (framer-motion, reduced-motion support). Deploy: Dockerfiles,
compose, Caddy config, provision.sh, GitHub Actions (lint/test/build + deploy on main).

**Accept:** provisioned VPS serves the PWA over HTTPS; login works; `/health` green;
a deliberate Claude-limit simulation makes the chain answer via Nebius and records it.

> **Amendment (2026-07-12, replan):** the last clause — the *live* Claude-limit → Nebius
> chain-and-record — **cannot be exercised in M0** and is **deferred to M3 (chat)**. M0 has
> **no chat/admin surface, no chain caller, and no `agent_runs` writer** (routers = `auth` +
> `health` only), so nothing invokes `.complete()` or records a run. For M0 this clause is
> satisfied by the **21 registry unit tests** (fallback chain with fakes); the live
> claude-max path + "records it" are verified in M3 when the chat endpoint exists. Consistent
> with [ADR-012](adr/012-m0-implementation-stack.md). **All other M0 accept clauses pass** (HTTPS
> PWA, login, `/health` green, TLS Full-strict).

**M0 build decisions (grilled 2026-07-12 — [ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md),
[ADR-012](adr/012-m0-implementation-stack.md)):** M0 is split into (a) a **local-first
build** — complete `server/`/`web/`/`deploy/`, verified to **boot end-to-end locally**
(dockerized `pgvector` dev DB, `alembic upgrade head`, `/health` green, login/session,
registry fallback unit-tested with fakes, web builds + shell clicks through) — and (b) a
later **provisioning session** for the live VPS/Cloudflare/Supabase/GitHub + `claude login`
that satisfies the remote half of the accept criteria. Stack: **uv**, Alembic + plain SQL
(no ORM), Argon2id, **pnpm**; `claude-max` implemented but health-guarded. Deploy artifacts
are written now but dormant until (b).

**Open decisions — RESOLVED 2026-07-12 (grilled):** TLS cert method → **Cloudflare Origin CA**
([ADR-017](adr/017-tls-cloudflare-origin-ca.md)); SSH hardening → non-root **`deploy`** user
([ADR-018](adr/018-vps-ssh-hardening-non-root-deploy-user.md)).

**Tasks** — full detail in [08-logs/m0.md](08-logs/m0.md).
- [x] Local-first build (server + web + deploy + CI, boots end-to-end locally) · [log](08-logs/m0.md#local-first-build-done-2026-07-12)
- [x] ADR-017/018 code (TLS Origin CA + non-root `deploy` SSH hardening) — `ae08d43` · [log](08-logs/m0.md#adr-017018-code-done-2026-07-12-pushed--originmain-at-ae08d43)
- [x] M0b provisioning (VPS/Cloudflare/Supabase/GitHub, deploy live at `braindan.cc`) · [log](08-logs/m0.md#m0b-provisioning-2026-07-12)
- [x] **M0 / M0b Accept COMPLETE** — HTTPS PWA, login, `/health` green, TLS Full-strict; chain-and-record deferred to M3 · [log](08-logs/m0.md#m0--m0b-accept-done-2026-07-12)

## M1 — Capture end-to-end (usable week 1)

Capture endpoints + pipeline (transcribe → organize/split per plane → write notes with
frontmatter contract → index stub) + vault git auto-backup **per [ADR-014](adr/014-vault-history-durability.md)**
(fast-forward-only push, gc/reflog pinned, atomic writes, `.trash` git-tracked, nightly
`git bundle --all` → R2 WORM, `/srv/data` + `pg_dump` → R2, weekly integrity drill;
monthly-CI restore + DR rehearsal are fast-follows). Web: capture screen (record
button + visualizer, text input, recent-captures strip with live status, retry).

**Accept:** voice memo from the phone becomes correctly plane-classified note(s) in the
vault < 30s, visible in GitHub history; organizer failure still yields an Inbox note; a
nightly WORM bundle lands in R2 and the weekly integrity drill passes the fingerprint check.

**M1 build decisions (grilled 2026-07-12 — [ADR-019](adr/019-conversational-capture-minimal-in-m1.md);
implements [ADR-014](adr/014-vault-history-durability.md), [ADR-005](adr/005-planes-and-atomic-notes.md)).**
Build-ready detail below; nothing here is left to implementer discretion.

- **Execution model.** API returns `202`; the pipeline runs **in-process** via a
  `CapturePipeline` service (`asyncio.create_task`), state in `captures.status`. No broker.
  **Boot-time sweep** marks orphaned in-flight captures (non-terminal, non-`failed`) as
  `failed` ("interrupted by restart") — retryable, nothing hangs silently (rule 7).
- **Pipeline order.** `transcribe` (voice only) → `organize` → `write` → `index` (stub) →
  **then** generate the follow-up nudge as a trailing, non-blocking step (notes land first,
  protecting <30s). **Never-lose:** the `captures` row (raw_text, or audio saved under
  `DATA_PATH` as `{id}.{ext}`) is persisted **before any model call**. Audio capped at 25 MB
  (whisper limit) → clear error if exceeded.
- **Transcription.** `registry.transcribe()` → `openai`/whisper, **no fallback**; STT-down ⇒
  capture `failed` + retry (accepted for M1). *(Replaced in the M1 replan by the Groq→OpenAI STT
  chain, [ADR-020](adr/020-stt-fallback-chain-groq-primary.md).)*
- **Organizer.** `registry.distill()` (cheap chain). Prompt-instructed JSON
  `{ notes:[{title,plane,planes[],tags[],body}] }`, robust parse (tolerate code fences). A
  **pure, unit-tested `validate_organizer_output`**: `plane` must be a configured plane else
  Inbox; `planes[]` filtered to valid planes and made a superset of `plane`; caps on note/tag
  counts. **Organic tagging** in the prompt — emotional tone + the what/why around feelings,
  free-form (no rigid taxonomy). **Inbox fallback** (chain exhausted / unparseable / zero
  valid notes): one note, `plane=Inbox`, title=first 8 words, body=full raw. Only infra
  failures (STT / vault-write) mark a capture `failed`; the organizer never does.
- **Note writing.** `<YYYY-MM-DD> <Sanitized Title>.md` in the plane folder; numeric-suffix
  collisions (` 2`, ` 3`); **atomic write** (temp + `os.replace`, ADR-014). Frontmatter per
  [02 §2](02-data-model.md) (`id`=capture id, local-TZ `created`, source, source_ref, plane,
  planes, tags, related). Siblings: `related:` frontmatter **+** a `## Related` section with
  `[[wikilinks]]`.
- **Conversational capture (minimal, [ADR-019](adr/019-conversational-capture-minimal-in-m1.md)).**
  Migration **002** adds `captures.follow_up_question` / `follow_up_answer` (nullable). One
  gentle nudge (≤20 words, versioned prompt constant) generated after a **successful** organize
  (none on the Inbox-fallback path). `POST /captures/{id}/follow-up {answer}` → **Pass 2**:
  re-organize original+answer, `git rm` the Pass-1 `note_paths` (soft-delete), write the
  enriched set, overwrite `note_paths` (**replace, not augment**). No server-side expiry.
- **Index stub = no-op.** `notes`/`chunks` stay empty until M2's real indexer/reindex; the
  step only transitions `written → indexed`. No frontmatter parser in M1. Keeps the supersede
  path pure filesystem+git.
- **Durability ([ADR-014](adr/014-vault-history-durability.md)).** A `VaultBackupService`
  owns **all** git ops behind one lock (file writes are concurrent+atomic; git staging/commit/
  push serialize): debounced ~60s commit + 04:55 sweep + `POST /admin/backup`; **ff-only push,
  heal-on-reject** (merge, never force/rebase/reset); gc/reflog pins set idempotently at
  startup (provision.sh doesn't). **Empty-repo bootstrap** creates the folder skeleton
  (`Inbox/`, `Summaries/Daily|Weekly/`, plane folders + `.gitkeep`) + initial commit +
  `push -u`.
- **Scheduler.** M1 introduces an **in-process APScheduler** (existing `enable_scheduler`
  flag; off by default, on in prod) running **only** the durability jobs; **M4 extends the same
  scheduler** with the 03:00–05:00 agent window. Jobs are also exposed via a **CLI entrypoint**
  so a future external scheduler can drive them without rework. **All four R2 jobs land in M1**
  (via boto3): nightly `git bundle --all` → WORM, weekly integrity drill, nightly `pg_dump` →
  R2, nightly `/srv/data` sync → R2. Each writes `agent_runs` (`vault-backup`,
  `integrity-drill`, `db-backup`, `data-sync`).
- **`/health` fourth leg.** Degrades if the latest `integrity-drill` `agent_run` failed or is
  overdue (>~8 days) — ADR-014 §6. Contract change recorded in [03-api](03-api.md).
- **Web (online-only).** Capture screen: record button + Web-Audio `AnalyserNode` visualizer,
  text input, optimistic confirm; recent-captures strip via new **`GET /captures?limit=20`**,
  TanStack Query **polling** (~2s while in-flight, stop when idle); pending nudge shown inline
  with an answer input. **Offline text queue stays M5, voice-offline stays v2.**
- **API contract additions** (recorded in [03-api](03-api.md)): `GET /captures?limit=` (list),
  `POST /captures/{id}/follow-up`, follow-up fields on `GET /captures/{id}`, `/health` fourth
  leg.

**M1 replan (grilled 2026-07-13 — recorded decisions).** The first live drive surfaced Accept-blockers
and prompted three decisions (full narrative in [08-logs/m1.md](08-logs/m1.md#m1-replan-2026-07-13)):
**(1)** STT fallback chain `["groq","openai"]`
([ADR-020](adr/020-stt-fallback-chain-groq-primary.md)); **(2)** one `agent_runs` row per capture +
`capture_interactions` view, migration 003
([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)); **(3)** global `CLAUDE_MAX_EFFORT`
(default `medium`) on every claude-max call. Plus three web must-fix and the vault-push SSH-wiring fix.

**M1 close POSTPONED to end of M2 (decided 2026-07-13).** M1's live Accept has a time-dependent tail —
a nightly WORM bundle lands in R2 and the weekly integrity drill passes — confirmable only after real
overnight/weekly cycles. Rather than block, **M1 stays open and closes at the end of M2**: the M1 Accept
confirmation (nightly bundle + drill green, any overnight-surfaced findings) folds into the M2 close.
The captured-voice-note→GitHub-history half is already satisfied (capture `a1e1e9b9` notes are in
`PSB-vault`); what remains is the scheduled-backup evidence.

**Tasks** — full detail in [08-logs/m1.md](08-logs/m1.md).
- [x] 1 — migration 002 + `CapturePipeline` (capture domain core) · [log](08-logs/m1.md#task-1)
- [x] 2 — capture routers + lifespan wiring — `8c65a0c` · [log](08-logs/m1.md#task-2)
- [x] 3 — durability: Slice A git `VaultBackupService` `884855f` · Slice B1 R2 jobs `7f3c4a7` · Slice B2 scheduler + `/health` leg `1e3c420` · [log](08-logs/m1.md#task-3--durability-three-slices)
- [x] 4 — web capture screen (06) · [log](08-logs/m1.md#web-capture-screen-06)
- [x] 5 — M1 replan (STT chain + `agent_runs` logging + `CLAUDE_MAX_EFFORT` + web 3 must-fix + vault SSH wiring) — server `d9b21e8`, web `4d988ea`+`26a1c09`, deploy `a56038f` · [log](08-logs/m1.md#m1-replan-2026-07-13)
- [x] 6 — polish batch (English-only vault, valid tags, git hygiene, reorganize) — `d469277` (shipped + deployed) · [log](08-logs/m1.md#polish-batch-2026-07-13)
- [x] 7 — **M1 live Accept** (scheduled-backup evidence: WORM bundle in R2 + integrity-drill verifies its fingerprint; `/health backups:true`) — **closed at the M2 Accept** (all four R2 jobs green after fixing `db-backup`/`data-sync`) · [log](08-logs/m2.md#task-9--live-m2-accept-in-progress-2026-07-13)

## M2 — Indexing & search

Chunking (pure, tested), indexer (hash skip, transactional upsert), full rescan +
`/admin/reindex`, `/search` with plane filters. Web: standalone search screen + admin controls.

**Accept:** reindex over a seeded vault; paraphrased query finds the right note; DB wipe +
reindex restores search; editing a note via git push is picked up by nightly pull+rescan.
**M2 also closes M1** (postponed 2026-07-13): fold in the M1 backup tail (a nightly WORM bundle +
the weekly integrity drill pass the fingerprint check) once the overnight/weekly cycles have run.

**M2 build decisions (grilled 2026-07-13 — [ADR-022](adr/022-embeddings-self-hosted-nomic.md),
[ADR-023](adr/023-semantic-relatedness-graph.md), [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)).**
Build-ready detail below; nothing left to implementer discretion. Facts confirmed against the M1
code: the registry already exposes `embed()`; `notes`/`chunks` exist (migration 001, currently
empty — the M1 index step is a no-op stub); chunking policy is in [02 §4](02-data-model.md).

- **Embeddings — self-hosted nomic ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)).**
  `nomic-embed-text-v1.5` via an **Ollama** sidecar (OpenAI-compatible; drop-in
  `OpenAICompatibleProvider`, base-URL only), **768-dim**, **single-provider, no hot fallback** (one
  index = one vector space). **Asymmetric prefixes mandatory**: `search_document:` when indexing,
  `search_query:` when searching. Nebius `Qwen3-Embedding` (multilingual) is the documented
  **cold-swap** (one config + reindex) if English-centric search disappoints; a same-model
  hosted-nomic **hot fallback is a noted future** (option B). Migration 004 resizes the empty
  `vector(1536)` columns → 768.
- **Indexer.** `file → strip frontmatter + `sb:related` block → sha256 (skip if unchanged) → parse
  frontmatter → chunk ([02 §4](02-data-model.md)) → batch-embed → **per-note transaction** (delete
  old chunks + upsert note + insert chunks) + `notes.embedding = mean-pool(chunk vectors)`. On embed
  failure: **skip-and-continue → run `partial`** (leave existing rows, retry re-indexes stale notes),
  429 backoff. Deletions reconciled (cascade); rename = new path. Fully idempotent.
- **`content_hash`** covers **frontmatter + body minus the `sb:related` block** — metadata edits
  reindex; the graph's own writes never re-trigger reindex (feedback-loop fix,
  [ADR-023](adr/023-semantic-relatedness-graph.md)).
- **Relatedness graph ([ADR-023](adr/023-semantic-relatedness-graph.md)) — full materialized graph.**
  Canonical **`note_links`** table (directional, `score`) **+** a rendered, delimited `sb:related`
  `## Related notes` wikilink block in each note body (Obsidian-visible), **distinct** from the
  co-capture `related:` field. Similarity = top-K over `notes.embedding` cosine above a floor;
  **`RELATED_TOP_K`=5**, **`RELATED_MIN_SCORE`=0.5**, both settings **tuned live** during Accept.
  **Recomputed nightly only** (+ on `/admin/reindex`); the real-time capture path never touches it.
  Churn-gated (rewrite a file only when its block changed).
- **Search.** `POST /search` → **note-grouped** results (best chunk = snippet), `top_k`=10,
  `planes` = `notes.planes` membership overlap ([ADR-005](adr/005-planes-and-atomic-notes.md)), no
  hard floor (`SEARCH_MIN_SCORE` off by default). `GET /notes/{id}` = read-only preview (body from
  the vault file + the note's `note_links` neighbours).
- **Reindex.** `POST /admin/reindex` **async**: `202 {run_id}`, `agent="reindex"` run
  (`details.trigger="manual"`, counts + `partial` in `details`), **single-flight** (409 if a
  reindex/rescan is running). The **one combined nightly `reindex` job** (03:40, existing scheduler)
  = `git pull` → rescan → recompute graph → render blocks → commit+push (one vault lock),
  `details.trigger="nightly"`.
- **Tags ([ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)).** Forward-only reuse: the
  live tag vocabulary (distinct `notes.tags`) is injected into the organizer (+ M4 distiller) prompt
  — *prefer existing, coin new only if none fits*. Existing cruft: a **manual** two-step
  `POST /admin/tags/consolidate` (propose → apply, reusing the reorganize vault-write path); **not**
  wired into the nightly job.
- **Web (online-only).** A standalone **Search tab** (query box, plane-filter chips, note cards with
  title/plane/snippet/tags/score, **read-only preview on expand** via `GET /notes/{id}`) + a
  lightweight **Admin tab** with buttons: **Reindex** (shows the run's live status/counts),
  **Backup now** (`/admin/backup`), **Consolidate tags** (Propose → review → Apply). Admin tab is
  movable later.
- **API/contract additions** ([03-api](03-api.md)): `POST /search` note-grouped, `GET /notes/{id}`,
  `POST /admin/reindex` async, `POST /admin/tags/consolidate`. Schema ([02](02-data-model.md)):
  migration 004 (`notes.embedding`, `note_links`, resize to 768). Infra ([07](07-infrastructure.md)):
  `ollama` sidecar.

**Tasks** — full detail in [08-logs/m2.md](08-logs/m2.md).
- [x] 1 — migration 004 + Ollama/nomic provider wiring — `c66b562` · [log](08-logs/m2.md#task-1)
- [x] 2 — pure text chunker — `fdd0f60` · [log](08-logs/m2.md#task-2)
- [x] 3 — indexer service (real index step replacing the M1 stub) — `684604e` · [log](08-logs/m2.md#task-3)
- [x] 4 — `POST /search` + `GET /notes/{id}` (note-grouped preview) — `6e0fa21` · [log](08-logs/m2.md#task-4)
- [x] 5 — relatedness graph recompute + `sb:related` render (ADR-023) — `73ed641` · [log](08-logs/m2.md#task-5)
- [x] 6 — combined nightly `reindex` job (`reindex_all` + `RelatednessGraph.recompute`, `git pull` first, one commit+push under the vault lock, single-flight) + `POST /admin/reindex` async wrapper — `a059c18`+`8cde827` · [log](08-logs/m2.md#task-6)
- [x] 7 — tag reuse in the organizer prompt + `POST /admin/tags/consolidate` (propose → apply) — `b404709` · [log](08-logs/m2.md#task-7)
- [x] 8 — web Search tab + Admin tab (+ two small server endpoints `GET /planes` and `GET /activity/runs/{id}`, 03-api v2.3) · [log](08-logs/m2.md#task-8)
- [x] 9 — **live M2 Accept** (paraphrased query finds the right note; DB-wipe + reindex restores search; git-push edit picked up) — **also closed the M1 backup tail**; surfaced + fixed 4 prod issues (model-pull → `ollama-init`; co-capture `## Related` embedding leak; `db-backup` pg_dump-missing; `data-sync` WORM lock) · [log](08-logs/m2.md#task-9--live-m2-accept-in-progress-2026-07-13)

## M3 — Chat + UI-editable model routing

Chat endpoints + retrieval + citations + sessions + per-conversation model picker +
fallback banner. Web: chat screen with reveal-animated render, source cards, model picker.
**Also lands** the UI-editable per-group model routing + per-task effort engine
([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)) and the Settings → Models panel.

**Accept:** questions over known vault content answered with correct [n] citations on both
Claude and Nebius; "not in your notes" behavior verified; sessions persist. **Plus the deferred M0
clause** (the live Claude-limit → Nebius chain-and-record) — satisfied by a Settings-driven
Nebius-primary drive on prod (the chain answers via Nebius, recorded in `chat_messages.model`) **+**
the existing 21 registry fallback unit tests for the mechanism; an optional one-shot "force Claude
down" exercises the literal live fallback.

**M3 build decisions (grilled 2026-07-13 — [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)).**
Build-ready detail below; nothing left to implementer discretion. Facts confirmed against the code:
the registry already does the `["claude-max","nebius"]` chat/distill fallback chain and returns
`model_used`/`fallback_used`; `SearchService.search(query, top_k, planes)` is the retrieval
primitive; `chat_sessions`/`chat_messages`/`app_settings` all exist (revision 001) so **M3 needs no
migration**; `claude --print` is **blocking** (no native token stream); effort is a per-call `--effort`
on the claude-max CLI, today a constructor constant.

- **Passive top-k retrieval (agentic stays v2).** `query → embed → SearchService note-grouped top_k
  (optional planes) → numbered context blocks → chat model → answer with [n] → persist`. Chat
  context uses a chat-specific `CHAT_CONTEXT_TOP_K` (default 8).
- **Transport = non-streaming.** `POST /chat` returns the full JSON body (answer + sources +
  model_used + fallback_used); the web animates a **client-side reveal**. Registry/providers
  untouched. True token streaming is a **post-v1** upgrade (would need a streaming provider interface
  + SSE + mid-stream persistence — out of scope).
- **Grounding = hybrid, biased to grounding.** Note-questions answered **only** from retrieved
  context + `[n]` citations, with an explicit "that's not in your notes" when the context is silent;
  clearly-general questions answered normally (no forced citation). System prompt biases toward the
  notes whenever the question could be about the user's memory. Reply in the user's language.
- **Multi-turn retrieval query = LLM condensation.** Before retrieval, the last N turns + the new
  message are condensed into one standalone search query via the **conspect** chain at **low effort**.
  **Skipped on turn 1** (nothing to resolve → embed the raw message); **degrades to embedding the raw
  message** if condensation's providers are all down (never a hard dependency).
- **Citations = cited-only, renumbered.** Parse `[n]` markers from the answer; `sources[]` = **only
  the notes actually cited**, renumbered `[1..m]` to match the rendered answer; uncited/invalid
  markers dropped; general/"not in notes" answers carry empty `sources[]`. Pure, unit-tested
  parse-and-renumber. Source = a note (best chunk snippet), matching note-grouped search.
- **"Not in your notes" = prompt-driven, no score floor.** Always retrieve, let the grounding-biased
  prompt decide (consistent with search's `SEARCH_MIN_SCORE=0`); no brittle cosine cutoff.
- **Sessions.** Implicit creation (`POST /chat` with no `session_id` creates + returns one). **User
  message persisted before the LLM call**, assistant message after (never-lose). History window =
  `CHAT_HISTORY_MAX_MESSAGES` (default **20**), oldest-trimmed for the answer prompt; full history
  stays in `chat_messages`. Title = **first-message truncation** (LLM-generated titles = fast-follow).
  Web = list / open / new only; **rename + delete deferred** (M5/polish, no endpoints in M3).
- **Fallback recording (deferred M0 clause).** `model_used` → `chat_messages.model`;
  `model_used`+`fallback_used` returned for the composer banner. **Chat stays out of `agent_runs`**
  (that table = autonomous agent/job work → the M4 feed; per-turn chat logging would flood it).
  Condensation's own model is internal plumbing, not separately recorded.
- **Model routing engine ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)).** Two
  UI-configurable groups — **Chat** (`chat_chain`) and **Conspect** (`distill_chain`) — each
  `{active, fallback, per-provider effort}`. **Per-group, per-provider effort** (Chat-Claude and
  Conspect-Claude independent; Nebius has no effort knob; default = global `CLAUDE_MAX_EFFORT`). A
  **`ModelRoutingService`** reads `app_settings` (env config as defaults, cache busted on save);
  **effort promoted to a per-call arg** through `registry.chat/distill` → `claude_max.complete(effort=…)`.
  A bad saved model id degrades to the config default chain (rule 7), never a hard failure.
- **Model picker vs Settings default.** Settings `chat` group = the default active/fallback/effort;
  the composer picker overrides just the **active** model per conversation (existing `model` param;
  the registry's `_resolve_chain` keeps the group's fallback + effort underneath).
- **API/contract additions** ([03-api](03-api.md)): `POST /chat` finalized (+ optional `planes`);
  `GET /chat/models` real registry ids + `CHAT_MODEL_LABELS`; `GET /chat/sessions` + `GET
  /chat/sessions/{id}`; enriched `GET /settings`; new `PUT /settings/models` (per group);
  `PUT /settings/agents` **superseded**. No schema change ([02](02-data-model.md) — routing lives in
  `app_settings` jsonb). New config: `CHAT_HISTORY_MAX_MESSAGES` (20), `CHAT_CONTEXT_TOP_K` (8),
  `CHAT_MODEL_LABELS` (id→label), chat condensation effort (low).
- **Web (online-only).** Chat screen per [06](06-web-app.md): conversation list + thread, composer
  with **model picker + plane-filter chips**, client-side **reveal animation**, expandable `[n]`
  **source cards** (title/plane badge/snippet), discreet **fallback banner** when `fallback_used`,
  "not in your notes" empty state. New **Settings → Models** panel: per-group active/fallback
  dropdowns + effort selector for effort-supporting models, saved via `PUT /settings/models`.

**Tasks** — full detail in 08-logs/m3.md (created at implementation).
- [ ] 1 — model routing engine: `ModelRoutingService` over `app_settings` (config defaults, cache-bust
      on save) + effort promoted to a per-call arg (`registry.chat/distill` → `claude_max.complete(effort=)`)
- [ ] 2 — chat service: retrieval (condensation → note-grouped top-k) + prompt (hybrid/grounded) +
      `[n]` cited-only parse-and-renumber + session/message persistence
- [ ] 3 — chat routers: `POST /chat`, `GET /chat/models`, `GET /chat/sessions[/{id}]` (+ `planes`)
- [ ] 4 — settings routers: enriched `GET /settings` + `PUT /settings/models` (supersede `PUT /settings/agents`)
- [ ] 5 — web chat screen (06): list/thread, composer (picker + plane chips), reveal, source cards, banner
- [ ] 6 — web Settings → Models panel (per-group active/fallback/effort)
- [ ] 7 — **live M3 Accept** (both-model cited answers; "not in notes"; sessions persist; deferred M0
      clause: Settings-driven Nebius-primary drive recorded + fallback unit tests)

## M4 — Slack agent + activity feed

Connector contract + Slack connector (fetch/normalize) + shared distiller + cursors +
`agent_runs` + scheduler with the 03:00–05:00 staggered window + `/activity` +
`/agents/{name}/run`. Web: activity feed with expandable run details, agents in Settings
(model chain config + run-now).

**Accept:** nightly run distills yesterday's Slack into plane-correct atomic linked notes;
feed shows the run (model, fallback, notes); rerun after forced failure resumes from
cursor without duplicates.

## M5 — Background intelligence

Daily summary + weekly review jobs (per-plane sections), summaries endpoints, PWA polish
(manifest, offline shell, offline text-capture queue), Settings completion.

**Accept:** morning after a captured day: daily summary exists in vault + feed, and is
retrievable via chat; weekly review lands Sunday; reruns overwrite.

## v2 backlog (do not build in v1)

Instagram spike (ADR-009) · more connectors (WhatsApp, email, calendar) · note editing in
web · hybrid keyword+vector search ·
Cloudflare Access second wall · voice offline queue · entity extraction ·
**conversational capture — full version** (adaptive *multi-turn* interviewer/therapist nudges
during ingestion — a minimal one-nudge form ships in M1 per [ADR-019](adr/019-conversational-capture-minimal-in-m1.md);
v2 is the multi-turn, gentle-not-strict experience) ·
**share ChatGPT/Claude conversations as an ingestion source** (slots into the existing
`source`/`source_ref` contract; a shared-conversation URL becomes `source_ref`) ·
**undo a manual ingestion** (soft-delete a capture's notes via `git rm` — history kept —
using `captures.note_paths[]`; builds on the M1 supersede path from ADR-019) ·
**demo/presentation layer** (a separate, limited access point showing a curated/redacted
view of the brain with wow-factor, for showing other people) ·
**explorable memory-graph visualization** (clean, navigable graph of notes/links — useful to
the user, not just decorative; extends related-notes suggestions & graph features) ·
**multi-tenant deployment** (far horizon — a handful of users, not mass scale; keep M1+
architecture from actively precluding it, e.g. jobs invokable via CLI/external scheduler) ·
backup fast-follows (monthly CI restore drill, semi-annual DR rehearsal — [ADR-014](adr/014-vault-history-durability.md)).

**Priority v2 candidates (agreed 2026-07-12):**
- **Agentic retrieval over the vault** — instead of passive top-k context, the chat model
  gets tools (`search`, `read_note`, `list_by_plane`, `follow_links`) and navigates the
  vault itself (Claude-Code-style). The server already sits next to both the files and
  the index; this is a chat-pipeline upgrade, not an architecture change.
- **MCP server layer** — expose the same services (search/read/capture) as an MCP server
  on the VPS (token-authenticated), so Claude and other LLM clients can talk to the vault
  directly from any conversation, without opening the web app. REST and MCP stay thin
  interfaces over shared services — no logic duplication.

## Testing policy

Pure logic (chunking, frontmatter, filename sanitization, cursor math) → unit tests, no
mocks. Services → fake providers + tmp vault + test DB schema. Connectors → recorded
fixture payloads (no live Slack in tests). No live LLM calls in CI; each milestone has a
manual smoke script documented in the code repo.
