# Implementation Plan

**Version:** 3.1 · **Status:** Approved 2026-07-13 (3.1 = deep re-review pass: **grilling agendas**
added to M3/M4/M6 — proposals from the 2026-07-13 re-review (entity substrate, thin-hub+derived-
profile, `occurred`, vocabulary seeds, edge metadata, tiered organizing, graph-aware retrieval,
FTS leg, session segmentation, dedup, drainers) — **recorded as agenda, not yet decided**; plus
consistency fixes. 3.0 = **THE MIND-GRAPH PIVOT** — vision/direction
change grilled decision-by-decision ([ADR-026](adr/026-graph-native-storage-obsidian-removed.md) ·
[027](adr/027-typed-vocabulary-governance.md) · [028](adr/028-one-service-layer-mcp-peer-surface.md) ·
[029](adr/029-conversational-ingestion-stance-gate-review-queue.md)): Obsidian removed, typed-node
**graph store**, MCP peer surface, conversational ingestion + review queue, the map, ops console,
reflection/life agents. **The old M3 (chat) moves to M4, carried intact and retargeted to nodes;
old M4 (Slack) → M9; old M5 → absorbed into M8/M10.** 2.x history in git.)
**Rule:** ship in phases; every phase ends usable. A phase starts only when the previous one's
acceptance criteria pass. Code lives in `second-brain/` (monorepo, ADR-006).
**Process:** every session runs under [09-session-protocol.md](09-session-protocol.md). **Each
milestone below M3+ carries approved scope; its build-ready detail is grilled in a planning
session when it comes up** (as M1/M2/old-M3 were).

**How this file is organised.** Each milestone carries **scope · acceptance** (+ build decisions
once grilled) and a **task checklist**; verbose per-task narratives live in **[08-logs/](08-logs/)**
([m0](08-logs/m0.md) · [m1](08-logs/m1.md) · [m2](08-logs/m2.md)).

> **Pivot note (2026-07-13).** M0–M2 shipped and were accepted on the **note model** (Markdown
> note vault, wikilinks, `sb:related`). That system is live at `braindan.cc` and remains so until
> M3 lands. The sections below are the shipped record; the pivot does **not** reopen them. M3
> replaces the note model with the typed graph (fresh start — the old vault is archived,
> [ADR-026](adr/026-graph-native-storage-obsidian-removed.md)); the durability machinery
> ([ADR-014](adr/014-vault-history-durability.md)), provider registry, STT chain, embeddings
> sidecar, auth, deploy — all carry over unchanged.

## M0 — Foundations ✅ (accepted 2026-07-12)

Monorepo skeleton, full schema migration 001, provider registry (`openai`+`nebius`+`claude-max`
+ fallback chain), auth, `/health`; web scaffold + theming + shell; deploy (Docker/Caddy/
provision/Actions). **Accept passed** (HTTPS PWA at `braindan.cc`, login, `/health` green, TLS
Full-strict); the live chain-and-record clause was deferred to chat (now **M4**) and covered by
21 registry unit tests meanwhile. Key ADRs: 011/012/017/018.
Tasks (all done, detail in [08-logs/m0.md](08-logs/m0.md)): local-first build · ADR-017/018 code
`ae08d43` · M0b provisioning · Accept.

## M1 — Capture end-to-end ✅ (accepted 2026-07-13, closed at the M2 Accept)

Capture endpoints + pipeline (STT chain [ADR-020] → organizer → vault write → index stub),
ADR-014 durability (VaultBackupService, four R2 jobs, integrity drill, `/health backups` leg),
conversational-capture nudge [ADR-019], `agent_runs` interaction logging [ADR-021], web capture
screen. **Accept passed** incl. the postponed backup tail (WORM bundle + drill green).
Tasks (all done, detail in [08-logs/m1.md](08-logs/m1.md)): 1 migration+pipeline · 2 routers ·
3 durability `884855f`/`7f3c4a7`/`1e3c420` · 4 web · 5 replan `d9b21e8`+ · 6 polish `d469277` ·
7 live Accept.

## M2 — Indexing & search ✅ (accepted 2026-07-13)

Self-hosted nomic embeddings via `ollama` [ADR-022], chunker, indexer, `/search` + note preview,
`note_links` relatedness graph [ADR-023], combined nightly reindex + async `/admin/reindex`, tag
reuse + consolidation [ADR-024], web Search + Admin tabs. **Accept passed live on prod** (4 prod
issues surfaced + fixed). *Pivot note: the rendered `## Related notes` block and its hash/strip
machinery — built here — are deleted by [ADR-026](adr/026-graph-native-storage-obsidian-removed.md);
the DB-side similarity graph concept survives as derived `similar` edges.*
Tasks (all done, detail in [08-logs/m2.md](08-logs/m2.md)): 1 migration 004+provider `c66b562` ·
2 chunker `fdd0f60` · 3 indexer `684604e` · 4 search+preview `6e0fa21` · 5 graph `73ed641` ·
6 nightly job `a059c18`+`8cde827` · 7 tags `b404709` · 8 web tabs · 9 live Accept.

