# Connectors (Ingestion Agents)

**Version:** 3.0 · **Status:** Approved 2026-07-18 (3.0 = the **conversation substrate**:
connectors produce normalized *messages* into source-generic tables; deterministic
**sessionization** + the **generalized stance-gated distiller** turn them into memories
([ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md)); media handled by the
multi-modal substrate ([ADR-057](adr/057-multimodal-media-ingestion-substrate.md)); **Instagram
promoted from deferred to M10** (export-first; ADR-009 resolved); **Slack → M13**; **Telegram
removed** ([ADR-059](adr/059-roadmap-restructure-telegram-removed-slack-m12.md)). 2.0 =
mind-graph pivot: distillation emits typed nodes/edges through the organizer; stance gate +
review queue; 6-month default lookback. 1.0 = initial contract.)
**Key ADRs:** [008 connectors-on-vps](adr/008-connectors-run-on-vps.md) ·
[009 instagram-deferred → resolved](adr/009-instagram-connector-deferred.md) ·
[029 stance gate](adr/029-conversational-ingestion-stance-gate-review-queue.md) ·
[048 distiller loop](adr/048-m6-chat-distiller-build-decisions.md) ·
[057 media substrate](adr/057-multimodal-media-ingestion-substrate.md) ·
[058 instagram + conversation substrate](adr/058-instagram-dm-connector-and-conversation-substrate.md)

A connector is an ingestion agent: it pulls what happened in a source, and the shared machinery
turns it — **via the organizer, the single writer of graph structure** — into typed,
plane-classified, edge-linked nodes in the graph store. Adding a source must never touch the rest
of the system (vision P5).

**Lookback policy (all connectors):** default ingestion horizon = **6 months into the past**;
per-connector override in the UI (`PUT /settings/connectors/{name}`). Applies to first syncs and
API backfills. **Export-based historic imports are exempt by nature** — the user explicitly
curates their scope in the triage manifest (ADR-058 §2).

**Stance gate (conversational sources):** distillation anchors on the **user's** stance —
commitments, agreements, decisions. Stance-unclear candidates go to the **review queue**
(agree/disagree/maybe), never guessed ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).
Backfill-scale runs add filing protections (salience floor + per-run cap, skips logged and
re-distillable — ADR-058 §7); the gate's *judgment* is never source-specific.

## The conversation substrate (v3 contract, [ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md) §3–§6)

Conversational sources converge on **source-generic tables** — `connector_threads` /
`connector_messages` / `connector_media` ([02-data-model](02-data-model.md)) — holding normalized
messages as **connector raw** (the durable server-side replay source). Two producer shapes feed
them:

- **API fetcher** (runs on the VPS, scheduled): `fetch(cursor) → normalized messages`, upserted
  by `(source, thread_id, message_id)`; cursor persisted only after successful materialization.
  No LLM calls in fetch.
- **Local import tool** (runs on the user's machine against an official export): parse +
  normalize + user triage, then batched **idempotent upload** into the same tables (same upsert
  key). Media: photos/voice upload raw; videos upload as locally-processed summaries
  (ADR-057 §2).

Downstream is shared, source-blind, and always re-runnable over the stored raw:

1. **Sessionization** — deterministic code (no LLM): chronological sort, split on a >6h gap
   (config); pathology guard splits at the largest internal gap past a token cap. One session =
   one distillation unit; **no summary-chaining** (ADR-058 §4).
2. **Media derivation** — the ADR-057 stage (photo descriptions, voice STT) with status-tracked
   resumability; a session distills when its media are derived (or marked `unavailable` →
   explicit placeholder).
3. **Distillation** — the generalized M6 loop (ADR-048 → ADR-058 §6): per-session
   multi-candidate distill on the rendered transcript (`[HH:MM] Name:` lines, day dividers,
   media placeholders, reactions; the ADR-057 §5 attribution contract binding), stance gate,
   endorsed → **`captures` row (`source=<source>`, `source_ref` = session, `anchor_at` = session
   time)** → organizer; unclear → review. P10 holds via captures replay; re-distillation from
   stored sessions is separately possible, targeted (ADR-058 §9).

**Definition of a new conversational connector:** one producer (fetcher module *or* import tool)
+ a config entry. Sessionization, distillation, review, media, transcripts — all inherited.

**Traceability:** memory → capture → `source_ref` → session; the session-transcript endpoint +
web view render the conversation with inline media ([ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md) §11).

## Instagram DM connector (M10 — [ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md))

**Decision:** ingest **DMs only** (group chats excluded — user call; `message_requests` never
inventoried), **export-first** for history, API only as the gated freshness path.

- **Historic spine — official data export**, processed by the local prep tool
  (`second-brain/tools/instagram-export/`): mojibake repair → inventory → **opt-in CSV triage
  manifest** (whitelist; `name_override`/aliases for deleted accounts; auto-skip tier for <5-msg
  one-sided threads; manifest local + gitignored — real names never committed) → local video
  processing → resumable, idempotent upload. Full design: ADR-058 §2.
- **Ongoing — spike-gated** (§10): the account is professional, so `/me/conversations` may serve
  the user's own DMs without App Review (app-role exemption). Time-boxed spike proves own-DM
  read, history depth, echoes, media URLs; **webhooks preferred**, frequent polling second.
  Pass → daily fetcher into the substrate; fail → **periodic manual re-export** (a cheap delta:
  upsert + watermarks skip everything already ingested).
- **Backfill campaign:** admin-triggered, single-flight, resumable; **parallel Claude distill
  workers** (default 4) + serial organize; review-queue protections on (§7–§8).
- `source_ref`: `instagram:<thread_id>/<session>`; sender attribution structural; the user's
  account name marked first-person via the manifest.

## Slack connector (M13 — deferred by [ADR-059](adr/059-roadmap-restructure-telegram-removed-slack-m12.md); spec unchanged, inherits the v3 substrate)

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
  names, keep thread grouping, drop joins/leaves/bot noise pre-LLM — **into
  `connector_messages`** (v3: the per-channel cursor persists in `connector_cursors`; sessions,
  distillation, media, transcripts all inherited from the substrate).
- **Distill:** shared (v3) — plane bias `professional`, personal exchanges →
  `personal`/`friends`; `conversation` nodes edged (`involves`) to `person` nodes;
  `source_ref: "slack:<channel_id>/<thread_ts|ts>"`.
- **Schedule:** daily 03:00. Lookback: the 6-month default on first sync. Volume guard: max
  messages per run (config, default 2000) — overflow logged in run details, cursor advances
  only through what was processed.

## Deferred connectors (v2+)

| Source | Status | Notes |
|---|---|---|
| **WhatsApp** | ideas backlog — **next natural substrate consumer** | Official chat export (per-chat txt/zip incl. media) → a new import tool over the same substrate; group support would be grilled here (ADR-058 §12). |
| **LLM-chat exports** (ChatGPT/Claude history) | backlog, promoted by the pivot | Shared-conversation URLs / export files → the same stance-gated distillation; `source_ref` = the share URL / export locator. 6-month default lookback applies to backfills. |
| Facebook Messenger, email, calendar, browser history, journals | ideas backlog | Each enters through the same contract when prioritized. |

*(Telegram: removed entirely — [ADR-059](adr/059-roadmap-restructure-telegram-removed-slack-m12.md), superseding ADR-033 #7.)*
