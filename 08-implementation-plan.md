# Implementation Plan

**Version:** 3.5 · **Status:** Approved 2026-07-13 (3.5 = [ADR-034](adr/034-external-inspirations-round-2-profile-tiering.md)
round-2 review (3 repos): **evidence-tiered profiles** adopted into the M3 profile job (stub/
snapshot/full by graph degree — caps nightly LLM spend); awesome-second-brain + COG two-way-sync
saved as grilling references; NicholasSpisak variant skipped outright.
3.4 = **ADR-033 external inspirations adopted
in full** (obsidian-second-brain review): identity capsule (M4/M5), contradiction sweep (M6),
freshness stamps + staleness interviews (M3-profiles/M8/M10), reflection enrichments (M10),
graph-health (M8), research-via-MCP pattern (M5), **Telegram capture promoted into M9 as a
must-have ingestion surface** (pull-forward eligible); rejections safeguarded in the ADR.
3.3 = **prior-art research pass adopted in full**
([ADR-032](adr/032-prior-art-adoptions.md)): edge `until`, resolution short-circuit + entropy
guard, observation profiles, day/night effort, RRF + English condensation + recency + expansion
guards, MCP pagination + `build_context`, M6 augment/re-split/salience, M7 rendering guidance,
continents architecture + PPR + serendipity + gap-prompts to backlog, explicit rejections
recorded. 3.2 = **M3 GRILLED TO BUILD-READY**
([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[031](adr/031-m3-organizer-and-contract-extensions.md)):
agenda → build decisions + 10-task list; M4/M6 addenda **ratified** (re-check at kickoff); repo
named `PSB-graph`; zero-manual-VPS cutover. 3.1 = re-review agendas + consistency fixes.
3.0 = **THE MIND-GRAPH PIVOT** — vision/direction
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
  **consolidation job** (approve → retro-walk propose → apply) — M3 scope per
  [ADR-035](adr/035-vocabulary-consolidation-scope-m3.md): edges apply, nodes propose-only.
- Web: capture/search/admin retargeted (type icons, node previews with edges).

**Accept (draft — refined at the M3 grilling):** a voice capture in Romanian becomes typed,
entity-resolved, edge-linked English nodes in the fresh graph store < 30s, visible in the new
repo's history; a person mentioned twice across captures resolves to **one** `person` node whose
`GET /nodes/{id}` lists both memories; an unclassifiable capture lands in `inbox/`; a vocabulary
proposal appears in Settings, approval runs consolidation; DB wipe + reindex restores search +
canonical **and** derived edges identically.

**M3 build decisions (GRILLED 2026-07-13 — [ADR-030](adr/030-entity-substrate-and-lifecycle.md) ·
[ADR-031](adr/031-m3-organizer-and-contract-extensions.md); contract detail in
[02](02-data-model.md) v3.1 / [03](03-api.md) v3.1 / [04](04-pipelines.md) v3.1).** Build-ready;
nothing left to implementer discretion:
- **Pipeline shape:** synchronous-full organize stays (option A); MCP `capture` burst-queued;
  tiered organizing = documented evolution path only.
- **Entity substrate:** `aliases[]`/`disambig` on entity-like types; GIN alias index;
  retrieval-bounded structured candidate injection; `< ENTITY_MATCH_MIN_CONF` (0.8, live-tuned)
  ⇒ edge pending + `entity-ambiguity` review item — never guessed. **+ADR-032:** single exact
  alias hit auto-links with **no LLM round-trip**; intra-capture dedup pass; short/low-entropy
  aliases never fuzzy auto-link.
- **Review queue lands in M3, kind-generic** (`entity-ambiguity`, `vocab-proposal` now;
  `stance-candidate` M6; `dedup-proposal` M6+): one lifecycle, items decidable in place;
  minimal admin Review list now, polished UX M6. Vocab proposals = a queue kind (no table).
- **Entity lifecycle:** thin canonical hubs + **derived profiles** (nightly for touched
  entities, DB-side, embedded, in `GET /nodes/{id}`; **format = categorized observation lines
  carrying `(as of …)` stamps** — ADR-032/[033](adr/033-external-inspirations-obsidian-second-brain.md);
  **evidence-tiered by graph degree** — stub (no LLM) → snapshot (3+) → full profile (8+, config
  `PROFILE_TIER_*`) — [ADR-034](adr/034-external-inspirations-round-2-profile-tiering.md));
  currency via edge `since` **+ optional `until`** (close a superseded relation — invalidate,
  never delete; ADR-032).
- **Merge + backfill:** `POST /admin/entities/merge` propose→apply, immediate apply after a
  forced commit+push; permanent `merged_into` tombstones; nightly backfill scan.
- **Contract:** `occurred`/`occurred_end` partial-ISO (ranges in DB, `occurred ?? created`,
  never fabricated); **9 node types** (+`place`/`event`/`project`/`topic`) & **6 edge rels**
  (+`at`); edges `{rel,to,conf?,since?,until?}` (DB `score` serves both origins);
  `organizer_version` stamped; injection-hygiene rules (a)–(d) are organizer law.
  **Day/night effort defaults** (ADR-032 via ADR-025): sync organize low, nightly jobs high.
- **Bootstrap/cutover — zero manual VPS steps:** repo **`PSB-graph`**; `GRAPH_STORE_REPO`
  config; idempotent app-level bootstrap (init+skeleton+push -u); deploy workflow prints the
  VPS deploy pubkey into the Actions log; user actions = GitHub UI only (create repo, paste
  key, archive `PSB-vault` after Accept).

**Tasks** — detail in [08-logs/m3.md](08-logs/m3.md).
- [x] 1 — migration 005 + config/vocab plumbing (`GRAPH_STORE_PATH/REPO`, `NODE_TYPES`,
      `EDGE_RELS`, `ENTITY_MATCH_MIN_CONF`, burst/profile settings) — done 2026-07-13,
      review clean, verified vs real pgvector; commit `f2b9549` (log: task 1)
- [x] 2 — graph-store service + code rename (store writer: type folders, slug+short-id,
      new frontmatter contract; bootstrap; ADR-014 machinery untouched) — done 2026-07-13
      (**combined 2+3+5**, user's call), review clean, commit `2798412`+`c54fed2` (log: task 2)
- [x] 3 — organizer v3 (typed nodes, occurred, entity resolution vs alias index, edges
      conf/since, hygiene, vocab proposals, organizer_version, English-only carried) — done
      2026-07-13 (combined with 2+5); review clean; **fuzzy + alias-accretion are follow-ups**
- [x] 4 — review queue read/resolve (`GET /review` + `POST /review/{id}`: entity pick/new/maybe
      materializes the pending edge file+DB via writer+reindex; vocab approve = verdict + queued
      `vocab-consolidation` marker, **mutation deferred to task 7** — user call) — done 2026-07-14;
      write path enriched with `pending_edges` (src node ids); 265 tests, real-DB SQL smoke, review
      clean; commit `c53eadc` (log: task 4)
- [x] 5 — indexer/search retarget (id-keyed, whole-file hash, canonical-edge materialization,
      derived `similar` edges, `types` filter, `GET /nodes/{id}` with edges+tombstone) — done
      2026-07-13 (combined with 2+3); `profile` stubbed null (job = task 6)
- [x] 6 — entity services (merge propose/apply + tombstones, backfill scan job, profile-refresh
      job) — done 2026-07-14; `POST /admin/entities/merge` (retarget→alias-union→tombstone→reindex→
      force-commit), nightly `entity-backfill` (qualifying-alias auto-link, watermark) + tiered
      `profile-refresh` (stub/snapshot/full by degree → `node_profiles`, served by `GET /nodes/{id}`);
      migration 006; 294 tests, review clean (3 minors fixed); commits `f5795fe`/`9e544c3`/`f6f0678`
      (log: task 6). **Open before Accept:** real-DB SQL smoke + profile-embedding-in-search +
      alias-accretion (log follow-ups)
- Vocabulary surface (`GET /types`, `PUT /settings/vocabulary`, consolidation job).
  **Scope fixed by [ADR-035](adr/035-vocabulary-consolidation-scope-m3.md):** approval writes
  the addition to `app_settings` (effective vocab = seeds ∪ approved, read via a provider at the
  organizer / `GET /types` / entity-substrate call sites), then the `vocab-consolidation` job
  re-walks — **edges propose→apply** (confirm-gated frontmatter rewrites, ADR-024 envelope),
  **nodes propose-only**. Split into 7a/7b at build:
  - [x] 7a — **governance core** (forward-live): `PgVocabularyStore` over `app_settings`,
        `EffectiveVocabulary` provider threaded into the organizer/validate + resolver + backfill/
        merge/profile + review mint, `GET /types`, `PUT /settings/vocabulary`, review-branch
        delegation to one `VocabularyService`, and the `vocab-consolidation` run an approval opens
        (records the now-live mutation, feed-visible — replaces task 4's SKIPPED marker). Done
        2026-07-14; **318 tests (+24), ruff clean; independent review APPROVE — no must-fix** (2
        minors fixed: mint forward-live test + FOR-UPDATE race note); commits `dd3c5be`/`410b5d2`
        (log: task 7a). **Open before Accept:** real-DB smoke of the `PgVocabularyStore` SQL.
  - [x] 7b — **edge retro-consolidation apply**: `NodeWriter.retype_edge`, the LLM re-walk propose
        + `POST /admin/vocab/consolidate` confirm-apply (edges only); node re-typing stays
        propose-only (its folder-move/re-slug/`node_paths` apply machinery is a deferred follow-up +
        own ADR, ADR-035). **Mechanics pinned by [ADR-036](adr/036-edge-retro-consolidation-walk-retypings-only-m3.md)
        (2026-07-14, task-7b kickoff):** the walk proposes **re-typings of existing edges only**
        (bounded edge inventory, config cap; inventing new edges from node bodies deferred), via an
        **on-demand two-step** endpoint mirroring ADR-024 tags (the 7a approve-time marker run
        unchanged) — resolves the ADR-035 §2 / 04-pipelines wording split. Done 2026-07-14;
        **344 tests (+26), ruff clean; independent review APPROVE after fixes** (1 must-fix —
        prefix-colliding rel match — + 3 minors, all resolved); commit `542459d` (log: task 7b).
        **Open before Accept:** real-DB smoke of the `PgEdgeConsolidationStore` SQL.
- [ ] 8 — web retarget (capture strip node_paths, search type icons, node preview with
      edges/profile, Review list, Settings → Vocabulary)
- [ ] 9 — deploy/CI (`GRAPH_STORE_REPO`, `/srv/graph-store` mount, pubkey-print step, defaults.env).
      **GitHub-side prep already done (2026-07-14): `PSB-graph` created + VPS deploy key pasted with
      write access** — so task 9's first `push -u` should work first try; the pubkey-print step is now
      a convenience, not a blocker.
- [ ] 10 — **live M3 Accept** (per accept draft above + threshold tuning + cutover: verify
      capture→node→PSB-graph push, then user archives PSB-vault)

## M4 — Chat (the grilled old-M3 plan, carried + retargeted to nodes)

All build decisions from the 2026-07-13 chat grilling stand
([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)): model-routing engine +
per-group/per-provider effort, non-streaming + client-side reveal, LLM query-condensation,
hybrid grounding, cited-only `[n]` renumbering, implicit sessions, Settings → Models panel.
**Retarget only:** retrieval returns nodes; source cards show type + plane; "not in your
memories". Retrieval stays **passive top-k** (full agentic traversal = backlog).

**M4 addendum — RATIFIED 2026-07-13 (M3 grilling), refined by [ADR-032](adr/032-prior-art-adoptions.md);
re-check at M4 kickoff before build:**
(a) **graph-aware retrieval lite** — after top-k, inject each hit's 1-hop canonical-edge
neighbors as `{rel, title, type}` lines (pure SQL, no extra LLM call, **config-capped**) +
entity-seeded expansion (**falls back to vector/FTS seeds, never a hard gate**; expansion
function **PPR-swappable**). (b) **hybrid FTS leg** — Postgres `tsvector` ⊍ vector top-k **fused
by RRF (rank-based, k=60)**, degenerate-signal suppression, FTS→0 on non-English raw queries.
(c) **condensation renders the query in English** (one prompt line). (d) **mild recency prior**
on `occurred ?? created`. (e) **temporal filters** (`as_of`/date-window) on `/search`.
(f) **identity capsule** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)) —
the derived ~200-token "who I am / current state" preamble (sleep-cycle-refreshed) injected
into the chat system prompt. (g) **challenge mode** — a chat preset that argues against an idea
using the user's own cited history (ADR-033 #4).
LLM-free at read time throughout — no reranker, no retrieval-loop LLM (ADR-032 rejections).

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

**Scope.** Token-authenticated MCP server on the VPS exposing the service layer: `search`
(+temporal filters), `get_node`, `traverse` (center+depth+rel, **cursor-paginated**),
`build_context` (get_node+traverse in one round-trip — [ADR-032](adr/032-prior-art-adoptions.md)),
`list_planes`/`list_types`, `capture` (full organizer pipeline, `source: mcp`, burst-queued).
`build_context` level-0 serves the **identity capsule** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)).
No logic of its own; smallest milestone, biggest compounding effect (external LLMs start
feeding the brain early). **Canonical usage pattern documented at M5 (ADR-033 #6):
research-via-MCP** — the calling LLM queries the graph for what's known, finds gaps, does
external research on its own plan, submits the distillate via `capture` with source refs
(zero marginal cost; a server-side research connector is backlog only if this proves
insufficient).
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

**M6 addendum — RATIFIED 2026-07-13 (M3 grilling), extended by [ADR-032](adr/032-prior-art-adoptions.md)
(dedup verbs gain **augment** — same event, new fact; nightly **re-split proposals** for bloated
nodes; salience = **graph degree + user pins + edge conf**, no LLM scoring); re-check at M6
kickoff before build:**
(a) **segment sessions before the salience gate** (a real session = 90% retrieval + one buried
decision across planes; per-session granularity skips the gem or distills the noise); bias
sarcasm/hedged/affect-laden statements toward review rather than auto-endorse; idempotent
re-distillation. (b) **review-queue ergonomics** — salience score, batch actions, periodic
"maybe" digest (no-expiry stands, but an untriaged pile stalls the feature). (c) **dedup via the
queue** — near-duplicate candidates (high cosine + shared entities + overlapping `occurred`)
become "possible duplicate — merge / keep / link" review items, using M3's merge primitive.
(d) **inbox drainer** — a nightly job re-attempts organization of `inbox/` nodes with the
now-richer entity registry. (e) **contradiction sweep** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md) #2)
— nightly reconciler: clear supersession → close the old edge with `until` + profile refresh +
feed flag (one-tap revert); ambiguous → review kind **`contradiction`**. Umbrella framing for
the whole 03:00–05:00 roster: **the sleep cycle** (capture fast by day; consolidate, link,
dedup, reconcile, drain, reflect by night).