---

# The graph roadmap (the pivot and after)

## M3 — Graph core (THE PIVOT)

**Scope.** The typed graph replaces the note model end-to-end
([ADR-026](adr/026-graph-native-storage-obsidian-removed.md)/[027](adr/027-typed-vocabulary-governance.md)):
- **Graph store**: new format (folder=type, slug+short-id filenames, `type`/`edges` frontmatter,
  `inbox/` fallback), fresh bootstrap, **fresh GitHub repo** (old `PSB-vault` archived);
  `GRAPH_STORE_PATH`/`/srv/graph-store` rename.
- **Organizer v3**: typed nodes, **entity resolution** (person/idea mentions → existing nodes or
  new ones), typed edges, vocabulary proposals (no fit → `memory` + proposal filed).
- **Migration 005**: `nodes`/`edges` tables replace `notes`/`note_links`;
  `captures.note_paths → node_paths`; no data migrated.
- **Indexer/search retargeted**: id-keyed, whole-file hash, canonical-edge materialization,
  derived `similar` edges (DB-only — no file rendering); `types` filter on `/search`.
- **API rename + additions**: `GET /nodes/{id}` (with edges), `GET /types`,
  `PUT /settings/vocabulary`; full nodes/graph vocabulary in code and OpenAPI.
- **Vocabulary governance**: Settings panel (types + proposals + approve), the
  **consolidation job** (approve → retro-walk propose → apply).
- Web: capture/search/admin retargeted (type icons, node previews with edges).

**Accept (draft — refined at the M3 grilling):** a voice capture in Romanian becomes typed,
entity-resolved, edge-linked English nodes in the fresh graph store < 30s, visible in the new
repo's history; a person mentioned twice across captures resolves to **one** `person` node whose
`GET /nodes/{id}` lists both memories; an unclassifiable capture lands in `inbox/`; a vocabulary
proposal appears in Settings, approval runs consolidation; DB wipe + reindex restores search +
canonical **and** derived edges identically.

**Before implementation: an M3 build-ready grilling session** (entity-resolution mechanics,
migration 005 DDL, proposal storage, bootstrap details, archive procedure).

**M3 grilling agenda (from the 2026-07-13 deep re-review — PROPOSALS, each needs a user
decision; they are contract-level and cheap now, LLM re-walks over the whole graph later):**
1. **Entity resolution substrate** — `aliases[]` + one-line `disambig` frontmatter on
   person/idea nodes; a DB alias index for *retrieval-bounded* candidate generation (organizer
   sees only matching candidates as structured data, never the registry, never node bodies);
   ambiguous/low-confidence matches → the review queue, never guessed.
2. **Entity merge + backfill primitive** — duplicates are inevitable: define
   `merge(loser→survivor)` (DB reverse-index → batch-rewrite inbound edges in the agent window
   under an ADR-014 checkpoint → tombstone `merged_into:` → reindex) + retroactive edge
   backfill when a new entity matches older unlinked/inbox nodes.
3. **Entity lifecycle: thin hub + derived profile** — hub files stay thin/canonical (identity,
   aliases, edges); the readable "who is Alex now" **profile is derived** (regenerated from the
   1-hop neighborhood, DB-side, embeddable, shown by `get_node`/map) — never LLM
   read-modify-write on capture. Fact currency via edge metadata (`since`) or a `supersedes`
   edge, decided here.
4. **Event time** — organizer-extracted `occurred` (+ granularity: day/month/year/range)
   alongside `created`; reflection windows and map timelines run on `occurred ?? created`.
5. **Vocabulary seeds** — add `place` (+ `happened_at`-style edge) and a container type
   (`event`/`project`) at M3; decide `topic` vs `idea` and whether `about` needs splitting
   before it becomes the misc-bucket of edges.
6. **Edge metadata + organizer versioning** — edges become `{rel, to, conf?, since?}`;
   nodes/captures stamped with `organizer_version` so later quality-retrofits are targetable.
