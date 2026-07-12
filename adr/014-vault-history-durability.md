# ADR-014: Vault history durability — WORM snapshots, rewrite protection, restore drills

**Status:** Accepted · 2026-07-12
**Extends:** [ADR-001](001-vault-on-vps-with-git-backup.md) (vault-on-VPS + git backup — its core decision stands; this hardens durability)
**Relates to:** [002 supabase](002-supabase-pgvector-for-index.md) · [010 agent-window](010-agent-window-3-5am.md) · updates [02-data-model §5](../02-data-model.md) and [07-infrastructure](../07-infrastructure.md)

## Context
Hard requirement: the **full history of the vault (the memory) must be impossible to lose.**
This is *not* about preventing edits — notes may change freely — it is about durability of
history. Today the vault is a git repo on the VPS pushed to **one** private GitHub remote
(ADR-001), and 02-data-model §5 accepted losing DB operational history in a worst case.

**How git actually loses history** (the threat model that drives everything):

| Operation | Loses history? | Why |
|---|---|---|
| Edit/clobber + commit, `git rm` + commit, **merge** | **No** | prior content stays reachable from the branch tip; merges keep both parents |
| **Rebase / force-push / gc-after-rewrite** | **Yes** | commits become unreachable, then pruned |
| **Total loss of every copy** | **Yes** | VPS disk + GitHub gone at once |

So history is lost **only** by rewrite or total-copy-loss. That collapses "never lose" into
four guarantees: (1) always commit, (2) make rewrite impossible, (3) ≥2 independent copies
with ≥1 immutable, (4) prove restores.

Cost note: options were priced against the tight budget (Topic C). GitHub branch-protection
/ rulesets are **not available on the free plan for private repos** (need GitHub Pro ≈ $4/mo
≈ the price of the whole VPS); object storage for a Markdown vault fits **free tiers** (~€0).

## Decision

**1. Copies.** GitHub stays the **live, rewritable primary**. Add the mandatory immutable
copy: a nightly `git bundle --all` (entire history in one self-contained, `git bundle
verify`-able, cloneable file) uploaded to **Cloudflare R2 with versioning + object-lock
(WORM)**. No second live remote (adds a rewritable copy, no durability gain). No GitHub Pro.

**2. Rewrite protection (all €0).** The **R2 object-lock is the append-only guarantee** — a
force-push/rebase/total-wipe of GitHub cannot touch the immutable R2 history. Plus: the
server only ever fast-forward `git push`, **never force/rebase/reset**; on a non-fast-forward
rejection it `pull`s **as a merge** and re-pushes (and thereby *heals* GitHub if a client
ever rewound it). The server repo pins gc/reflog: `gc.pruneExpire=never`,
`gc.reflogExpire=never`, `gc.reflogExpireUnreachable=never`, auto-gc off.

**3. Write/delete discipline.** Debounced ~60s batch commits. The nightly agent window
(03:00–05:00, the only place existing notes get reorganized) **begins with a checkpoint —
commit + push + WORM bundle — after pulling latest** (see 5), so a buggy organizer can only
add a commit on top of an immutably-captured "before." The vault's `.gitignore` excludes
**only** Obsidian UI cache (`.obsidian/workspace*`) + OS cruft — **not `.trash`, not notes**
(the indexer's `vault_ignore` ≠ the git ignore; soft-deleted notes stay committed/backed up).
Deletes go through `git rm` (content stays in history); note writes are atomic
(write-temp-then-`rename`). Every run records the paths it wrote/modified/deleted in
`agent_runs.details` (auditable, one `git revert` away).

**4. Obsidian sync safety.** Merge-only everywhere; **rebase/force/reset forbidden on both
ends.** obsidian-git: sync method = merge, order commit→pull(merge)→push, never force-push.
The server never hard-resets to remote and never rebases. Conflicts keep both sides (merge /
markers) surfaced in the activity feed — never resolved by discarding. Obsidian is
**read/explore-mostly**; the server is the de-facto sole writer.

**5. Pull-before-write.** Scheduled agent/connector runs pull (merge) latest **before**
writing, **best-effort and non-blocking** — if the remote is unreachable they proceed on the
local full-history copy and the push's merge-retry reconciles later (never block a write on
the network). Ordering: pull → checkpoint → run. Interactive captures are additive (unique
ids) and rely on pull-before-push instead of a per-batch pull.

