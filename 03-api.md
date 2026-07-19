# API Contract

**Version:** 3.2 · **Status:** Approved 2026-07-13 (3.2 = prior-art adoptions
[ADR-032](adr/032-prior-art-adoptions.md): search RRF hybrid + temporal filters; MCP traverse
pagination + `build_context`. 3.1 = **M3 grilled**
([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[031](adr/031-m3-organizer-and-contract-extensions.md)):
review queue → M3 kind-generic, `POST /admin/entities/merge`, `GET /nodes/{id}` gains
aliases/occurred/`profile`/tombstone-resolve. 3.0 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–[029](adr/029-conversational-ingestion-stance-gate-review-queue.md):
full rename (notes→**nodes**, vault→**graph store**) landing with M3; graph/traverse endpoints; the
MCP peer surface (M5); review queue (M6); ops/observability endpoints (M8). Endpoints marked with
their milestone; unmarked = live from M0–M2 (renames apply at M3). 2.x history in git.)

This HTTP API is the **only** seam between `web/` and `server/`
([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)). The server publishes OpenAPI
at `/api/openapi.json`. **The REST API and the MCP server are thin peers over the same service
layer** ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)) — MCP tool list at the end.

- Base path: `/api/v1` (Caddy serves the PWA at `/`, proxies `/api`).
- Auth: session cookie (httpOnly, Secure, SameSite=Lax) via `/auth/login`
  ([ADR-007](adr/007-auth-password-session-cloudflare.md)); all endpoints require it except
  `/auth/login` and `/health`. **MCP (M5): OAuth 2.1** ([ADR-046](adr/046-m5-mcp-server-oauth-connectors.md)) at root-level routes (`/mcp` + `/.well-known/oauth-*`, `/authorize`, `/token`, `/register`), separate from the web session and independently revocable — see the MCP section below.
- Errors: `{"detail": "..."}` + proper status. 401 on missing/expired session.

## Auth

| | |
|---|---|
| `POST /auth/login` | `{ "password": "…" }` → sets session cookie. Rate-limited (5/min/IP). |
| `POST /auth/logout` | revokes current session |
| `GET /auth/me` | `{ "authenticated": true, "session_created_at": … }` |

## Capture

> **M9.6 ([ADR-061](adr/061-composite-multi-part-capture.md)) reshaped this surface — SHIPPED.** The
> three one-shot `POST /capture/{text,voice,image}` endpoints are **removed** (§8); every web capture
> goes through the server-side **draft** flow below — one capture carries a typed text body + 0..N
> photos + ≤1 voice, organized in **one blended pass** at submit. MCP `capture` + chat-distiller
> ingest are **unchanged** (they create committed captures directly through the same organizer).

**Composite draft lifecycle** ([ADR-061](adr/061-composite-multi-part-capture.md) §3 — one active draft):

| | |
|---|---|
| `POST /capture/draft` | open-or-**resume** the single active draft → `DraftView { capture_id, status:"draft", text_body, parts:[{ id, kind, status, part_ordinal, mime_type }], created_at }`. Idempotent (resumes the existing draft; a racing concurrent open resolves to the winner) |
| `POST /capture/{id}/part` | multipart `kind` (`photo`\|`voice`) + `file` → `DraftPart`. Raw persists immediately; **derivation is deferred to submit**. `≤1 voice` enforced (`409`); `400` bad type/size; `409` not an open draft; `404` unknown |
| `DELETE /capture/{id}/part/{mediaId}` | remove a draft part (the 'x') — hard-removes raw + row → `204`; `409` not a draft; `404` unknown |
| `PUT /capture/{id}/text` | `{ "text" }` edit the draft's text body (empty allowed) → the fresh `DraftView`; `409` not a draft; `404` unknown |
| `POST /capture/{id}/submit` | commit the draft → `202 { capture_id }`; flips `draft→received`, spawns the blended organize. `400` if no non-empty part (`≥1` required); `409` not an open draft (idempotent) |
| `DELETE /capture/{id}/draft` | discard the open draft (raw files + rows) → `204`; `409` not a draft |

**Capture read:**

| | |
|---|---|
| `GET /captures?limit=20` | recent captures, newest first: `[{ capture_id, kind, status, raw_text, text_body, node_paths[], node_refs[], source, media[], run_id, follow_up_question, follow_up_answer, error, created_at, updated_at }]`. **M9.6 T4:** `kind` gains **`composite`**; `media` is a **list** `[{ id, kind, status, part_ordinal }]` (0..N photos + ≤1 voice, part-ordinal order — was a single item); `text_body` = the person's typed words; `run_id` = the capture's most recent processing `agent_runs` id (the **Activity-tab deep-link**, live during processing — [ADR-061](adr/061-composite-multi-part-capture.md) §10). (`node_refs`/`source` M8.1 T4; `kind` `image` + `status` `deriving` M9 T3; voice media M9 T4 — see the addendum.) |
| `GET /captures/{id}` | pipeline state (same shape as above) |
| `POST /captures/{id}/retry` | re-run from first incomplete step; `409` unless `failed` |
| `DELETE /captures/{id}` | **(M9.7)** general capture remove — git-rm content nodes + DB-delete (cascades `chunks`/`edges`/`node_media`), **purge** the capture's `media` rows + raw files, **tombstone** the capture (`removed_at`; replay-excluded, kept for audit) → `204`. Entity hubs preserved ([ADR-038](adr/038-reorganize-preserves-shared-entity-hubs.md)); removed captures vanish from `GET /captures`/search/chat/audit. `409` open draft (use Discard); `404` unknown/already-removed. Idempotent, tombstone-last ordering (the [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §11 core generalized; client UX must double-confirm) |
| `POST /captures/{id}/follow-up` | `{ "answer" }` → Pass-2 re-organize, replaces the capture's nodes ([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)); `202`; `409` if no nudge pending |
| `PUT /captures/{id}/anchor` | **(M8.2, [ADR-056](adr/056-temporal-correctness-date-tokens.md) §5 — the anchor edit)** `{ "anchor": iso-datetime }` corrects the capture's recorded-at, then triggers a **background one-capture reorganize** re-resolving every relative date against the new anchor; `202`; `404` unknown. The stored anchor is data (never wall-clock), so the replay is deterministic |

