# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

**Status (2026-07-13).** Design approved 2026-07-12 (grilled decision-by-decision).
**M0 / M0b ACCEPT COMPLETE** — the code monorepo `../second-brain/` is **deployed live at
`https://braindan.cc`** (PWA over HTTPS, login, `/health` green, Cloudflare TLS Full-strict).
**M1 (capture end-to-end) code-complete** — voice/text capture → organized atomic notes with
full ADR-014 vault durability; its live-Accept backup tail folds into the M2 close. **M2
(indexing & search) IN PROGRESS** — Tasks 1–6 done (nomic-via-Ollama embeddings, chunker,
indexer, `/search` + note preview, the materialized `note_links` relatedness graph, and the
combined nightly `reindex` job + async single-flight `POST /admin/reindex`); next is organizer
tag reuse + `POST /admin/tags/consolidate`.

> The per-milestone status, task checklist (done/open), and the full implementation logs live
> in **[08-implementation-plan.md](08-implementation-plan.md)** + **[08-logs/](08-logs/)** — that
> is the authoritative source for where the build is. This paragraph is a snapshot.

> **Planning/replanning sessions start with `/grilling`; implementation sessions build
> against the approved plan (no grilling). Every session follows
> [09-session-protocol.md](09-session-protocol.md).**

## Reading order

| Doc | Contents |
|---|---|
| [00-vision.md](00-vision.md) | Why the system exists, principles, planes, non-goals, success criteria |
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA client, VPS service, agents, storage |
| [02-data-model.md](02-data-model.md) | Vault layout (planes), note frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the only seam between web and server) |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, indexing, chat, analysis pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract, Slack connector spec, deferred connectors |
| [06-web-app.md](06-web-app.md) | PWA screens, design language (premium, animated), auth UX |
| [07-infrastructure.md](07-infrastructure.md) | VPS, Docker Compose, Caddy, Cloudflare, CI/CD, secrets, backups |
| [08-implementation-plan.md](08-implementation-plan.md) | Phased delivery: per-milestone scope, acceptance, build decisions + a **task tracker** (done/open) |
| [08-logs/](08-logs/) | Per-milestone **implementation logs** (what was built, reviews, verification) — the append-only detail behind the tracker |
| [09-session-protocol.md](09-session-protocol.md) | How every session runs: grill → record → pause → respawn-friendly commits |
| [adr/](adr/) | Architecture Decision Records — the *why* behind every choice |
| [templates/CLAUDE.md](templates/CLAUDE.md) | Ready-to-copy implementation rules for the code monorepo |

## Development workspace layout (local machine)

```
PersonalSecondBrain/          # workspace folder, not a repo
├── second-brain-docs/        # THIS repo — documentation
├── second-brain/             # code monorepo: server/ + web/ + deploy/
└── ObisidanVault/            # local dev vault (scratch, not canonical)
```

Production vault lives on the VPS (see [ADR-001](adr/001-vault-on-vps-with-git-backup.md)).

## Cold start — instructions for a fresh implementation session

If you are an AI (or human) picking this up with no prior context:

0. Follow the [session protocol](09-session-protocol.md): **planning/replanning** sessions
   `/grilling` first, record decisions to docs, and **pause before implementation** so the
   user can continue or respawn; **implementation** sessions build against the approved plan
   (no grilling), pausing between tasks. Commit + push docs at every pause.
1. Read the docs in the order above; skim every ADR — they are binding.
2. The code monorepo `../second-brain/` **may not exist yet**. If missing, create it per
   [01-architecture.md](01-architecture.md) layout (`server/`, `web/`, `deploy/`), git-init
   it, and copy [templates/CLAUDE.md](templates/CLAUDE.md) to its root as `CLAUDE.md`.
3. Implement strictly by phases in [08-implementation-plan.md](08-implementation-plan.md),
   starting at the first milestone whose acceptance criteria don't pass yet. Do not skip ahead.
4. Anything ambiguous or contradictory: fix the docs first (new ADR if architectural),
   then implement. Never silently diverge from these documents.
5. Things intentionally NOT decided yet (ask the user when reached): domain name,
   Cloudflare account setup, Supabase project credentials, Slack app creation,
   GitHub repo names for vault backup and code.

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).

## License

Source-available under the **PolyForm Noncommercial License 1.0.0** ([LICENSE.md](LICENSE.md)):
free for any noncommercial purpose, attribution required (keep the `Required Notice:` line).
**Commercial use requires a separate paid license** — see [COMMERCIAL.md](COMMERCIAL.md).
