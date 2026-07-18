# ADR-059: Roadmap restructure — Telegram removed, Slack deferred to M12, M9 rebuilt as multi-modal + Instagram

**Status:** Accepted · 2026-07-18 · **Supersedes** [ADR-033](033-external-inspirations-obsidian-second-brain.md)
#7 (Telegram capture as a must-have ingestion surface) · **Rewrites** the former §M9 of
[08-implementation-plan.md](../08-implementation-plan.md) (Slack + Telegram) · **Companion to**
[ADR-057](057-multimodal-media-ingestion-substrate.md) / [ADR-058](058-instagram-dm-connector-and-conversation-substrate.md).

## Context

The old M9 bundled the Slack connector with Telegram capture (ADR-033 #7 had promoted Telegram to
must-have). At the M9 planning session (2026-07-18) the user redirected: the priority is
multi-modal ingestion (images/video/voice) because the Instagram export — the personally most
valuable conversational source, already downloaded — requires it; Slack is not currently urgent;
Telegram is not wanted at all.

## Decision

- **Telegram capture is removed entirely** — not deferred: dead. User decision 2026-07-18,
  superseding ADR-033 #7's "must-have" promotion. (PWA + MCP remain the capture surfaces; M9's
  PWA photo capture covers the mobile-quick-capture need Telegram was meant to serve.)
- **The former M9 is replaced by two milestones:** **M9 — multi-modal ingestion foundation**
  (ADR-057) and **M9.5 — Instagram DM connector** (ADR-058), in that order (M9.5 consumes M9's
  substrate).
- **Slack moves to M12** — after M10 (reflection) and M11 (life-manager). Its spec in
  [05-connectors.md](../05-connectors.md) survives intact, and it inherits M9.5's conversation
  substrate + generalized distiller when built (a fetcher, not a new pipeline).

## Consequences

- ✅ The roadmap follows actual user value: the richest personal source (10 years of DMs) lands
  before a workplace source.
- ✅ Slack, when it comes, is cheaper than the old M9 planned — the M9.5 substrate does the heavy
  lifting.
- ⚠️ Anything downstream that cited "Telegram (M9)" (02-data-model source enum notes, ADR-046
  examples) reads as historical; the source enum gains `instagram` instead.