## Connectors & media (M9/M9.5 — [ADR-057](adr/057-multimodal-media-ingestion-substrate.md)/[058](adr/058-instagram-dm-connector-and-conversation-substrate.md); shapes draft-level, re-checked at task kickoff)

| | |
|---|---|
| `GET /media/{id}` | **(M9, ADR-057 §7)** streams a stored media file (photo/voice/thumbnail); session-gated. Serves **ad-hoc voice audio** from M9 T4 ([ADR-060](adr/060-node-media-linkage-and-voice-unification.md) §5); `FileResponse` Range/206 supports audio scrubbing (verified in the T6 phone drill) |
| `POST /connector/import/threads` | **(M9.5)** authenticated bulk upsert from the local prep tool: thread metadata (+ `name_override`/aliases) — idempotent |
| `POST /connector/import/messages` | **(M9.5)** batched message upsert by `(source, thread_id, message_id)` — idempotent, safe to re-send |
| `POST /connector/import/media` | **(M9.5)** multipart media upload (photos/voice raw; video = summary text + optional thumbnail) tied to message refs — idempotent |
| `GET /connector/sessions/{id}` | **(M9.5, ADR-058 §11)** the session transcript: rendered messages + media refs (photos inline via `GET /media/{id}`, voice playable) — the memory→source traceability surface |
| `GET /entities?type=person&q=…&limit=` | **(specced M9.5 ADR-058 §11; BUILT M9.8 T2, ADR-064 §2)** alphabetical browse/search over entity hubs → `[{ id, type, title, aliases }]` — feeds the shared merge picker (search-as-you-type; **diacritic-folded** name/alias match ranked exact-title→prefix→exact-alias→title-contains→alias-contains then alphabetical; empty `q` = alphabetical browse; `type` narrows to one entity-like kind, all when omitted; `limit` 1..50 default 20). Read-only, no model, session-gated; tombstones excluded; `/search` stays query-shaped |
| `POST /admin/connector/backfill` | **(M9.5, ADR-058 §8)** start the distill campaign (confirm-gated, single-flight, resumable; optional per-run model override + concurrency); `202`/`409`; progress via `agent_runs` |
| `POST /admin/connector/rederive` | **(M9.5, ADR-058 §9)** targeted media re-derivation — only `status=unavailable` or explicit ids |
| `POST /admin/connector/redistill` | **(M9.5, ADR-058 §9)** targeted session re-distill — only degraded sessions or an explicit list; idempotent via candidate dedup keys |

## Chat (M4 — carries the grilled chat plan, retargeted to nodes; [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))

Non-streaming; the web animates a client-side reveal. Retrieval = passive node-grouped top-k
(graph-aware expansion is backlog). Hybrid grounding, cited-only `[n]`. Fallback recorded on
`chat_messages.model`; chat does not write `agent_runs`.

