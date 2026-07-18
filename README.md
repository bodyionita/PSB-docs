# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

## Current status (snapshot — 2026-07-19)

> This section is a **snapshot, updated in place** each session. The full session-by-session
> history lives in **[08-logs/status-history.md](08-logs/status-history.md)** (append-only —
> a superseded snapshot entry moves there verbatim). The authoritative build state is
> **[08-implementation-plan.md](08-implementation-plan.md)** + **[08-logs/](08-logs/)**.

**Live at `https://braindan.cc` — M0 through M8.2 all closed.** The graph-native stack
([ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–029 pivot, M3): capture
(voice/text/MCP) → organizer (single writer; typed nodes + edges, entity resolution, alias
accretion, diacritic folding) → `PSB-graph` git store → index/hybrid search (vector ⊍ FTS RRF +
recency + profile leg). Grounded cited **chat** with UI-editable model routing
(`chat`/`conspect`/`quick`; provider ≠ model, [ADR-045](adr/045-provider-model-effort-separation.md))
+ provider observability (M4). **MCP server behind self-hosted OAuth 2.1** — connector-verified
on Claude (mobile/web) *and* ChatGPT (M5); identity capsule L0. **Pipeline scheduling primitive**
(M5.5, [ADR-047](adr/047-pipeline-scheduling-primitive.md)). **Chat distiller** with stance gate,
watermarks, salience, dedup sweep + merge core, inbox drainer, one-tap remove (M6,
[ADR-048](adr/048-m6-chat-distiller-build-decisions.md)/[049](adr/049-dedup-sweep-merge-core-build-decisions.md)).
**Map/Explore** constellation UI (M7/M8.1), **ops console + observability** (M8,
[ADR-053](adr/053-m8-ops-console-observability-build-decisions.md)), **graph-health**.
**Interiority + temporal correctness** (M8.2, [ADR-055](adr/055-interiority-inner-voice-first-class.md)/[056](adr/056-temporal-correctness-date-tokens.md)):
"LLMs classify, code computes" (hard rule 12), anchored symbolic time-refs → `[[t:…]]` tokens,
inner-voice extraction; prod reprocessed (41/41 captures, 160 nodes). Durability: everything
derived rebuilds from the store (`reprocess-all-from-raw`, vision P10,
[ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md)); reindex parity verified live.

**Where we are (2026-07-19):** **M9.6 composite capture — IMPLEMENTATION IN PROGRESS (T1 done).**
A single-session build against the approved plan ([08 §M9.6](08-implementation-plan.md) +
[ADR-061](adr/061-composite-multi-part-capture.md)); the user directed all M9.6 tasks in one pass
(no per-task pause), agreeing to pushes + postponing manual/live drills to T6. **T1 (draft lifecycle
+ schema) is committed** (`e785554`, code repo — **not yet deployed**; M9.6 deploys once at T6):
migration **019** (`captures.text_body`, `media.part_ordinal`, `captures_single_active_draft` partial
unique index), the server draft surface (open/resume · part add/remove with ≤1-voice + ordinals ·
text edit · submit ≥1-part gate · discard · 7-day GC; orphan-sweep skips drafts), draft endpoints +
`DraftView`, and a **baseline sequential composite `_process`** so submit works end-to-end. Full
suite **1026 pass**, ruff clean. See the **T1 progress note** in [08 §M9.6](08-implementation-plan.md).
**Next:** continue the sequential M9.6 build — **T2** (concurrent-bounded derivation + cached indexed
part markers + per-part `agent_runs`) → **T3** (per-node attribution + organizer `parts:[…]`) →
**T4** (CaptureView media→list + fold the one-shot endpoints + Activity deep-link) → **T5** (compose
web) → **T6** (deploy + live Accept, folding the M9 T6 drills). Independent review runs as a
server-diff `/code-review` pass after T4.

> **Planning/replanning sessions start with `/grilling`; implementation sessions build
> against the approved plan (no grilling). Every session follows
> [09-session-protocol.md](09-session-protocol.md).**

## Reading order

| Doc | Contents |
|---|---|
| [00-vision.md](00-vision.md) | Why the system exists, principles, planes, non-goals, success criteria |
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA + MCP surfaces, VPS service, agents, graph store |
| [02-data-model.md](02-data-model.md) | Graph-store layout, typed-node frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the web↔server seam) + the MCP tool surface |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, chat-distillation, indexing, chat, agent pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract (conversation substrate, stance gate), Instagram + Slack specs |
| [06-web-app.md](06-web-app.md) | PWA screens, design language (premium, animated), auth UX |
| [07-infrastructure.md](07-infrastructure.md) | VPS, Docker Compose, Caddy, Cloudflare, CI/CD, secrets, backups |
| [08-implementation-plan.md](08-implementation-plan.md) | Phased delivery: per-milestone scope, acceptance, build decisions + a **task tracker** (done/open) |
| [08-logs/](08-logs/) | Per-milestone **implementation logs** + the archived **[status history](08-logs/status-history.md)** |
| [09-session-protocol.md](09-session-protocol.md) | How every session runs: grill → record → pause → respawn-friendly commits |
| [adr/](adr/) | Architecture Decision Records — the *why* behind every choice |

