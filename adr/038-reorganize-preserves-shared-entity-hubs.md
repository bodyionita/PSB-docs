# ADR-038: reorganize preserves shared entity hubs (M3)

**Status:** Accepted · 2026-07-14 · Fixes a graph-integrity bug found in the M3 task-10 live Accept ·
refines the reorganize path of [ADR-019](019-capture-followup-two-pass.md) and the entity substrate
of [ADR-030](030-entity-substrate-and-lifecycle.md). Decided in the M3 organizer-quality replanning
session ([09](../09-session-protocol.md)); recorded before coding.

## Context

The task-10 live Accept produced **dangling edges**: a `conversation` node's `involves`/`about`
edges pointed at entity ids (`person/Horia Fenwick`, a `topic`) with **no node file** in the store.

Root cause (confirmed in code): a capture runs `_replace_notes_via_reorganize` on its follow-up pass
(ADR-019 §2). That path does `remove_nodes(record.node_paths)` then re-organizes. `node_paths`
includes the **entity hubs the first pass minted** — so the follow-up **deletes the person/topic hub
files from disk**. The re-index step only indexes the *newly written* files, so the deleted hub
**lingers in the DB alias index**. The fresh pass's resolver then finds it as an exact-alias
candidate and **links** the new edges to that id (rather than re-minting) — while its file is gone.
Result: edges → deleted nodes.

The deeper flaw: a capture's reorganize treats minted entity hubs as *its own disposable nodes*, but
**hubs are shared substrate** referenced across captures (ADR-030: thin canonical hubs + derived
profiles, one node per entity, lifecycle owned by merge/backfill — not by whichever capture first
mentioned it).

## Decision

**A capture's reorganize removes only the content nodes it owns; entity hubs are never deleted by a
reorganize.**

1. **Type-aware removal.** `remove_nodes` on the reorganize path removes only nodes whose type is a
   **content type** (`memory`/`conversation`/`insight`/`idea`). Nodes of an **entity type**
   (`person`/`place`/`event`/`project`/`topic` — the `entity_types` set the resolver mints) are
   **skipped** and left on disk. The set of entity types is read from config (rule 9), not hardcoded.
2. **Hub lifecycle stays with the entity endpoints.** Hubs are created on first mention (mint) and
   removed/merged only via the explicit entity lifecycle (`POST /admin/entities/merge`, backfill,
   and a future GC) — never as a side effect of re-organizing one capture.
3. **Orphan hubs are tolerated.** If a reorganize's fresh pass stops referencing a hub the capture
   originally minted, that hub becomes an **orphan** (thin: no body, just aliases/disambig). Orphans
   are left in place. Deleting them eagerly risks removing a hub another capture is about to
   reference; a later **lightweight GC** (or the existing merge/backfill machinery) can prune hubs
   with zero inbound edges. No orphan sweep is added now.

## Rationale

- Removing only content nodes makes the reorganize idempotent w.r.t. shared state (rule 6): re-running
  a capture can never destroy an entity another node depends on, so a written edge always has a live
  target.
- It matches the ADR-030 ownership model — the graph has exactly one hub per entity, owned by the
  entity substrate, not by a capture.
- Tolerating orphans keeps the fix minimal and never-lose-safe; an orphan thin hub is harmless and
  cheaply reaped later.

## Consequences

- `_replace_notes_via_reorganize` / `NodeWriter.remove_nodes` gain the content-type filter; the
  first-pass write path is unchanged. Re-index after a reorganize continues to materialize edges from
  the (now consistent) files.
- Existing prod data already carries dangling edges from before this fix; they are healed by the
  `reprocess-all-from-raw` op ([ADR-042](042-reprocess-all-from-raw-and-data-survival.md)), not by a
  hand-patch.
- Follow-up: a **GC for zero-inbound-edge orphan hubs** (own ADR when built).
- `04-pipelines` (reorganize step) + `02-data-model` (hub lifecycle note) updated in the same change
  set; `08-logs/m3.md` records the bug + fix.
- ❌ Rejected: *delete-then-guarantee-re-mint* (id churn every reorganize) and *evict-deleted-entities
  -from-the-index* (still destroys/recreates hubs, churning ids + edges) — both treat the symptom, not
  the shared-substrate ownership error.