7. **Organizer economics (pipeline shape)** — tiered organizing: cheap synchronous pass at
   capture (never-lose write, type guess, alias-index candidate match, inbox fallback, nudge)
   + a **nightly batch organize** in the agent window (full ER/edges/splitting with the whole
   day in view — better ER + enables dedup, and caps Claude-Max window spend). Decide sync-all
   vs tiered.
8. **Injection hygiene** — all organizer prompt injections retrieval-bounded (never O(corpus));
   ingested text (Slack/MCP) treated as untrusted data behind hard delimiters; node bodies
   never fed to the resolver.

- [ ] M3 grilled to build-ready detail (planning session over the agenda above)
- [ ] Tasks defined at that grilling; 08-logs/m3.md created at implementation

## M4 — Chat (the grilled old-M3 plan, carried + retargeted to nodes)

All build decisions from the 2026-07-13 chat grilling stand
([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)): model-routing engine +
per-group/per-provider effort, non-streaming + client-side reveal, LLM query-condensation,
hybrid grounding, cited-only `[n]` renumbering, implicit sessions, Settings → Models panel.
**Retarget only:** retrieval returns nodes; source cards show type + plane; "not in your
memories". Retrieval stays **passive top-k** (full agentic traversal = backlog).

**M4 grilling addendum (2026-07-13 deep re-review — PROPOSALS, user decision pending):**
(a) **graph-aware retrieval lite** — after top-k, inject each hit's 1-hop canonical-edge
neighbors as `{rel, title, type}` lines (pure SQL, no extra LLM call) + entity-seeded expansion
(query resolves to a person/idea → union in its `involves`/`about` neighborhood); without this
the typed graph is unused on the primary read surface. (b) **hybrid FTS leg** — a Postgres
`tsvector` union with vector top-k (nomic is English-only and weak on proper nouns — this is a
graph full of names).

**Accept:** questions over known graph content answered with correct `[n]` node citations on
both Claude and Nebius; "not in your memories" verified; sessions persist; **plus the deferred
M0 clause** (Settings-driven Nebius-primary drive recorded in `chat_messages.model` + the 21
registry fallback unit tests).

- [ ] 1 model routing engine (`ModelRoutingService` + per-call effort)
- [ ] 2 chat service (condensation → node-grouped top-k → grounded prompt → cited-only parse → persistence)
- [ ] 3 chat routers (`POST /chat`, `GET /chat/models`, `GET /chat/sessions[/{id}]`)
- [ ] 4 settings routers (enriched `GET /settings` + `PUT /settings/models`)
- [ ] 5 web chat screen (list/thread, picker + plane chips, reveal, source cards, banner)
- [ ] 6 web Settings → Models panel
- [ ] 7 live M4 Accept (incl. deferred M0 clause)

## M5 — MCP server ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md))

**Scope.** Token-authenticated MCP server on the VPS exposing the service layer: `search`,
`get_node`, `traverse`, `list_planes`/`list_types`, `capture` (full organizer pipeline,
`source: mcp`). No logic of its own; smallest milestone, biggest compounding effect (external
LLMs start feeding the brain early).
**Accept (draft):** from a Claude conversation on any device: a `capture` lands as organized
nodes; `search`+`get_node`+`traverse` answer a question about known graph content; token
revocation locks it out; MCP-driven runs visible in activity.

- [ ] M5 grilled to build-ready detail · tasks defined there

## M6 — Chat-distiller + review queue ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md))

**Scope.** The nightly stance-gated chat-distiller (connector pattern, salience gate,
endorsed/rejected/unclear), `review_queue` table + endpoints, the web **Review** surface
(agree/disagree/maybe), feed flags + one-tap remove for auto-ingested items,
`POST /chat/sessions/{id}/remember`.
**Accept (draft):** a substantive chat session yields an `insight`/`conversation` node overnight
(feed-flagged, removable); a pure-retrieval session is skipped and logged; a stance-unclear
candidate waits in Review and ingests only on agree.

**M6 grilling addendum (2026-07-13 deep re-review — PROPOSALS, user decision pending):**
(a) **segment sessions before the salience gate** (a real session = 90% retrieval + one buried
decision across planes; per-session granularity skips the gem or distills the noise); bias
sarcasm/hedged/affect-laden statements toward review rather than auto-endorse; idempotent
re-distillation. (b) **review-queue ergonomics** — salience score, batch actions, periodic
"maybe" digest (no-expiry stands, but an untriaged pile stalls the feature). (c) **dedup via the
queue** — near-duplicate candidates (high cosine + shared entities + overlapping `occurred`)
become "possible duplicate — merge / keep / link" review items, using M3's merge primitive.
(d) **inbox drainer** — a nightly job re-attempts organization of `inbox/` nodes with the
now-richer entity registry. Umbrella framing for the whole 03:00–05:00 roster: **the sleep
cycle** (capture fast by day; consolidate, link, dedup, drain, reflect by night).

