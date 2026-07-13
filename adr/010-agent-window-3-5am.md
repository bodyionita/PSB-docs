# ADR-010: All scheduled agent work runs 03:00–05:00 Europe/Bucharest

**Status:** Accepted · 2026-07-12
**Scope/terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md)–[029](029-conversational-ingestion-stance-gate-review-queue.md), 2026-07-13):**
the window stands; the stagger example is illustrative — Slack ingest is now **M9**, daily-summary/
weekly-review become the **M10** reflection agent's `insight` runs, "vault-backup" → store backup;
the **chat-distiller (M6)** joins the window.

## Context
The primary LLM provider is the user's Claude Max subscription (ADR-004), whose usage
windows are shared with his interactive Claude Code sessions during the day. Agent runs
(Slack distillation, summaries) are batch work with no freshness requirement finer than
daily.

## Decision
All scheduled jobs run inside a **03:00–05:00 local window**, staggered:
03:00 Slack ingest → 03:40 full rescan → 04:10 daily summary → 04:40 weekly review
(Sundays) → 04:55 vault-backup sweep. Manual triggers remain available any time via
`POST /agents/{name}/run` (user's explicit choice = user's budget).

## Consequences
- ✅ Agents consume Claude quota when the user is asleep, not competing with his work;
  RAM/CPU spikes also land off-hours.
- ✅ Stagger prevents provider rate-limit collisions and RAM stacking on the 4GB VPS.
- ⚠️ Slack content lands with up to ~24h latency — accepted for v1 (chat/search covers
  "right now" needs via capture).
- ⚠️ Missed window (VPS down) ⇒ next night covers it: cursors make ingestion catch up,
  summaries are re-runnable on demand.
