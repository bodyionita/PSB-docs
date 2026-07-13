# API Contract

**Version:** 2.4 · **Status:** Approved 2026-07-13 (2.4 = **M3 chat + model routing** —
[ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md): `POST /chat` finalized
(non-streaming, cited-only `[n]` sources, `planes`), `GET /chat/models` real registry ids + labels,
`GET /chat/sessions[/{id}]`, enriched `GET /settings`, new `PUT /settings/models` per group
(`PUT /settings/agents` superseded). 2.3 = M2 web Task 8 needs: new `GET /planes`
for the Search-tab plane-filter chips, and `GET /activity/runs/{id}` **pulled forward from M4**
so the Admin tab can show a reindex/tags-apply run's live status + counts. 2.2 = M2 indexing/search:
`POST /search` now note-grouped, `POST /admin/reindex` async→`202`+run, new `GET /notes/{id}` preview,
`POST /admin/tags/consolidate` — [ADR-022](adr/022-embeddings-self-hosted-nomic.md),
[ADR-023](adr/023-semantic-relatedness-graph.md), [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md);
2.1 = M1 capture additions: `GET /captures` list, `POST /captures/{id}/follow-up`, follow-up
fields, `/health` `backups` leg)

This HTTP API is the **only** seam between `web/` and `server/`
([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)). The server publishes
OpenAPI at `/api/openapi.json`; the web client may generate its types from it.

- Base path: `/api/v1` (Caddy serves the PWA at `/` and proxies `/api` to FastAPI).
- Auth: session cookie (httpOnly, Secure, SameSite=Lax) set by `/auth/login`
  ([ADR-007](adr/007-auth-password-session-cloudflare.md)). All endpoints require it
  except `/auth/login` and `/health`.
- Errors: `{"detail": "..."}` + proper status. 401 on missing/expired session.

## Auth

| | |
|---|---|
| `POST /auth/login` | `{ "password": "…" }` → sets session cookie. Rate-limited (5/min/IP). |
| `POST /auth/logout` | revokes current session |
| `GET /auth/me` | `{ "authenticated": true, "session_created_at": … }` |

## Capture

| | |
|---|---|
| `POST /capture/text` | `{ "text": "...", "created_at"?: iso }` → `202 { "capture_id", "status": "received" }`; pipeline continues in background |
| `POST /capture/voice` | multipart `file` (m4a/webm/ogg/mp3/wav, ≤25 MB) → same `202` body |
| `GET /captures?limit=20` | recent captures for the capture-screen strip, newest first: `[{ capture_id, kind, status, raw_text, note_paths[], follow_up_question, follow_up_answer, error, created_at, updated_at }]` (M1; `/activity` supersedes it in M4) |
| `GET /captures/{id}` | pipeline state: `{ capture_id, kind, status, raw_text, note_paths[], follow_up_question, follow_up_answer, error, created_at, updated_at }` |
| `POST /captures/{id}/retry` | re-run from first incomplete step; `409` unless `failed` |
| `POST /captures/{id}/follow-up` | `{ "answer": "..." }` → **Pass 2** re-organize (original + answer), replacing the capture's notes ([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)); `202`. `409` if no `follow_up_question` pending |

## Chat (M3, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))

Non-streaming (M3): `POST /chat` returns the full answer in one JSON body; the web animates a
client-side reveal (true token streaming is post-v1). Retrieval is passive note-grouped top-k over
the vault index; the answer is grounded + `[n]`-cited for note-questions (hybrid — general questions
answered normally). Fallback resolution is recorded on the assistant message (`chat_messages.model`);
chat does **not** write `agent_runs`.

| | |
|---|---|
| `GET /chat/models` | chat models from the registry, **real ids** + display labels (`CHAT_MODEL_LABELS`): `[{ "id": "claude-max", "label": "Claude (Max)", "default": true }, { "id": "nebius", "label": "Nebius · Llama 3.1 70B" }]`. `default` = the Chat group's active model. The composer picker overrides only the active model per conversation (fallback + effort inherited from the Chat group — ADR-025) |
| `GET /chat/sessions` | conversation list for the sidebar, newest first: `[{ id, title, created_at, last_model }]` |
| `GET /chat/sessions/{id}` | one thread's messages: `{ id, title, messages: [{ role, content, model, sources, created_at }] }`; `404` if unknown |
| `POST /chat` | `{ "message", "session_id"?: uuid, "model"?: id, "planes"?: [..], "top_k"?: int }` → `{ "session_id", "answer", "model_used", "fallback_used", "sources": [{ note_id, vault_path, title, snippet, score, planes }] }`. Implicit session creation when `session_id` omitted. `sources` = **only the notes actually cited**, renumbered `[1..m]` to match the `[n]` in `answer` (empty for general / "not in your notes" answers). `model_used` may differ from requested/default when fallback fired (UI shows the banner). `planes` scopes retrieval on `notes.planes` membership; `top_k` defaults to `CHAT_CONTEXT_TOP_K` |

## Search & notes

