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
  `/auth/login` and `/health`. **MCP: bearer token**, separate and independently revocable.
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
| `POST /capture/text` | `{ "text", "created_at"?: iso }` → `202 { capture_id, status: "received" }`; pipeline continues in background |
| `POST /capture/voice` | multipart `file` (m4a/webm/ogg/mp3/wav, ≤25 MB) → same `202` |
| `GET /captures?limit=20` | recent captures, newest first: `[{ capture_id, kind, status, raw_text, node_paths[], follow_up_question, follow_up_answer, error, created_at, updated_at }]` |
| `GET /captures/{id}` | pipeline state (same shape as above) |
| `POST /captures/{id}/retry` | re-run from first incomplete step; `409` unless `failed` |
| `POST /captures/{id}/follow-up` | `{ "answer" }` → Pass-2 re-organize, replaces the capture's nodes ([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)); `202`; `409` if no nudge pending |

## Chat (M4 — carries the grilled chat plan, retargeted to nodes; [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))

Non-streaming; the web animates a client-side reveal. Retrieval = passive node-grouped top-k
(graph-aware expansion is backlog). Hybrid grounding, cited-only `[n]`. Fallback recorded on
`chat_messages.model`; chat does not write `agent_runs`.

| | |
|---|---|
| `GET /chat/models` | registry ids + labels; `default` = the Chat group's active model |
| `GET /chat/sessions` | `[{ id, title, created_at, last_model }]` newest first |
| `GET /chat/sessions/{id}` | `{ id, title, messages: [{ role, content, model, sources, created_at }] }` |
| `POST /chat` | `{ "message", "session_id"?, "model"?, "planes"?, "top_k"? }` → `{ session_id, answer, model_used, fallback_used, sources: [{ node_id, store_path, type, title, snippet, score, planes }] }`. Implicit session creation; `sources` = cited nodes only, renumbered `[1..m]` (empty for general / "not in your memories" answers) |
| `POST /chat/sessions/{id}/remember` | **(M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md))** distill this session now (manual trigger of the chat-distiller for one session) → `202 { run_id }` |

## Search & graph

| | |
|---|---|
| `POST /search` | `{ "query", "top_k"?: 10, "planes"?, "types"?, "as_of"?: date, "since"?/"until"?: date }` → **node-grouped** scored results, no LLM. `search_query:` prefix, cosine over `chunks`; **M4 ([ADR-032](adr/032-prior-art-adoptions.md)): ⊍ tsvector FTS fused by RRF (k=60)** + mild recency prior; date params filter on the `occurred` range. `planes` filters membership; `types` filters node type. `[{ node_id, store_path, type, title, plane, planes[], tags[], snippet, score }]` |
| `GET /nodes/{id}` | node detail: `{ node_id, store_path, type, title, plane, planes[], tags[], aliases[], disambig, occurred, body, profile, edges: [{ rel, dir, node_id, type, title, origin, score?, since? }] }` — body from the store file; `profile` = the **derived** entity profile ([ADR-030](adr/030-entity-substrate-and-lifecycle.md), null for content nodes); edges = canonical + derived, both directions. Tombstones 302-resolve to `merged_into`. `404` if unknown |
| `GET /nodes/{id}/neighbors` | **(M7 map; same service as MCP `traverse`)** one-hop expansion for the explorer: grouped by rel/origin, paginated |
| `GET /planes` | plane vocabulary for filter chips: `{ planes: [..], inbox: "inbox" }` |
| `GET /types` | **(M3)** node + edge type vocabularies (config + approved additions) + pending proposals ([ADR-027](adr/027-typed-vocabulary-governance.md)) |

## Review queue (**M3** minimal + M6 full UX — [ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[029](adr/029-conversational-ingestion-stance-gate-review-queue.md))

Kind-generic: `entity-ambiguity` + `vocab-proposal` (M3), `stance-candidate` (M6), `dedup-proposal` (M6+).

