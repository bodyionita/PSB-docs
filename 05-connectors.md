# Connectors (Ingestion Agents)

**Version:** 2.0 · **Status:** Approved 2026-07-13 (2.0 = mind-graph pivot: distillation emits
typed nodes/edges **through the organizer**; **stance gate + review queue** for conversational
sources [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md); **default
6-month lookback**, UI-overridable; the in-app **chat-distiller** is a connector too. 1.0 =
initial contract.)
**Key ADRs:** [008 connectors-on-vps](adr/008-connectors-run-on-vps.md) · [009 instagram-deferred](adr/009-instagram-connector-deferred.md) · [028](adr/028-one-service-layer-mcp-peer-surface.md) · [029](adr/029-conversational-ingestion-stance-gate-review-queue.md)

A connector is an ingestion agent: it periodically pulls what happened in a source, and the
shared distiller turns it — **via the organizer, the single writer of graph structure** — into
typed, plane-classified, edge-linked nodes in the graph store. Adding a source must never touch
the rest of the system (vision P5).

**Lookback policy (all connectors):** default ingestion horizon = **6 months into the past**;
per-connector override in the UI (`PUT /settings/connectors/{name}`). Applies to first syncs and
backfills (Slack history, LLM-chat exports, etc.).

**Stance gate (conversational sources):** distillation anchors on the **user's** stance —
commitments, agreements, decisions. Stance-unclear candidates go to the **review queue**
(agree/disagree/maybe), never guessed ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).

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
`SourceItem`s + connector context, applies the stance gate where the source is conversational,
and hands candidates to the **organizer** (typed nodes + edges, contract in
[02-data-model.md §2](02-data-model.md)). Connectors only know how to fetch and normalize. This
keeps LLM behavior consistent across sources and avoids duplicated prompt logic. The **in-app
chat-distiller** ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)) is a
connector under this contract whose "external source" is `chat_sessions`.

Runtime: registered connectors are declared in config; the scheduler runs them in the
agent window; state in `connector_cursors`; every run in `agent_runs`
([04-pipelines.md §2](04-pipelines.md)).

**Definition of a new connector:** one module implementing `fetch` + a config entry.
Nothing else changes.

## Slack connector (M9)

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
- **Distill:** group by conversation/topic; only substantive content becomes nodes
  (decisions, commitments, information, personal moments) — **stance-gated per ADR-029**
  (unclear stance → review queue); split per plane (default bias `professional`, personal
  exchanges → `personal`/`friends`); `conversation` nodes edged (`involves`) to `person` nodes;
  `source_ref: "slack:<channel_id>/<thread_ts|ts>"`.
- **Schedule:** daily 03:00. Lookback: the 6-month default on first sync. Volume guard: max
  messages per run (config, default 2000) — overflow logged in run details, cursor advances
  only through what was processed.

## Deferred connectors (v2+)

| Source | Status | Notes |
|---|---|---|
| **LLM-chat exports** (ChatGPT/Claude history) | backlog, promoted by the pivot | Shared-conversation URLs / export files → the same stance-gated distillation as the in-app chat-distiller; `source_ref` = the share URL / export locator. 6-month default lookback applies to backfills. |
| Instagram DMs | deferred ([ADR-009](adr/009-instagram-connector-deferred.md)) | Requires business account + Meta app review; API only covers business-messaging conversations. Needs a feasibility spike before committing. |
| WhatsApp, email, calendar, browser history, journals | ideas backlog | Each enters through the same contract when prioritized. |
