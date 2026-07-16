# ADR-047: The pipeline is the scheduling primitive (sequential steps, one start time)

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision) · **Supersedes** the
staggered-cron *mechanism* of [ADR-010](010-agent-window-3-5am.md) (the 03:00–05:00 window
stands; the per-job stagger does not); builds on the jobs-observability contract
([01-architecture.md](../01-architecture.md) invariant 4) and rule 7 (everything visible).
Lands in **milestone M5.5** (a small orchestration-only milestone inserted before M6, so a
scheduling regression can't masquerade as a chat-distiller bug).

## Context

Nightly agent + durability work was scheduled as **N independent cron jobs staggered by
hand** inside the 03:00–05:00 window (ADR-010): `data-sync 03:10 → db-backup 03:25 →
reindex 03:40 → profile-refresh 04:10 → backfill 04:20 → identity-capsule 04:35 →
store-sweep 04:55 → bundle 04:57`, plus a weekly integrity-drill. Two problems, both
sharpened by M6 adding four more jobs (chat-distiller, inbox-drainer, dedup-sweep,
maybe-digest) with **real ordering dependencies** (the distiller must write its nodes before
reindex; dedup needs fresh embeddings *after* reindex):

1. **Fragile ordering.** The stagger only *approximates* "run B after A." If a step overruns
   its budgeted slot (a slow reindex, a big backfill), the next step starts anyway and works
   on half-built state. Correctness depended on hand-tuned minute gaps that no one can
   guarantee.
2. **RAM stacking.** The stagger existed *because* two heavy jobs firing at once could OOM
   the VPS — but overruns can still overlap them, which is exactly what it was meant to
   prevent.

Both dissolve if steps run **sequentially, each starting only when the previous completes**,
from a **single scheduled start**.

## Decision

**The pipeline is the only schedulable unit. A bare job/agent is never cron-scheduled.**

1. **Pipeline** = a named, ordered list of **steps** + a schedule (one cron) + config. The
   runner executes each step, **awaits its completion, then starts the next** — regardless of
   how long any step takes. One start time, N sequential steps, no minute-tuning.
2. **Everything is a pipeline, even single-step work.** There is exactly one schedulable unit
   type. A lone job that needs a schedule is wrapped in a one-step pipeline.
3. **Multiple pipelines** may exist. Cadence maps to a pipeline: a **`nightly`** pipeline
   (daily steps) and a **`weekly`** pipeline (Sunday steps) rather than one pipeline with
   per-step day-of-week conditionals (which would re-introduce the per-step scheduling logic
   pipelines exist to remove).
4. **Per-step failure policy (`on_fail`).** Each step is tagged `continue` (its failure is
   recorded and the pipeline proceeds — the rule-7 default, so one flaky LLM call never costs
   the night its backups) or `halt` (its failure aborts the remaining steps — reserved for a
   foundational precondition). Set per step in the pipeline definition; the M6 nightly roster
   is `continue`-dominant.
5. **Visibility (rule 7 / invariant 4).** A pipeline run opens a **parent `agent_runs` row**;
   **each step keeps its own child `agent_runs` row** (linked by a new `parent_run_id`), so
   the existing per-job observability is unchanged and the parent records the sequence +
   per-step status. Richer visualization (a pipeline timeline) is deferred to the **M8** ops
   console.
6. **Jobs stay independently invokable** (CLI `python -m app.cli <job>` + `POST
   /agents/{name}/run`, invariant 4). The pipeline owns the *schedule*, not the *only* way to
   run a step. A manual run mid-pipeline is serialized by the existing single-flight locks
   (the store git lock + per-service guards), unchanged.
7. **The in-process scheduler registers one cron per pipeline**, not one per job. Pipeline
   definitions (name, cron, ordered steps, per-step `on_fail`) live in config (rule 9); the
   scheduler decides *when* a pipeline starts, the pipeline decides *what runs in what order*.

## Consequences

- **M5.5** builds the runner + config-defined pipelines and **migrates every existing nightly
  job** off its own cron into the `nightly`/`weekly` pipelines. Pure orchestration: **no job
  changes what it does**; the DB-wipe → reindex durability drill still passes. `agent_runs`
  gains `parent_run_id`.
- **M6** adds its four jobs as **steps** (born into the pipeline, never as crons): distiller
  near the front, inbox-drain before reindex, dedup late (post-embedding), maybe-digest in the
  `weekly` pipeline.
- The 03:00–05:00 window (ADR-010) is now enforced by *sequencing from a 03:00 start*, not by
  stagger; peak RAM is one step at a time by construction.
- ❌ Rejected: keeping per-job crons (fragile ordering, the reason for this ADR); one pipeline
  with conditional weekly steps (re-adds per-step scheduling logic); a full DAG scheduler with
  per-step dependency edges (over-built — a linear ordered list + `on_fail` covers the roster;
  revisit only if true fan-out/fan-in appears); aborting the whole pipeline on any failure
  (one transient blip would cost the night its durability steps).