| | |
|---|---|
| `POST /search` | `{ "query", "top_k"?: int=10, "planes"?: [..] }` → **note-grouped** scored results, no LLM call. Query embedded with the `search_query:` prefix ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)); pgvector cosine over `chunks`, **deduped to one row per note** (best-scoring chunk = snippet). `planes` filters on **`notes.planes` membership** (array overlap, not folder — [ADR-005](adr/005-planes-and-atomic-notes.md)). Response: `[{ note_id, vault_path, title, plane, planes[], tags[], snippet, score }]`, ranked by best chunk. No hard score floor by default (`SEARCH_MIN_SCORE` setting, off = 0). |
| `GET /notes/{id}` | read-only note preview for the search UI expand: `{ note_id, vault_path, title, plane, planes[], tags[], body, related: [{ note_id, vault_path, title, score }] }`. `body` is read from the **vault file** (fidelity, incl. any Obsidian edits); `related` = this note's semantic neighbours from `note_links` ([ADR-023](adr/023-semantic-relatedness-graph.md)). `404` if unknown. No in-app editing (v2). |

## Meta

| | |
|---|---|
| `GET /planes` | the configured plane vocabulary for the Search-tab plane-filter chips ([ADR-005](adr/005-planes-and-atomic-notes.md)): `{ planes: [..], inbox: "Inbox" }` — `planes` = the `PLANES=` config list (primary homes); `inbox` is the always-present system plane, not part of `PLANES`. Session-gated, no LLM. The web filters `POST /search` on `notes.planes` membership using these values. (Added M2 so the web needs no server-config duplication — ADR-006.) |

## Activity feed

| | |
|---|---|
| `GET /activity?limit=50&before=<cursor>` | merged feed: agent runs + captures + errors, newest first. Run entries: `{ id, agent, status, started_at, finished_at, model_used, fallback_used, summary }` |
| `GET /activity/runs/{id}` | full run incl. `details` jsonb (expandable view): `{ id, agent, status, started_at, finished_at, model_used, fallback_used, summary, details, error }`; `404` if unknown. **Implemented in M2** (pulled forward from the M4 feed) so the Admin tab can poll a reindex / tags-apply run's live status + `details` counts. The merged `GET /activity` list stays M4. |

## Settings ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))

Model routing is UI-editable per **group** (`chat` = `chat_chain`, `conspect` = `distill_chain`) and
persisted in `app_settings`; env config (`chat_chain` / `distill_chain` / `CLAUDE_MAX_EFFORT`) is the
default when unset. Saves bust the `ModelRoutingService` cache (apply on next request, no restart).

| | |
|---|---|
| `GET /settings` | saved routing for both groups **plus** the choices the UI needs (all registry-sourced — no hardcoded model/effort lists in the web): `{ "models": { "chat": { "active": id, "fallback": id, "effort": { "<provider>": "medium" } }, "conspect": { … } }, "available": { "chat": [id…], "conspect": [id…] }, "providers": { "<id>": { "label", "supports_effort": bool, "effort_levels": ["low",…] } } }` |
| `PUT /settings/models` | `{ "group": "chat"\|"conspect", "active": id, "fallback"?: id, "effort"?: { "<provider>": level } }` — saves one group's routing; unknown model id → `422`. Busts the routing cache |
| `PUT /settings/agents` | **superseded by `PUT /settings/models`** ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)); the `conspect` group replaces it |

## Summaries

| | |
|---|---|
| `GET /summaries?period=daily&limit=14` | most recent summaries: `[{ period, period_start, content, note_path }]` |

(Generating/re-running summaries goes through `POST /agents/{name}/run` with
`daily-summary` / `weekly-review`.)

## Agents & admin

| | |
|---|---|
| `GET /agents` | registered connectors/jobs + schedule + last run |
| `POST /agents/{name}/run` | trigger on demand (e.g. `slack-ingest`, `daily-summary`); `409` if already running |
| `POST /admin/reindex` | vault reconciliation, **async** ([ADR-023](adr/023-semantic-relatedness-graph.md)): `202 { run_id }`, work runs in the background and opens an `agent="reindex"` run (`details.trigger="manual"`) — counts (`indexed, skipped, deleted, failed`) land in the run's `details`; `partial` on embed failures (skip-and-continue). **Single-flight**: `409` if a reindex or the nightly rescan is already running. Runs the full pass (rescan → recompute `note_links` → render `## Related notes` blocks → commit+push). |
| `POST /admin/tags/consolidate` | tag-vocabulary cleanup, two-step ([ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)). **Propose** (`{ "apply": false }` / default) → `{ plan_id, merges: [{ canonical, variants[] }] }`, no writes. **Apply** (`{ "apply": true, "plan": [{ canonical, variants[] }] }`) → rewrites affected notes' frontmatter tags + reindexes them → `202 { run_id }`. Never-lose (git-tracked, revertible). |
| `POST /admin/backup` | force vault git commit+push → `{ committed, pushed }` |
| `POST /admin/captures/{id}/reorganize` | maintenance re-run: re-organize a capture's stored raw text and **replace** its notes (e.g. re-deriving notes after the English-only organizer change). `202` → `{ capture_id }`; `404` if unknown. Raw input untouched (never-lose); notes replaced only on a successful organize, kept on the Inbox-fallback |
| `GET /health` | no auth: `{ status, db, vault, git_remote, backups }`, `503` when degraded. `backups` (M1, [ADR-014](adr/014-vault-history-durability.md) §6) is false when the latest `integrity-drill` run failed or is overdue (>~8 days) |
