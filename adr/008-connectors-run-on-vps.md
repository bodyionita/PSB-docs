# ADR-008: Ingestion agents run on the VPS under one connector contract

**Status:** Accepted · 2026-07-12 · (supersedes the Supabase-Edge-Functions direction)

## Context
Ingestion agents were initially imagined as Supabase Edge Functions on cron (born when no
always-on machine existed). With the VPS decision (ADR-003), that premise died: edge
functions would mean a second runtime (Deno), duplicated secrets, execution-time limits
hostile to LLM distillation, and a staging table solely to reach the vault they can't
touch.

## Decision
Connectors are modules inside the VPS service, implementing one contract
(`name`, `default_schedule`, `fetch(cursor) → items + next_cursor`), scheduled by
APScheduler in the agent window, writing directly to the vault. Distillation is a shared
service, not per-connector. Cursors in `connector_cursors`, every run in `agent_runs`.
"Agent" is realized as an LLM-driven pipeline run, not a separate deployed artifact.

## Consequences
- ✅ One runtime, one secret store, direct vault access, no staging infrastructure.
- ✅ New source = one `fetch` module + config entry (vision P5).
- ✅ A heavy connector can still be extracted to its own process later — the contract
  doesn't know where it runs.
- ⚠️ Connector bugs share the process with the API; mitigated by per-run isolation
  (exceptions end the run as `failed`, never crash the service).
- ❌ Rejected: Edge Functions + staging + drain (complexity with zero remaining benefit);
  fully autonomous scheduled AI agents (non-deterministic, unidempotent, unpriceable).
