# ADR-050: A pipeline step's status is its own job's run, not the transitive scope

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision) · **Refines**
[ADR-047](047-pipeline-scheduling-primitive.md) §4–5 (per-step `on_fail` + status inference).

## Context

The M6 live Accept ran `python -m app.cli pipeline nightly` on prod. One start correctly drove
all 11 steps in dependency order under a single parent `agent_runs` run — but the summary read
**8 succeeded, 2 failed, 1 skipped**, with **`chat-distiller`** and **`inbox-drain`** reported
**failed**. No data was lost and both jobs did their work (4 endorsed; the drainer correctly found
2 captures still un-organizable). The "failure" was an artefact of how step status is computed.

The mechanism, traced end-to-end:

1. A capture whose organize can't produce valid typed nodes degrades to the **`inbox/` fallback**
   (rule 2 — never lose input). By deliberate design that organize run is closed
   **`status=failed`** with context, so the degradation is visible (rule 7 / vision P8;
   `capture_pipeline.py`). The data is safe (the capture lives as an `inbox/` node; the drainer
   retries it next window).
2. `chat-distiller` endorses a candidate → materializes a `captures` row → spawns a **background
   organize** (`asyncio.create_task`); `inbox-drain` **awaits** `reorganize_capture` inline. Both
   open a nested run under **`agent="capture"`**.
3. `child_run_scope` (ADR-047 §5) collects **every** `agent_runs` row opened while a step runs —
   via a contextvar the background task inherits — and stamps each with the pipeline
   `parent_run_id`. So the nested `capture` run is **flattened into the step's child set**, a
   direct child of the pipeline parent.
4. `_step_status` marked a step **failed if *any* run in that set** was not `{succeeded, skipped}`.
   A single benign `inbox/`-fallback `capture` run therefore failed the whole enclosing step.

So a **data-safe, expected degradation of one capture** surfaced as a **failed pipeline step**,
overstating the problem (the distiller's and drainer's *own* runs both closed `succeeded`) and — had
either step been `on_fail=halt` — would have wrongly aborted the roster on a benign inbox fallback.

This is a status-fidelity defect, not a data or ordering defect (P10 held; the pipeline continued
correctly under `continue`). It was grilled decision-by-decision at the M6 Accept.

## Decision

**A pipeline step's pass/fail is the outcome of the step's *own job run* — the run whose
`agent == step.name` — never the transitive set of every run opened under its scope.**

1. **Own-run status, by agent-name match.** `PipelineStepDef.name` already *is* the
   `agent_runs.agent` id (it is also the `_step_funcs` key), and each job's `run_scheduled` opens
   its own row under `start(AGENT)` with `AGENT == step.name`. `_step_status` derives the step's
   status from the child run(s) matching that agent; runs the job *spawns* under a different agent
   (`"capture"`, …) are **not** step-gating.
2. **`raised → failed`, unchanged.** If the step's function raises, the step is `failed` and any
   run it left `running` is closed `failed` (rule 7). A step that opened **no** own run (e.g.
   `store-sweep`) stays `skipped`.
3. **Nested runs stay visible.** Spawned `capture`/organize runs keep `parent_run_id = the pipeline
   parent` and remain fully visible in the feed (rule 7); they simply no longer determine the step's
   status. The step's reported `child_run_id` becomes its **own** run, not a random nested one.
4. **The inbox-fallback organize run keeps `status=failed`.** We do **not** relabel it — rule 7 /
   ADR-021's "an unusable organize is visible as `failed`" stands. The fix lives **only** at the
   step-rollup layer; organize / REST / feed semantics are untouched.
5. **Halt semantics preserved in spirit.** A `halt` step aborts when its **own** job fails or
   raises — a genuine, roster-blocking failure — never on a benign nested degradation. No genuine
   step failure is hidden: if a job's own pass raises (e.g. the distiller's `_distill_all`), its own
   run closes `failed` → the step fails → `halt` still aborts.

## Consequences

- `chat-distiller` and `inbox-drain` (and any future step that funnels through the organizer — rule
  2b) report their **own** outcome; a night with N inbox fallbacks no longer reads as N failed
  steps. Operators see the true signal: the job succeeded, and M separate `capture` runs degraded to
  `inbox/` (each visible, each retried by the drainer).
- The change is **general** — it keys on the step.name↔agent invariant, not on any one job — and is
  **server-only, no migration** (`agent_runs` schema unchanged; this is read-side status inference).
- A regression test asserts: a step whose scope contains a `failed` nested `capture` run is
  `succeeded` when its own run is `succeeded`; the step's own `failed`/raised run still fails (and a
  `halt` step still aborts on it); a nested run stays visible under the parent.

## Alternatives considered (rejected)

- **Relabel the inbox-fallback organize run** to a new non-failure status (e.g. `degraded`) added to
  the step's "OK" set. Rejected: larger blast radius (changes what `failed` means for organize across
  REST, the feed, and tests) and it softens rule 7's "an unusable organize is visible as `failed`" —
  the wrong layer to fix a step-rollup problem.
- **Identify the own run by first-opened id.** Works today (jobs open their own row before spawning
  nested work) but is positional and fragile — it breaks silently if a job ever spawns a nested run
  before opening its own. The agent-name match rests on a documented invariant instead.
- **Stop capturing background organize tasks in the scope.** Would drop the nested run's
  `parent_run_id` linkage (hurting visibility) and wouldn't fix `inbox-drain`'s *inline* reorganize
  at all.

## Out of scope (logged separately at the M6 Accept, not this ADR)

- **Why 2 of 4 chat-distilled captures fell to `inbox/`** — organizer quality on synthesized
  claim-text; a distinct concern, and the inbox-drainer already retries.
- **The `claude` Max CLI 300s hang** before the Nebius fallback on a nudge call — a VPS
  provider-health signal (visible on the Providers card, [ADR-044](044-provider-observability-surface.md)),
  independent of M6.
