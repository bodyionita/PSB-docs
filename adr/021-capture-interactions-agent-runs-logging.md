# ADR-021: Capture model interactions logged to `agent_runs` + Supabase-native exploration

**Status:** Accepted · 2026-07-12
**Terminology/milestone note ([ADR-026](026-graph-native-storage-obsidian-removed.md), 2026-07-13):**
"note" → node, `note_paths` → `node_paths`, "vault" → graph store; the in-app activity UI this ADR
calls "M4" is now the **M8** ops-console/activity restructure. The logging decision stands unchanged.
**Relates to:** vision **P8 "everything visible"** · [CLAUDE.md rule 3 (fallback recorded) + rule 7 (agent work in `agent_runs`)] · [002 Supabase/pgvector](002-supabase-pgvector-for-index.md) · [020 STT fallback chain](020-stt-fallback-chain-groq-primary.md) · updates [04-pipelines](../04-pipelines.md), [02-data-model](../02-data-model.md), [08-implementation-plan §M1](../08-implementation-plan.md)

## Context
The M1 live drive raised: *where do the model interactions behind a capture surface, so they can
be explored easily — ideally free/included in the current architecture?* Today the capture
pipeline records only the `captures` row (`status`, `error`, `raw_text`, `note_paths`) — which
drives the recent-captures strip — and writes **nothing** to `agent_runs`. Which STT/organize/
nudge model answered, whether a fallback fired, and the OpenAI 429 itself live **only in the
container's stdout logs** — nowhere queryable.

Two forces make this a now-decision rather than a nice-to-have:
- **Rule 3, newly triggered by [ADR-020](020-stt-fallback-chain-groq-primary.md):** the STT chain
  must *record* its fallback resolution. It needs a home.
- **Rule 7 / vision P8:** agent/job work lands in `agent_runs` with a human-readable summary +
  details JSON. The capture pipeline runs **async, fire-and-forget after the 202** — it *is* a
  background job and belongs in `agent_runs` like every other.

## Decision

**1. One `agent_runs` row per capture-pipeline run** (`agent = "capture"`).
- `status`: `running` → `succeeded` / `failed` (mirrors the durability jobs).
- `model_used`: the organize model; `fallback_used`: **true if any step fell back** (STT or
  organize).
- `details` JSON captures the sub-steps:
  ```json
  {
    "capture_id": "…",
    "kind": "voice|text",
    "stt": { "provider": "groq", "fallback_used": false, "error": null },
    "organize": { "model": "claude-max", "fallback_used": false },
    "nudge": { "model": "claude-max" },
    "timings_ms": { "transcribe": 0, "organize": 0, "total": 0 }
  }
  ```
- The **`captures` table is unchanged** — it stays lean for the strip; per-capture *provenance*
  and interaction detail live in `agent_runs`. (Rejected: `model_used`/`fallback_used` columns on
  `captures` — a capture makes three model calls each with its own fallback; one column-pair
  can't represent that, and a `details` JSON on `captures` would just duplicate `agent_runs`.)

**2. A `capture_interactions` Postgres view** (migration 003, explicit SQL per
[ADR-011](011-alembic-migrations-plain-sql-no-orm.md)) flattens the `details` JSON of
`agent='capture'` rows into readable columns — `capture_id, kind, stt_provider, stt_fallback,
organize_model, fallback_used, status, error, started_at, duration_ms`. Turns the raw JSON blob
into a clean, sortable table for the dashboard; the same view a future in-app activity screen
(M4) can read.

**3. Exploration is Supabase-native — no new infra.**
- **Supabase dashboard** (Table Editor + SQL Editor) is the interface — free/included, works the
  moment the rows exist. Filter `agent_runs` to `agent='capture'`, or open `capture_interactions`.
- **Supabase MCP** (official, **read-only**, project-scoped) is the optional conversational path
  ("show the last 10 failed captures and why"), added interactively via `claude mcp` on the dev
  machine when wanted — not wired from a build session.
- **Raw docker/stdout logs stay as-is** (container logs for incident debugging on the box); the
  *structured* interaction log in `agent_runs` covers "explore what the models did." Aggregating
  raw logs into Supabase (Logflare etc.) is explicitly **out of scope for v1**.
- **No custom in-app UI before M4** — the dashboard + view cover exploration until the planned
  activity feed lands.

## Consequences
- ✅ Satisfies rule 3 (STT resolution recorded), rule 7 / P8 (capture work now visible), and the
  "explore easily, free/included" ask — all with the existing Supabase project.
- ✅ Reuses the existing `AgentRunStore` seam + its in-memory fake, so the capture pipeline's new
  logging is unit-testable without a live DB.
- ✅ `capture_interactions` is the seam the M4 activity feed reads — no rework later.
- ⚙️ The capture pipeline gains an `agent_runs` open/close around the run; logging never changes
  capture behavior (a logging-store failure is swallowed — the capture still runs). Run status
  follows the **capture** outcome, not the model calls: a genuine capture failure (STT chain
  exhausted, vault write error) ends the run `failed` with context (rule 7); but an **organize
  outage that degrades to an Inbox note is a capture *success*** (never-lose — the note is written
  and indexed, rule 2), so the run ends `succeeded` — marking it `failed` would falsely flag a
  saved note in the activity feed. The degradation stays visible via `details.organize.inbox_fallback`
  (and the `capture_interactions.inbox_fallback` column) + the run summary, so a dashboard can find
  "organize was down" captures without treating them as failures.
- ⚙️ Adds migration 003 (a view; no table/column change).
- ↩️ Supabase MCP wiring, raw-log aggregation, and the in-app activity UI are deferred (MCP =
  user's call; aggregation = out of v1 scope; UI = M4).
