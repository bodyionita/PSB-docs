# ADR-042: reprocess-all-from-raw op + the data-survival principle (M3)

**Status:** Accepted · 2026-07-14 · Establishes a standing principle + the mechanism to honor it,
prompted by the M3 task-10 Accept surfacing multiple defects in already-ingested data. Builds on the
never-lose invariant ([CLAUDE.md rule 2](../templates/CLAUDE.md)), store-is-truth ([rule 1](../templates/CLAUDE.md)),
and idempotency ([rule 6](../templates/CLAUDE.md)). Decided in the M3 organizer-quality replanning
session; recorded before coding.

## Context

The Accept fixes (ADR-038…041) change how nodes are produced and their on-disk/DB shape. The 4 prod
captures already ingested carry the old defects (dangling edges, over-extracted person nodes,
mangled diacritics). There was no sanctioned way to heal already-ingested data through a fix — only
per-capture admin reorganize, which leaves old shared hubs behind.

**Standing principle (user, binding for all future work in this repo):** *already-ingested data must
try to survive bug fixes and format changes.* A fix that silently corrupts, drops, or leaves stale
already-ingested nodes is worse than the bug. Because raw inputs are always retained (rule 2),
nothing forces data loss: prefer **re-processing the original raw inputs as a fresh ingestion** and
cleaning up old derived artifacts. If a fix does **not** auto-heal previously-ingested data, **stop
and tell the user**, and let them choose migrate vs delete — never leave silently-broken history.

## Decision

**Add a first-class, reusable `reprocess-all-from-raw` admin operation; it is the standing mechanism
for the data-survival principle.**

1. **What it does.** Resets the **capture-derived** graph, then replays **every capture's raw input**
   through the current (fixed) pipeline in **original chronological order** (so entity accretion
   rebuilds naturally), using each capture's combined text where a follow-up answer exists (ADR-019).
   Each replay is a normal organize→resolve→write→index, so it inherits every current fix for free.
2. **Reset contract — what is preserved vs rebuilt.**
   - **Always kept:** raw `captures` (text/audio/source_ref) and the graph store **git history**
     (removals are unlinks; history retains prior content, ADR-014 §3).
   - **Preserved (human governance):** approved **vocabulary** additions (`app_settings`); standing
     entity **merges** are re-applied **best-effort** (see limitation).
   - **Rebuilt from raw:** all node files, the DB index (`nodes`/`chunks`/`edges`/`node_profiles`),
     entity hubs + alias index, and the **review queue** (entity-ambiguity / vocab-proposal items are
     capture-derived and re-minted by the replay).
3. **Safety + visibility.** Destructive of derived state → admin-gated, **confirm-required** (two-step,
   ADR-024 envelope shape), runs in the background with an `agent_runs` row + human-readable summary
   (rule 7). Idempotent (rule 6): running it twice yields the same graph. Raw is never touched, so a
   bad reprocess is recovered by fixing code and reprocessing again.
4. **Merge re-apply limitation (documented).** A full rebuild regenerates entity hubs with **new
   ids**, so re-applying a prior merge means re-identifying the surviving/absorbed hubs by
   alias/name, not id. There are **zero merges today**, so this is a no-op for the M3 reprocess; the
   general re-identify-and-re-apply is a **documented follow-up inside the op** (own refinement when a
   merge exists to preserve). Until built, the op **reports** any standing merges it could not
   re-apply rather than dropping them silently (principle: surface, don't lose).

## Rationale

- One reusable operation turns "we changed the format" from a data-loss risk into a routine replay,
  which is exactly the principle. It reuses the whole capture pipeline, so it can never drift from how
  fresh captures are processed.
- Preserving governance (approved vocab) but rebuilding derived nodes keeps the human decisions that
  aren't capture-derived while regenerating everything that is — the least-surprising contract.
- Chronological replay makes alias accretion (ADR-040) and entity resolution deterministic and
  faithful to how the data actually arrived.

## Consequences

- New `POST /admin/reprocess` (name TBD at build) + service + CLI (`python -m app.cli reprocess-all`),
  behind confirm. New reset helpers on the store writer + DB. No change to the capture request path.
- **M3 usage:** after ADR-038…041 land, this op heals the 4 prod captures — validated by a **local
  dry-run against a local DB + captures** before the prod run (the op is destructive; the local run
  proves it), per the task-11 sequencing.
- The principle is lifted into [00-vision](../00-vision.md) as a standing operating principle so every
  future organizer/indexer/migration change plans its data-survival path.
- `03-api` (§Admin: reprocess) + `04-pipelines` (reprocess flow) + `02-data-model` (reset contract) +
  `08-implementation-plan`/`08-logs/m3.md` updated in the same change set.
- ❌ Rejected: *rebuild absolutely everything incl. approved vocab* (silently undoes governance every
  reprocess); *one-off throwaway wipe for these 4* (doesn't build the standing mechanism the principle
  needs); *per-capture reorganize only* (leaves old shared hubs as orphans — partial clean).
