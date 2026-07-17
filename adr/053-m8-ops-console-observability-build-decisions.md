# ADR-053: M8 ops console & activity-restructure build decisions

**Status:** Accepted · 2026-07-17 (grilled decision-by-decision) · **Implements** the M8 scope in
[08-implementation-plan.md](../08-implementation-plan.md) · **Delivers** the jobs-observability
contract of [01-architecture.md](../01-architecture.md) invariant 4 · **Builds on**
[ADR-047](047-pipeline-scheduling-primitive.md) (pipeline = scheduling primitive; ambient
`parent_run_id` contextvar), [ADR-050](050-pipeline-step-status-is-the-jobs-own-run.md) (a step's
status = its own run), the `agent_runs` record (vision P8 / rule 7), and ADR-033 #5 (graph-health).

## Context

M8 is the observability milestone: it makes invariant 4 real ("every background action is persisted
and visible — **live while running** (status + logs), manually triggerable, schedule inspectable").
The pieces the contract already stubbed ([03-api.md](../03-api.md) §Activity & ops): a merged
`GET /activity` feed (categorized tabs), a live `GET /activity/runs/{id}/logs` tail, a `GET /agents`
roster, and `POST /agents/{name}/run` manual triggers — plus ADR-033 #5's `graph-health` job + panel
card, and the 06 §3/§6 web restructure (categorized Activity tabs; the M2 Admin panel absorbed into
the ops console).

The grill surfaced one load-bearing fact: **no per-run log capture exists today.** `agent_runs`
stores only an end-of-run `summary` + `details` JSON — nothing incremental. Every other M8 piece is
projection/CRUD over data that already exists (`agent_runs`/`captures`/`review_queue`, the scheduler's
`pipeline_defs`/`_step_funcs`, APScheduler `next_run_time`). So the "live log tail" is the one
genuinely new subsystem, and it drives the milestone's migration and most of its cost. This ADR
records the shape, grilled decision-by-decision (2026-07-17).

## Decisions

**1. Live logs = `app.*`-scoped logging handler → bounded per-run buffer → durable
`agent_run_logs` table.** A process-wide `logging.Handler` captures records **scoped to the active
run** via a `_current_run_id` contextvar (the same ambient-contextvar pattern ADR-047 §5 established
for `parent_run_id`, so **no job body changes**). The handler filters to **our namespace (`app.*`)
at `INFO`+ only** — this keeps clean, meaningful progress lines and structurally excludes library
`DEBUG` chatter, which is exactly where connection strings / bearer tokens / request bodies leak
(rule 11: a secret must never reach a tracked/rendered surface, and `agent_run_logs` is DB the UI
renders). Job-level failures still land in `agent_runs.error` + container logs as today; a
third-party `WARNING`/`ERROR` our code doesn't re-log won't appear in the tail (acceptable — rule 7
covers it at the run level).

**2. Write path = non-blocking in-memory buffer + async flusher (stdlib logging is sync; we're async
end-to-end, rule 8).** `Handler.emit()` runs synchronously inside whatever coroutine called
`logger.info(...)`, so it must not `await` a DB write. It appends `(seq, ts, level, message)` to a
**bounded per-run in-memory deque** (cheap, non-blocking). A small async flusher persists new lines to
`agent_run_logs` on a **~1s cadence** and once more **on run finish** (guaranteeing a completed run's
logs are fully durable). The poll endpoint reads the **table**, so the ~1s flush lag is the liveness
bound and the buffer never has to be shared across workers. The buffer is **bounded**: overflow
drops-oldest and records a `"…N lines elided"` marker (rule 7 — no silent caps). The only loss window
is lines emitted in the ≤1s before a hard crash mid-run — acceptable (a hard crash also fails the run
and it gets re-run).

**3. `GET /activity/runs/{id}/logs` = poll, not stream.** A cursor-paginated read over
`agent_run_logs` (`?after_seq=`), returning ordered lines + a `running` flag so the client knows when
to stop polling. **No SSE/WebSocket** — consistent with "true token streaming → backlog"; the web
polls at ~1s only while a run is active. (The endpoint already existed as an M8 stub;
`GET /activity/runs/{id}` stays the single-run status read.)

**4. Merged `GET /activity` = UNION-of-views over existing tables, keyset-paginated, 3 tabs.** The
feed is a **projection**, not a new events table (a table would be a derived copy that drifts from the
source rows — against rule 1's durable-truth spirit). A `UNION ALL` over `agent_runs` + `captures` +
`review_queue`, each normalized to a common `{id, category, kind, ts, title, snippet, ref}` shape,
ordered `ts DESC`, **keyset-paginated on `(ts, id)`** via the `before=` cursor. Three tabs
(`?category=`):
- **agents/jobs** — scheduled `agent_runs` (pipeline parents + their step children, nested).
- **conversations** — chat-derived events (auto-endorsed `source=chat` captures — the M6
  "recently auto-recorded" list folds in here — + chat-distiller outcomes).
- **manual actions** — human-initiated: manual runs, review verdicts, hand-triggered admin ops.

**5. Category is assigned by *origin*, not by table — via a new `agent_runs.trigger` column.** An
`agent_runs` row can be both a job and manually triggered (a hand-run `reindex`). To land a manual
reindex under **manual actions** rather than looking like a nightly job, `agent_runs` gains a
**`trigger` column (`scheduled` | `manual`, default `scheduled`)**, set through a `_trigger`
contextvar the manual endpoint sets around the call (mirrors `parent_run_id` — **no job-body churn**).

**6. Ops surface: agents and pipelines are separate resources; a job belongs to 0..N pipelines.**
- **`GET /agents`** → a **flat roster** of individual jobs. Each: `{name, category, pipelines:
  [names], last_run: {status, finished_at, run_id}}`. A job's schedule is *derived* from its pipeline
  memberships. **Membership is many-to-many** — a step name may appear in several pipeline defs (today
  each happens to sit in one, but nothing may assume it).
- **`GET /pipelines`** (new) → pipelines as their own resource: `{name, cron, next_run, steps:
  [ordered names], last_run}`, sourced from the live scheduler (`pipeline_runners()` + APScheduler
  `next_run_time`). This is the home the schedule view lives in and where **pipeline editing** and
  **whole-pipeline manual runs** grow.
- **`POST /agents/{name}/run`** (one zero-arg job standalone) and **`POST /pipelines/{name}/run`**
  (a whole pipeline; the CLI verb from ADR-047 §6, now over HTTP). `409` if already running.

**7. Single-flight = one in-process JobRunner guard the scheduler and the manual endpoint both route
through.** A manual trigger is a new concurrency source (hand-run `reindex` while the nightly pipeline
is on `reindex`). A small **in-process per-agent guard** (running-set / per-name lock) serializes
them — **authoritative because the scheduler must run in a single process** (two schedulers would
double-fire, so there is no cross-worker race). The JobRunner seam is the single home for the guard +
the `_trigger`/`_current_run_id` contextvar setup + the log-handler attach/detach; job bodies stay
untouched.

**8. One unified, ordered ops console — everything runnable is present.** The split is *mechanical*,
not an agent/admin wall:
- **Zero-arg jobs** — every pipeline step + `graph-health` + `reindex` + `db-backup`/`store-backup`:
  a bare `POST /agents/{name}/run`, a plain **Run** button, listed in **nightly-pipeline order** so
  running them by hand in a sensible sequence is natural.
- **Parameterized ops** — `reprocess` (confirm), `entities/merge` (which two nodes), `tags/consolidate`
  + `vocab/consolidate` (two-step propose→apply): they **cannot** collapse to a bare Run (an
  entity-merge is meaningless without its two nodes), so they keep their existing `/admin/*` workflows
  and input controls, **rehomed into the same console** rather than a separate Admin tab.

**9. `graph-health` = read-only reporter; findings in `agent_runs.details`; nightly-tail.** The
checks (ADR-033 #5): orphan nodes (no edges), `inbox/` depth, pending-review aging, memories missing
`occurred`, alias-less entities, tombstone integrity (no dangling `merged_into`), freshness flags
(stale `(as of …)` observations). It **only computes and reports** — never creates review items,
never mutates the graph (trivially idempotent, rule 6); acting on a flag is M10's reflection agent /
manual (auto-vs-manual remediation revisitable then). Findings are written to the run's **`details`
JSON**; the console's graph-health card reads the **latest** `graph-health` run (no new table — the
Q4 feed call again). Runs as the **last step of the nightly pipeline** (reports on the settled
post-reindex/post-dedup state; `on_fail: continue`). Thresholds (review-aging days, freshness window,
sample-offender count) are **config knobs** (rule 9). The freshness check is the only more-than-a-count
query (it parses profile `(as of …)` stamps).

**10. `store-sweep` gets its own run row.** Carried M5.5-task-3 follow-up tagged M8:
`run_store_sweep` opens no `agent_runs` row, so it shows as a phantom `skipped` step every night.
M8 is the observability milestone rendering exactly this — give it a run row like every other step so
the feed/console shows it honestly.

**11. Web = one Activity tab, Feed/Ops segmented sub-views.** No new top-level tab (there are already
8). A segmented control:
- **Feed** — the 3 categorized tabs, infinite-scroll (`before=`), tap-to-expand run details,
  pipeline parent→child nesting, fallback badges; the auto-recorded one-tap-remove list inline in
  Conversations.
- **Ops** — pipelines (schedule + next-run + ordered steps + whole-pipeline Run), the agent roster in
  pipeline order with per-agent Run + last-run status, the **live log-tail panel** (polls
  `GET /activity/runs/{id}/logs` while active), the graph-health card, and the rehomed parameterized
  admin ops.
Liveness: TanStack Query `refetchInterval` active **only while something is running** (log tail +
run-status ~1s; feed slower / manual refresh).

## Consequences

- **One migration:** `agent_run_logs` table (`run_id` FK, `seq`, `ts`, `level`, `message`) + a
  `agent_runs.trigger` column (`scheduled` default). Both additive; rebuildable op-state (rule 1 —
  they are not graph truth).
- **One new subsystem** (log capture: handler + contextvar + buffer + flusher + JobRunner seam) and
  otherwise **projection/CRUD** over existing tables + the live scheduler. No organizer touch, no new
  store surface, no vendor SDK.
- New endpoints: `GET /activity` (merged feed), `GET /pipelines`, `POST /agents/{name}/run`,
  `POST /pipelines/{name}/run` (`GET /activity/runs/{id}/logs` + `GET /agents` were M8 stubs, now
  implemented). Config knobs: log buffer size / flush cadence + graph-health thresholds.
- The M2 Admin panel and the M6 chat "recently auto-recorded" list are both **absorbed** into the
  Activity screen — the ops console and the Conversations feed respectively.

## Fenced out of M8 (recorded)

- **Pipeline editing** — needs `pipeline_defs()` to move config→DB + the scheduler to read from
  there; its own grill + ADR. M8 is read + trigger only.
- **Connectors in the roster** — M9 (Slack) adds itself when real.
- **graph-health auto-remediation** (auto-drain on `inbox/` depth, staleness interviews, orphan
  cleanup) — M10 / revisitable.
- **True log streaming (SSE/WebSocket)** — backlog; M8 polls.
- Known residuals untouched: the `claude` CLI 300s-hang-before-fallback, the Sunday nightly/weekly
  RAM overlap (ADR-047 §3), the `occurred`-enrichment review kind (its own backlog session).

## Alternatives considered

- **Explicit `run.log(msg)` progress API** (jobs call it at checkpoints) — rejected as the primary
  mechanism: touches every job body and only shows what we remember to instrument; the `app.*`
  handler captures real output for free. A job wanting richer output just logs more at `INFO`.
- **Tail the container stdout / journald** — rejected: fragile, infra-coupled, hard to scope to one
  run.
- **In-memory-only ring buffer (no table)** — rejected: a job that fails at 03:00 is exactly when you
  want its logs at 09:00; the table is cheap and survives restart.
- **Capture everything at `INFO`+ (all namespaces)** — rejected: library noise + the DEBUG/INFO
  secret-leak surface on a UI-rendered store (rule 11).
- **A dedicated `activity_events` table** — rejected: a derived copy of `agent_runs`/`captures`/
  `review_queue` that drifts; the source rows *are* the events (rule 1).
- **Pipeline-grouped `GET /agents`** (steps nested under their pipeline) — rejected on the user's
  call: pipelines are a first-class resource of their own (schedule, future editing, whole-pipeline
  run); a job belongs to 0..N of them, so grouping is the wrong shape.
- **Per-line synchronous DB insert from the handler** — rejected: a blocking DB round-trip on the
  event loop per log line (rule 8), plus ordering headaches.
