# Infrastructure & Operations

**Version:** 1.3 · **Status:** Approved 2026-07-13 (1.3 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md): vault → **graph store** (`/srv/graph-store`,
`GRAPH_STORE_PATH`) with a **fresh GitHub repo** (old `PSB-vault` archived, its R2 bundles retained);
Obsidian references removed — renames land with M3. 1.2 = M2 Accept `ollama-init`; 1.1 = `ollama` sidecar
[ADR-022](adr/022-embeddings-self-hosted-nomic.md))
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [003](adr/003-single-service-on-vps.md) · [007](adr/007-auth-password-session-cloudflare.md) · [022](adr/022-embeddings-self-hosted-nomic.md)

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
| Graph-store WORM bundle + DB `pg_dump` + raw-input sync — Cloudflare R2 ([ADR-014](adr/014-vault-history-durability.md)) | 0 (free tier) |
| Database — Supabase | 0 (free tier) |
| DNS / TLS / proxy — Cloudflare | 0 (free tier) |
| Domain (~€10–12/yr, amortized) | ~1 |
| **Total** | **~€7/mo** |
- **Web hosting:** the PWA is served by Caddy on the VPS, **single origin** (`/` = web,
  `/api` = API) — Vercel/Cloudflare Pages/Netlify were evaluated and rejected (no cost
  benefit for a single-user static PWA + they force cross-origin cookie rework of ADR-007).
  See [ADR-013](adr/013-web-stays-on-vps-single-origin.md).
- **DNS/TLS edge:** Cloudflare, proxied DNS (origin IP hidden). Caddy terminates TLS on
  origin with a **Cloudflare Origin CA** certificate (Cloudflare "Full (strict)") — explicit
  `tls` directive, no ACME; cert+key delivered by CI ([ADR-017](adr/017-tls-cloudflare-origin-ca.md)).
  Origin needs **:443 only** (no :80). Cloudflare Access = optional future second wall.
- **Database:** Supabase (managed Postgres + pgvector) — zero RAM cost on the VPS,
  managed backups ([ADR-002](adr/002-supabase-pgvector-for-index.md)).

## Topology on the VPS (Docker Compose)

```
deploy/docker-compose.yml
├── caddy      # :443 — serves web/dist statically, proxies /api → api:8000
├── api        # FastAPI + scheduler + agents (single app container, ADR-003)
│              # volumes: /srv/graph-store (graph-store git repo — /srv/vault pre-M3),
│              #          /srv/data (audio files),
│              #          claude CLI credentials volume
├── ollama     # M2 — self-hosted embeddings (nomic-embed-text-v1.5), ADR-022
│              # OpenAI-compatible /v1/embeddings on the internal network;
│              # model volume; not exposed off-box (no published port); healthchecked
└── ollama-init # M2 — one-shot: pulls the embedding model into ollama's volume, then exits.
               # Runs on every `up` (self-healing, idempotent no-op once present) so a fresh
               # box / recreated volume needs no manual `ollama pull` (ADR-022)
```

- The `api` image includes Python 3.12, the app, git, and the **Claude Code CLI**
  (provider `claude-max` shells the Agent SDK; OAuth done once via
  `docker compose exec api claude login`, credentials persisted on a volume).
- **`ollama` (M2, [ADR-022](adr/022-embeddings-self-hosted-nomic.md)):** an infra sidecar (not an
  app-logic split — consistent with [ADR-003](adr/003-single-service-on-vps.md), which is about the
  *application* being one service). The `api` container reaches it as an `OpenAICompatibleProvider`
  at `http://ollama:11434/v1` (base-URL config, no API key). The model is pulled **automatically**
  by the `ollama-init` one-shot (gated on the ollama healthcheck), which runs on every `up` and is
  a fast idempotent no-op once the blobs are present — so a fresh box or a recreated `ollama_models`
  volume self-provisions with **no manual `ollama pull`**. Resident ~0.1–0.3 GB on the 4 GB box.
  The app **tolerates ollama-not-ready** — a capture's index step fails retryably and the next
  rescan converges; a transient outage never corrupts the index (single-provider, one vector space).
  *(History: the live M2 Accept caught the prod box running without the model — the sidecar predated
  M2 provisioning — surfacing as a `partial`/all-embeds-failed reindex; `ollama-init` removes that
  class of manual-step gap for good.)*