## M7 — The map (neighborhood explorer)

**Scope.** Desktop-first point-and-click graph exploration over `GET /nodes/{id}/neighbors`
(same service as MCP `traverse`): centered node, one-hop edges, click-to-expand, type
shapes/icons, plane colors, canonical-vs-derived edge styling, breadcrumbs; entry from search/
node lists; phone = tappable-list degradation. **Growth path (post-M7, own planning):** curated
"world/continents" overviews (custom-designed together) → aerial whole-graph only if performant
and genuinely pleasant.
**Accept (draft):** starting from a search hit, reach a `person` node in one click and see their
constellation; expand three hops without jank; edge styling distinguishes typed relations from
similarity.

- [ ] M7 grilled to build-ready detail · tasks defined there

## M8 — Ops console & activity restructure

**Scope.** The jobs-observability contract made UI (01 invariant 4): every job manually
triggerable by category, **live status + log tail while running**
(`GET /activity/runs/{id}/logs`), schedule registry (cadence + next run, `GET /agents`);
activity becomes **categorized tabs** (agents/jobs · conversations · manual actions) recording
automatic and manual events; merged `GET /activity`.
**Accept (draft):** every registered job (9+ already) is listed with schedule + next run and can
be run now; a running reindex streams its log live; a review verdict and a manual backup both
appear in the right activity tab.

- [ ] M8 grilled to build-ready detail · tasks defined there

## M9 — Slack connector (the old M4, stance-gated)

**Scope.** Connector contract implementation for Slack ([05-connectors.md](05-connectors.md)):
user-token fetch/normalize, shared stance-gated distillation into typed nodes
(`conversation`/`person` edges), cursors, **6-month default lookback** (UI-overridable), volume
guard.
**Accept (draft):** nightly run distills yesterday's Slack into plane-correct, entity-resolved
nodes; unclear-stance items appear in Review; rerun after forced failure resumes from cursor
without duplicates; feed shows the run.

- [ ] M9 grilled to build-ready detail · tasks defined there

## M10 — Reflection agent (+ push notifications)

**Scope.** The "therapist": scheduled (≥ daily) + on-demand reflection over 1d/1w/1m/1y windows
— what went well, what to work on, improvements — producing `insight` nodes through the
organizer; **absorbs the old daily-summary/weekly-review** (retire `summaries`); **PWA push
notifications** (morning digest) land here.
**Accept (draft):** the morning after a captured day: a push notification links to a fresh
reflection `insight` retrievable via chat; weekly/monthly views on demand; reruns overwrite.

- [ ] M10 grilled to build-ready detail · tasks defined there

## M11 — Life-manager agent

**Scope (deliberately thin — full grilling session required before build).** Schedule, tasks,
goals across professional/personal planes. Open questions parked for its planning session:
`task`/`goal` node types? calendar integration? advisor vs state-manager?

- [ ] M11 planning session (full grill) · everything else defined there

## Backlog (do not build unprompted)

**Chat/retrieval:** graph-aware chat context (one-hop canonical-edge expansion) → agentic
traversal (chat model gets `search`/`get_node`/`traverse` tools) · LLM session titles · session
rename/delete · true token streaming (streaming provider interface + SSE).
**Graph:** node editing in web · undo a manual ingestion (soft-delete via `git rm`,
`captures.node_paths`) · entity extraction beyond person/idea · hybrid keyword+vector search.
**Sources:** LLM-chat exports connector (promoted by the pivot — stance-gated like the
chat-distiller) · WhatsApp · Instagram spike ([ADR-009](adr/009-instagram-connector-deferred.md)) ·
email · calendar.
**Map:** curated "world/continents" overviews (custom design session) → aerial whole-graph mode.
**Platform:** PWA offline text-capture queue + offline shell polish · voice offline queue ·
Cloudflare Access second wall · demo/presentation layer (curated/redacted show-off view) ·
multi-tenant (far horizon; keep jobs CLI-invokable) · backup fast-follows (monthly CI restore
drill, semi-annual DR rehearsal — [ADR-014](adr/014-vault-history-durability.md)).

## Testing policy

Pure logic (chunking, frontmatter/edge parsing, slugs, entity-resolution scoring, cursor math,
citation renumbering, stance classification post-processing) → unit tests, no mocks. Services →
fake providers + tmp graph store + test DB schema. Connectors/distillers → recorded fixture
payloads. No live LLM calls in CI; each milestone has a manual smoke script documented in the
code repo.
