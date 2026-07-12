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

**M0b provisioning progress (in flight — 2026-07-12).** Live bring-up, step by step. No
secrets recorded here (ADR-016 / protocol §Security):

- **Domain:** `braindan.cc` registered via Cloudflare Registrar; zone active; **apex `A` record
  proxied** (orange-cloud) → the VPS. Origin IP is held out of the docs (hidden behind
  Cloudflare); it goes into the GitHub `VPS_HOST` secret at Step 8.
- **VPS:** Hetzner **CX23** (2 vCPU / 4 GB, Ubuntu 24.04, Germany), SSH keys-only, Hetzner
  firewall 22+443, backups off (ADR-014/015). **Not yet provisioned** — `provision.sh` not run.
- **Supabase:** project (free tier), region `eu-central-1`, ref `aegauzpsyybfknddxlbw`.
  Connection = **Session pooler + `?sslmode=require`** (the `DATABASE_URL` is a secret, not
  stored here). `vector`/`pgcrypto` left for migration 001 (not dashboard-enabled).
- **Code repo:** `PSB` on GitHub, `main` pushed; CI (lint/test/build + **gitleaks** secrets
  job) live. **Made PUBLIC 2026-07-12** so the VPS clones/pulls `/srv/app` anonymously over
  HTTPS — no code-repo deploy key needed. Safe because no secret is ever in git (ADR-016;
  hook + gitleaks on full history; verified only `*.env.example` + non-secret `defaults.env`
  are tracked). Reversible: to go private, add a read-only PSB deploy key on the box and use
  the SSH remote. Vault-backup repo: `PSB-vault` (deploy key generated on the VPS at Step 9).
- **Backups (R2) — Step 7 DONE 2026-07-12:** bucket **`braindan-backups`** created + an
  **Object Read & Write** API token scoped to it (creds are Step 8 secrets `R2_ACCOUNT_ID` /
  `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY`, not stored here). WORM immutability is an R2
  **bucket lock rule** (Settings → Bucket lock rules), added **post-creation** — R2's equivalent
  of S3 "object-lock" named in ADR-014/07; there is no create-time toggle. Retention kept
  **bounded** (not indefinite) since a lock rule overrides lifecycle deletes and blocks emptying.
- **Secrets/deploy:** per [ADR-016](adr/016-secrets-via-github-actions-ci-renders-env.md) —
  GitHub `production` environment (**not yet populated**); CI renders `deploy/.env`. The
  ADR-016 implementation was **independently reviewed**; 2 must-fix (secret-rendering `$`
  corruption; transport vs ADR §3) + a caddy `BRAINDAN_DOMAIN` gap were fixed (code committed
  **and pushed** — `origin/main` at `70f5d63`).

**Remaining M0b:** ~~Step 7 R2 bucket + token~~ (done) → ~~Step 8 populate `production` secrets~~
(done — all set **except `VPS_HOST`, intentionally held** so the deploy job stays dormant until
the box is prepared; `ORIGIN_CERT_PEM`/`ORIGIN_KEY_PEM` in; `VPS_USER=deploy`; `SLACK_USER_TOKEN`
deferred to M4) → **Step 9 (next)** run `provision.sh` **as root** — a two-pass vault-key bootstrap: bootstrap
= `apt install git` + clone public `PSB` to `/srv/app`, then `CI_DEPLOY_PUBKEY=<psb_ci_deploy.pub>
bash deploy/provision.sh`; **pass 1** installs Docker + `deploy` user + UFW + prints the vault
deploy pubkey, then **defers hardening** (root stays enabled, since the vault can't clone until
its key is on GitHub); add that pubkey to `PSB-vault` deploy keys (write); **pass 2** (re-run as
root) clones the vault and hardens SSH (final). `passwd deploy` for sudo. Then set `VPS_HOST` →
trigger deploy (`workflow_dispatch`) → Step 10 `claude login` → Step 11 Cloudflare **Full
(strict)** + verify. Note: prod `/health` green requires the vault git remote present (verified
in `system_health.py`), so the vault clone is not optional.