- `ENABLE_SCHEDULER=true` on exactly one instance (there is only one).

## Graph-store backup & history durability (ADR-001 + [ADR-014](adr/014-vault-history-durability.md) — machinery unchanged by the pivot)

**Live copy — GitHub (rewritable primary):**
- `/srv/graph-store` is a git repo with a private GitHub remote (deploy key, write access) —
  the **fresh `PSB-graph` repo**, created at M3 ([ADR-031](adr/031-m3-organizer-and-contract-extensions.md);
  the old `PSB-vault` repo is archived read-only after the M3 Accept, its R2 WORM bundles retained,
  new bundles land as `graph-store-*` in the same WORM bucket).
- **Zero-touch cutover (ADR-031):** the app bootstraps the store itself (idempotent: init +
  skeleton + `push -u` to `GRAPH_STORE_REPO` from `defaults.env`); the deploy workflow **prints
  the VPS's existing deploy public key into the Actions log** so the user's entire manual surface
  is GitHub UI — create `PSB-graph`, paste the key (write access), archive `PSB-vault` later.
  No SSH, no VPS steps.
- Auto commit+push: after every pipeline write batch (debounced ~60s) + a 04:55 sweep +
  `POST /admin/backup`. Commit messages: `capture: 2 nodes` / `chat-distill: 1 node` etc.
- The server only fast-forward pushes — **never force/rebase/reset**; on non-fast-forward it
  pulls (merge) and re-pushes (healing GitHub if a client rewound it). gc/reflog pinned:
  `gc.pruneExpire=never`, `gc.reflogExpire=never`, `gc.reflogExpireUnreachable=never`, auto-gc off.
- `.gitignore` excludes **only** OS cruft — **not `.trash`, not nodes** (indexer `store_ignore`
  ≠ git ignore; soft-deleted nodes stay committed). The `*.md eol=lf` `.gitattributes` is kept
  (correct regardless of editor).

**Immutable copy — Cloudflare R2 (WORM, the never-lose guarantee):**
- Nightly `git bundle --all` (full history, self-contained, `git bundle verify`-able) →
  R2 bucket with **versioning + object-lock** (on R2 the WORM mechanism is a **bucket lock
  rule**, added post-creation in bucket Settings — there is no S3-style create-time toggle;
  keep retention bounded, as a lock rule overrides lifecycle deletes). A checkpoint bundle is also taken at the
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
(+ `pg_dump` restore for operational history). Occasional manual exploration/editing: clone the
repo with any editor — any external client must be **merge-only, never force-push**
(ADR-014 §4, generalized by [ADR-026](adr/026-graph-native-storage-obsidian-removed.md)).

## CI/CD (GitHub Actions on the code monorepo)

- **PR/push:** lint (ruff, eslint), typecheck (mypy optional, tsc), tests (pytest, vitest),
  web build.
- **Push to `main`** (or manual `workflow_dispatch`): all of the above → render `.env` + origin
  TLS files → `scp` them to the VPS → **`scp` the built `web/dist`** (uploaded as an artifact by
  the `web` job; `web/dist` is gitignored, so the box's clone never has it — CI is its sole
  delivery path, mirroring the `.env`/cert flow) → SSH to VPS → `git pull --ff-only && docker
  compose up -d --build && alembic upgrade head`. Caddy then serves `/srv/app/web/dist` at `/`.
  Rollback = revert commit and push.
