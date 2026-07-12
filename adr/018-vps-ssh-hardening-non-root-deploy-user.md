# ADR-018: VPS SSH hardening — non-root `deploy` user; root login disabled; automated in provision.sh

**Status:** Accepted · 2026-07-12
**Relates to:** [003 single service on VPS](003-single-service-on-vps.md) · [015 compute tier Hetzner CX23](015-compute-tier-hetzner-cx23.md) · [016 secrets via GitHub Actions](016-secrets-via-github-actions-ci-renders-env.md) · [09-session-protocol §Security](../09-session-protocol.md). **Updates** [07-infrastructure](../07-infrastructure.md) §Provisioning.

## Context
The M0b `provision.sh` ran entirely as **root**, created no user, and never touched
`sshd_config`; the deploy workflow SSHes in as `VPS_USER` (value TBD) to run docker compose.
That leaves **root SSH login enabled** and hands CI a root-capable key — the largest default
exposure on a fresh VPS. The deploy identity does not actually need root: docker-group
membership is enough to run compose.

Honest scope note: any user in the **`docker` group is effectively root-equivalent** on the
host (`docker run -v /:/host …`). So the deploy identity is root-capable regardless. The wins
here are therefore **SSH attack-surface reduction and auditability**, not an ironclad
"CI key can never reach root" barrier.

## Decision
1. **Non-root `deploy` user.** Created by `provision.sh`, in the **`docker` + `sudo`** groups,
   owning `/srv/*`. The human sets an **account password** (used only for `sudo`/console — never
   in git, never handled by the agent) so interactive root is available via `sudo`.
2. **Deploy as `deploy`.** `VPS_USER = deploy`. Both the operator's **personal** public key and
   the **CI deploy** public key go in `deploy`'s `authorized_keys`. The vault deploy key and
   app run under this account's tree (moved off `/root`).
3. **SSH hardened, keys-only.** `sshd_config`: **`PermitRootLogin no`**,
   **`PasswordAuthentication no`** (SSH is key-only; the account password is for `sudo`/console,
   which `PasswordAuthentication no` does not affect). Root retains power via `sudo`; direct
   root SSH is closed.
4. **Automated, guarded, in `provision.sh` as the final step.** Ordering: create `deploy` →
   populate `authorized_keys` (both keys) → **verify the file is non-empty** → write the
   `sshd_config` changes → **validate with `sshd -t`** → reload → print a loud banner:
   *"open a second session as `deploy@host` and confirm it works before closing this one."*
   Hetzner's web console is the rescue hatch if a login ever breaks.

## Consequences
- ✅ Root SSH — the most brute-forced login — is closed; access is via a named, keyed,
  auditable account. SSH stays keys-only.
- ✅ Fully **reproducible**: hardening lives in the script, so it can't be skipped or
  mis-typed during a 2am recovery; preserves the < 30-min DR target.
- ✅ You keep full control: root via `sudo`, plus the Hetzner console backstop.
- ⚙️ Partial blast-radius win only — a leaked deploy key can still reach root via the docker
  group (documented, accepted for a single-user box; the SSH-surface + audit gains stand).
- ⚙️ Provisioning now depends on the operator's personal pubkey + the CI deploy pubkey being
  available to the script (pasted/parameterized), and on setting the `deploy` account password.
- ↩️ **Reopen trigger:** multiple operators or stricter isolation needs → split the CI **deploy**
  identity (docker group, no sudo) from a personal **admin** identity (sudo, no CI key), and/or
  a rootless-docker / non-docker-group deploy path to actually bound the CI blast radius.
- ❌ Rejected: deploy-as-root (keeps the largest surface open, root-capable CI key); manual
  post-provision `sshd` edit (breaks reproducibility; a DR step a human must remember).
