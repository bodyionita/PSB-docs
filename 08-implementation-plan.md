# Implementation Plan

**Version:** 2.0 · **Status:** Approved 2026-07-12
**Rule:** ship in phases; every phase ends usable. A phase starts only when the previous
one's acceptance criteria pass. Code lives in `second-brain/` (monorepo, ADR-006).
**Process:** every session runs under [09-session-protocol.md](09-session-protocol.md)
(grill → record → pause; commit + push docs at pauses).

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

**M0 build decisions (grilled 2026-07-12 — [ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md),
[ADR-012](adr/012-m0-implementation-stack.md)):** M0 is split into (a) a **local-first
build** — complete `server/`/`web/`/`deploy/`, verified to **boot end-to-end locally**
(dockerized `pgvector` dev DB, `alembic upgrade head`, `/health` green, login/session,
registry fallback unit-tested with fakes, web builds + shell clicks through) — and (b) a
later **provisioning session** for the live VPS/Cloudflare/Supabase/GitHub + `claude login`
that satisfies the remote half of the accept criteria. Stack: **uv**, Alembic + plain SQL
(no ORM), Argon2id, **pnpm**; `claude-max` implemented but health-guarded. Deploy artifacts
are written now but dormant until (b).

**M0 progress (local-first build done — 2026-07-12).** Code monorepo `../second-brain/`
created (`server/` + `web/` + `deploy/` + CI, CLAUDE.md from template). Verified locally:

- **Server:** pydantic-settings config, asyncpg pool, Argon2id auth (login/session/logout +
  per-IP rate limit), health-guarded provider registry with fallback chain
  (`claude-max → nebius`, OpenAI-compatible client), `/health`. 21 unit tests pass (registry
  fallback with fakes, security, rate limit, config, migration-head); `ruff` clean.
- **DB:** dockerized `pgvector/pgvector:pg16` (`deploy/docker-compose.dev.yml`);
  `alembic upgrade head` applies revision `001` = full schema (11 tables, `vector` +
  `pgcrypto` extensions, HNSW cosine index) — confirmed in the running DB.
- **Boot flow:** `/health` green (db+vault); `POST /auth/login` sets an httpOnly cookie;
  `GET /auth/me` confirms the session; logout revokes it — all exercised against the live
  server, plus wrong-password/no-cookie → 401.
- **Web:** React+Vite+TS (strict) PWA, 5 themes, animated shell + auth + 4 screens.
  `pnpm build` (tsc strict + vite) and `pnpm lint` green; driven in a browser end-to-end
  (login → all four tabs switch → theme switch persists → logout). Fixed a real bug found
  during that drive: `AnimatePresence mode="wait"` hung transitions because screens contain
  infinite (`repeat`) animations that never complete an exit — switched to enter-only keyed
  transitions.
- **Toolchain pinned:** server on **uv**/pip + Python 3.12; web on **Node 24** (`.nvmrc`) +
  **pnpm 9** (`packageManager` field, Corepack). CI uses Node 24.

Deferred to the provisioning session (accepted M0 gap, ADR-012): live VPS/Cloudflare/
Supabase/GitHub remotes, `claude login` (real `claude-max` path), and the remote half of
the accept criteria. Deploy artifacts are written but unexercised.

## M1 — Capture end-to-end (usable week 1)

Capture endpoints + pipeline (transcribe → organize/split per plane → write notes with
frontmatter contract → index stub) + vault git auto-backup. Web: capture screen (record
button + visualizer, text input, recent-captures strip with live status, retry).

**Accept:** voice memo from the phone becomes correctly plane-classified note(s) in the
vault < 30s, visible in GitHub history; organizer failure still yields an Inbox note.

## M2 — Indexing & search

Chunking (pure, tested), indexer (hash skip, transactional upsert), full rescan +
`/admin/reindex`, `/search` with plane filters. Web: search UI (can ship inside chat screen).

**Accept:** reindex over a seeded vault; paraphrased query finds the right note; DB wipe +
reindex restores search; editing a note via git push is picked up by nightly pull+rescan.

## M3 — Chat

Chat endpoints + retrieval + citations + sessions + per-conversation model picker +
fallback banner. Web: chat screen with streaming render, source cards, model picker.

**Accept:** questions over known vault content answered with correct [n] citations on both
Claude and Nebius; "not in your notes" behavior verified; sessions persist.

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
web · related-notes suggestions & graph features · hybrid keyword+vector search ·
Cloudflare Access second wall · voice offline queue · entity extraction.

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
