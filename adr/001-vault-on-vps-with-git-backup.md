# ADR-001: Vault lives on the VPS; git is backup, history and the Obsidian bridge

**Status:** Accepted · 2026-07-12 · (supersedes the v1 idea of a local vault + Obsidian Sync)
**Partially superseded by [ADR-026](026-graph-native-storage-obsidian-removed.md) (2026-07-13):**
the files-on-VPS + git substrate stands; the Obsidian bridge and the "vault" naming are removed
(the store is now the **graph store** of typed nodes).
**Extended by:** [ADR-014](014-vault-history-durability.md) — hardens durability (R2 WORM
snapshots, rewrite protection, merge-only sync, restore drills). This ADR's core decision
(vault on VPS + git backup + Obsidian bridge) stands unchanged.

## Context
v1 assumed the vault on the user's machine with Obsidian Sync. Requirement changed: the
system must be fully available without any personal machine running. Obsidian Sync cannot
be written from a server (no public API, client-side E2E encryption), so a cloud-first
design needs the vault where the pipelines run.

## Decision
The canonical vault is a folder on the VPS. It is a git repository auto-committed and
pushed to a private GitHub repo after every write batch (debounced), on a nightly sweep,
and on demand. Obsidian becomes an optional exploration client via the same repo
(obsidian-git plugin on laptop/phone).

## Consequences
- ✅ Markdown stays the source of truth (vision P1) with off-site backup, full per-note
  history, and trivial recovery (`git clone` + reindex).
- ✅ Pipelines write files directly — no staging queues, no sync daemons.
- ⚠️ Manual edits made in a cloned vault must be pushed; the server pulls before nightly
  rescan to pick them up (merge conflicts resolved favoring remote HEAD is out of scope v1
  — single writer in practice).
- ❌ Rejected: Obsidian Sync as transport (no API, E2E-encrypted); Supabase Storage as
  vault (no folder semantics, no versioning, poor Obsidian interop).
