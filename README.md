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

**Where we are (2026-07-19):** **M9.7 code COMPLETE — Batch A + B built, reviewed, committed
locally; only T7 (deploy + live Accept) remains, and it's the user's.** The two M9.6-drill FAILs
+ the general-remove scope-add ([ADR-062](adr/062-chat-screenshot-self-attribution.md), grilled
to build-ready) were implemented this session against the approved [08 §M9.7](08-implementation-plan.md):
- **Batch A {T1,T2,T3}** (`e91d1cb`/`1a8a809`/`ab173db`): **T1** vision emits disciplined
  per-message chat-screenshot lines (`[side · sender]` + `quoting` insets, identity-agnostic —
  ADR-062 §2); **T2** the capture pipeline streams milestone `logger.info` lines (per-part derive +
  stage transitions) live through the M8 run-log tail, **zero new schema**; **T3** the openRun
  RunDetail renders the live `RunLogTail` + a structured `derive.parts[]` block (ADR-061 §10, *by
  reuse* — the §C.2 fix).
- **Batch B {T4,T5,T6}** (`8ec8de6`/`dcefee7`/`f83268e`): **T4** organizer **v9** maps `[right]`→the
  user's own words (no self-entity, phantom-sender ban), `[left · Name]`→named person, `quoting`→the
  quoted party, own-chat default + "not mine" override (the §E.2 fix, ADR-062 §3); **T5** general
  capture remove `DELETE /captures/{id}` (shared hub-preserving core, media purge, tombstone-last —
  ADR-062 §R); **T6** the double-confirm Remove affordance on the capture card.
- **Every task independently reviewed — no must-fix** anywhere. Integration gate green: server
  ruff+format + **1037 pytest**; web `tsc` + whole-project `eslint` + `vite build`. **No migrations**
  (no schema changes). Commits are **local on `main`, not pushed** — a code push auto-deploys to prod
  (= T7's deploy), which is the user's call.

**Next:** **T7 — deploy + live Accept (the user, agent-guided):** push/deploy; **A/B** the real
screenshot on `llama-4-scout-17b` vs `qwen2.5-vl-72b` (Settings flip; pick primary on evidence —
ADR-062 §4); **migration** rederive+reorganize existing photo captures (§5) and confirm the drill's
misattributed capture reads correctly; **re-drill** m9.6-accept-drill §C.2 (live per-part lines now
stream) + §E.2 (attribution under v9); **remove-drill** a junk capture (double-confirm → gone from
Recents/search/chat, nodes + media gone, hubs preserved, `reprocess-all` doesn't resurrect it);
then flip **M9.6 T6 + M9.7** done together and update this snapshot.

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
