# Personal Second Brain — Vision

**Version:** 3.0 (supersedes v2.0 — pivot to a **typed mind graph**, Obsidian removed;
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–[029](adr/029-conversational-ingestion-stance-gate-review-queue.md))
**Status:** Approved 2026-07-13

## Vision

A mind graph of my life, available anywhere, any time, to me and to my AI tools. It ingests
everything — spoken thoughts, written notes, ideas, conversations with people across sources,
and my discussions with LLMs — and holds it **organized, categorised, and mapped with meaningful
relations**: memories linked to the people they involve, ideas linked to where they came from,
conversations linked to what they decided. I can question it in chat, feed it and query it from
any LLM via MCP, and walk it visually as a map of my own mind. The goal is not storage; it is an
extension of memory and a reflection partner: information → knowledge → understanding,
compounding over time.

## What it is, concretely

- A **typed knowledge graph** as the canonical memory: nodes (`memory`, `person`, `idea`,
  `conversation`, `insight` — a governed, growing vocabulary) connected by typed edges
  (`involves`, `about`, `part_of`, `led_to`, `follows`), stored as plain Markdown files in a
  git-versioned **graph store** on my own server, fully recoverable
  ([ADR-026](adr/026-graph-native-storage-obsidian-removed.md), [027](adr/027-typed-vocabulary-governance.md)).
- A **secure personal web app** (PWA on phone + desktop): talk or type to capture (voice in any
  language, incl. Romanian), chat over the whole memory, review what conversations proposed to
  remember, watch what the system did, choose the AI model that answers.
- An **MCP server**: other LLMs (Claude anywhere, other clients) query the graph and feed it
  with the same discipline as the app — one service layer, thin surfaces
  ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)).
- **Conversations become memory**: in-app LLM chats are distilled — anchored on *my* stance,
  not the model's words — and people-conversations (Slack first) flow in through connectors;
  anything with unclear stance waits in a review queue for my agree/disagree
  ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).
- **The map**: point-and-click visual exploration of the graph — start anywhere, expand
  outward, see the constellations around a person or an idea. Desktop-first.
- **Background intelligence**: reflection over days/weeks/months/years (what went well, what to
  work on), schedule/goal awareness — agents that produce insight, not just summaries, every
  run observable live.

## Planes of life

Nodes carry **planes** — configurable life areas used for scoping, filtering and coloring
(no longer folders — [ADR-026](adr/026-graph-native-storage-obsidian-removed.md)):

`Professional` · `Personal` · `Family` · `Friends` · `Health` · `Ideas` (initial set; config-driven)

- Every node has one primary `plane` + full `planes[]` membership; membership truth is
  frontmatter ([ADR-005](adr/005-planes-and-atomic-notes.md), the surviving half).
- One source may yield multiple atomic nodes in different planes, connected by typed edges.
- The organizer may say "don't know" → the node lands in `inbox/`, never guessed.

## Principles

| # | Principle | Consequence |
|---|-----------|-------------|
| P1 | **The graph store is the single source of truth** | Typed-node Markdown files on the VPS, git-versioned. Database holds only derived/rebuildable index + operational state. |
| P2 | **Always available, no personal machine required** | Everything runs on an always-on VPS; my laptop being off changes nothing. |
| P3 | **Frictionless capture** | < 10 seconds from thought to captured, from any device or any LLM (MCP). No titles/types/planes asked. |
| P4 | **AI organizes, not me** | Typing, entity resolution, edges, planes, titles, tags and splitting are pipeline work — and the **organizer is the single writer of graph structure**, whatever the surface ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)). |
| P5 | **Everything feeds the brain** | New sources = new connectors implementing one contract (default 6-month lookback, UI-overridable); my own chats feed it too, stance-gated ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)). |
| P6 | **Model independence with a preference** | Claude (Max subscription, Agent SDK) is the primary mind; automatic fallback to Nebius; every model call goes through the provider registry. |
| P7 | **The data is mine and recoverable** | Plain Markdown + git history + managed Postgres. Any component can burn down without memory loss. |
| P8 | **Transparency** | Every background job is visible **while it runs** (live status + logs), manually triggerable, and its schedule inspectable — no cron-only ghosts; activity is recorded for automatic *and* manual actions. |
| P9 | **The brain records my stance, not the model's output** | LLM statements become memory only through my uptake; unclear stance goes to review, never guessed. |
| P10 | **Ingested data survives bug fixes** | Already-ingested data must try to survive fixes and format changes; because raw inputs are always retained (P7), prefer **reprocessing raw as a fresh ingestion** over data loss. A fix that doesn't auto-heal old data must **surface a migrate-vs-delete choice**, never leave it silently broken ([ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md)). |

## Non-goals (v1)

- Multi-user, collaboration, public sharing.
- Native mobile apps (the PWA is the app).
- Connectors beyond Slack in v1 ([ADR-009](adr/009-instagram-connector-deferred.md)) — the
  contract anticipates WhatsApp, Instagram, LLM-chat exports, email, calendar.
- Editing node bodies from the web app (git covers manual editing for now).
- Local/offline LLMs.

## Success criteria

- Capture in < 10s from phone lock screen to confirmation — and equally from any MCP-connected LLM.
- Any memory findable in seconds via chat, with correct source citations.
- Ask about a person and get their whole constellation: shared memories, conversations,
  decisions, ideas — assembled from typed edges, not text search luck.
- The map answers "what does my mind look like around X?" by pointing and clicking.
- Conversational ingestion runs unattended; a morning glance at the activity surfaces tells me
  what was learned, what was skipped, and what awaits my review.
- Zero data loss tolerated: raw captures always preserved; graph store restorable from git;
  index restorable by reindex.
- Insights I wouldn't have produced myself: reflection agents surface patterns across planes
  and timescales.
