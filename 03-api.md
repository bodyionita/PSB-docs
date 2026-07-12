# API Contract

**Version:** 2.1 · **Status:** Approved 2026-07-12 (2.1 = M1 capture additions: `GET /captures`
list, `POST /captures/{id}/follow-up`, follow-up fields, `/health` `backups` leg)

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

## Chat

| | |
|---|---|
| `GET /chat/models` | available chat models from provider registry: `[{ "id": "claude", "label": "Claude (Max)", "default": true }, { "id": "nebius/…", … }]` |
| `GET /chat/sessions` · `GET /chat/sessions/{id}` | history for the UI |
| `POST /chat` | `{ "message", "session_id"?: uuid, "model"?: id, "top_k"?: int }` → `{ "session_id", "answer", "model_used", "fallback_used", "sources": [{ note_path, title, snippet, score, planes }] }` — `model_used` may differ from requested when fallback fired (UI shows banner) |

## Search

| | |
|---|---|
| `POST /search` | `{ "query", "top_k"?, "planes"?: [..] }` → scored chunks, no LLM call |

## Activity feed

| | |
|---|---|
| `GET /activity?limit=50&before=<cursor>` | merged feed: agent runs + captures + errors, newest first. Run entries: `{ id, agent, status, started_at, finished_at, model_used, fallback_used, summary }` |
| `GET /activity/runs/{id}` | full run incl. `details` jsonb (expandable view) |

## Settings

| | |
|---|---|
| `GET /settings` | current app settings incl. agent model chain and available models |
| `PUT /settings/agents` | `{ "conspect_model": id, "fallback_model": id }` — applies to future runs; separate from per-chat picker by design |

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
| `POST /admin/reindex` | vault reconciliation → `{ indexed, skipped, deleted }` |
| `POST /admin/backup` | force vault git commit+push → `{ committed, pushed }` |
| `POST /admin/captures/{id}/reorganize` | maintenance re-run: re-organize a capture's stored raw text and **replace** its notes (e.g. re-deriving notes after the English-only organizer change). `202` → `{ capture_id }`; `404` if unknown. Raw input untouched (never-lose); notes replaced only on a successful organize, kept on the Inbox-fallback |
| `GET /health` | no auth: `{ status, db, vault, git_remote, backups }`, `503` when degraded. `backups` (M1, [ADR-014](adr/014-vault-history-durability.md) §6) is false when the latest `integrity-drill` run failed or is overdue (>~8 days) |
