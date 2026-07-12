# Infrastructure & Operations

**Version:** 1.0 · **Status:** Approved 2026-07-12
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [003](adr/003-single-service-on-vps.md) · [007](adr/007-auth-password-session-cloudflare.md)

## Hosting

- **VPS:** Hetzner **CX23** (2 vCPU / 4GB RAM / 40GB NVMe, ~€5.49/mo + €0.50 IPv4), EU,
  Ubuntu LTS ([ADR-015](adr/015-compute-tier-hetzner-cx23.md)). 4GB is the comfortable floor:
  the Node-based Claude CLI (ADR-004) drives RAM, bursting on every `claude-max` chat and the
  nightly window — sub-4GB risks OOM and swap thrash in exactly that window. Free tiers
  (Oracle Always Free) were rejected: idle-reclaim targets our bursty workload; ADR-014 keeps
  data safe through a reclaim but not uptime. Cost comparison vs PaaS is in ADR-003.

### All-in monthly cost

| Component | €/mo |
|---|---|
| VPS — Hetzner CX23 (x86, 4GB) | ~5.49 |
| IPv4 | 0.50 |
| Web hosting (Caddy on the same box — [ADR-013](adr/013-web-stays-on-vps-single-origin.md)) | 0 |
| Vault WORM bundle + DB `pg_dump` + raw-input sync — Cloudflare R2 ([ADR-014](adr/014-vault-history-durability.md)) | 0 (free tier) |
| Database — Supabase | 0 (free tier) |
| DNS / TLS / proxy — Cloudflare | 0 (free tier) |
| Domain (~€10–12/yr, amortized) | ~1 |
| **Total** | **~€7/mo** |
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

## Vault backup & history durability (ADR-001 + [ADR-014](adr/014-vault-history-durability.md))

**Live copy — GitHub (rewritable primary):**
- `/srv/vault` is a git repo with a private GitHub remote (deploy key, write access).
- Auto commit+push: after every pipeline write batch (debounced ~60s) + a 04:55 sweep +
  `POST /admin/backup`. Commit messages: `capture: 2 notes` / `slack-ingest: 3 notes` etc.
- The server only fast-forward pushes — **never force/rebase/reset**; on non-fast-forward it
  pulls (merge) and re-pushes (healing GitHub if a client rewound it). gc/reflog pinned:
  `gc.pruneExpire=never`, `gc.reflogExpire=never`, `gc.reflogExpireUnreachable=never`, auto-gc off.
- `.gitignore` excludes **only** `.obsidian/workspace*` + OS cruft — **not `.trash`, not notes**
  (indexer `vault_ignore` ≠ git ignore; soft-deleted notes stay committed).

**Immutable copy — Cloudflare R2 (WORM, the never-lose guarantee):**
- Nightly `git bundle --all` (full history, self-contained, `git bundle verify`-able) →
  R2 bucket with **versioning + object-lock**. A checkpoint bundle is also taken at the
  start of the agent window (after pull), before any reorganization. Object-lock makes the
  history append-only at the storage layer — immune to force-push, rebase, or account loss.
- A **fingerprint** (HEAD sha, monotonic commit count, file count) is recorded per bundle
  (`agent_runs` + manifest object) for drills.
- **Weekly integrity drill** on the VPS: `git bundle verify` + clone + fingerprint check on
  both R2 and GitHub; failure → `failed` `agent_run` + `/health` degraded. Fast-follows:
  monthly full-restore drill in GitHub Actions, semi-annual manual DR rehearsal.

**Operational-state backup (second independent copy, not never-lose):**
- Nightly `pg_dump` → R2 (versioned) so Supabase isn't a SPOF; restore-to-last-nightly.
- Nightly sync of `/srv/data` raw inputs (audio) → R2 so un-transcribed input isn't VPS-disk-only.

**Recovery:** `git clone` (GitHub or the R2 bundle) → mount → `POST /admin/reindex`
(+ `pg_dump` restore for operational history). Occasional manual exploration: clone/pull the
same repo on laptop/phone, open in Obsidian (obsidian-git, **merge-only**, never force-push).

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
`PLANES`, model routing config, and **R2 backup creds** (`R2_ACCOUNT_ID`,
`R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET` — [ADR-014](adr/014-vault-history-durability.md)).
Claude Max credentials live only in the CLI volume. The monthly CI restore drill (fast-follow)
needs R2-read + `OPENAI_API_KEY` as GitHub Actions secrets.

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