## M7 — The map (neighborhood explorer)

**Scope.** Desktop-first point-and-click graph exploration over `GET /nodes/{id}/neighbors`
(same service as MCP `traverse`): centered node, one-hop edges, click-to-expand, type
shapes/icons, plane colors, canonical-vs-derived edge styling, breadcrumbs; entry from search/
node lists; phone = tappable-list degradation. **ADR-032 build guidance:** `react-force-graph`
**2D canvas build only**; plex-style animated re-center; **rel-based zones** over pure physics;
per-hop fanout cap with "show more" — never a whole-graph client layout. **Growth path
(post-M7, own planning):** curated "world/continents" overviews = **nightly server-side layout
(UMAP/community detection) served as static tiles, clusters LLM-named once/night** — never live
client GPU layout → aerial whole-graph only if performant and genuinely pleasant.
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
**+ ADR-033 #5: a `graph-health` job + panel card** — orphan nodes, `inbox/` depth,
pending-review aging, memories missing `occurred`, alias-less entities, tombstone integrity,
freshness flags (stale `(as of …)` observations).
**Accept (draft):** every registered job (9+ already) is listed with schedule + next run and can
be run now; a running reindex streams its log live; a review verdict and a manual backup both
appear in the right activity tab.

- [ ] M8 grilled to build-ready detail · tasks defined there

