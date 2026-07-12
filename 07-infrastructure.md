# Infrastructure & Operations

**Version:** 1.0 · **Status:** Approved 2026-07-12
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [003](adr/003-single-service-on-vps.md) · [007](adr/007-auth-password-session-cloudflare.md)

## Hosting

- **VPS:** Hetzner CX22 (2 vCPU / 4GB RAM / 40GB disk, ~5€/mo incl. IPv4). Ubuntu LTS.
  Cost comparison vs PaaS documented in ADR-003 (PaaS ≈ 2× price + fragile Claude CLI OAuth
  on ephemeral containers + single-host volumes anyway).
- **Web hosting:** the PWA is served by Caddy on the VPS, **single origin** (`/` = web,
  `/api` = API) — Vercel/Cloudflare Pages/Netlify were evaluated and rejected (no cost
  benefit for a single-user static PWA + they force cross-origin cookie rework of ADR-007).
  See [ADR-013](adr/013-web-stays-on-vps-single-origin.md).
- **DNS/TLS edge:** Cloudflare, proxied DNS (origin IP hidden). Caddy terminates TLS on
  origin (Cloudflare "Full (strict)"). Cloudflare Access = optional future second wall.
- **Database:** Supabase (managed Postgres + pgvector) — zero RAM cost on the VPS,
  managed backups ([ADR-002](adr/002-supabase-pgvector-for-index.md)).

## Topology on the VPS (Docker Compose)

```
deploy/docker-compose.yml
├── caddy      # :443 — serves web/dist statically, proxies /api → api:8000
└── api        # FastAPI + scheduler + agents (single container, ADR-003)
               # volumes: /srv/vault (vault git repo), /srv/data (audio files),
               #          claude CLI credentials volume
```

- The `api` image includes Python 3.12, the app, git, and the **Claude Code CLI**
  (provider `claude-max` shells the Agent SDK; OAuth done once via
  `docker compose exec api claude login`, credentials persisted on a volume).
- `ENABLE_SCHEDULER=true` on exactly one instance (there is only one).

## Vault backup (ADR-001)

- `/srv/vault` is a git repo with a private GitHub remote (deploy key, write access).
- Auto commit+push: after every pipeline write batch (debounced ~60s) + a 04:55 sweep +
  `POST /admin/backup`. Commit messages: `capture: 2 notes` / `slack-ingest: 3 notes` etc.
- Recovery: `git clone` → mount → `POST /admin/reindex`.
- Occasional manual exploration: clone/pull the same repo on laptop/phone, open in
  Obsidian (obsidian-git plugin for auto-pull/push).

## CI/CD (GitHub Actions on the code monorepo)

- **PR/push:** lint (ruff, eslint), typecheck (mypy optional, tsc), tests (pytest, vitest),
  web build.
- **Push to `main`:** all of the above → SSH to VPS → `git pull && docker compose up -d --build`.
  Rollback = revert commit and push.
- Docs repo has no pipeline (it's the contract, not a deployable).

## Secrets

All on the VPS in `deploy/.env` (never in git; `.env.example` documents every key):
`API_PASSWORD_HASH`, `SESSION_SECRET`, `DATABASE_URL`, `OPENAI_API_KEY`,
`NEBIUS_API_KEY`, `SLACK_USER_TOKEN`, `GITHUB_DEPLOY_KEY` (file), `VAULT_PATH`,
`PLANES`, model routing config. Claude Max credentials live only in the CLI volume.

## Observability

- Structured logs (JSON) from the api container → `docker logs` / journald.
- The product-level view is the **activity feed** (`agent_runs`) — operational visibility
  is a feature, not an afterthought (vision P8).
- `GET /health` checks db + vault + git remote; Cloudflare health notification (free) or
  UptimeRobot pings it.

## Provisioning (scripted, reproducible)

`deploy/provision.sh` (run once per fresh VPS): create user, harden SSH (keys only),
UFW (443/22), install Docker, clone repos, restore vault from GitHub, `claude login`,
start compose. Target: **full disaster recovery < 30 minutes**.