| | |
|---|---|
| `GET /chat/models` | `{ models: [{ id, label, effort }], default }` — **`id` is the MODEL id** (the raw vendor string, e.g. `claude-opus-4-8`; [ADR-045](adr/045-provider-model-effort-separation.md) — was a provider id); `label` is a **friendly display name** derived from that model (`claude-opus-4-8` → "Claude Opus 4.8"; `meta-llama/Llama-3.3-70B-Instruct` → "Llama 3.3 70B"; unknown → raw id), `effort` = the reasoning effort the Chat group applies to that model (`null` for effort-less models like Nebius or one with no configured Chat-group effort); `default` = the Chat group's active model |
| `GET /chat/sessions` | `[{ id, title, created_at, last_model }]` newest first |
| `GET /chat/sessions/{id}` | `{ id, title, messages: [{ role, content, model, sources, created_at }] }` |
| `POST /chat` | `{ "message", "session_id"?, "model"?, "planes"?, "top_k"? }` → `{ session_id, answer, model_used, fallback_used, effort_used, sources: [{ node_id, store_path, type, title, snippet, score, planes }] }`. The `model` override + `model_used` are **model ids** ([ADR-045](adr/045-provider-model-effort-separation.md)); `chat_messages.model` stores the model id going forward (legacy provider-id rows kept + label-tolerated, ADR-045 §4). Implicit session creation; `sources` = cited nodes only, renumbered `[1..m]` (empty for general / "not in your memories" answers), **each + `media_kinds: []` from M9 T4** ([ADR-060](adr/060-node-media-linkage-and-voice-unification.md) §7 — the source-card glyph); `effort_used` = the reasoning effort applied to the answering model (`null` for effort-less models like Nebius) — feeds the per-message "answered by `<label>` · `<effort>`" caption on a fresh turn (**not persisted** — history renders the model label without effort) |
| `POST /chat/sessions/{id}/remember` | **(M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)/[048](adr/048-m6-chat-distiller-build-decisions.md) §6)** distill this session now — runs the **same** single distill pass **synchronously** on the delta-after-watermark, returns `200 { endorsed, to_review }` (or `{ skipped: "reason" }`); the resulting `captures` **organize in the background** (like a normal capture). Advances the same `chat_distill_state` watermark ⇒ idempotent with the nightly run; **same salience + stance gate** (no force-endorse). `404` unknown session |
| `GET /chat/auto-recorded?limit=` | **(M6, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §12)** the chat-scoped "recently auto-recorded" audit list — auto-endorsed chat memories newest-first: `[{ capture_id, node_paths, title, snippet, salience, source_ref, created_at }]` (feeds the one-tap-remove surface; M8's general feed absorbs this later) |
| `POST /chat/auto-recorded/{capture_id}/remove` | **(M6, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §11)** one-tap remove of an auto-endorsed chat memory — git-rm the node file(s) + DB-delete (`nodes`/`chunks`/`edges`) + **tombstone the capture** (`captures.removed_at`, replay/reprocess-excluded so it can't resurrect). Soft-delete (git history kept). `404` unknown/already-removed |

## Search & graph

| | |
|---|---|
| `POST /search` | `{ "query", "top_k"?: 10, "planes"?, "types"?, "as_of"?: date, "since"?/"until"?: date }` → **node-grouped** scored results, no LLM. `search_query:` prefix, cosine over `chunks` **⊍ derived-profile embeddings** (`node_profiles.embedding`, best-per-node, raw union — searching an entity's name surfaces its hub node with the profile as snippet; [ADR-037](adr/037-profile-embedding-in-search-m3.md)); **M4: ⊍ tsvector FTS over `chunks` ⊍ `node_profiles` (migration 008, mirrors the vector union) fused by RRF (k=60; FTS self-suppresses on non-English / zero-lexeme queries)** + mild recency prior (bounded multiplicative). `since`/`until` = `occurred`-range window; `as_of` = simple node-date filter (`occurred_start ≤ as_of`; bitemporal edge-validity `as_of` is backlog). `planes` filters membership; `types` filters node type. Result shape unchanged (a profile hit carries the entity's fields + the profile text as `snippet`): `[{ node_id, store_path, type, title, plane, planes[], tags[], snippet, score }]`. **M9 T4 ([ADR-060](adr/060-node-media-linkage-and-voice-unification.md) §7): + `media_kinds: []`** (distinct kinds of the node's attached media, e.g. `["photo"]` — drives the tiny list glyph; no thumbnails in lists) |
| `GET /nodes/{id}` | node detail: `{ node_id, store_path, type, title, plane, planes[], tags[], aliases[], disambig, occurred, occurred_end, interiority, body, profile, edges: [{ rel, dir, node_id, type, title, origin, score?, since? }] }` — body from the store file; `profile` = the **derived** entity profile ([ADR-030](adr/030-entity-substrate-and-lifecycle.md), null for content nodes); **`interiority`** (M8.2 T3.5, [ADR-055](adr/055-interiority-inner-voice-first-class.md) §3c) = `internal`\|`external`\|`mixed`, **null** on unstamped entity hubs — drives the `NodePreview` inner-voice marker; edges = canonical + derived, both directions. **M9 T4 ([ADR-060](adr/060-node-media-linkage-and-voice-unification.md) §1/§7): + `media: [{ id, kind, status, capture_id }]`** — the node's attached media via `node_media` (content-node links only; render via `GET /media/{id}`; `capture_id` feeds the "see raw capture" sheet over `GET /captures/{id}`; empty list when none). Tombstones 302-resolve to `merged_into`. `404` if unknown |
| `PUT /nodes/{id}/date-token` | **(M8.2, [ADR-056](adr/056-temporal-correctness-date-tokens.md) §5 — the mechanical token edit)** `{ old, start, end?, label? }` — rewrite the body date token `old` (the exact `[[t:START[/END][\|label]]]` string — the edit anchor, no text-span bookkeeping) to a new date built from `start` (+ optional `end` for a range, `label`), all **partial-ISO** (`2025` / `2025-07` / `2025-07-07`). When `old` is the node's **event** date (its resolved `occurred` span matches), `occurred`/`occurred_end` move too; the node's chunks are re-embedded. **No LLM, instant.** → `{ node_id, occurred_updated, occurred, occurred_end }`. `422` malformed id; `404` unknown/merged; `400` a bad token/date or a token not in the body |
| `GET /nodes/{id}/neighbors` | **(M7 map, [ADR-051](adr/051-m7-map-build-decisions.md) + [ADR-052](adr/052-map-zones-keyed-by-rel.md); same `GraphService.neighbors` as MCP `traverse`)** one-hop expansion for the explorer. One route, **two modes**: **no `rel`** → the **grouped** first page `{ center:{node_id,type,title,plane,planes[],interiority}, zones:[{ rel, neighbors:[…≤`map_zone_fanout`], total, next_cursor }, …] }` — **one zone per distinct `rel`** ([ADR-052](adr/052-map-zones-keyed-by-rel.md): zones keyed by `rel`, so canonical + derived `similar` collapse to **one** zone — `similar` is the only dual-origin rel; per-neighbor `origin` carries the solid/faint styling, no zone-level `origin`), each independently capped (**seed 8**) with its own `total` + `next_cursor`; **with `rel`** (+ optional `cursor`, `direction`) → that **single zone's** next flat page (thin over the M5 rel-filtered keyset — the "show more" call; a zone *is* a rel, so the rel-only cursor resumes it exactly). `direction=both`; each neighbor carries `origin`/`dir`/`score`/`since`/`until` + **`interiority`** (M8.2 T3.5, [ADR-055](adr/055-interiority-inner-voice-first-class.md) §3c — `internal`\|`external`\|`mixed`\|null; the map inner-voice marker, on neighbors **and** the `center`) (renders arrowheads, faint-derived, dashed-superseded, inner-voice marks without a second fetch). Tombstoned endpoints excluded; **`until`-closed edges returned** (rendered dashed+dimmed). Unknown node → `center=null` + empty `zones` |
| `GET /planes` | plane vocabulary for filter chips: `{ planes: [..], inbox: "inbox" }` |
| `GET /types` | **(M3)** effective vocabularies + pending proposals ([ADR-027](adr/027-typed-vocabulary-governance.md)): `{ node_types[], edge_rels[], entity_like_types[], proposals: [{ id, vocab, value, excerpt, created_at }] }`. Lists = config seeds ∪ approved additions (`app_settings`); `proposals` = still-pending `vocab-proposal` review items (resolve by their `id`) |

## Review queue (**M3** minimal + M6 full UX — [ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[029](adr/029-conversational-ingestion-stance-gate-review-queue.md))

Kind-generic: `entity-ambiguity` + `vocab-proposal` (M3), `stance-candidate` (M6), `dedup-proposal` (M6), `occurred-enrichment` (M8.2), `entity-dedup` (M9.8, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §4). M6 payloads/ergonomics: [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §7–§9; the `occurred-enrichment` kind: [ADR-056](adr/056-temporal-correctness-date-tokens.md) §7.

| | |
|---|---|
| `GET /review?status=pending&kind=` | decidable-in-place items: `[{ id, kind, payload, excerpt, source, source_ref, salience?, created_at }]` — `payload` carries candidates (`{id,name,disambig,aliases}`) or proposed content per kind; **stance-candidate** payload = `{candidate_text, referenced_entity_names, salience(high\|med\|low), why_unclear, anchor_at}` (`anchor_at` = the anchoring-message timestamp used to stamp the capture on **agree** — conversation time, not decision time); **dedup-proposal** ([ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md)) payload = `{node_a, node_b, signals:{cosine, shared_entity_ids, shared_entity_titles, occurred_overlap}, default_survivor}` (canonical `least→greatest` ids); **occurred-enrichment** ([ADR-056](adr/056-temporal-correctness-date-tokens.md) §7) payload = `{node_id, title, type}` (the undated content node a nightly step flagged — `node_id` is a clickable `NodeChip`); **entity-dedup** ([ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §4) payload = `{node_a, node_b, default_survivor, type, titles, signals:{name_match:{kind,score}, shared_count, jaccard}}` (canonical `least→greatest` hub ids — a **lower-confidence** duplicate hub pair; high-confidence pairs are surfaced inline on graph-health instead). Ordered by salience (high first). Include `status=maybe` to list parked items with **aging** |
| `GET /review/{id}` | **(M8.1 follow-up)** one review item by id, **any status** (incl. resolved) → `{ id, kind, payload, excerpt, source, source_ref, status, resolution, created_at }`. The detail the Activity **Reviewed** feed row expands to (its kind, what was filed, and the recorded `resolution`). `422` malformed id; `404` unknown. Read-only. |
| `POST /review/{id}` | resolution per kind — entity-ambiguity: `{ "choice": node_id \| "new" \| "maybe" }` (materializes the pending edge); vocab-proposal: `{ "verdict": "approve"\|"reject" }` (approve queues retro-consolidation); **stance-candidate (M6):** `{ "verdict": "agree"\|"disagree"\|"maybe" }` — agree materializes a `captures` row (the auto-endorse path); **dedup-proposal (M6, [ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md)):** `{ "action": "merge"\|"keep"\|"link", "survivor"? }` — **merge** folds the loser into the survivor (`survivor` id, else `payload.default_survivor`) via the shared merge-core → `resolved`; **keep** dismisses → `discarded`; **link** writes a canonical `similar` edge between the pair (kept distinct) → `resolved`; **entity-dedup (M9.8, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §4):** `{ "action": "merge"\|"keep", "survivor"? }` — **merge** folds the loser hub into the survivor **with the entity alias union** (the shared `fold_entities`) **and records a durable `entity_merges` decision** (survives a reprocess, §1) → `resolved`; **keep** dismisses (not a duplicate) → `discarded`; the entity counterpart to the content-only `dedup-proposal` merge; **occurred-enrichment ([ADR-056](adr/056-temporal-correctness-date-tokens.md) §7):** `{ "answer" }` — a **natural-language** date ("summer 2019", "last Tuesday ~6pm") classified into a symbolic time-ref (rule 12, fail-closed) + resolved against the answer's own date, then applied via the mechanical tier (§5: set `occurred`/`occurred_end` + re-embed) → `resolved`; `"maybe"` parks it; `"skip"`/`"unknown"`/`"dismiss"` leave the node undated → `discarded`; an **uninterpretable** phrase is a `400` (item stays decidable — rephrase, never a guessed date). **`maybe` is re-openable** — a parked item accepts a later agree/disagree ([ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §7). `404`/`409`/`400` |
| `POST /review/batch` | **(M6)** `{ ids[], action }` — resolve many at once, best-effort per item → `{ results: [{id, ok, error?}] }` |

## Activity & ops

| | |
|---|---|
| `GET /activity?category=&limit=50&before=` | **(M8, [ADR-053](adr/053-m8-ops-console-observability-build-decisions.md))** the merged categorized feed — a **UNION-of-views projection** over `agent_runs` + `captures` + `review_queue` (no new events table), each row normalized to `{ id, category, kind, ts, title, snippet, ref }`, newest first, **keyset-paginated on `(ts, id)`** via `before=`. `category` ∈ **`agents_jobs`** (scheduled runs; pipeline parents nest their step children via `parent_run_id`) · **`conversations`** (auto-endorsed `source=chat` captures — the M6 auto-recorded list folds in here — + chat-distiller outcomes) · **`manual_actions`** (manual runs, review verdicts, hand-triggered admin ops). **Category is by *origin*, not table:** a hand-run `reindex` lands under `manual_actions` via `agent_runs.trigger` |
| `GET /activity/runs/{id}` | full run incl. `details` (live-pollable) |
| `GET /activity/runs/{id}/logs?after_seq=` | **(M8)** the live log tail — cursor-paginated lines `[{ seq, ts, level, message }]` (1-based `seq`; the response also carries `next_after_seq` = the max `seq` returned, unchanged when empty) + a `running` flag (poll while `running`). Backed by the durable `agent_run_logs` table (an `app.*`/`INFO`+ logging handler → bounded per-run buffer → ~1s async flush + on-finish flush, ADR-053 §1/§2). **Poll, not stream** (SSE/streaming is backlog). **Client guidance:** the on-finish flush is async (~≤1s after the run flips terminal) and one page is capped (`run_log_tail_max_lines`), so a client must **keep paging on `next_after_seq` until a page comes back empty** rather than stop the instant `running==false` — else it can miss the last tail. Jobs-observability contract, 01 invariant 4 |
| `GET /agents` | **(M8)** a **flat roster** of individual jobs — `[{ name, category, pipelines: [names], running, last_run: { status, finished_at, run_id } }]` (`running` = live single-flight status from the JobRunner, for the console's live badge). **Scheduling is per-pipeline (M5.5, [ADR-047](adr/047-pipeline-scheduling-primitive.md)):** a job's cadence is *derived* from the pipelines it is a step of; **membership is 0..N (many-to-many)** — a step may appear in several pipeline defs, and an on-demand-only job has `pipelines: []` and no schedule. Schedule detail lives on `GET /pipelines` |
| `GET /pipelines` | **(M8, [ADR-053](adr/053-m8-ops-console-observability-build-decisions.md))** pipelines as a first-class resource — `[{ name, cron, next_run, steps: [ordered names], last_run }]`, from the live scheduler (`pipeline_runners()` + APScheduler `next_run_time`). A pipeline run is a parent `agent_runs` row, each step a child (`parent_run_id`). (Future home of pipeline *editing* — deferred, ADR-053) |
| `POST /agents/{name}/run` | **(M8)** manual trigger of one **zero-arg** job standalone (invariant 4 — every job runnable, no cron-only ghosts), stamped `trigger=manual`; `409` if that agent (or a live pipeline it is a step of) is running. Serialized by the in-process **JobRunner** single-flight guard the scheduler shares (authoritative — the scheduler runs single-process). Returns **`202 { agent }`** (runs in the background; the mint-inside-the-body run id is discovered via `last_run.run_id`); `404` unknown; `503` when the scheduler is off. Parameterized ops (`reprocess`/`entities merge`/tag+vocab consolidate) keep their own `/admin/*` endpoints |
| `POST /pipelines/{name}/run` | **(M8)** manual trigger of a **whole pipeline** (the ADR-047 §6 CLI verb over HTTP); returns **`202 { pipeline }`**; `409` if it is already running; `404` unknown; `503` when the scheduler is off |

> **M8.1 addendum ([ADR-054](adr/054-m8.1-ui-navigation-consolidation.md) — server built, task 1):**
> the `GET /activity` `agent_runs` branch returns **only parentless runs** (`parent_run_id IS NULL`;
> `parent_ref` kept on the wire, now always null on that branch). Every feed row also carries
> **`status`** (the source row's lifecycle status) and **`source`** (a Captures row's origin badge
> `COALESCE(source, kind)` → `text`/`voice`/`mcp`/`chat`; null on the non-capture branches).
> `GET /activity/runs/{id}` gains a **recursive `children[]` tree** — each node
> `{ id, name, status, ts, summary, children[] }`, siblings ordered **ascending ts (early→late)**;
> empty for a leaf run. (Per [ADR-050](adr/050-pipeline-step-status-is-the-jobs-own-run.md) a step's
> spawned `capture` runs parent to the pipeline *root*, so current trees are one level deep — steps
> and spawned runs are siblings; the render is recursive so any future deeper nesting shows its true
> depth.) The **`conversations` category is renamed `captures`** and widened to **all** captures
> regardless of source (was chat-only); the normalized row `kind` is **`capture`** (was
> `chat_capture`); one-tap-removed captures stay excluded (`removed_at`); chat rows keep the one-tap
> remove semantics. Expandable full detail (raw text + node paths + status + source) is the existing
> **`GET /captures/{id}`**, whose `CaptureView` now also carries **`source`**. graph-health `details`
> offender samples are already structured `sample: [{ id, label }]` (since M8 Batch B T4). **Carve-out
> refining [ADR-054](adr/054-m8.1-ui-navigation-consolidation.md) §5:** the six node-checks
> (orphan-nodes, inbox-depth, memories-missing-occurred, alias-less-entities, tombstone-integrity,
> stale-observations) carry a `nodes.id` → the web renders them as clickable `NodeChip`s; the
> **`pending-review-aging`** check's offenders carry a `review_queue.id` (a review item, *not* a
> single node — stance-candidate/dedup-proposal items intrinsically have no one node id), so the web
> links those to the **Review** item, not `NodePreview`. The web distinguishes by check name (stable
> constants). No server change.
> **M8.1 addendum (task 4 — the T2-replan fold-in):** `CaptureView` (both `GET /captures` and
> `GET /captures/{id}`) also carries **`node_refs: [{ id, store_path, type, title }]`** — a read-time
> `node_paths → nodes.id` join (a `LEFT JOIN LATERAL` in the store, **no migration**), because
> `node_paths` are store-path *projections* ([02](02-data-model.md) §Identity), not the frontmatter-uuid
> identity that `GET /nodes/{id}` / the web `NodeChip` require. `node_refs` preserves `node_paths` order
> (`array_position`); a path with no live `nodes` row (not yet indexed, or tombstoned/merged away) is
> **silently absent** — never null, never an error; an empty `node_paths` yields `[]`. `node_paths`
> itself is unchanged. This is what makes a capture's node chips clickable → `NodePreview`.
> **M8.2 addendum ([ADR-056](adr/056-temporal-correctness-date-tokens.md)):** node body text may
> carry inline date tokens `[[t:START[/END][|label]]]` — clients render, never show raw. **Landed
> (Task 3):** the two edit endpoints — **token edit** `PUT /nodes/{id}/date-token` (mechanical) and
> **capture-anchor edit** `PUT /captures/{id}/anchor` (→ background reorganize) — plus the
> **`occurred-enrichment`** review kind (`answer` on `POST /review/{id}`). Note: a token/enrichment
> edit lives on the derived node, so a later `reprocess-all` re-derives it from raw and reverts it
> ([ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md) — this is a per-node derived-edit
> caveat; manual **merges** are now reprocess-durable via `entity_merges`, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §1;
> the **anchor** edit is reprocess-safe too — it writes `captures.created_at`, which the replay reads).
> **M8.2 addendum (Task 3.5 — interiority on the read API, [ADR-055](adr/055-interiority-inner-voice-first-class.md) §3c):**
> a **replan seam** — T3's consumer sweep wired `interiority` into LLM-bound renders + the chat boost +
> the capsule slice but never exposed it on the two web read endpoints, so the web could not render the
> §3c marker. Now **`GET /nodes/{id}`** carries **`interiority`** (`internal`\|`external`\|`mixed`\|null)
> and **`GET /nodes/{id}/neighbors`** carries it on every neighbor **and** the `center`. `SearchResultItem`
> is deliberately unchanged (the ADR names only Map/`NodePreview`). Contract wiring, no new ADR; the
> web marker is `internal` = full / `mixed` = subtle / `external`\|null = none.
> **M9.8 addendum (T4 — the entity-dedup inline feed, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §4):**
> the conservative entity-hub dedup detector's **high-confidence** pairs are surfaced **inline** the
> same way the graph-health card is read — off the **latest `entity-dedup` run's `details`** (via the
> roster `last_run.run_id` → `GET /activity/runs/{id}`). `details.high_confidence` is
> `[{survivor:{id,title}, loser:{id,title}, type, signals:{name_match, shared_count, jaccard}}]` (the
> survivor is the higher-degree hub — the web pre-fills the shared merge picker from it); the run also
> carries `pairs_scanned` / `low_confidence_filed` / `already_tracked`. **Lower-confidence** pairs file
> an `entity-dedup` review item instead (above). No new endpoint — the one-click Merge reuses
> `POST /admin/entities/merge` (which already unions aliases + records the durable `entity_merges`
> decision, T1); the detector never auto-merges (rule 2).

## Settings ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md), [ADR-027](adr/027-typed-vocabulary-governance.md))

| | |
|---|---|
| `GET /settings` | model routing for **all groups** (`chat`/`conspect`/`quick`, ADR-025 + [ADR-043](adr/043-quick-routing-tier-m4.md); **+ `vision` at M9** — [ADR-057](adr/057-multimodal-media-ingestion-substrate.md) §4, Groq-primary seed, effort N/A) + available **models** incl. which support effort + their levels (registry-sourced; no hardcoded lists in web). **([ADR-045](adr/045-provider-model-effort-separation.md)) the routable unit is a MODEL id** (the raw vendor string, e.g. `claude-opus-4-8`) — the provider is an attribute of the model; each model option carries its `provider`. `active`/`fallback` are model ids; effort is keyed by model id |
| `PUT /settings/models` | save **one** group's `{active, fallback, effort_by_model}` (model ids — [ADR-045](adr/045-provider-model-effort-separation.md); was `effort_by_provider`/provider ids); `422` on unknown id; busts the routing cache (forward-live, no restart) |
| `PUT /settings/vocabulary` | **(M3)** `{ review_id, verdict: "approve"\|"reject" }` → resolve a pending type proposal. Approve writes the type to the live vocabulary (`app_settings`, forward-live) + opens the `vocab-consolidation` job; reject discards. Returns the updated review item. Same choke point as `POST /review/{id}` for a `vocab-proposal` (ADR-027 §4 / [ADR-035](adr/035-vocabulary-consolidation-scope-m3.md)). `404`/`409`/`400` as `POST /review/{id}` |
| `PUT /settings/connectors/{name}` | **(M9.5 API-connector path / M12)** per-connector config incl. lookback override (default 6 months) |

## Admin

| | |
|---|---|
| `POST /admin/reindex` | async full pass: rescan store → materialize canonical edges → recompute derived edges → `202 { run_id }`; single-flight `409`. (No render/commit step — similarity never touches files, ADR-026) |
| `POST /admin/tags/consolidate` | two-step tag cleanup (propose → apply), [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md) |
| `POST /admin/vocab/consolidate` | **(M3, [ADR-036](adr/036-edge-retro-consolidation-walk-retypings-only-m3.md))** two-step edge retro-consolidation for an approved edge rel: propose (`{rel, apply:false}` → `200 {plan_id, retypings:[{src_id, to, from_rel, to_rel}]}`, bounded edge inventory, no writes) → apply (`{rel, apply:true, plan}` → `202 {run_id}`, rewrites edge `rel:` frontmatter + reindex + force-commit). Edges only; node re-typing stays propose-only (ADR-035). `400` unknown/empty rel or empty apply plan; `503` distill down on propose |
| `POST /admin/entities/merge` | **(M3, [ADR-030](adr/030-entity-substrate-and-lifecycle.md))** two-step: propose (`{loser, survivor, apply:false}` → inbound-edge inventory) → apply (`apply:true`) — immediate, after a forced commit+push; unions aliases, writes the tombstone, reindexes |
| `POST /admin/nodes/{id}/delete` | **(M9.8 T5, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §5)** delete a genuinely **zero-degree orphan hub** — git-rm the file (`NodeWriter.remove_nodes`) + prune its index rows (`chunks`/`edges`/`node_media` cascade off `nodes`) + force-commit, in the background under an `agent_runs` row → `202 {run_id}`. Validation is synchronous: `404` unknown/tombstone; `400` a **content** node (route it to `DELETE /captures/{id}` — capture-remove tombstones the owning capture so a reprocess can't replay the raw + resurrect it, whereas a bare git-rm would); `409` a **still-referenced** node (any live canonical edge either direction → route to Merge). Never auto-deletes (rule 2); a reprocess won't recreate an unreferenced hub |
| `POST /admin/reprocess` | **(M3 task 11, [ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md))** reusable `reprocess-all-from-raw` (the data-survival op, vision P10). Confirm-gated: `{confirm:false}` → `200 {captures, nodes, merges}` preview (no writes); `{confirm:true}` → `202 {run_id}` — reset derived state (node files + `nodes`/`chunks`/`edges`/`node_profiles`, `captures.node_paths`) and the **capture-derived `review_queue` kinds only** (`entity-ambiguity`/`vocab-proposal`/`dedup-proposal`/`entity-dedup`; **`stance-candidate` preserved** — [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §7), replay every capture's raw **chronologically** through the current pipeline (incl. chat-endorsed captures → P10; `removed_at`-tombstoned captures skipped), **re-apply durable standing merges** (`entity_merges`, [ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §1 — matched by surface form now the hubs+edges exist, before the recompute so similarity/profiles reflect the merged graph), recompute derived edges, **rebuild derived profiles** (the reset truncates `node_profiles`, so a profile-refresh runs before the commit → the profile search leg [ADR-037] is live immediately, not empty until the nightly job), force commit+push. The run `summary`/`details` carry per-heal totals (`coerced`/`accreted`/`profiles_refreshed`/`standing_merges_reapplied`) for auditability. Raw + approved vocab preserved; `{merges}` in the preview = durable merges that will re-apply (no longer dropped). Single-flight `409` |
| `POST /admin/backup` | force store git commit+push → `{ committed, pushed }` |
| `POST /admin/captures/{id}/reorganize` | re-organize a capture's raw text, replace its **content** nodes ([ADR-038](adr/038-reorganize-preserves-shared-entity-hubs.md): entity hubs are shared substrate — never deleted); `202` |
| `GET /admin/providers` | **(M4 follow-up, [ADR-044](adr/044-provider-observability-surface.md); [ADR-045](adr/045-provider-model-effort-separation.md))** provider observability — **one row per PROVIDER** (5: `claude`, `nebius`, `groq`, `openai`, `ollama`): `{ id, label, capabilities:[chat\|stt\|embedding], reachable, last_error:{message,at}\|null, last_success_at, consecutive_failures }`. `label` is the **friendly provider name** ("Claude", "Nebius", …), **not** a model — ADR-045 (`claude` serves Opus+Sonnet as *models*, one health signal for one CLI/credential; no per-model breakdown, no raw id in the UI). `reachable` is a **live `Provider.health()` probe** (config-reachability, **not** a success guarantee); `last_error` is **sticky** (a later success does not clear it) with `last_success_at`/`consecutive_failures` beside it — state is **in-memory** on the registry (resets on redeploy, no persistence). Session-gated; **no LLM call**. `/health` stays public + untouched |
| `POST /admin/mcp/revoke-all` | **(M5 task 3, [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md) §2)** the **"revoke all MCP access"** switch — flags every live MCP access + refresh token in one UPDATE → `{ revoked }`. Instant + total (single-user; per-connector management → M8). Session-gated (the OAuth flow itself is the public surface; revocation is the private control). A connector must re-run the OAuth flow after this |
| `GET /health` | no auth: `{ status, db, store, git_remote, backups }`, `503` when degraded ([ADR-014](adr/014-vault-history-durability.md) §6) |

## MCP server (M5, [ADR-028](adr/028-one-service-layer-mcp-peer-surface.md) + [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md))

Remote MCP server (`mcp` SDK, **Streamable HTTP**) at **`https://braindan.cc/mcp`**, mounted on
the `api` app (single origin, single container). Exposes the same services — **no logic of its
own**. Tool results are **LLM-optimized Markdown** (not JSON — token-efficient, cross-model
Claude+GPT; **IDs rendered verbatim + labeled** so the model chains calls). Hub edge lists are
capped inline at a config N with a `traverse` overflow pointer. The MCP `initialize`
**`instructions`** field carries an authored usage capsule (what the brain is · the six tools +
when to use each · the `search`→`build_context`→`capture` loop · temporal filters · the
research-via-MCP pattern); each tool carries a rich `description` + annotations (`readOnlyHint`
on reads, a write marker on `capture`). An invokable **research-via-MCP Prompt** encodes
[ADR-033](adr/033-external-inspirations-obsidian-second-brain.md) #6. A read-only resource
**`identity://me`** serves the identity capsule ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)
#1) up-front without a node.

| tool | maps to |
|---|---|
| `search(query, top_k?, planes?, types?, as_of?, since?/until?)` | SearchService (as `POST /search`, incl. RRF hybrid + temporal filters — [ADR-032](adr/032-prior-art-adoptions.md)) |
| `get_node(id)` | as `GET /nodes/{id}` |
| `traverse(id, rel?, direction?=both, cursor?)` | new **`GraphService.neighbors`** one-hop primitive (also serves `GET /nodes/{id}/neighbors`, M7) — center+rel filter, `direction` ∈ {out, in, both}, **cursor-paginated** (LLM context is finite) |
| `build_context(id, depth?)` | convenience: get_node + neighbors bundled in one round-trip, **depth ≤ 2** with fanout caps ([ADR-032](adr/032-prior-art-adoptions.md), Basic-Memory pattern); **level-0 = the identity capsule** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md) #1) |
| `list_planes()` / `list_types()` | vocabulary listings |
| `capture(text)` | the **full organizer pipeline**, identical to `POST /capture/text` (`source: mcp`, **burst-queued**, **fast ack** — returns id+status, LLM verifies via `search`) — external LLMs never write nodes/edges directly |

**Auth — OAuth 2.1** ([ADR-046](adr/046-m5-mcp-server-oauth-connectors.md), refines ADR-028 §5;
`authlib`). The `api` app is authorization + resource server, root-level routes (Caddy-proxied,
Cloudflare un-cached):

| endpoint | purpose |
|---|---|
| `GET /.well-known/oauth-protected-resource` | RFC 9728 — points to the AS; resource id `https://braindan.cc/mcp` (RFC 8707) |
| `GET /.well-known/oauth-authorization-server` | RFC 8414 — AS metadata |
| `POST /register` | RFC 7591 open **Dynamic Client Registration** (inert alone) |
| `GET/POST /authorize` | the choke point — **Argon2id password + explicit consent** (valid PWA session short-circuits to consent), **CSRF + rate-limited**, **PKCE** |
| `POST /token` | code→token + refresh; **opaque HMAC-hashed DB tokens** (~1h access, long-lived sliding refresh) |

Tokens are **independently revocable** (ADR-028 §5) — a **"revoke all MCP access"** control
(`POST /admin/mcp/revoke-all`, session-gated) is the M5 mechanism (single-user; per-connector
management → M8). Access tokens are opaque, HMAC-hashed in `mcp_tokens` (~1h), with a long-lived
**sliding refresh** rotated on each `/token` use (reuse of a rotated-out refresh revokes the
client — replay guard); `/authorize` requires **PKCE S256** + a double-submit-CSRF-protected,
rate-limited password/consent gate. Single full-access scope (`brain`) in M5.
Token distribution is the add-connector OAuth flow (no manual token). **Accept gate = a real
Claude connector** (mobile app / claude.ai web) live; **ChatGPT is a fast-follow before M6**
(may add thin `search`/`fetch` aliases for its deep-research connector).
