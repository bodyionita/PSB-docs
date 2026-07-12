# Implementation Plan

**Version:** 2.1 · **Status:** Approved 2026-07-12 (2.1 = M1 grilled to build-ready detail; v2
backlog additions)
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

**Next M1 task:** **`VaultBackupService`** (ADR-014) + durability/R2/scheduler + `/health` 4th
leg — replaces the `LoggingVaultBackup` stub with the real git-backed service (ff-only push,
heal-on-reject, one git lock; empty-repo bootstrap; debounced ~60s commit + 04:55 sweep +
`POST /admin/backup`; the four R2 jobs + integrity drill via APScheduler/CLI; `/health` `backups`
leg). Then the web capture screen (06). Code committed locally (not pushed — user's call).

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