CI trigger gap fixed 2026-07-12: `ci.yml` gained `workflow_dispatch` + deploy runs on it
(`origin/main` `ff7ad4b`), matching `provision.sh`'s "Actions → Run workflow" instruction.
Web-delivery gap fixed 2026-07-12 (`702c5f6`): `web/dist` is gitignored, so the `web` job
uploads it as an artifact and the `deploy` job downloads + `scp`s it to `/srv/app/web/dist`
before `compose up` — otherwise the box would serve an API with no PWA (M0b "serves the PWA").

**Open decisions — RESOLVED 2026-07-12 (planning pass, grilled):**
- **TLS cert method** → **Cloudflare Origin CA**, cert+key CI-rendered via the ADR-016 path
  ([ADR-017](adr/017-tls-cloudflare-origin-ca.md)).
- **SSH hardening** → non-root **`deploy`** user (docker+sudo, owns `/srv/*`), root SSH
  disabled, keys-only, automated + guarded in `provision.sh`
  ([ADR-018](adr/018-vps-ssh-hardening-non-root-deploy-user.md)).

**Code for ADR-017/018 — DONE 2026-07-12 (implementation session; pushed — `origin/main` at `ae08d43`).**
In `../second-brain/`: `deploy/Caddyfile` → explicit `tls /etc/caddy/origin.crt
/etc/caddy/origin.key` (no ACME); `deploy/docker-compose.yml` → dropped `:80`, mounts the two
cert files read-only; `.github/workflows/ci.yml` deploy job → renders `origin.crt`/`origin.key`
(printf, no shell re-parse) and `scp`s them mode 600 alongside `.env`; `deploy/provision.sh` →
creates the `deploy` user (docker+sudo, owns `/srv/*`), seeds `authorized_keys`, guarded sshd
hardening as the final step; `deploy/.env.example` documents the two PEM secrets; `.gitignore`
+ `.githooks/pre-commit` make `origin.{crt,key}` untrackable.

- **Independent review (fresh agent, diff vs ADR-016/017/018 + CLAUDE.md invariants): no
  must-fix findings.** Secrets discipline holds, the cert path triangle (Caddy `tls` ↔ compose
  mount ↔ scp target) is consistent, hardening is guarded/fail-safe. 6 minor items raised; 5
  applied (widened the authorized_keys guard regex to cover FIDO2 `sk-`/options-prefixed keys;
  warn when only the CI key is present; tolerate an sshd reload-command miss; `*.crt` added to
  the pre-commit block; ADR-017 wording). **Logged follow-up (not blocking):** running the
  *production* compose by hand before CI renders the cert files makes Docker create dirs at
  `origin.{crt,key}` — harmless in the CI flow (scp precedes `up`), noted for manual use.

## M1 — Capture end-to-end (usable week 1)

Capture endpoints + pipeline (transcribe → organize/split per plane → write notes with
frontmatter contract → index stub) + vault git auto-backup **per [ADR-014](adr/014-vault-history-durability.md)**
(fast-forward-only push, gc/reflog pinned, atomic writes, `.trash` git-tracked, nightly
`git bundle --all` → R2 WORM, `/srv/data` + `pg_dump` → R2, weekly integrity drill;
monthly-CI restore + DR rehearsal are fast-follows). Web: capture screen (record
button + visualizer, text input, recent-captures strip with live status, retry).

**Accept:** voice memo from the phone becomes correctly plane-classified note(s) in the
vault < 30s, visible in GitHub history; organizer failure still yields an Inbox note; a
nightly WORM bundle lands in R2 and the weekly integrity drill passes the fingerprint check.

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
Cloudflare Access second wall · voice offline queue · entity extraction ·
**conversational capture** (short LLM "interviewer/therapist" nudges during voice ingestion
— e.g. "expand on this", "how did you feel when that happened" — to draw out missing detail;
gentle, not strict; grill UX + prompt design at the capture-pipeline stage) ·
backup fast-follows (monthly CI restore drill, semi-annual DR rehearsal — [ADR-014](adr/014-vault-history-durability.md)).

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