**6. Restore drills.** Each nightly bundle records a **fingerprint** — HEAD sha + commit
count (**monotonic non-decreasing** ⇒ truncation/rewrite alarm) + tracked-file count — in
`agent_runs` and as a manifest object beside the bundle. Drills:
- **Weekly integrity drill on the VPS (mandatory now):** `git bundle verify` + clone from
  the R2 bundle + assert fingerprint on both R2 and GitHub. Failure → `failed` `agent_run`
  **and** `GET /health` degraded (fires the existing Cloudflare/UptimeRobot notification).
- **Fast-follows (recorded, not built now):** monthly full-restore drill in **GitHub
  Actions** (bundle → ephemeral pgvector service container → reindex → known query returns
  hits; environment shares nothing with prod); semi-annual manual DR rehearsal
  (`provision.sh` from scratch, timing the <30-min target).

**7. DB operational history — vault is the only never-lose tier.** All memory *content*
already reaches the vault (every capture ends as a note: organized, or an Inbox fallback on
model failure), so total DB loss never loses memory. Additionally: nightly-sync `/srv/data`
raw inputs to R2 (closes the "un-transcribed audio is VPS-disk-only" gap); nightly `pg_dump`
→ R2 (versioned) as a **second independent copy** so Supabase is no longer a single point of
failure. The DB does **not** get the vault's monotonic/object-lock treatment — operational
data (agent runs, chat) may **restore-to-last-nightly**; losing *today's* run/chat log in a
total-Supabase disaster is acceptable. **Chat history is classified operational, not memory**
(insights worth keeping are captured into the vault as notes, not preserved as raw logs).

> **Amendment (2026-07-13, M1).** Two refinements from the live drive:
> - **Pull-first push.** `VaultBackupService` now does a proactive merge-`pull` of the remote
>   *before* every push (after committing local work, so the tree is clean), so edits made on
>   GitHub or another device are integrated and the push is a plain fast-forward. Still
>   **merge-only, never rebase/reset** (§5 invariant preserved), and **best-effort**: an
>   unreachable remote or a conflicting merge never loses the local commit — it's aborted-clean
>   and the next backup reconciles. The non-ff heal-on-reject stays as a backstop.
> - **Housekeeping files reconciled on boot.** `ensure_ready` idempotently keeps the vault's
>   `.gitignore` (now also ignores `.idea/`) and a new **`.gitattributes`** (`* text=auto
>   eol=lf`, `*.md text eol=lf`) matching canonical content — committing only when they change.
>   The `.gitattributes` stops the CRLF↔LF churn Obsidian/Windows editing produced. §3 still
>   holds: the ignore set is editor/OS cruft only — never notes, never `.trash`.

## Consequences
- ✅ No single event — VPS disk loss, GitHub account/repo loss or takedown, a bad pipeline
  run, or a bad Obsidian merge/force-push — can destroy committed vault history.
- ✅ Append-only is enforced at the **storage layer** (R2 object-lock), so it needs neither a
  paid git host nor trusting every client. Total added cost ≈ **€0/mo** (R2 free tier).
- ✅ Supabase removed as a SPOF for operational state; recovery paths are *proven*, not assumed.
- ⚙️ New nightly jobs: `git bundle --all` → R2, `/srv/data` sync → R2, `pg_dump` → R2; a
  weekly restore-drill job; gc/reflog config at provisioning. New secrets: R2 credentials
  (VPS + CI). Reflected in 07-infrastructure and M1.
- ↩️ Fast-follows tracked: monthly CI restore drill, semi-annual DR rehearsal, optional
  GitHub Pro (self-defended live remote), optional second live remote (hot failover).
- ❌ Rejected: second live remote now (rewritable, no durability gain); GitHub Pro (≈ the
  VPS's whole price for no gain over WORM); full WORM/object-lock on the DB (operational, not
  crown jewels); treating chat history as never-lose memory.