## Development workspace layout (local machine)

```
PersonalSecondBrain/            # workspace folder, not a repo
├── second-brain-docs/          # THIS repo — documentation (public GitHub — no personal data!)
├── second-brain/               # code monorepo: server/ + web/ + deploy/ + tools/
├── PSB-graph/                  # local clone of the (private) production graph-store repo
├── instagram-…/                # the user's unzipped Instagram data export (local only,
│                               # NEVER committed anywhere — M9.5 prep-tool input)
└── _archive/                   # pre-pivot artifacts (ADR-026): ObisidanVault/ (dead dev
                                # scratch) + PSB-vault/ (pre-pivot vault repo — ARCHIVED
                                # 2026-07-14 at the M3 close; see _archive/README.md)
```

The production graph store lives on the VPS ([ADR-001](adr/001-vault-on-vps-with-git-backup.md) +
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)).

## Cold start — instructions for a fresh implementation session

If you are an AI (or human) picking this up with no prior context:

0. Follow the [session protocol](09-session-protocol.md): **planning/replanning** sessions
   `/grilling` first, record decisions to docs, and **pause before implementation** so the
   user can continue or respawn; **implementation** sessions build against the approved plan
   (no grilling), pausing between tasks. Commit + push docs at every pause.
1. Read the docs in the order above; skim every ADR — they are binding.
2. The code monorepo lives at `../second-brain/` (server/, web/, deploy/, tools/;
   its `CLAUDE.md` carries the binding implementation rules).
3. Implement strictly by phases in [08-implementation-plan.md](08-implementation-plan.md),
   starting at the first milestone whose acceptance criteria don't pass yet. Do not skip ahead.
4. Anything ambiguous or contradictory: fix the docs first (new ADR if architectural),
   then implement. Never silently diverge from these documents.
5. Things intentionally NOT decided yet (ask the user when reached): Slack app creation (M12);
   exact vision model ids (M9 T1 — verified against live Groq/Nebius catalogs at build).
   Already provisioned: domain (`braindan.cc`), Cloudflare, Supabase, code repo, `PSB-graph`
   (+ VPS deploy key, write access).
6. Update the **Current status** snapshot above **in place** at every pause; move the
   superseded entry verbatim to [08-logs/status-history.md](08-logs/status-history.md)
   (append-only). Never let state live only in chat.

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).
- **This repo is public: no real names, no relationship details, no personal data** — use
  placeholders (P1/P2) in logs; the triage manifest and export data stay local, gitignored.

## License

Source-available under the **PolyForm Noncommercial License 1.0.0** ([LICENSE.md](LICENSE.md)):
free for any noncommercial purpose, attribution required (keep the `Required Notice:` line).
**Commercial use requires a separate paid license** — see [COMMERCIAL.md](COMMERCIAL.md).
