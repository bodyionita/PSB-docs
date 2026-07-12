# Personal Second Brain — Vision

**Version:** 2.0 (supersedes v1.0 — architecture pivoted to always-available cloud service)
**Status:** Approved 2026-07-12

## Vision

A conversational second brain, available anywhere, any time. It captures everything —
spoken thoughts, written notes, and (via automated agents) my digital conversations —
organizes it across the planes of my life, and lets me question, explore and understand
my own history. The goal is not storage; it is an extension of memory and a reflection
partner: information → knowledge → understanding, compounding over time.

## What it is, concretely

- A **secure personal web app** (PWA on phone + desktop): talk or type to capture, chat
  over the whole memory, watch what the system did overnight, choose the AI model that
  answers.
- A **Markdown knowledge vault** as the canonical memory, hosted on my own server, fully
  versioned and recoverable, explorable in Obsidian whenever I want to wander manually.
- A set of **ingestion agents** running on a schedule that pull in what I said and
  discussed elsewhere (Slack first; more sources over time), distill it, and file it into
  the right plane of my life.
- **Background intelligence**: daily and weekly analysis that surfaces themes, decisions,
  patterns and open questions — insight, not just summaries.

## Planes of life

The vault is organized into **planes** — configurable top-level life areas:

`Professional` · `Personal` · `Family` · `Friends` · `Health` · `Ideas` (initial set; config-driven)

- Every note has exactly one **primary plane** (its folder) and may declare additional
  planes in frontmatter — membership truth is frontmatter, folders are navigation.
- One source (e.g. a Slack conversation) may yield **multiple atomic notes** in different
  planes, cross-linked and sharing a source reference ([ADR-005](adr/005-planes-and-atomic-notes.md)).
- The classifier may say "don't know" → the note lands in a global `Inbox/`, never guessed.

## Principles

| # | Principle | Consequence |
|---|-----------|-------------|
| P1 | **The vault is the single source of truth** | Markdown files on the VPS, git-versioned. Database holds only derived/rebuildable index + operational state. |
| P2 | **Always available, no personal machine required** | Everything runs on an always-on VPS; my laptop being off changes nothing. |
| P3 | **Frictionless capture** | < 10 seconds from thought to captured, from any device. No titles/tags/folders asked. |
| P4 | **AI organizes, not me** | Planes, titles, tags, splitting and linking are pipeline work. |
| P5 | **Agents feed the brain** | New sources = new connectors implementing one contract; the rest of the system doesn't change. |
| P6 | **Model independence with a preference** | Claude (Max subscription, Agent SDK) is the primary mind; automatic fallback to Nebius; every model call goes through the provider registry. |
| P7 | **The data is mine and recoverable** | Plain Markdown + git history + managed Postgres. Any component can burn down without memory loss. |
| P8 | **Transparency** | Everything the system does in the background is visible in the activity feed. |

## Non-goals (v1)

- Multi-user, collaboration, public sharing.
- Native mobile apps (the PWA is the app).
- Instagram and other connectors beyond Slack ([ADR-009](adr/009-instagram-connector-deferred.md)) — the contract anticipates them.
- Editing notes from the web app (Obsidian/git covers manual editing for now).
- Local/offline LLMs.

## Success criteria

- Capture in < 10s from phone lock screen to confirmation.
- Any memory findable in seconds via chat, with correct source citations.
- Slack ingestion runs nightly unattended; a morning glance at the activity feed tells me
  what was learned.
- Zero data loss tolerated: raw captures always preserved; vault restorable from git;
  index restorable by reindex.
- Insights I wouldn't have produced myself: weekly reviews surface patterns across planes.
