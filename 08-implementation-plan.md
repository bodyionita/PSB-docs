# Implementation Plan

**Version:** 2.2 · **Status:** Approved 2026-07-13 (2.2 = M2 grilled to build-ready detail —
[ADR-022](adr/022-embeddings-self-hosted-nomic.md)/[023](adr/023-semantic-relatedness-graph.md)/[024](adr/024-tag-vocabulary-reuse-and-consolidation.md);
M1 close folded into M2; 2.1 = M1 grilled to build-ready detail; v2 backlog additions)
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

> **Amendment (2026-07-12, replan):** the last clause — the *live* Claude-limit → Nebius
> chain-and-record — **cannot be exercised in M0** and is **deferred to M3 (chat)**. M0 has
> **no chat/admin surface, no chain caller, and no `agent_runs` writer** (routers = `auth` +
> `health` only), so nothing invokes `.complete()` or records a run. For M0 this clause is
> satisfied by the **21 registry unit tests** (fallback chain with fakes); the live
> claude-max path + "records it" are verified in M3 when the chat endpoint exists. Consistent
> with [ADR-012](adr/012-m0-implementation-stack.md) (real `claude-max` unverified until a
> live surface). `claude login` (Step 10) is already done on the box, so M3 lights up with
> zero further infra work. **All other M0 accept clauses pass** (HTTPS PWA, login, `/health`).

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
- **Code repo:** `PSB` on GitHub, `main` pushed. **CI caveat:** the workflow file was
  **invalid YAML from the ADR-016 deploy step until `e30cdc0` (2026-07-12)** — an unquoted
  colon in a step name — so **no CI jobs actually ran in that window** (server/web/gitleaks/
  deploy all skipped). Fixed + validated with a real YAML parser; the next push re-enables CI.
  **Made PUBLIC 2026-07-12** so the VPS clones/pulls `/srv/app` anonymously over HTTPS — no
  code-repo deploy key needed. Public-safety was confirmed **directly** (`git ls-files`: only
  `*.env.example` placeholders + non-secret `defaults.env` tracked; no secret ever in git per
  ADR-016 + pre-commit hook) — not via gitleaks, which wasn't running; gitleaks now re-enabled
  gives the automated backstop. Reversible: to go private, add a read-only PSB deploy key on
  the box + SSH remote. Vault-backup repo: `PSB-vault` (deploy key generated on the VPS at Step 9).
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

**Steps 9–9.5 DONE — DEPLOY IS LIVE 2026-07-12.** `provision.sh` ran (two-pass, `deploy` user +
SSH hardened + vault cloned); `VPS_HOST` set; the deploy workflow runs green. **The PWA is served
over HTTPS at `https://braindan.cc`** and **`GET /api/v1/health` = 200, all three legs green**
(`db` ✓ Supabase + migration 001 applied, `vault` ✓, `git_remote` ✓). Getting the first green
deploy took a chain of fixes to things that were latent while CI never actually ran (the workflow
had been **invalid YAML since the ADR-016 step** — `e30cdc0` — so no jobs executed):
- `ci.yml` **invalid YAML** (unquoted colon in a step name) → nothing ran (`e30cdc0`).
- **web CI** `pnpm` version not found (action can't read `web/package.json` through
  working-directory) → pin `9.15.0`; **server CI** ruff/pytest are an *extra* that plain
  `uv sync` skips → `uv sync --extra dev` (`682ca00`).