## M9 — Connectors: Slack (stance-gated) + Telegram capture

**Scope.** (a) **Slack** connector ([05-connectors.md](05-connectors.md)): user-token
fetch/normalize, shared stance-gated distillation into typed nodes (`conversation`/`person`
edges), cursors, **6-month default lookback** (UI-overridable), volume guard.
(b) **Telegram capture** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md) #7 —
**must-have ingestion surface**, user 2026-07-13): a private bot polled **from the VPS**;
voice/text messages enter the **same capture pipeline** (STT chain, organizer, never-lose;
`source: telegram`). **Small and independent — depends only on the M3 capture pipeline and may
be pulled forward ahead of the rest of M9 at the user's call.** Open question for its grilling:
**image capture** (vision-routed photos — new scope, decide there; ADR-033 references the
prior-art implementation).
**Accept (draft):** nightly run distills yesterday's Slack into plane-correct, entity-resolved
nodes; unclear-stance items appear in Review; rerun after forced failure resumes from cursor
without duplicates; feed shows the run; **a voice note sent to the Telegram bot from the phone
becomes organized nodes with no PWA open**.

- [ ] M9 grilled to build-ready detail · tasks defined there

## M10 — Reflection agent (+ push notifications)

**Scope.** The "therapist": scheduled (≥ daily) + on-demand reflection over 1d/1w/1m/1y windows
— what went well, what to work on, improvements — producing `insight` nodes through the
organizer; **absorbs the old daily-summary/weekly-review** (retire `summaries`); **PWA push
notifications** (morning digest) land here.
**+ ADR-033 #3/#4 enrichments:** the emerge taxonomy (recurring themes 3+ never named ·
energize-vs-drain patterns · unstated implications · emerging directions; prompt law: *"surface
what they haven't named yet, don't restate what they know"*) · **belief timeline** ("how my view
of X evolved" — `occurred`-ordered memories + `until`-closed edges) · **staleness interviews**
("Is Alex still at Google?" — from stale `(as of …)` profile observations) · catch-up on demand.
**Accept (draft):** the morning after a captured day: a push notification links to a fresh
reflection `insight` retrievable via chat; weekly/monthly views on demand; reruns overwrite.

