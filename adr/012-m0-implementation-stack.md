# ADR-012: M0 implementation stack & local-first delivery

**Status:** Accepted · 2026-07-12
**Related:** [008-implementation-plan](../08-implementation-plan.md) ·
[ADR-004](004-provider-registry-claude-primary-nebius-fallback.md) ·
[ADR-006](006-monorepo-with-strict-server-web-decoupling.md) ·
[ADR-011](011-alembic-migrations-plain-sql-no-orm.md)

## Context
M0's acceptance criteria are stated against a live VPS (HTTPS, Cloudflare, Supabase,
GitHub, `claude login`). Those external accounts are deliberately deferred (README cold
start §5) and the first build happens on the developer's local Windows machine. This ADR
pins the concrete toolchain and the local-first delivery split agreed during grilling, so a
fresh session inherits them without re-deciding.

## Decision

**Delivery split.** Build the *complete* M0 codebase (`server/`, `web/`, `deploy/`) and
verify it **boots end-to-end locally**; defer live provisioning (Hetzner, Cloudflare,
Supabase project, GitHub remotes, VPS `claude login`) to a separate hands-on session.
Everything ships ready-to-deploy; the deploy artifacts are written now but dormant.

**Server toolchain.**
- **uv** for env + lockfile + running (`pyproject.toml` + `uv.lock`); same in the Dockerfile.
- Python 3.12, FastAPI, asyncpg (plain SQL), pydantic-settings, APScheduler, ruff.
- **Argon2id** (`argon2-cffi`) for the login password hash; a `scripts/hash_password.py`
  generates `API_PASSWORD_HASH`.
- Migrations: **Alembic, plain SQL, no ORM** ([ADR-011](011-alembic-migrations-plain-sql-no-orm.md)).

**Local dev database.** Dockerized `pgvector/pgvector:pg16` via
`deploy/docker-compose.dev.yml` on localhost. Prod swaps `DATABASE_URL` → Supabase; same
extension, same migrations, disposable.

**Provider registry in M0.** Full registry with real interfaces and a working
OpenAI-compatible client (serves Nebius + OpenAI). `claude-max` (Agent SDK / CLI) is
**implemented but health-guarded**: absent/failed credentials ⇒ the chain falls back and
records `model_used` / `fallback_used`. On the VPS, `claude login` lights up the real path
with **zero code change**. Fallback is verified locally with **fake providers in unit
tests** — no live LLMs in CI ([08-implementation-plan.md](../08-implementation-plan.md)
testing policy). `/health` never calls an LLM, so the service boots with no API keys.

**Web toolchain.** React + Vite + TypeScript (strict), **pnpm**, TanStack Query,
framer-motion. M0 scope = full premium *foundation* with stubbed features: themed auth
screen wired to real `/auth/login` + `/auth/me`, animated app shell, four placeholder
screens (Capture/Chat/Activity/Settings). A single `api/` module is the only place that
knows server URLs (ADR-006). See [06-web-app.md](../06-web-app.md) for themes + name.

**Verification bar for M0 (this session).** docker dev DB up → `alembic upgrade head` →
FastAPI up → `GET /health` green → `POST /auth/login` sets cookie → `GET /auth/me` confirms
session → registry fallback unit tests pass → `pnpm build` + web shell clicks through.

## Consequences
- ✅ A fully working, locally-verified foundation; "accounts + money + DNS" isolated to its
  own session.
- ✅ Dev/prod parity on Python (uv), DB (pgvector), and the provider seam.
- ⚠️ Deploy artifacts (Dockerfiles, compose, Caddy, provision.sh, CI) are written but
  unexercised until provisioning — a known, accepted gap for M0.
- ⚠️ Real `claude-max`, Supabase, Cloudflare, and Slack behavior are unverified until the
  provisioning session.