- **scp** `tar: empty archive` (multiline `source:` didn't parse) → comma-separated list
  (`a0bf990`); then `Permission denied` (`umask 077` → 600, scp container is another user) →
  render 644 on the ephemeral runner, VPS chmod 600 stays (`5950c3e`).
- **`alembic`** `TypeError: connect() got unexpected kwarg 'sslmode'` — SQLAlchemy+asyncpg forwards
  `sslmode`; raw asyncpg (app) parses it fine, so only `migrations/env.py` needed to translate
  `sslmode` → asyncpg `ssl=` connect_arg (`881346e`). DATABASE_URL secret unchanged.
- **`/health` `git_remote:false`** — git **dubious ownership** (`deploy`-owned vault, root
  container) → `Dockerfile` `git config --global --add safe.directory /srv/vault` (`ed4a7ff`);
  also unblocks M1 vault backup.
- **Compose interpolating the `$`-laden argon2 hash** (WARN spam; would corrupt
  `API_PASSWORD_HASH`) → project interpolation uses non-secret `defaults.env` (`ed4a7ff`) **and**
  the api `env_file` uses **`format: raw`** (`d9700a1`) so secrets pass literally.

**Remaining for M0/M0b accept:** ~~(a) **verify login** in the browser after the `d9700a1`
redeploy (confirms the raw-hash fix end-to-end)~~ — **DONE 2026-07-12**: CI `d9700a1`
(run 29197844854) green on all four jobs (`server`/`secrets`/`web`/`deploy`); live
`/api/v1/health` = `{status:ok, db:true, vault:true, git_remote:true}`; PWA root 200 over HTTPS;
login mechanism confirmed (wrong password → clean `401 "Invalid password"`, **not** a `500` —
proves the raw-hash argon2 verify parses) **and real-password login confirmed working in the
browser by the user**. The `d9700a1` raw-hash fix is verified end-to-end.
~~(b) **Step 10** `claude login` on the box~~ — **DONE 2026-07-12** (user ran
`docker compose exec api claude login`; OAuth creds persisted on the CLI volume). Effect is
**not observable in M0** — no chain caller exists — but persists for M3; see the Accept
amendment above.
~~(c) the accept criterion **"Claude-limit simulation → chain answers via Nebius and records
it"**~~ — **DEFERRED to M3** (replan 2026-07-12): M0 has no chat surface / chain caller /
`agent_runs` writer, so it can't be exercised live; satisfied for M0 by the 21 registry unit
tests. See the Accept amendment above.
~~(d) **Step 11** confirm Cloudflare SSL mode = **Full (strict)**~~ — **DONE + VERIFIED
2026-07-12**: site loads `HTTP/2 200` via Cloudflare with health green and **no `526`**, so
the edge validates the Origin CA cert (strict confirmed end-to-end from outside).

**→ M0 / M0b ACCEPT COMPLETE (2026-07-12)** — all live clauses pass (HTTPS PWA, login,
`/health` green, TLS Full-strict); the live claude-max chain-and-record clause is formally
deferred to M3 per the amendment. Next milestone is **M1** (see below).

**Follow-up (M1, non-blocking):** give `PSB-vault` an **initial commit/branch**
so the auto-backup has a branch to push to (empty repo clones fine and health is green, but M1
push needs a HEAD).

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

**M1 build decisions (grilled 2026-07-12 — [ADR-019](adr/019-conversational-capture-minimal-in-m1.md);
implements [ADR-014](adr/014-vault-history-durability.md), [ADR-005](adr/005-planes-and-atomic-notes.md)).**
Build-ready detail below; nothing here is left to implementer discretion.

- **Execution model.** API returns `202`; the pipeline runs **in-process** via a
  `CapturePipeline` service (`asyncio.create_task`), state in `captures.status`. No broker.
  **Boot-time sweep** marks orphaned in-flight captures (non-terminal, non-`failed`) as
  `failed` ("interrupted by restart") — retryable, nothing hangs silently (rule 7).
- **Pipeline order.** `transcribe` (voice only) → `organize` → `write` → `index` (stub) →
  **then** generate the follow-up nudge as a trailing, non-blocking step (notes land first,
  protecting <30s). **Never-lose:** the `captures` row (raw_text, or audio saved under
  `DATA_PATH` as `{id}.{ext}`) is persisted **before any model call**. Audio capped at 25 MB
  (whisper limit) → clear error if exceeded.
- **Transcription.** `registry.transcribe()` → `openai`/whisper, **no fallback**; STT-down ⇒
  capture `failed` + retry (accepted for M1).
- **Organizer.** `registry.distill()` (cheap chain). Prompt-instructed JSON
  `{ notes:[{title,plane,planes[],tags[],body}] }`, robust parse (tolerate code fences). A
  **pure, unit-tested `validate_organizer_output`**: `plane` must be a configured plane else
  Inbox; `planes[]` filtered to valid planes and made a superset of `plane`; caps on note/tag
  counts. **Organic tagging** in the prompt — emotional tone + the what/why around feelings,
  free-form (no rigid taxonomy). **Inbox fallback** (chain exhausted / unparseable / zero
  valid notes): one note, `plane=Inbox`, title=first 8 words, body=full raw. Only infra
  failures (STT / vault-write) mark a capture `failed`; the organizer never does.
- **Note writing.** `<YYYY-MM-DD> <Sanitized Title>.md` in the plane folder; numeric-suffix
  collisions (` 2`, ` 3`); **atomic write** (temp + `os.replace`, ADR-014). Frontmatter per
  [02 §2](02-data-model.md) (`id`=capture id, local-TZ `created`, source, source_ref, plane,
  planes, tags, related). Siblings: `related:` frontmatter **+** a `## Related` section with
  `[[wikilinks]]`.
- **Conversational capture (minimal, [ADR-019](adr/019-conversational-capture-minimal-in-m1.md)).**
  Migration **002** adds `captures.follow_up_question` / `follow_up_answer` (nullable). One
  gentle nudge (≤20 words, versioned prompt constant) generated after a **successful** organize
  (none on the Inbox-fallback path). `POST /captures/{id}/follow-up {answer}` → **Pass 2**:
  re-organize original+answer, `git rm` the Pass-1 `note_paths` (soft-delete), write the
  enriched set, overwrite `note_paths` (**replace, not augment**). No server-side expiry.
- **Index stub = no-op.** `notes`/`chunks` stay empty until M2's real indexer/reindex; the
  step only transitions `written → indexed`. No frontmatter parser in M1. Keeps the supersede
  path pure filesystem+git.
- **Durability ([ADR-014](adr/014-vault-history-durability.md)).** A `VaultBackupService`
  owns **all** git ops behind one lock (file writes are concurrent+atomic; git staging/commit/
  push serialize): debounced ~60s commit + 04:55 sweep + `POST /admin/backup`; **ff-only push,
  heal-on-reject** (merge, never force/rebase/reset); gc/reflog pins set idempotently at
  startup (provision.sh doesn't). **Empty-repo bootstrap** creates the folder skeleton
  (`Inbox/`, `Summaries/Daily|Weekly/`, plane folders + `.gitkeep`) + initial commit +
  `push -u` — this **subsumes the `PSB-vault` initial-commit follow-up** (done in code).
- **Scheduler.** M1 introduces an **in-process APScheduler** (existing `enable_scheduler`
  flag; off by default, on in prod) running **only** the durability jobs; **M4 extends the same
  scheduler** with the 03:00–05:00 agent window. Jobs are also exposed via a **CLI entrypoint**
  so a future external scheduler (e.g. an eventual multi-tenant deployment) can drive them
  without rework. **All four R2 jobs land in M1** (via boto3): nightly `git bundle --all` →
  WORM, weekly integrity drill, nightly `pg_dump` → R2, nightly `/srv/data` sync → R2. Each
  writes `agent_runs` (`vault-backup`, `integrity-drill`, `db-backup`, `data-sync`).
- **`/health` fourth leg.** Degrades if the latest `integrity-drill` `agent_run` failed or is
  overdue (>~8 days) — ADR-014 §6. Contract change recorded in [03-api](03-api.md).
- **Web (online-only).** Capture screen: record button + Web-Audio `AnalyserNode` visualizer,
  text input, optimistic confirm; recent-captures strip via new **`GET /captures?limit=20`**,
  TanStack Query **polling** (~2s while in-flight, stop when idle); pending nudge shown inline
  with an answer input. **Offline text queue stays M5, voice-offline stays v2** (06-vs-08
  wording reconciled in favour of 08).
- **API contract additions** (recorded in [03-api](03-api.md)): `GET /captures?limit=` (list),
  `POST /captures/{id}/follow-up`, follow-up fields on `GET /captures/{id}`, `/health` fourth
  leg.

**Paused after grilling (2026-07-12) — recorded, not yet implemented** per the
[session protocol](09-session-protocol.md). Next: an **implementation session** builds M1
against this spec (no grilling), pausing between tasks with independent review at each.

**M1 progress — Task 1 done (capture domain core, 2026-07-12).** First implementation task
("Migration 002 + CapturePipeline") built + reviewed + verified in `../second-brain/server/`.
Delivered (no routers/web/durability yet — those are later M1 tasks):

- **Migration 002** (`versions/002_capture_follow_up.py`) — nullable `follow_up_question` /
  `follow_up_answer` on `captures`, hand-authored SQL (ADR-011). Applies to head + downgrade
  round-trips cleanly on the dev DB; `compute_head()` test bumped 001→002.
- **Pure logic (unit-tested, no mocks):** `app/capture/organizer.py` — versioned organizer +
  nudge prompt constants, fence-tolerant `parse_organizer_json`, pure `validate_organizer_output`
  (plane→configured-plane-else-Inbox, `planes[]` filtered + superset of `plane`, note/tag caps),
  `inbox_fallback_note` (first-8-words title). `app/capture/notes.py` — filename sanitization,
  frontmatter/`## Related` wikilink rendering (02 §2), and a filesystem `NoteWriter` (atomic
  temp+`os.replace`, numeric-suffix collision handling across on-disk + in-batch siblings,
  `/`-separated vault-relative paths, `remove_notes` for the supersede path).
- **`CapturePipeline`** (`services/capture_pipeline.py`) — in-process (`asyncio.create_task`,
  202 semantics), order transcribe(voice, no fallback)→organize→write→index-stub→trailing nudge;
  never-lose ordering (row/audio persisted before any model call); organizer failure ⇒ Inbox
  fallback (never `failed`); only STT + vault-write mark `failed`; nudge non-blocking + skipped
  on the Inbox path; follow-up **Pass 2 replaces** note_paths (soft-delete + rewrite), one nudge
  only; **boot-time `sweep_orphans`** (non-terminal ⇒ failed, retryable).
- **`CaptureStore` seam** — protocol + asyncpg `PgCaptureStore` (plain SQL) + in-memory fake, so
  the pipeline unit-tests with **no live DB** (CI runs pytest DB-less). **`VaultBackup` protocol**
  + `LoggingVaultBackup` stub — the real git-backed `VaultBackupService` (ADR-014) is a later task.
- **New settings** (rule 9, in `.env.example`): `DATA_PATH`, `INBOX_PLANE`, `AUDIO_MAX_BYTES`
  (25 MB), `ORGANIZER_MAX_NOTES/TAGS`; `tzdata` pinned for deterministic `zoneinfo`.

*Interpretation recorded (flag for user):* 02 §2's frontmatter example shows lower-case
`plane: professional` while `PLANES`/folders are capitalized and 02 says "primary plane ==
folder". Resolved in favour of the stronger rule: **planes are identified by their configured
spelling** (`Professional`), used verbatim as folder + frontmatter; the organizer's output is
normalized to that spelling case-insensitively (so a model returning `professional` maps to
`Professional`), unknown ⇒ `Inbox`. Keeps M2's folder-derived `notes.plane` and frontmatter
`planes` consistent. Not an ADR (normalization detail, not architecture) — revisit if lower-case
frontmatter was actually intended.

- **Independent review** (fresh agent, diff vs M1 spec + ADR-019/014/005/011 + CLAUDE.md rules):
  **2 must-fix found and resolved**, both untested failure paths — (1) the trailing nudge could
  flip an already-`indexed` capture to `failed` (only `ProviderUnavailable` was caught) → nudge
  now swallows *all* errors (ADR-019 §1 non-blocking); (2) Pass-2 re-organize on an unavailable
  chain deleted the good Pass-1 notes and replaced them with an Inbox dump → now guards on
  `used_fallback`, keeps the original notes, and fails retryably (ADR-019 §2 "enrich, not
  degrade"). Regression tests added for both. Minors: nudge now built from the organize result
  (not raw transcript, ADR-019 §1); `written` status set after files land; settings documented.
  Accepted-as-scoped minors: per-capture fallback not persisted (no column in M1), single-user
  TOCTOU on the follow-up guard.
- **Verification:** 56 server unit tests + `ruff` green (CI parity: fakes + tmp vault, no DB).
  Against the dockerized dev DB: migration 002 up/down/up; `PgCaptureStore` CRUD/sweep smoke;
  full `CapturePipeline` end-to-end smoke (text → vault note + nudge, then follow-up Pass 2
  replace) driving the **real** DB + filesystem with fake providers.

**M1 progress — Task 2 done (capture routers + wiring, 2026-07-12).** The HTTP capture surface
(03-api §Capture) is live over `CapturePipeline`, committed locally (`8c65a0c`, **not pushed —
user's call**). Delivered:

- **Router** (`app/routers/capture.py`) — all six endpoints, session-gated at the router level
  (`dependencies=[Depends(require_session)]`; only `/auth/login` + `/health` stay public):
  `POST /capture/text` (`{text, created_at?}` → `202 {capture_id, status:"received"}`),
  `POST /capture/voice` (multipart `file`, `UnsupportedAudio` → `400`), `GET /captures?limit=`
  (default 20, `1..100`, newest-first), `GET /captures/{id}` (`404` if missing),
  `POST /captures/{id}/retry` (`409` unless `failed`), `POST /captures/{id}/follow-up`
  (`{answer}` → `202`, `409` if no nudge pending). Router only validates + delegates + maps
  domain errors (rule 5); wire models (`CaptureView.from_record`, `CaptureAcceptedResponse`, …)
  in `models.py`.
- **Pipeline additions** — read-through `get`/`list_recent`; **`retry_capture`** re-drives from
  the first incomplete step: a **held follow-up answer** re-runs Pass 2 (enrich, never degrade —
  ADR-019 §2), otherwise the **recorded** notes are removed before re-running `_process`
  (idempotent, rule 6). New `CaptureStore.reset_for_retry` clears the failure back to `received`
  (+ asyncpg impl + in-memory fake).
- **Lifespan wiring** (`main.py`) — builds `CapturePipeline` (`PgCaptureStore` + `NoteWriter` +
  the `LoggingVaultBackup` M1 stub) onto `app.state.capture_pipeline`, runs **boot
  `sweep_orphans`** after the migration-head check, and **`drain()`s** in-flight tasks on
  shutdown before `db.disconnect()`. Router mounted under `api_prefix`.

- **Independent review** (fresh agent, diff vs 03-api + ADR-019 + CLAUDE.md rules): **no must-fix**
  — six endpoints match the contract exactly, auth gating verified (401 test exercises the real
  gate), retry branching + never-lose/idempotency traced correct, layering clean, lifespan
  collaborator/order correct. Minors applied: retry docstring no longer over-claims duplicate
  safety; added a test asserting a provided `created_at` reaches the pipeline. **Logged
  follow-ups (non-blocking):** (1) a note that lands in a batch crashing *before*
  `set_note_paths` records it isn't tracked, so a retry can leave a `" 2"` duplicate — pre-existing
  `_process` ordering, fix by recording `note_paths` incrementally or sweeping by frontmatter
  `id`; (2) the whole voice upload is read into memory before the 25 MB check (Starlette spools
  to a temp file; fine for single-user M1); (3) the `follow-up` 202 body reuses `status:"received"`
  though that capture goes `indexed→organizing` (contract pins a body only for text/voice; UI
  polls `GET /captures` for real status); (4) `models.py` importing `CaptureRecord` from the
  service layer for the mapper (no cycle; cosmetic).
- **Verification:** 73 server unit tests + `ruff` green (CI parity: fakes + tmp vault, no DB).
  Router tested end-to-end via FastAPI `TestClient` over a fake pipeline (status codes, shapes,
  error→code mapping, auth gate); `retry_capture`'s four paths tested against the real filesystem
  with fakes; `create_app()` imports clean and OpenAPI registers all six routes under `/api/v1`.

**M1 progress — Task 3 / durability Slice A done (git-backed `VaultBackupService`, 2026-07-12).**
The durability task is being built in two reviewable slices; **Slice A (the git side)** is done,
committed locally (`884855f`, **not pushed — user's call**). Replaces the `LoggingVaultBackup`
no-op stub the pipeline depended on with the real service — the **one owner of all vault git ops
behind a single `asyncio` lock** (ADR-014 §2–3):

- **`GitRepo`** (`services/git_repo.py`) — thin subprocess wrapper (async via `to_thread`) behind
  a **`GitClient` protocol** (fakeable, mirrors the `CaptureStore` seam). Only ever
  **fast-forward** pushes (`git push HEAD:branch`, no `--force`); the sole reconciliation is a
  **merge** (`pull --no-rebase --no-edit`) — never rebase/force/reset. `is_merging`/`abort_merge`
  guards; push/pull **network timeouts are soft failures** (commits kept local, retried).
- **`VaultBackupService`** — **debounced ~60s batch commits** (coalesced, deduped message); every
  stage/commit/push serialises under one lock; **gc/reflog pins** (`gc.pruneExpire`/`reflogExpire`/
  `reflogExpireUnreachable=never`, `gc.auto=0`) + commit identity applied **idempotently at
  startup**; **empty-repo bootstrap** (`Inbox/`, `Summaries/Daily|Weekly/`, plane folders +
  `.gitkeep`, the ADR-014 §3 `.gitignore` — excludes only Obsidian UI cache + OS cruft, never
  notes/`.trash` — initial commit + `push -u`; subsumes the `PSB-vault` initial-commit follow-up);
  **heal-on-reject** (non-ff push → pull-merge → re-push, healing GitHub if a client rewound it);
  best-effort push (unreachable remote ⇒ commits stay local, next backup reconciles — §5).
- **`POST /admin/backup`** → `{committed, pushed}` (03-api), session-gated; forces an immediate
  commit + push, folding in any pending debounced reasons.
- **Lifespan** — builds the service, `ensure_ready()` at boot (init if needed → pins → bootstrap),
  wires it into `CapturePipeline` (replacing the stub), and on shutdown **drains the pipeline →
  flushes the backup → disconnects the DB** (order matters: a draining capture may enqueue a
  commit). New non-secret settings (`vault_git_remote/branch`, `vault_backup_debounce_seconds`,
  `git_user_name/email`) in `.env.example`.

- **Independent review** (fresh agent, diff vs ADR-014 + CLAUDE.md rules): **1 must-fix resolved** —
  cancelling the debounce timer *mid-commit* (via `flush`/`backup_now`) raised `CancelledError`
  into the in-flight batch, releasing the lock while the uncancellable `to_thread` git subprocess
  kept running, so the canceller launched a **second concurrent git command** (broke the single-
  lock invariant). Fixed by **`asyncio.shield`ing the commit phase** so the canceller blocks on the
  lock until the in-flight batch finishes; **regression test verified it fails without the shield**.
  3 minors also applied: an in-progress-merge commit guard (never capture conflict markers),
  reschedule-on-failed-batch (don't wait for the nightly sweep), and soft push/pull timeouts.
- **Verification:** 89 server unit tests + `ruff` green. Orchestration (debounce/coalesce, heal-on-
  reject success+abort, no-remote, no-op flush, backup_now fold-in, the shield regression) tested
  with a `FakeGitRepo`; **real-git integration** (bootstrap→push to a bare remote, a genuine non-ff
  heal-merge preserving both sides) tested against actual git, **skipped when git is absent** so CI
  stays green; `create_app()` imports clean and `/admin/backup` registers under `/api/v1`.

**M1 progress — Task 3 / durability Slice B1 done (R2 job layer, 2026-07-12).** The four R2/WORM
jobs + `agent_runs` writer + CLI, committed locally (`7f3c4a7`, **not pushed — user's call**).
Driven for now by the CLI; the in-process scheduler + `/health` 4th leg are **Slice B2**.

- **`ObjectStore` seam + `R2ObjectStore`** (`services/object_store.py`) — Cloudflare R2 via the S3
  API; **boto3 imported lazily** inside the impl so the module graph / tests / CI never pull it.
  `build_object_store(settings)` → `None` when creds are absent (dev) ⇒ the R2 jobs record a
  **skipped** run. New R2 settings (`r2_account_id/access_key_id/secret_access_key/bucket/
  endpoint_url`; endpoint derives `https://<account>.r2.cloudflarestorage.com`) in `.env.example`.
- **`AgentRunStore`** (`services/agent_runs.py`, plain SQL, no ORM) — every job **opens/closes an
  `agent_runs` row** (`running` → `succeeded`/`failed`/`skipped`) with a summary + `details` jsonb
  (vision P8); `latest(agent, status=)` feeds the drill and (Slice B2) `/health`.
- **Git bundle support** — `GitRepo` gains `bundle_all` / `commit_count` / `tracked_file_count` /
  `verify_bundle` / `clone_from`; `VaultBackupService` gains a **`Fingerprint`** (HEAD sha +
  monotonic commit count + file count) with `write_bundle` / `snapshot_fingerprint` **both under
  the one git lock** (live-repo git never runs outside the lock, ADR-014 §2).
- **`BackupJobs`** (`services/backup_jobs.py`) — the four jobs + the 04:55 sweep:
  **`vault-backup`** (`git bundle --all` → R2 WORM + `.manifest.json`, records the fingerprint),
  **`integrity-drill`** (download the last *good* bundle → `git bundle verify` + clone → assert
  fingerprint vs manifest + **monotonic** live count), **`db-backup`** (`pg_dump` → R2),
  **`data-sync`** (`DATA_PATH` raw inputs → R2). Never-crash (rule 7): each ends as a `failed` run
  with context; all blocking IO (boto3/pg_dump/git/file/tempdir) via `asyncio.to_thread`. The sweep
  is a plain `backup_now` (the git commit), not one of the four named runs.
- **CLI** (`app/cli.py`) — `python -m app.cli {vault-backup|integrity-drill|db-backup|data-sync|
  vault-sweep}` for a future external scheduler (no rework); `ensure_ready()` first. `boto3` added
  to deps (prod only).

- **Independent review** (fresh agent, diff vs ADR-014 §1/§6/§7 + CLAUDE.md rules): **1 must-fix
  resolved** — the **monotonic truncation alarm wasn't actually enforced**: the drill only compared
  live vs the *latest* bundle, so a persistent commit-count DROP got silently re-recorded as the
  new baseline and never alarmed. Fixed by failing the **`vault-backup`** run when its count drops
  below the last **succeeded** bundle (so the good baseline is preserved) **and** drilling the last
  *succeeded* bundle — a truncation now always alarms. 3 minors applied: wrap the `agent_runs`
  row-open so a DB-down never crashes a job (rule 7); CLI `ensure_ready()` + a use-CLI-**or**-
  scheduler note; a note that the drill's live-repo check stands in for the "and GitHub" side.
  **Logged follow-ups:** an independent GitHub-side fingerprint fetch; data-sync re-uploads the
  whole corpus nightly with flattened keys (fine for the flat uuid-named inputs today).
- **Verification:** 105 server unit tests + `ruff` green. Jobs tested with fake R2/agent-runs/git
  (fingerprint upload, monotonic-regression → failed run, skip-when-disabled, drill pass +
  regression + mismatch + no-bundle, db/data uploads, row-open-failure resilience); **real-git**
  bundle→R2→verify→clone→drill **round-trip** against actual git (skipped when git absent); CLI
  arg-handling + job wiring smoke; `boto3` confirmed **not** imported without creds.

**M1 progress — Task 3 / durability Slice B2 done (scheduler + `/health` backups leg, 2026-07-12).**
The final durability piece — the in-process APScheduler and the `/health` 4th leg — committed
locally (`1e3c420`, **not pushed — user's call**). This closes the durability task (Slices A + B1 +
B2). Delivered:

- **`BackupScheduler`** (`services/scheduler.py`) — wraps APScheduler's `AsyncIOScheduler`; the
  job→schedule map is **pure data** (`job_specs()`, so it unit-tests without a loop), `start()`/
  `shutdown()` are the only parts touching APScheduler. Registers the **five** durability jobs on
  crontab windows in `scheduler_tz`, staggered inside the ADR-010 03:00–05:00 band and clear of the
  reserved M4 slots (Slack 03:00 / rescan 03:40 / summary 04:10 / review 04:40): `data-sync` 03:10,
  `db-backup` 03:25, `integrity-drill` Sun 04:30, **`vault-sweep` 04:55** (ADR-010), **`vault-backup`
  bundle 04:57** (WORM right after the sweep, so the bundle includes it — ADR-014 §3 "checkpoint =
  commit+push+bundle"). `misfire_grace_time` (default 1h) + `coalesce` + `max_instances=1` realise
  ADR-010's "missed window ⇒ next night covers it" (a long outage skips, restart jitter still fires,
  a job never overlaps itself on the 4GB VPS).
- **Lifespan wiring** (`main.py`) — when `enable_scheduler` (off by default; exactly one prod
  instance sets it), builds `BackupJobs` via the shared **`build_backup_jobs`** factory (moved from
  `cli.py` into `backup_jobs.py` so CLI and scheduler share one constructor) and `start()`s it inside
  the lifespan event loop; on shutdown **stops the scheduler first** (a job may enqueue a vault
  commit), then drains the pipeline → flushes the backup → disconnects the DB.
- **`/health` 4th leg** (`system_health.py`, `models.py`, `routers/health.py`) — `backups` degrades
  when the latest `integrity-drill` run **failed** or the last **succeeded** one is **overdue**
  (`integrity_drill_max_age_days`, default 8 = weekly + 1 night grace); a currently-*running* drill
  doesn't flip health while the last success is fresh, and a DB-read blip degrades rather than errors
  `/health` (rule 7). **Prod-gated exactly like `git_remote`** (reported always, gates `ok` only in
  prod). Contract already recorded in [03-api](03-api.md).
- **Settings** (rule 9 — no hardcoded schedules): five crontab strings + `scheduler_misfire_grace_
  seconds` + `integrity_drill_max_age_days`, all in `.env.example`.

- **Independent review** (fresh agent, diff vs Slice B2 spec + ADR-010/014 §6 + 03-api + CLAUDE.md
  rules): **no must-fix.** Verified: all five jobs scheduled to the right coroutines; times in-window,
  staggered, sweep@04:55, bundle-after-sweep; misfire/coalesce/max_instances consistent with ADR-010;
  lifespan start/shutdown ordering correct + `enable_scheduler`-gated; backups-leg prod gating mirrors
  `git_remote`; **no TZ bug** (`agent_runs.started_at` is `timestamptz` ⇒ asyncpg returns aware UTC);
  DB-read error degrades, never crashes `/health`; all schedules/thresholds are settings. One minor
  applied (folded the date-age computation inside the rule-7 `try` so the "never errors /health"
  guarantee is robust even to a future naive `started_at`). **Logged follow-ups (non-blocking):**
  (1) `shutdown(wait=False)` doesn't await an in-flight backup job, so a job finishing after
  `db.disconnect()` could log an error on the closed pool — background task, idempotent, service
  never crashes; guard `finish()` if clean teardown logs matter. (2) A freshly-provisioned prod box
  reports `backups:false` (503) until the first weekly drill — **provisioning runbook should run
  `python -m app.cli integrity-drill` once** to establish a green baseline. (3) The Sunday 04:30
  drill could occasionally overlap the future M4 04:40 review if a clone runs long — an M4-era
  scheduling concern, nothing now.
- **Verification:** **116 server unit tests + `ruff` green** (was 105; +11: scheduler job-spec
  mapping + real-APScheduler registration asserting ids/triggers/misfire wiring without firing a job,
  and the backups-leg logic across never-ran / failed / fresh-success / overdue / running-while-fresh
  / DB-error / prod-gated / dev-ignored). Plus an **out-of-band in-process smoke** driving a real
  `AsyncIOScheduler` to actually fire a job coroutine end-to-end (records a `data-sync` `agent_run`),
  confirming APScheduler awaits the async job methods through the full `agent_runs` path.

**M1 progress — web capture screen (06) done, 2026-07-12.** The last M1 surface — the PWA capture
screen over the (already-done) server side — is built in `../second-brain/web/`, committed locally
(**not pushed — user's call**). **M1 is now code-complete;** what remains is the *live* end-to-end
Accept (below), which is a deployed-stack drive, not a code task. Delivered:

- **API layer** (`api/types.ts`, `api/client.ts`) — added `CaptureView` / `CaptureAcceptedResponse`
  + `CaptureKind`/`CaptureStatus` unions (field names match the server `CaptureView` exactly); five
  capture methods (`captureText`, `captureVoice`, `listCaptures`, `retryCapture`, `submitFollowUp`).
  The shared `request()` now detects `FormData` and **omits the JSON `Content-Type`** so the browser
  sets the multipart boundary for the voice upload. **All URLs stay in `api/` (ADR-006 / rule 4);**
  ids are path-encoded.
- **`useRecorder`** (`features/capture/useRecorder.ts`) — `MediaRecorder` (picks `audio/webm`, falls
  back to `audio/mp4`=m4a / `audio/ogg`; all server-accepted per 03-api) + a **Web-Audio
  `AnalyserNode`** reading mic **RMS each rAF frame into a framer `MotionValue`**, so the orb reacts
  at ~60fps **without React re-renders** (06 "transform/opacity only"). Secure-context /
  permission / getUserMedia errors are surfaced; the mic is released on stop **and on unmount**
  (tab switch); `AudioContext.resume()` inside the tap gesture defeats Chrome's autoplay-suspend.
- **`CaptureScreen`** (rewritten) — oversized **tap-to-toggle** record orb with a **voice-reactive
  glow** (idle → breathing halo, recording → amplitude-scaled glow, uploading → spinner), a quick
  **text field** (`POST /capture/text`), an **optimistic "captured ✓"** flash on any successful
  submit, and the recent strip. Autonomous animations gated on `useReducedMotion` (06 respect-RM).
- **`RecentCaptures`** (`features/capture/{RecentCaptures.tsx,useCaptures.ts}`) — `GET /captures?limit=20`
  via **TanStack Query**, `refetchInterval` = **2s only while any capture is in-flight** (terminal =
  `indexed`/`failed`), idle otherwise (08 M1); every write invalidates the list. Animated status
  pills across the full lifecycle (`received → transcribing → organizing → written → indexed`/`failed`),
  note-path chips once written, **failed → Retry** (`POST …/retry`), and the **pending nudge shown
  inline** with an answer input (`POST …/follow-up` → Pass 2, shown until `follow_up_answer` lands).
  **Online-only** — the offline text queue stays M5, per the M1 build decisions.

- **Review (self-review; see caveat).** The session-protocol *independent* review (fresh agent) could
  **not** be run this session — the harness restricted spawning an unrequested subagent, and the
  `/code-review` skill that loaded is a generic manual checklist, not the diff reviewer. What ran was
  a rigorous **implementer self-review** against 03-api + ADR-019 + the 06/08 M1 spec + CLAUDE.md rule 4.
  **3 findings fixed:** (1) a failed **voice upload** surfaced nothing (text did) → error now shown;
  (2) `AudioContext` could stay **suspended** under autoplay policy, killing the visualizer →
  `resume()` added in the gesture; (3) the voice snippet duplicated/contradicted the status pill →
  simplified. **Recommend a true independent review** (fresh agent or `/code-review ultra`) before
  M1 is declared closed, to satisfy the protocol's independence requirement.
- **Verification (static).** `tsc --noEmit` + `eslint --max-warnings 0` + `vite build` (452 modules)
  all green under the strict config (`noUncheckedIndexedAccess`, `verbatimModuleSyntax`, no-explicit-any).
  **Not exercised live:** the screen mounts only behind login, so an end-to-end drive needs the
  deployed backend + DB + session + a real mic — out of reach in this session. The server side already
  carries 116 tests.

**M1 live Accept — still pending (deployed-stack drive).** All M1 code is done, but the milestone's
**Accept** ("voice memo → correctly plane-classified note(s) in the vault < 30s, visible in GitHub
history; organizer failure still yields an Inbox note; a nightly WORM bundle lands in R2 and the
weekly integrity drill passes the fingerprint check") is a **live run on the deployed box**, not a
unit test. Next real step: **deploy web + run the Accept end-to-end** on `braindan.cc` (record from
the phone, watch the strip go `received→…→indexed`, confirm the note + git history, and let one
nightly cycle produce a WORM bundle + drill). Then M1 closes and M2 (indexing/search) begins.

**M1 replan — recorded 2026-07-12 (grilled; live-drive findings + independent review).** The
first live drive on `braindan.cc` surfaced two Accept-blockers and prompted three decisions plus a
failed independent review. All decisions recorded; **implementation pending** (paused per the
session protocol before building). Scope:

1. **STT fallback chain ([ADR-020](adr/020-stt-fallback-chain-groq-primary.md)).** Voice 429'd
   because STT was single-provider (OpenAI). Fix: `STT_CHAIN = ["groq","openai"]` (Groq
   `whisper-large-v3` primary, OpenAI `whisper-1` fallback), reusing `OpenAICompatibleProvider`;
   registry gains `_transcribe_over_chain`; `transcribe()` returns a result carrying
   `model_used`/`fallback_used` (rule 3). No per-provider retry in v1. New config `STT_CHAIN`,
   `GROQ_BASE_URL`, `GROQ_STT_MODEL` + secret `GROQ_API_KEY` (human-entered on the box).
2. **Capture interaction logging ([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)).**
   Capture writes **one `agent_runs` row per run** (`agent="capture"`, model/fallback + `details`
   JSON of stt/organize/nudge + timings); migration 003 adds the `capture_interactions` view.
   Explore via Supabase dashboard / view / optional Supabase MCP. Docker logs left as-is; no custom
   UI before M4.
3. **Claude Max effort.** Global `CLAUDE_MAX_EFFORT` (default `medium`) threaded into every
   claude-max CLI call (`--effort`), shipping in v1; per-task effort deferred (post-v1).
4. **Web capture must-fix (independent review, 3 findings — live-only, static gates missed them).**
   (a/b) The recent-strip stops polling at `indexed`, but ADR-019 generates the follow-up nudge
   *after* `indexed`, so the nudge — and the answered-nudge Pass-2 re-cycle — usually never render
   live; fix by polling a grace window past `indexed` until the nudge settles / a just-submitted
   follow-up re-enters flight. (c) `RecentCaptures` never imports `useReducedMotion` — the strip's
   infinite pulsing status dot ignores `prefers-reduced-motion` (06 / CLAUDE.md). Minors logged:
   server error `detail` discarded on capture/follow-up failures (25 MB / format rejection gives no
   hint); transient glow + row entrance not RM-gated.
5. **Vault push blocker — root-caused to a container SSH-wiring gap (deploy config).**
   `VaultBackupService` logged "vault push failed (remote unreachable)" — the *designed*
   best-effort degradation (commits kept local), but the Accept needs "note visible in git
   history". Diagnosis (2026-07-12): the vault repo at `/srv/vault` pushes to `origin =
   git@github.com:bodyionita/PSB-vault.git` over **SSH from inside the api container**, but
   `deploy/Dockerfile` installs `git` **without `openssh-client`**, and `deploy/docker-compose.yml`
   mounts `/srv/vault` but **no deploy key** and sets **no `GIT_SSH_COMMAND`** — even though
   `.env.example` promises "GITHUB_DEPLOY_KEY is mounted as a file for vault git push." So the
   container has no ssh binary / key / known_hosts and every push fails auth. Fix (deploy config,
   not app code): (a) add `openssh-client` to the Dockerfile; (b) mount the `provision.sh`-generated
   vault deploy key read-only into the container + set `GIT_SSH_COMMAND` (with a host-key policy —
   pin GitHub's key or `accept-new`); (c) ensure the `PSB-vault` GitHub repo exists and the deploy
   key is added with **write** access (`provision.sh` prints it). Until (a)+(b) land, provisioning
   the box alone won't make the push succeed.

**Provisioning status (2026-07-12):** `GROQ_API_KEY` **added to GitHub Actions secrets** (user).
Vault push still blocked pending the SSH-wiring fix above.

**Build order when implementation resumes:** STT chain (+ Groq key on the box) → capture
`agent_runs` logging + migration 003 view → `CLAUDE_MAX_EFFORT` → web 3 must-fix → vault remote
provisioning → **re-run the M1 live Accept**. Each task gets an independent review before its pause
(session protocol); the M1 Accept stays open until the live drive passes.

**M1 replan — IMPLEMENTED 2026-07-12 (code done, reviewed, static-verified; live Accept pending).**
All replan code is built and committed locally (**not pushed — user's call**):
- **Server** (`d9b21e8`): STT chain `[groq,openai]` + `_transcribe_over_chain` + `TranscriptResult`
  (rule 3); Groq as an `OpenAICompatibleProvider`; one `agent_runs` row per capture run
  (model/fallback + `details` stt/organize/nudge/timings), `PgAgentRunStore.finish` persists
  model/fallback; **migration 003** `capture_interactions` view; global `CLAUDE_MAX_EFFORT` → every
  claude-max `--effort`.
- **Web** (`4d988ea`, `26a1c09`): the 3 must-fix — trailing nudge + Pass-2 render live via a
  bounded `isSettling` poll window; strip pulse gated on `useReducedMotion`.
- **Deploy** (`a56038f`): the vault-push SSH-wiring fix (openssh-client + entrypoint installs the
  bind-mounted deploy key root-owned 0600 + pins GitHub's host key + `core.sshCommand`; compose
  mounts the key).
- **Independent review ran** (fresh agent, diff vs ADRs/plan/rules). Found **1 must-fix** — the
  answered-nudge Pass-2 didn't render live (optimistic write clobbered by its own invalidate;
  `isSettling` stopped on `follow_up_answer`) — **resolved in `26a1c09`** (polling keyed off
  `updated_at`, stops only for a nudge awaiting the user). Also reconciled a doc/code mismatch: an
  **Inbox-fallback capture is a *success*** (never-lose), so its run stays `succeeded` but is now
  flagged via `details.organize.inbox_fallback` + the `capture_interactions.inbox_fallback` column
  ([ADR-021](adr/021-capture-interactions-agent-runs-logging.md) updated; test added). Server-half
  verdict: contract-clean. Accepted minors (not blocking): `SETTLE_MS=20s` is a tuning bound not
  tied to nudge completion; a shutdown-cancelled task leaves a `running` row (consistent with the
  durability jobs — no agent_runs orphan sweep exists yet); the server-error-`detail` UI minor from
  the first review still stands.
- **Static-verified:** server 124 pytest + ruff green; web `tsc`+`eslint`+`vite build` green. **Not
  exercised live** (needs the deployed stack + real mic + Groq/OpenAI keys + a reachable vault
  remote).

**What remains for the M1 Accept (all operational — needs the user / the deployed box):**
1. **Push code** to `origin/main` (user's call) so CI deploys it.
2. **Box side of the vault fix (deploy §5c):** confirm the `PSB-vault` GitHub repo exists and the
   `provision.sh` vault deploy key is added to it with **write** access; confirm `/srv/vault` has
   `origin` set (re-run `provision.sh` if the first-pass clone never completed).
3. **Run the live Accept** on `braindan.cc`: record from phone → strip goes `received→…→indexed`
   with the nudge appearing → note visible in the vault **and GitHub history** → organizer-down
   still yields an Inbox note → one nightly WORM bundle + the weekly drill pass. Then M1 closes.

**Bug found during the live Accept — audio not durable across deploys (rule-2 violation; fixed).**
A voice capture from before a redeploy could not be retried afterwards. Root cause: `data_path`
defaults to the dev-relative `../data` and **prod never set `DATA_PATH`**, so the container wrote
audio to its *ephemeral* `/data` (WORKDIR `/app` → `/app/../data`), **not** the persistent
`/srv/data` bind mount — so a redeploy wiped un-processed audio and retry found no file. The R2
audio-sync job reads the same `data_path`, so it was backing up the wrong (ephemeral) directory
too. Fix: pin **`DATA_PATH=/srv/data`** in `deploy/defaults.env` (+ `.env.example`). **Requires a
redeploy to take effect**; audio captured before the fix on the ephemeral path is unrecoverable if
already wiped. Once deployed, audio persists on the R2-synced `/srv/data` volume and survives
redeploys, so retry works across deploys (matches the recovery matrix in 02-data-model). The user's
desired invariant ("if it can't process, store it somewhere between deploys") is exactly this fix.

**Bug found during the live Accept — `GROQ_API_KEY` never reached the container (fixed).** Voice
transcription failed with `groq: no API key configured; openai … 429`: the STT chain worked (Groq
primary → no key → fell back to a rate-limited OpenAI), but the container's `GROQ_API_KEY` was empty
despite the GitHub Actions secret existing. Root cause: the CI deploy workflow's `.env` render step
listed `OPENAI_API_KEY`/`NEBIUS_API_KEY`/… but **not `GROQ_API_KEY`**, so it was never written to
`deploy/.env` (07-infrastructure already listed it as a rendered secret — the workflow had drifted).
Fix: add `GROQ_API_KEY` to the render `env:` block + the `printf` in `.github/workflows/ci.yml`.
**Requires a redeploy.** Once rendered, Groq (free tier) is the STT primary and voice no longer
depends on the 429'd OpenAI.

**M1 polish batch — English-only vault, valid tags, vault git hygiene, re-organize (2026-07-13).**
From live use (grilled decisions recorded here + in the ADRs): (1) **Vault is English-only** —
`organizer-v2` translates non-English captures; note titles/bodies/tags are English (raw input
preserved in `captures.raw_text`). The UI **nudge still mirrors the capture's language** (decision:
vault English, conversational prompt personal). (2) **Tags are valid Obsidian tags** — `_slugify_tag`
(lower-case, hyphenated, no spaces, purely-numeric dropped). (3) **Vault git hygiene** ([ADR-014
amendment](adr/014-vault-history-durability.md)): `.gitignore` adds `.idea/`; new `.gitattributes`
forces `*.md eol=lf` (kills the Obsidian/Windows CRLF↔LF churn); both reconciled idempotently by
`ensure_ready`; **pull-first** merge before every push (integrates remote/other-device edits, still
merge-only). (4) **Re-organize** — `POST /admin/captures/{id}/reorganize` re-runs organize on a
capture's stored raw text and replaces its notes (the retranslation tool), sharing the Pass-2 core.
Verified: 129 pytest + ruff green. **SHIPPED 2026-07-13** — code pushed (`d469277`) and deployed;
reorganize run on the Romanian capture `a1e1e9b9-…` (both reorganizes `6aa45db`+`328b142` in the
`PSB-vault` remote), English output + commit verified by the user. Local clone one-time
`git config core.autocrlf false` applied (`.gitattributes` `*.md eol=lf`; working tree clean, no
CRLF churn).

**Post-ship vault review + one manual content fix (2026-07-13).** Reviewed the reorganized vault
end-to-end: all four notes are English, filenames English, tags valid English slugs — the polish
batch held. **One organizer-v2 translation error found and fixed by hand** (vault commit `70396dd`,
pushed): the app name **`Braindan`** (a "Brain" + "Bogdan" wordplay — the project's own domain is
`braindan.cc`) was corrupted to **`Braindone`** in the Ideas note's H1, body, filename, and both
sibling cross-links. Corrected all five occurrences directly in the vault clone (the never-lose
source of truth; nightly pull+rescan / `POST /admin/reindex` will reconcile the derived tables once
M2 lands). **Follow-up for M1-organizer:** LLM translation can mangle proper nouns / coined names —
consider passing known entities (app name, people) as a do-not-translate glossary, or a light
post-check; logged as a quality follow-up, not an M1 blocker.

**Two "odd" observations from live use — clarified as BY-DESIGN, with one open scope question.**
The user noticed (a) the two sibling notes from capture `a1e1e9b9` **share one `id`**, and (b) they
are **`related:` to each other** despite being unrelated in meaning (a park picnic vs. an app idea),
and asked whether the nightly run fixes this. Answer per [02-data-model.md](02-data-model.md) §2 +
[ADR-005](adr/005-planes-and-atomic-notes.md): both are the **documented design**, not bugs —
frontmatter `id` **is the capture id** (siblings from one capture share it; the `notes` table still
gives each file its own uuid pk keyed on unique `vault_path`), and `related:` means **"sibling
notes from the same source"** (co-capture linkage, not semantic similarity). **The nightly run does
NOT change this:** M2 nightly pull+rescan only rebuilds `notes`/`chunks` from the vault; nothing in
the current plan re-links notes by semantic relatedness or re-keys sibling ids. So the user's
underlying want — **links that reflect actual topical relatedness rather than mere co-capture** — is
a genuine **new scope item** (semantic backlinks / relatedness graph), not something an existing
milestone delivers. **Not decided here** (session protocol: don't resolve scope inline) — flagged
for a future planning/grilling pass; M2 (pgvector similarity) is the natural place to consider it.

**M1 close POSTPONED to end of M2 (decided 2026-07-13).** M1's live Accept has a time-dependent
tail — **a nightly WORM bundle lands in R2** and **the weekly integrity drill passes the
fingerprint check** — that can only be confirmed after real overnight/weekly cycles run on the box.
The app is now **left running overnight**, so those results (plus live signal on the nudge /
organizer-down→Inbox paths, and whether the semantic-relatedness observation actually bites in
practice) will exist by the next cycle. Rather than block on them, **M1 stays open and is closed at
the end of M2**: build M2 now, and fold the M1 Accept confirmation (nightly bundle + drill green,
any overnight-surfaced findings) into the M2 close. The captured-voice-note→GitHub-history half of
the Accept is already satisfied (capture `a1e1e9b9` notes are in `PSB-vault`); what remains is the
scheduled-backup evidence and any overnight findings. Start M2 without waiting.

## M2 — Indexing & search

Chunking (pure, tested), indexer (hash skip, transactional upsert), full rescan +
`/admin/reindex`, `/search` with plane filters. Web: standalone search screen + admin controls.

**Accept:** reindex over a seeded vault; paraphrased query finds the right note; DB wipe +
reindex restores search; editing a note via git push is picked up by nightly pull+rescan.
**M2 also closes M1** (postponed 2026-07-13): fold in the M1 backup tail (a nightly WORM bundle +
the weekly integrity drill pass the fingerprint check) once the overnight/weekly cycles have run.

**M2 build decisions (grilled 2026-07-13 — [ADR-022](adr/022-embeddings-self-hosted-nomic.md),
[ADR-023](adr/023-semantic-relatedness-graph.md), [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)).**
Build-ready detail below; nothing left to implementer discretion. Facts confirmed against the M1
code: the registry already exposes `embed()`; `notes`/`chunks` exist (migration 001, currently
empty — the M1 index step is a no-op stub); chunking policy is in [02 §4](02-data-model.md).

- **Embeddings — self-hosted nomic ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)).**
  `nomic-embed-text-v1.5` via an **Ollama** sidecar (OpenAI-compatible; drop-in
  `OpenAICompatibleProvider`, base-URL only), **768-dim**, **single-provider, no hot fallback** (one
  index = one vector space). **Asymmetric prefixes mandatory**: `search_document:` when indexing,
  `search_query:` when searching. Nebius `Qwen3-Embedding` (multilingual) is the documented
  **cold-swap** (one config + reindex) if English-centric search disappoints; a same-model
  hosted-nomic **hot fallback is a noted future** (option B). Migration 004 resizes the empty
  `vector(1536)` columns → 768.
- **Indexer.** `file → strip frontmatter + `sb:related` block → sha256 (skip if unchanged) → parse
  frontmatter → chunk ([02 §4](02-data-model.md)) → batch-embed → **per-note transaction** (delete
  old chunks + upsert note + insert chunks) + `notes.embedding = mean-pool(chunk vectors)`. On embed
  failure: **skip-and-continue → run `partial`** (leave existing rows, retry re-indexes stale notes),
  429 backoff. Deletions reconciled (cascade); rename = new path. Fully idempotent.
- **`content_hash`** covers **frontmatter + body minus the `sb:related` block** — metadata edits
  reindex; the graph's own writes never re-trigger reindex (feedback-loop fix,
  [ADR-023](adr/023-semantic-relatedness-graph.md)).
- **Relatedness graph ([ADR-023](adr/023-semantic-relatedness-graph.md)) — full materialized graph.**
  Canonical **`note_links`** table (directional, `score`) **+** a rendered, delimited `sb:related`
  `## Related notes` wikilink block in each note body (Obsidian-visible), **distinct** from the
  co-capture `related:` field. Similarity = top-K over `notes.embedding` cosine above a floor;
  **`RELATED_TOP_K`=5**, **`RELATED_MIN_SCORE`=0.5**, both settings **tuned live** during Accept.
  **Recomputed nightly only** (+ on `/admin/reindex`); the real-time capture path never touches it.
  Churn-gated (rewrite a file only when its block changed).
- **Search.** `POST /search` → **note-grouped** results (best chunk = snippet), `top_k`=10,
  `planes` = `notes.planes` membership overlap ([ADR-005](adr/005-planes-and-atomic-notes.md)), no
  hard floor (`SEARCH_MIN_SCORE` off by default). `GET /notes/{id}` = read-only preview (body from
  the vault file + the note's `note_links` neighbours).
- **Reindex.** `POST /admin/reindex` **async**: `202 {run_id}`, `agent="reindex"` run
  (`details.trigger="manual"`, counts + `partial` in `details`), **single-flight** (409 if a
  reindex/rescan is running). The **one combined nightly `reindex` job** (03:40, existing scheduler)
  = `git pull` → rescan → recompute graph → render blocks → commit+push (one vault lock),
  `details.trigger="nightly"`.
- **Tags ([ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)).** Forward-only reuse: the
  live tag vocabulary (distinct `notes.tags`) is injected into the organizer (+ M4 distiller) prompt
  — *prefer existing, coin new only if none fits*. Existing cruft: a **manual** two-step
  `POST /admin/tags/consolidate` (propose → apply, reusing the reorganize vault-write path); **not**
  wired into the nightly job.
- **Web (online-only).** A standalone **Search tab** (query box, plane-filter chips, note cards with
  title/plane/snippet/tags/score, **read-only preview on expand** via `GET /notes/{id}`) + a
  lightweight **Admin tab** with buttons: **Reindex** (shows the run's live status/counts),
  **Backup now** (`/admin/backup`), **Consolidate tags** (Propose → review → Apply). Admin tab is
  movable later.
- **API/contract additions** ([03-api](03-api.md)): `POST /search` note-grouped, `GET /notes/{id}`,
  `POST /admin/reindex` async, `POST /admin/tags/consolidate`. Schema ([02](02-data-model.md)):
  migration 004 (`notes.embedding`, `note_links`, resize to 768). Infra ([07](07-infrastructure.md)):
  `ollama` sidecar.

**Overnight snapshot at M2 start (2026-07-13).** The M1-tail collection ran before starting M2:
- **Box up** — `braindan.cc` root `200`; `/health` `db`/`vault`/`git_remote` green. `backups:false`
  → `503` (`degraded`) — the expected "no green integrity-drill baseline yet" state.
- **M1 non-scheduled paths confirmed working live:** `agent_runs` shows capture runs (4 succeeded /
  3 failed); the **Inbox-fallback** path fired (a capture → `Inbox/…`), **nudges** generated on 3
  captures, **2 answered** (Pass-2 follow-up works end-to-end). So nudge + Inbox-fallback are
  live-verified; the co-capture note→GitHub-history half of the M1 Accept was already satisfied.
- **Scheduled-backup evidence still pending, no action taken.** Zero `agent_runs` for
  `vault-backup`/`db-backup`/`data-sync`/`integrity-drill`. This is **consistent with no scheduled
  window (03:10–04:57; drill Sunday 04:30) having elapsed** since the app was left running — *not*
  evidence the scheduler is broken. Decision: **don't chase it preemptively**; let the next real
  overnight cycle produce the rows, check `agent_runs` then, and diagnose (likely just verify
  `ENABLE_SCHEDULER` in the prod `.env`, and seed a baseline with `python -m app.cli integrity-drill`)
  **only if it's actually empty**. The M1 close folds into the M2 close either way.

**Paused after grilling (2026-07-13) — recorded, not yet implemented** per the
[session protocol](09-session-protocol.md). Next: an **implementation session** builds M2 against
this spec (no grilling), pausing between tasks with an independent review at each. Suggested task
order: migration 004 + Ollama/nomic provider wiring → pure chunker → indexer service (real index
step replacing the M1 stub) → `/search` + `GET /notes/{id}` → relatedness graph (`note_links` +
`sb:related` render) → nightly combined `reindex` job → tag reuse + `/admin/tags/consolidate` → web
Search + Admin tabs → live M2 Accept (which also confirms the M1 backup tail).

**M2 progress (implementation) — Tasks 1–2 done, reviewed, committed 2026-07-13.**
- **Task 1 — migration 004 + Ollama/nomic provider wiring — DONE** (server commit `c66b562`,
  local, **not pushed**). Migration 004 (`004_embeddings_768_and_note_links.py`): resizes the
  still-empty `chunks.embedding` + adds `notes.embedding` at **`vector(768)`**, creates the
  directional scored **`note_links`** table (PK `(note_id, related_note_id)`, both FKs cascade),
  recreates HNSW cosine indexes at 768 — **verified up→down→up against a real pgvector DB** (dev
  compose). Provider wiring ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)): `embedding_model`
  → `nomic-embed-text`, `embedding_dim` → **768**, new `embedding_provider_id` (default `ollama`,
  the cold-swap seam) + `ollama_base_url` settings; `OpenAICompatibleProvider` gained a
  `requires_api_key` flag so the **keyless localhost** Ollama endpoint skips the key guard +
  Authorization header (keyed OpenAI/Nebius/Groq fallback semantics unchanged); the registry
  registers `ollama` as the **sole** embedding provider and OpenAI is **STT-only** now. Deploy:
  `ollama` sidecar in compose (expose-only, persistent model volume, `depends_on` start-order),
  `defaults.env`/`.env.example` wiring, a provision model-pull step. The asymmetric
  `search_document:`/`search_query:` **prefixes are deferred** to the indexer/search tasks (the
  generic provider stays prefix-free). **134 pytest + ruff green.**
  - **Independent review: no must-fix.** Verified migration/schema against 02 §3 + ADR-023, the
    provider-boundary + keyless change (no regression to keyed providers), config/secrets
    discipline, and scope (prefixes correctly deferred). Fixed one trivial docstring typo.
    **Logged follow-up (non-blocking):** `note_links` has no index on `related_note_id`, so a
    note-delete cascade seq-scans — negligible at M2 scale (small, rebuildable, nightly-recomputed);
    add only if reverse-neighbor queries or graph size warrant it later.
- **Task 2 — pure text chunker — DONE** (server commit `fdd0f60`, local, **not pushed**). New
  `app/indexing/` domain package. `app/indexing/chunking.py` (pure, no I/O, unit-tested no-mocks):
  `split_frontmatter`, `strip_related_block` (canonical `sb:related` delimiter constants live
  here), `chunk_text` (split on headings → paragraphs → hard-split with overlap **only** on hard
  splits, per [02 §4](02-data-model.md)), and `chunk_note` (strip frontmatter + `sb:related`, then
  chunk). `CHUNK_SIZE`/`CHUNK_OVERLAP` are caller-passed params (rule 9); the
  `search_document:`/`search_query:` prefixes stay **deferred** to embed time (the chunker also
  feeds hashing). 19 chunker tests; **153 pytest + ruff green.**
  - **Independent review: no must-fix.** Applied its four MINOR findings: **(1)** fence-aware
    heading split (a `#` inside a fenced code block is not a heading boundary), **(2)** positive-size
    / non-negative-overlap guards (a `size≤0` misconfig previously hung), **(3)** CRLF normalization
    inside the exported primitives (the Task-3 hash builder will call them on raw vault text, which
    can be CRLF), **(4)** reconciled a **doc contradiction**: [02 §4](02-data-model.md) now states
    that only the `sb:related` block is stripped from *both* the hash and embed paths, while
    frontmatter is stripped from the embed path but **kept in `content_hash`** (so tag/plane edits
    still reindex) — matching §3 + [ADR-023](adr/023-semantic-relatedness-graph.md).
- **Task 3 — indexer service (real index step, replacing the M1 stub) — DONE** (server commit
  `684604e`, local, **not pushed**). The `notes`/`chunks` index is now populated from the vault
  (04 §3, [ADR-022](adr/022-embeddings-self-hosted-nomic.md)/[ADR-023](adr/023-semantic-relatedness-graph.md)):
  - **`app/indexing/indexer.py`** — the `Indexer`. Per note: read → strip `sb:related` → **sha256
    `content_hash`** (skip if unchanged) → `parse_note_metadata` → `chunk_note` → **batch-embed a
    note's chunks in one `registry.embed` call** with the mandatory `search_document: {title}\n\n{chunk}`
    prefix (bounded backoff on `ProviderUnavailable`) → **per-note transaction** (upsert note +
    replace chunks) + `notes.embedding` = **mean-pool** of the chunk vectors. On embed failure:
    **skip-and-continue → `partial`**, existing rows left intact (a later reindex retries). Two
    entry points: `index_paths(paths)` (the capture/ingestion write path) and `reindex_all()`
    (full rescan + **deletion reconciliation**, cascade). The **relatedness graph is left
    untouched** — nightly-only per ADR-023. `content_hash` `strip()`s surrounding whitespace so the
    trailing `sb:related` block (the graph's own writes) never re-triggers a reindex.
  - **`app/indexing/frontmatter.py`** — pure frontmatter parser + `NoteMetadata` extraction
    (plane←folder / planes←`[plane]` / tags / source / source_ref / created←mtime fallbacks; title
    = H1 else filename stem). **No YAML dependency** — the parser matches the controlled render
    shape and reads user notes leniently.
  - **`app/indexing/store.py`** — `IndexStore` protocol + `PgIndexStore` (the per-note transaction
    in plain SQL, upsert on `vault_path`, `list_indexed_paths`/`delete_notes` for reconciliation).
  - **`db.py`** — registers a **text codec for the pgvector `vector` type** on every pooled
    connection, so embeddings pass as plain `list[float]` (no `::vector` casts / string juggling in
    query code); encode on INSERT, decode on SELECT.
  - **Capture pipeline wired** — the M1 no-op stub (both `_process` and the reorganize path) now
    calls `index_paths`. **Best-effort**: the vault is truth (rule 1) and the notes are already on
    disk, so an index/embed failure **never flips a written capture to `failed`** — it's logged and
    the outcome recorded to the capture's `agent_runs` `details.index` (ADR-021). New settings
    `EMBED_MAX_ATTEMPTS` / `EMBED_RETRY_BACKOFF_SECONDS` (rule 9).
  - **Independent review (`/code-review` high, protocol-sanctioned mechanism): 1 must-fix, fixed.**
    `index_paths` promised "never raised — the rest of the batch still indexes," but `_index_one`
    only caught `ProviderUnavailable`; an unexpected per-note store/DB error would abort the whole
    batch (breaks rule 7 / "never crashes the job"). **Fixed:** the per-note loop now catches any
    exception, logs it, counts the note `failed`, and continues. Added coverage (batch survives an
    unexpected store error; pipeline indexes exactly the written notes + logs the outcome).
  - **173 pytest + ruff green** (18→20 new across `test_frontmatter` + `test_indexer` +
    `test_capture_pipeline`). **Verified end-to-end against a real pgvector DB** (dev compose, at
    head 004): `vector` codec round-trips (768-dim `list` in/out), per-note transaction lands
    note+chunks, mean-pool stored, hash-skip on unchanged reindex, cosine query runs over the HNSW
    index, and `reindex_all` reconciles a deleted file's row away.
- **Task 4 — `/search` + `GET /notes/{id}` — DONE** (server commit `6e0fa21`, local, **not
  pushed**). The read side of the index (03-api §Search, 04 §4, [ADR-022](adr/022-embeddings-self-hosted-nomic.md)/[ADR-023](adr/023-semantic-relatedness-graph.md)):
  - **`app/search/store.py`** — `SearchStore` protocol + `PgSearchStore`. `search_chunks` =
    pgvector cosine over `chunks`, **`DISTINCT ON (note)` best chunk = snippet**, `planes` **array
    overlap** (`&&`, membership not folder — [ADR-005](adr/005-planes-and-atomic-notes.md)),
    re-ranked by score desc, `top_k`, optional `min_score` floor. `get_note` = note metadata +
    `note_links` neighbours.
  - **`app/search/service.py`** — `SearchService`. Embeds the query with the **mandatory
    `search_query:` prefix** (ADR-022 — the asymmetric counterpart of the indexer's
    `search_document:`), clamps `top_k` to `SEARCH_MAX_TOP_K`, trims each hit's best chunk to a
    snippet. `get_note` reads the note **body from the vault file** (fidelity — reflects Obsidian
    edits, frontmatter stripped) + attaches neighbours. A down embedder surfaces as
    `ProviderUnavailable` (single provider, no hot fallback).
  - **`routers/search.py`** — `POST /search` (**503** when embeddings unavailable — it's a request
    path) + `GET /notes/{id}` (`uuid.UUID` path → **422** on malformed, **404** on unknown).
    Session-gated. New wire models (`SearchRequest`/`SearchResultItem`/`NotePreviewResponse`/
    `RelatedNoteItem`) + settings (`SEARCH_TOP_K_DEFAULT`/`SEARCH_MAX_TOP_K`/`SEARCH_MIN_SCORE`/
    `SEARCH_SNIPPET_MAX_CHARS`, rule 9). Wired into `main.py` + `dependencies`.
  - **Independent review (`/code-review` high): no must-fix.** One consistency fix applied:
    `search_chunks` reached past the `Database` abstraction with `self._db.pool.fetch` — switched to
    the `acquire()` context manager like every other store.
  - **186 pytest + ruff green** (13 new across `test_search_service` + `test_search_router`).
    **Verified end-to-end against a real pgvector DB** (dev compose): note dedupe (one row/note),
    cosine ordering, `planes` overlap filter, `min_score` floor, and `get_note` metadata/related.
  - **Note:** `note_links` is still empty (graph recompute is the next task), so `related` is `[]`
    until then — the endpoint reads whatever the graph has materialised.
- **Task 5 — relatedness graph recompute + `sb:related` render — DONE** (server commit `73ed641`,
  local, **not pushed**). The topical-relatedness graph ([ADR-023](adr/023-semantic-relatedness-graph.md),
  02 §3, 04 §3) — a new `app/graph/` domain package with one entry point,
  `RelatednessGraph.recompute()`:
  - **`app/graph/store.py`** — `GraphStore` protocol + `PgGraphStore` (plain SQL, ADR-011).
    `compute_neighbors` = a per-note **LATERAL top-K nearest** over `notes.embedding` cosine, the
    `RELATED_MIN_SCORE` floor applied **after** the top-K cut (so a note keeps *at most* K,
    possibly fewer), `m.id <> n.id` (no self-links), NULL embeddings excluded. `replace_note_links`
    rewrites the whole **canonical** `note_links` table in one transaction. Distance written
    `m.embedding <=> n.embedding` (indexed operand left, HNSW-friendly) + an `m.id` **deterministic
    tiebreaker** on both the top-K cut and the render order, so exact-score ties can't reorder a
    block and spuriously churn the vault.
  - **`app/graph/renderer.py`** — pure, **idempotent** render/apply of the machine-owned
    `sb:related` `## Related notes` block: path-target + title-alias `[[wikilinks]]` (title falls
    back to the basename), placed at the end of the body, LF-normalized, strips a **stale** block
    when a note loses all neighbours. Strictly distinct from the co-capture `related:` frontmatter
    / human `## Related` section (untouched).
  - **`app/graph/service.py`** — `recompute()`: **always** replaces `note_links` (canonical,
    reflects current embeddings even at zero vault churn) → renders every indexed note's block
    **churn-gated** (rewrite only when the file content actually changed → a stable graph = **zero**
    writes, ADR-023 §5) → **atomic** temp+`os.replace` writes → requests a vault commit via the
    `VaultBackup` seam **only when ≥1 file changed** (the single git owner folds it into one
    commit+push under the vault lock; the nightly job/`/admin/reindex` wrapper is Task 6). The
    feedback-loop fix holds: the indexer's `content_hash` excludes the block, so these writes never
    re-trigger a reindex. `RELATED_TOP_K`=5 / `RELATED_MIN_SCORE`=0.5 settings (rule 9, tuned live
    at Accept). Wired onto `app.state` + a `get_relatedness_graph` accessor for Task 6.
  - **Independent review (protocol-sanctioned, fresh context): 1 must-fix, fixed.** A
    present-but-unreadable/unwritable note (bad encoding, permissions, disk-full) would raise out of
    the whole render, aborting it mid-run and leaving `note_links` replaced but blocks partially
    rendered (breaks rule 7). **Fixed:** the per-note loop now catches any exception, logs it, counts
    the note `failed`, and continues (skip-and-continue, mirroring the indexer) — with a new test
    (undecodable file → run completes, others still render). Also applied its two hardening MINORs
    (deterministic ordering; index-friendly operand order); MINOR-3 (one-time first-run
    normalization is not a churn bug) documented in the renderer.
  - **202 pytest + ruff green** (16 new across `test_graph_renderer` + `test_graph_service`).
    **Verified end-to-end against a real pgvector DB** (dev compose, head 004): LATERAL top-K +
    floor (near-duplicate notes link ~1.0, an orthogonal note is correctly excluded by the 0.5
    floor), directional `note_links` materialized, blocks rendered only into linked notes, and a
    **zero-churn second run** (churn gate holds).
- **Remaining M2 tasks** (unchanged order): nightly combined `reindex` job (calls `reindex_all` +
  `RelatednessGraph.recompute`, single-flight; `POST /admin/reindex` async wrapper, `git pull`
  first, one commit+push under the vault lock) → tag reuse + `/admin/tags/consolidate` → web Search
  + Admin tabs → live M2 Accept (also confirms the M1 backup tail).

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
web · hybrid keyword+vector search ·
Cloudflare Access second wall · voice offline queue · entity extraction ·
**conversational capture — full version** (adaptive *multi-turn* interviewer/therapist nudges
during ingestion — a minimal one-nudge form ships in M1 per [ADR-019](adr/019-conversational-capture-minimal-in-m1.md);
v2 is the multi-turn, gentle-not-strict experience) ·
**share ChatGPT/Claude conversations as an ingestion source** (slots into the existing
`source`/`source_ref` contract; a shared-conversation URL becomes `source_ref`) ·
**undo a manual ingestion** (soft-delete a capture's notes via `git rm` — history kept —
using `captures.note_paths[]`; builds on the M1 supersede path from ADR-019) ·
**demo/presentation layer** (a separate, limited access point showing a curated/redacted
view of the brain with wow-factor, for showing other people) ·
**explorable memory-graph visualization** (clean, navigable graph of notes/links — useful to
the user, not just decorative; extends related-notes suggestions & graph features) ·
**multi-tenant deployment** (far horizon — a handful of users, not mass scale; keep M1+
architecture from actively precluding it, e.g. jobs invokable via CLI/external scheduler) ·
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
