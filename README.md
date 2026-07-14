# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

**Status (2026-07-13).** **THE MIND-GRAPH PIVOT approved (2026-07-13, grilled decision-by-decision)**
— the vision is reframed as a **typed mind graph**: Obsidian removed entirely, the note vault
becomes a **graph store** of typed nodes (`memory`/`person`/`idea`/`conversation`/`insight`) with
typed edges, governed vocabulary growth, MCP as a peer surface (query + store), conversational
ingestion with a stance gate + review queue, a visual map, and an ops-console/observability
contract — **[ADR-026](adr/026-graph-native-storage-obsidian-removed.md) ·
[027](adr/027-typed-vocabulary-governance.md) · [028](adr/028-one-service-layer-mcp-peer-surface.md) ·
[029](adr/029-conversational-ingestion-stance-gate-review-queue.md)**; new roadmap **M3–M11** in
[08](08-implementation-plan.md). **M0 / M1 / M2 ACCEPT COMPLETE** on the pre-pivot note model —
deployed live at `https://braindan.cc` (capture → organize → index/search, full ADR-014
durability); that system stays live until M3 lands (fresh start: the old vault is archived, no
data migration). The previously grilled chat plan ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))
is carried intact to **M4**, retargeted to nodes.
**ADR-033 (2026-07-13): `obsidian-second-brain` review adopted in full** — identity capsule,
contradiction sweep, freshness stamps + staleness interviews, reflection enrichments (emerge
taxonomy/challenge/belief timeline), graph-health, research-via-MCP pattern, **Telegram capture
promoted into M9** (pull-forward eligible); storage-model ideas explicitly rejected (safeguarded
in the ADR). **ADR-034: round-2 repo review** — **evidence-tiered profiles** (stub/snapshot/full
by graph degree) into the M3 profile job; ecosystem references saved; the rest skipped.
**Prior-art research pass adopted in full (2026-07-13, [ADR-032](adr/032-prior-art-adoptions.md))** —
field survey validated the design (several areas ahead of SOTA); adoptions: edge `until`,
exact-alias short-circuit, observation-style profiles, day/night effort, RRF hybrid retrieval +
guards, MCP pagination/`build_context`, plus M6/M7/backlog refinements and explicit rejections.
**M3 GRILLED TO BUILD-READY (2026-07-13 — [ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[031](adr/031-m3-organizer-and-contract-extensions.md)):**
sync-full organize + MCP burst queue; alias/disambig entity substrate with bounded resolution;
kind-generic review queue in M3; thin hubs + derived profiles; merge/backfill; `occurred`;
9 types / 6 edge rels; edge `{conf,since}` + `organizer_version`; injection hygiene; repo
**`PSB-graph`** with a zero-manual-VPS cutover. M4/M6 addenda ratified (re-check at kickoff).
**M3 in progress: tasks 1–6 done** (2026-07-13/14; tasks 2/3/5 combined by user call — store writer +
rename + bootstrap, organizer v3 + entity resolution, indexer/search retarget). Task 6 (2026-07-14):
**entity services** — `POST /admin/entities/merge` (propose→apply: retarget inbound edges → union
aliases → `merged_into` tombstone → reindex → force-commit), a nightly **backfill** scan (qualifying-
alias auto-link of recent memories, watermark-bounded) and a nightly **profile-refresh** (derived
entity profiles **tiered by graph degree** — stub/snapshot/full, ADR-034 — into a new `node_profiles`
table, served by `GET /nodes/{id}`); migration 006. 294 tests green, independent review clean (3
minors fixed) — [08-logs/m3.md](08-logs/m3.md).
**Task 7a done (2026-07-14):** vocabulary governance core — `PgVocabularyStore` over `app_settings`,
an `EffectiveVocabulary` provider (config seeds ∪ approved additions) threaded into every writer so an
approved type is **forward-live**, `GET /types`, `PUT /settings/vocabulary`, the vocab-proposal branch
delegated to one `VocabularyService` (shared with `POST /review/{id}`), and a feed-visible
`vocab-consolidation` run on approve (replaces task 4's SKIPPED marker). Scope split by
**[ADR-035](adr/035-vocabulary-consolidation-scope-m3.md)** (edges apply / nodes propose-only). 318
tests green (+24), ruff clean, **independent review APPROVE — no must-fix** (2 minors fixed) —
[08-logs/m3.md](08-logs/m3.md) task 7a; commits `dd3c5be`/`410b5d2`.
**Task 7b done (2026-07-14):** edge retro-consolidation apply — `POST /admin/vocab/consolidate`, an
on-demand two-step (ADR-024 shape) that **re-types existing edges** onto a newly-approved rel:
`NodeWriter.retype_edge` (rel-only frontmatter rewrite, `conf`/`since`/`until` preserved, field-boundary
match, dedup-safe), a bounded recency-ordered edge inventory (`PgEdgeConsolidationStore`), and a
`vocab-consolidation` background apply (rewrite → reindex → force-commit, skip-and-continue). Scope
pinned by **[ADR-036](adr/036-edge-retro-consolidation-walk-retypings-only-m3.md)** (re-typings only;
new-edge invention + node re-typing deferred; reconciles the ADR-035 §2 / 04-pipelines wording split).
344 tests (+26), ruff clean, **independent review APPROVE after fixes** (1 must-fix prefix-colliding-rel
+ 3 minors, all resolved + regression-tested); commit `542459d`.
**Task 8 done (2026-07-14):** web retarget — the PWA moved note→node across all five surfaces
against the M3 API (03-api §Search & graph/§Review/§Settings): `SearchResultItem`/`NodeDetailResponse`
+ `getNode`/`types`/`review`/`vocabulary` clients, a `ui/nodeTypes` icon map (governed-type fallback),
search **type icons** + type/plane filters, a node preview with the derived **profile** + canonical/
derived **edges**, capture strip **node_paths** chips, a new **Review** tab (entity-ambiguity picker +
vocab approve/reject), and **Settings → Vocabulary**. No server code touched (ADR-006). tsc/eslint/vite
green; a **real-browser walkthrough vs a throwaway mock API** drove all five surfaces (console clean);
**independent review APPROVE-WITH-MINORS — no must-fix** (3 minors fixed) — [08-logs/m3.md](08-logs/m3.md)
task 8. Out of scope (M8/admin, endpoints exist but no web surface): edge-consolidation + entity-merge UIs.
**Task 9 done (2026-07-14):** deploy/CI retarget vault→graph store — `GRAPH_STORE_PATH=/srv/graph-store`
+ `GRAPH_STORE_REPO` in `defaults.env`/`.env.example`, `/srv/graph-store` mount + `graph_store_deploy_key`
bind in compose, matching `docker-entrypoint.sh`/`Dockerfile` (`safe.directory` — a functional miss)
/`provision.sh` (two-pass hardening guard intact), and a deploy-workflow step printing the VPS deploy
**public** key into the Actions log (best-effort; private key never leaves the box). App bootstrap still
owns remote-wire + `push -u` (ADR-031 §6); no VPS git steps. YAML + `sh -n` clean; **independent review
APPROVE — no must-fix** — [08-logs/m3.md](08-logs/m3.md) task 9; commit `e97671b`.
**Task 10 in progress (2026-07-14): live M3 Accept.** Green baseline (344 tests) + carried backend
smokes closed against **real local pgvector**: migration 006 applied; a real-DB SQL smoke
(`server/scripts/smoke_db.py`) drives the actual task-6/7a/7b `Pg*` stores — **23/23 green**. The
smoke surfaced a real gap → replanned: **`profile-embedding-in-search` was an unbuilt
[ADR-030](adr/030-entity-substrate-and-lifecycle.md) §4 requirement** (search was chunks-only; the
stored profile vector never queried). Mechanism pinned by **[ADR-037](adr/037-profile-embedding-in-search-m3.md)**
— a second per-profile vector leg unioned best-per-node with the chunk leg (all tiers, no weighting,
`SearchResultItem` unchanged, reindex-decoupled); **migration 007** adds the profile HNSW index.
**Built + independently reviewed (APPROVE, no must-fix)** — commits `d0fc52e`/`8ec472a`; 344 tests +
26-check real-DB smoke green, ruff clean. Alias *accretion* stays a documented follow-up (the exact
short-circuit serves "mentioned twice → one node"). **Live-Accept decision: go straight to prod** (no
local dry-run) — the remaining step is the **prod cutover**: a push to `main` auto-deploys (CI runs
`alembic upgrade head` on prod; migration 005 drops the pre-pivot note tables), then a joint live
Accept (capture→node→PSB-graph push < 30s, entity resolution, `inbox/` fallback, vocab consolidation,
DB-wipe/reindex parity, `ENTITY_MATCH_MIN_CONF` tuning) → user archives `PSB-vault`. Code committed
through `8ec472a`, **not pushed**.
**Cutover DONE + Accept STARTED then PAUSED to replan (2026-07-14).** Pushed `8ec472a`; CI green; prod
live on the graph schema (migrations 005/006/007; `/api/v1/health` all-green). Fixed a **deploy-key
gap** on the VPS (graph-store key was missing → per-capture pushes soft-failed; reused the freed
`vault_deploy_key`, force-recreated the api container → `PSB-graph` now populated). The 4 live captures
then surfaced **organizer-quality defects** — 🔴 dangling edges (reorganize deletes shared entity
hubs), person over-extraction, entity split (Horia/Horia Fenwick), diacritic mangling; `inbox/`
clarified as the model-failure-only fallback. User paused to **replan quality** and set a binding
principle: **already-ingested data must survive bug fixes** (reprocess raw, never silently drop) —
lifted to **vision P10**. Grilled decision-by-decision → **[ADR-038](adr/038-reorganize-preserves-shared-entity-hubs.md)**
(hubs shared, never deleted by reorganize) · **[039](adr/039-entity-types-are-mention-only.md)** (entity
types mention-only + coercion guard) · **[040](adr/040-token-overlap-retrieval-and-alias-accretion.md)**
(token-overlap retrieval + alias accretion) · **[041](adr/041-diacritic-folding-derived-content.md)**
(fold diacritics on all derived content, raw kept) · **[042](adr/042-reprocess-all-from-raw-and-data-survival.md)**
(reusable `reprocess-all-from-raw` op + the data-survival principle). New **task 11** in [08](08-implementation-plan.md)
(all must-fix; build → review → local-test the reprocess → deploy + reprocess prod → finish the
remaining Accept criteria → archive `PSB-vault`). Code pushed through `8ec472a`; **no new code this
session** (planning pass — paused before implementing task 11).
**Task 11 DONE + M3 Accept criteria ALL GREEN (2026-07-14).** Built/reviewed/deployed/prod-reprocessed
(through `f8a0e1b`); then the four remaining task-10 Accept criteria verified live (no code): **reindex
parity EXACT** (wipe → `/admin/reindex` rebuilt a byte-identical index from the `PSB-graph` store —
Rule-1/ADR-001 durability), **profile-in-search** (ADR-037 — ran the VPS `profile-refresh` after finding
`node_profiles` empty post-reprocess; person hubs surface via the profile leg, confirmed in the PWA),
**vocab-proposal→consolidation** round-trip (edge-rel propose → approve → forward-live + feed-visible run,
synthetic seed reverted), **`ENTITY_MATCH_MIN_CONF` kept at 0.8** (conservative, zero false merges).
Follow-up logged: reprocess/reindex leave `node_profiles` empty until the nightly job. **M3 is accepted —
the only remaining step is the user archiving `PSB-vault`** ([08-logs/m3.md](08-logs/m3.md) "Task 10/11").

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
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA + MCP surfaces, VPS service, agents, graph store |
| [02-data-model.md](02-data-model.md) | Graph-store layout, typed-node frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the web↔server seam) + the MCP tool surface |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, chat-distillation, indexing, chat, agent pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract (6-month lookback, stance gate), Slack spec, deferred connectors |
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
└── _archive/                 # pre-pivot artifacts (ADR-026): ObisidanVault/ (dead dev
                              # scratch) + PSB-vault/ (local clone of the pre-pivot vault
                              # repo — the GitHub repo stays ACTIVE until the M3 cutover;
                              # see _archive/README.md)
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
2. The code monorepo `../second-brain/` **may not exist yet**. If missing, create it per
   [01-architecture.md](01-architecture.md) layout (`server/`, `web/`, `deploy/`), git-init
   it, and copy [templates/CLAUDE.md](templates/CLAUDE.md) to its root as `CLAUDE.md`.
3. Implement strictly by phases in [08-implementation-plan.md](08-implementation-plan.md),
   starting at the first milestone whose acceptance criteria don't pass yet. Do not skip ahead.
4. Anything ambiguous or contradictory: fix the docs first (new ADR if architectural),
   then implement. Never silently diverge from these documents.
5. Things intentionally NOT decided yet (ask the user when reached): Slack app creation (M9),
   MCP token distribution (M5). Decided at the M3 grilling: graph-store repo = **`PSB-graph`**
   (`PSB-vault` archived after the Accept — zero manual VPS steps, ADR-031). Already provisioned:
   domain (`braindan.cc`), Cloudflare, Supabase, code repo, and **`PSB-graph` created + VPS deploy
   key added with write access (2026-07-14)** — the task-9 cutover's GitHub-side prep is complete.

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).

## License

Source-available under the **PolyForm Noncommercial License 1.0.0** ([LICENSE.md](LICENSE.md)):
free for any noncommercial purpose, attribution required (keep the `Required Notice:` line).
**Commercial use requires a separate paid license** — see [COMMERCIAL.md](COMMERCIAL.md).