| | |
|---|---|
| `GET /review?status=pending&kind=` | decidable-in-place items: `[{ id, kind, payload, excerpt, source, source_ref, created_at }]` — `payload` carries candidates (`{id,name,disambig,aliases}`) or proposed content per kind |
| `POST /review/{id}` | resolution per kind — entity-ambiguity: `{ "choice": node_id \| "new" \| "maybe" }` (choice materializes the pending edge, file + DB); vocab-proposal: `{ "verdict": "approve"\|"reject" }` (approve queues retro-consolidation); stance (M6): `{ "verdict": "agree"\|"disagree"\|"maybe" }` |

## Activity & ops

| | |
|---|---|
| `GET /activity?category=&limit=50&before=` | categorized feed (agents/jobs · conversations · manual actions — **tabs plural**, M8): runs + captures + review verdicts + errors, newest first |
| `GET /activity/runs/{id}` | full run incl. `details` (live-pollable) |
| `GET /activity/runs/{id}/logs` | **(M8)** live log tail for a running job (jobs-observability contract, 01 invariant 4) |
| `GET /agents` | registered jobs/connectors + category + **schedule (cadence, next run)** + last run |
| `POST /agents/{name}/run` | manual trigger (every job, no cron-only ghosts); `409` if running |

## Settings ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md), [ADR-027](adr/027-typed-vocabulary-governance.md))

| | |
|---|---|
| `GET /settings` | model routing per group + available models/providers (registry-sourced) |
| `PUT /settings/models` | save one group's routing; `422` on unknown id; busts the routing cache |
| `PUT /settings/vocabulary` | **(M3)** approve/reject a proposed node/edge type; approval queues the retro-consolidation job |
| `PUT /settings/connectors/{name}` | **(M9)** per-connector config incl. lookback override (default 6 months) |

## Admin

| | |
|---|---|
| `POST /admin/reindex` | async full pass: rescan store → materialize canonical edges → recompute derived edges → `202 { run_id }`; single-flight `409`. (No render/commit step — similarity never touches files, ADR-026) |
| `POST /admin/tags/consolidate` | two-step tag cleanup (propose → apply), [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md) |
| `POST /admin/entities/merge` | **(M3, [ADR-030](adr/030-entity-substrate-and-lifecycle.md))** two-step: propose (`{loser, survivor, apply:false}` → inbound-edge inventory) → apply (`apply:true`) — immediate, after a forced commit+push; unions aliases, writes the tombstone, reindexes |
| `POST /admin/backup` | force store git commit+push → `{ committed, pushed }` |
| `POST /admin/captures/{id}/reorganize` | re-organize a capture's raw text, replace its nodes; `202` |
| `GET /health` | no auth: `{ status, db, store, git_remote, backups }`, `503` when degraded ([ADR-014](adr/014-vault-history-durability.md) §6) |

## MCP tools (M5, [ADR-028](adr/028-one-service-layer-mcp-peer-surface.md))

Bearer-token-authenticated MCP server exposing the same services — no logic of its own:

| tool | maps to |
|---|---|
| `search(query, top_k?, planes?, types?, as_of?, since?/until?)` | SearchService (as `POST /search`, incl. RRF hybrid + temporal filters — [ADR-032](adr/032-prior-art-adoptions.md)) |
| `get_node(id)` | as `GET /nodes/{id}` |
| `traverse(id, rel?, depth?=1, cursor?)` | GraphService (as `GET /nodes/{id}/neighbors`) — center+depth+rel filter, **cursor-paginated** (LLM context is finite) |
| `build_context(id, depth?)` | convenience: get_node + traverse bundled in one round-trip ([ADR-032](adr/032-prior-art-adoptions.md), Basic-Memory pattern) |
| `list_planes()` / `list_types()` | vocabulary listings |
| `capture(text)` | the **full organizer pipeline**, identical to `POST /capture/text` (`source: mcp`, burst-queued) — external LLMs never write nodes/edges directly |