- Docs repo has no pipeline (it's the contract, not a deployable).

## Secrets & config ([ADR-016](adr/016-secrets-via-github-actions-ci-renders-env.md))

`deploy/.env` on the VPS is the file the stack reads (docker-compose `env_file`), but it is
**rendered by CI, never hand-filled and never in git** ([ADR-016](adr/016-secrets-via-github-actions-ci-renders-env.md)):

- **Secrets → GitHub Actions Secrets** (a `production` Environment; entered once in the UI,
  never seen by the agent, never committed): `API_PASSWORD_HASH`, `SESSION_SECRET`,
  `DATABASE_URL`, `OPENAI_API_KEY`, `NEBIUS_API_KEY`, `GROQ_API_KEY` (STT primary, M1 replan
  [ADR-020](adr/020-stt-fallback-chain-groq-primary.md)), `SLACK_USER_TOKEN`, and the **R2 backup
  creds** `R2_ACCOUNT_ID` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` ([ADR-014](adr/014-vault-history-durability.md)),
  the **origin TLS** PEM material `ORIGIN_CERT_PEM` / `ORIGIN_KEY_PEM` ([ADR-017](adr/017-tls-cloudflare-origin-ca.md)),
  plus deploy secrets `VPS_HOST` / `VPS_USER` (= `deploy`, [ADR-018](adr/018-vps-ssh-hardening-non-root-deploy-user.md)) / `VPS_SSH_KEY`.
- **Non-secret config → git** in `deploy/defaults.env` (versioned, reviewable):
  `BRAINDAN_DOMAIN`, `PLANES`, `NEBIUS_CHAT_MODEL`, `CLAUDE_MAX_MODEL`, `CLAUDE_MAX_EFFORT`
  (default `medium`, M1 replan), `STT_CHAIN` (default `groq,openai`), `GROQ_BASE_URL`,
  `GROQ_STT_MODEL` (default `whisper-large-v3`) — all [ADR-020](adr/020-stt-fallback-chain-groq-primary.md)/replan —
  `SCHEDULER_TZ`, `GRAPH_STORE_PATH` + `GRAPH_STORE_REPO` (ex-`VAULT_PATH`, renamed/added at M3 —
  [ADR-031](adr/031-m3-organizer-and-contract-extensions.md)), `NODE_TYPES`, `EDGE_RELS`,
  `ENTITY_MATCH_MIN_CONF`, `SESSION_COOKIE_SECURE`, `ENVIRONMENT`, `R2_BUCKET`; **M5 adds the
  MCP bearer-token secret** ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)) to the
  Actions secrets set.
- **Deploy renders** `.env = defaults.env + secrets`, `scp`s it to the VPS mode 600, then
  `compose up`. It **also renders the origin TLS files** `deploy/origin.crt` / `origin.key`
  from `ORIGIN_CERT_PEM` / `ORIGIN_KEY_PEM` and `scp`s them mode 600 for Caddy to mount
  ([ADR-017](adr/017-tls-cloudflare-origin-ca.md)). `.env.example` documents the full key set.

Not in this flow: **Claude Max** credentials live only in the CLI volume (`claude login`);
the graph-store git **deploy key** (`GITHUB_DEPLOY_KEY`, a file) is generated on the VPS and its
private half never leaves the box. The monthly CI restore drill (fast-follow) reuses the
GitHub `R2_*` + `OPENAI_API_KEY` secrets.

## Observability

- Structured logs (JSON) from the api container → `docker logs` / journald.
- The product-level view is the **activity feed** (`agent_runs`) — operational visibility
  is a feature, not an afterthought (vision P8).
- `GET /health` checks db + store + git remote (+ `backups`, [03-api](03-api.md)); Cloudflare health notification (free) or
  UptimeRobot pings it.

## Provisioning (scripted, reproducible)

`deploy/provision.sh` (run once per fresh VPS, as root) **preps the box**: create the non-root
**`deploy`** user (`docker` + `sudo` groups, owns `/srv/*`; operator + CI pubkeys in its
`authorized_keys`), **harden SSH** (`PermitRootLogin no`, `PasswordAuthentication no`, keys-only —
applied last, guarded: verify `authorized_keys` non-empty → `sshd -t` → reload → warn to test a
second session) per [ADR-018](adr/018-vps-ssh-hardening-non-root-deploy-user.md), UFW (**443/22**),
install Docker, clone repos, generate + register the graph-store deploy key, restore the graph store from GitHub,
`claude login`. It **does not write `deploy/.env`** — CI is the sole writer
([ADR-016](adr/016-secrets-via-github-actions-ci-renders-env.md)). **App start comes from the
deploy workflow**, so **disaster recovery = provision the box + trigger the deploy** (renders
`.env` + origin TLS files, `compose up`, `alembic upgrade head`). Target: **full disaster
recovery < 30 minutes**.
