# ADR-016: Secrets in GitHub Actions; CI renders `deploy/.env`; non-secret config in git

**Status:** Accepted · 2026-07-12
**Relates to:** [003 single-service-on-VPS](003-single-service-on-vps.md) · [006 monorepo + unified CI/CD](006-monorepo-with-strict-server-web-decoupling.md) · [012 M0 stack (deploy job dormant)](012-m0-implementation-stack.md) · [09-session-protocol §Security & secrets discipline](../09-session-protocol.md) (v1.4). **Updates** [07-infrastructure](../07-infrastructure.md) §Secrets and §Provisioning.

## Context
The VPS stack loads runtime config from `deploy/.env` (docker-compose `env_file`). The
original plan was to **fill `deploy/.env` by hand on the box** during provisioning — no
central store, no versioning, a manual step in every disaster recovery, and a live secret
sitting on disk that a human typed. The requirement now: a **central, secure,
auto-deployed** way to manage these values.

Constraints:
- **Hard rule** (09-session-protocol v1.4): no plaintext secret in git; the agent never
  handles secret values.
- **Minimal external dependencies** (ADR-003/015 ethos): don't stand up a secrets manager
  or a new runtime dependency for a single-user system.
- We **already run** a GitHub Actions deploy pipeline (ADR-006/012) that SSHes to the VPS.

Options weighed: **(a) encrypted-in-git (SOPS + age)** — versioned in-git ciphertext, but
adds tooling and one bootstrap key; **(b) GitHub Actions Secrets** — reuses the pipeline we
already run, adds nothing new, keeps values off the box until deploy; **(c) external manager
(Infisical/Doppler/Vault)** — best UX, but another account + a runtime dependency. Chosen:
**(b)**, for least new surface consistent with the ethos.

## Decision
1. **Secret values live in GitHub Actions Secrets**, grouped in a `production`
   **Environment**: `API_PASSWORD_HASH`, `SESSION_SECRET`, `DATABASE_URL`, `OPENAI_API_KEY`,
   `NEBIUS_API_KEY`, `SLACK_USER_TOKEN`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`,
   `R2_SECRET_ACCESS_KEY` (plus the deploy secrets `VPS_HOST` / `VPS_USER` / `VPS_SSH_KEY`).
   Entered once in the GitHub UI by the human; never seen by the agent, never in git.
2. **Non-secret config is versioned in git** as `deploy/defaults.env` (`BRAINDAN_DOMAIN`,
   `PLANES`, `NEBIUS_CHAT_MODEL`, `CLAUDE_MAX_MODEL`, `SCHEDULER_TZ`, `VAULT_PATH`,
   `SESSION_COOKIE_SECURE`, `ENVIRONMENT`, `R2_BUCKET`). Named `defaults.env` — **not**
   `.env.defaults` — so it does not collide with the `.env*` gitignore rule or the
   pre-commit secret-guard (the guard's `.env*` block stays strict, no carve-out).
3. **CI is the sole writer of `deploy/.env`.** The deploy job (on push to `main`, dormant
   until `VPS_HOST` exists) renders `.env` = `defaults.env` + the secrets, `scp`s it to
   `/srv/app/deploy/.env` **mode 600**, then `git pull && docker compose up -d --build &&
   alembic upgrade head`. The runner's copy is ephemeral. Transport is scp-of-a-file (not
   values piped through the SSH script) to avoid log leakage.
4. **`provision.sh` no longer fills `.env`.** It preps the box (Docker, UFW, clone, vault
   deploy key). The app is started by the deploy workflow. **Disaster recovery = provision
   the box, then trigger the deploy workflow.**
5. **Out of this flow (stay VPS-local, never in GitHub):** the Claude Max OAuth store
   (`claude login`, on a volume) and the vault git **deploy key** (generated on the VPS,
   private half never leaves the box).

## Consequences
- ✅ Central (GitHub), secure (encrypted at rest, masked in logs, off-box until deploy),
  auto-deployed, and consistent with the v1.4 no-secrets-in-git rule — the agent never
  touches a value.
- ✅ Non-secret config is reviewable/versioned in git; only the irreducible ~9 secrets sit
  in the GitHub store.
- ✅ No new service or runtime dependency; reuses the ADR-006 pipeline.
- ⚙️ Adding a secret is a two-step change: add to GitHub Secrets **and** add one line to the
  render step (accepted; low frequency).
- ⚙️ First boot / DR now depends on the deploy workflow running (not a hand-filled `.env`).
  `provision.sh` + 07-infra updated; `.env.example` becomes the documented full key set
  (secret + config), with `defaults.env` holding the real non-secret values.
- ↩️ **Reopen trigger:** collaborators or multiple environments arrive → revisit SOPS-in-git
  (versioned encrypted secrets) or an external manager.
- ❌ Rejected: SOPS + age (tooling + bootstrap key, overkill for one user); external manager
  (account + runtime dependency vs minimal-deps ethos); hand-filled `.env` on the box (no
  central store, manual DR step, human-typed live secret).
