# Connectors (Ingestion Agents)

**Version:** 1.0 · **Status:** Approved 2026-07-12
**Key ADRs:** [008 connectors-on-vps](adr/008-connectors-run-on-vps.md) · [009 instagram-deferred](adr/009-instagram-connector-deferred.md)

A connector is an ingestion agent: it periodically pulls what happened in an external
source, uses an LLM to distill it into atomic plane-classified notes, and writes them to
the vault. Adding a source must never touch the rest of the system (vision P5).

## Contract

```python
class Connector(Protocol):
    name: str                      # unique, e.g. "slack"
    default_schedule: str          # cron expr within the 03:00–05:00 window

    async def fetch(self, cursor: dict | None) -> FetchResult:
        """Pull raw items newer than cursor. No LLM calls here."""

class FetchResult:
    items: list[SourceItem]        # normalized: text, author, timestamp, source_ref, thread context
    next_cursor: dict              # persisted only after successful materialization
```

The **distillation stage is shared**, not per-connector: one `DistillerService` receives
`SourceItem`s + connector context and produces notes (split per plane, frontmatter
contract from [02-data-model.md §2](02-data-model.md)). Connectors only know how to fetch
and normalize. This keeps LLM behavior consistent across sources and avoids duplicated
prompt logic.

Runtime: registered connectors are declared in config; the scheduler runs them in the
agent window; state in `connector_cursors`; every run in `agent_runs`
([04-pipelines.md §2](04-pipelines.md)).

**Definition of a new connector:** one module implementing `fetch` + a config entry.
Nothing else changes.

## Slack connector (v1)

**Decision:** ingest **conversations I participate in** — DMs, group DMs and channels
where I'm active — including other people's messages in those threads (full context).

- **Auth:** user token (`xoxp-…`) from a personal Slack app installed on the workspace
  (bot tokens cannot see the user's DMs). Scopes: `channels:history`, `groups:history`,
  `im:history`, `mpim:history`, `users:read`, `channels:read`, `groups:read`, `im:read`, `mpim:read`.
  Known risk: an employer workspace may restrict app installs — discovered at token
  creation time; fallback would be periodic personal export (not built until needed).
- **Fetch:** `conversations.list` (member=true) → per conversation `conversations.history`
  since cursor (`latest_ts` per channel in `connector_cursors.cursor`), thread replies via
  `conversations.replies` for threads I participated in. Normalize: resolve user IDs to
  names, keep thread grouping, drop joins/leaves/bot noise pre-LLM.
- **Distill:** group by conversation/topic; only substantive content becomes notes
  (decisions, commitments, information, personal moments); split per plane
  (default bias `professional`, but e.g. personal exchanges → `personal`/`friends`);
  `source_ref: "slack:<channel_id>/<thread_ts|ts>"`.
- **Schedule:** daily 03:00. Volume guard: max messages per run (config,
  default 2000) — overflow logged in run details, cursor advances only through
  what was processed.

## Deferred connectors (v2+)

| Source | Status | Notes |
|---|---|---|
| Instagram DMs | deferred ([ADR-009](adr/009-instagram-connector-deferred.md)) | Requires business account + Meta app review; API only covers business-messaging conversations. Needs a feasibility spike before committing. |
| WhatsApp, email, calendar, browser history, journals | ideas backlog | Each enters through the same contract when prioritized. |