- [ ] M10 grilled to build-ready detail · tasks defined there

## M11 — Life-manager agent

**Scope (deliberately thin — full grilling session required before build).** Schedule, tasks,
goals across professional/personal planes. Grilling references:
[ADR-034](adr/034-external-inspirations-round-2-profile-tiering.md) (COG's pre-approved
two-way external sync — input to "advisor vs state-manager"). Open questions parked for its
planning session:
`task`/`goal` node types? calendar integration? advisor vs state-manager?

- [ ] M11 planning session (full grill) · everything else defined there

## Backlog (do not build unprompted)

**Chat/retrieval:** seeded **Personalized PageRank** expansion (SPRIG GraphRRF — replaces the
flat 1-hop union behind the same function; needs hub down-weighting) → agentic traversal (chat
model gets `search`/`get_node`/`traverse` tools; only if M5 MCP usage proves one-shot fails —
[ADR-032](adr/032-prior-art-adoptions.md)) · cross-encoder rerank (rejected for the hot path;
revisit only if RRF ordering demonstrably fails) · LLM session titles · session rename/delete ·
true token streaming (streaming provider interface + SSE).
**Graph:** node editing in web · undo a manual ingestion (soft-delete via `git rm`,
`captures.node_paths`) · entity extraction beyond person/idea · hybrid keyword+vector search.
**Sources:** LLM-chat exports connector (promoted by the pivot — stance-gated like the
chat-distiller) · WhatsApp · Instagram spike ([ADR-009](adr/009-instagram-connector-deferred.md)) ·
email · calendar.
**Map:** curated "world/continents" overviews (custom design session; architecture fixed by
[ADR-032](adr/032-prior-art-adoptions.md): nightly server-side UMAP/community layout → static
tiles, LLM-named clusters) → aerial whole-graph mode · **InfraNodus-style structural-gap
prompts** for the reflection agent ("much about X and Y, nothing linking them") ·
**serendipity resurfacing** (M10-adjacent: "On This Day" on the `occurred` range + weighted
random-walk digest with a "why this" path; tune the similarity floor first) · per-type
**field templates** (Tana-style, via ADR-027 governance — pairs with M11's task/goal types).
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
