# API Contract

**Version:** 2.0 · **Status:** Approved 2026-07-12

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
| `POST /capture/voice` | multipart `file` (m4a/webm/ogg/mp3/wav) → same `202` body |
| `GET /captures/{id}` | pipeline state: `{ capture_id, kind, status, raw_text, note_paths[], error, created_at, updated_at }` |
| `POST /captures/{id}/retry` | re-run from first incomplete step; `409` unless `failed` |

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

## Agents & admin

| | |
|---|---|
| `GET /agents` | registered connectors/jobs + schedule + last run |
| `POST /agents/{name}/run` | trigger on demand (e.g. `slack-ingest`, `daily-summary`); `409` if already running |
| `POST /admin/reindex` | vault reconciliation → `{ indexed, skipped, deleted }` |
| `POST /admin/backup` | force vault git commit+push → `{ committed, pushed }` |
| `GET /health` | no auth: `{ status, db, vault, git_remote }`, `503` when degraded |
