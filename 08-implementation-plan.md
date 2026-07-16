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
- [x] 8 — web retarget (capture strip node_paths, search type icons, node preview with
      edges/profile, Review list, Settings → Vocabulary) — done 2026-07-14; all five surfaces
      retargeted note→node against the M3 API; new **Review** tab + **Settings → Vocabulary** panel;
      `ui/nodeTypes` icon map (governed-type fallback); Admin/Chat/Activity copy de-noted. No server
      code touched (ADR-006). tsc/eslint/vite green; **real-browser walkthrough vs a mock API** drove
      all five surfaces (search cards, node preview profile+edges, Review resolve, vocab approve,
      capture node_paths) — console clean. **Independent review APPROVE-WITH-MINORS — no must-fix**
      (3 minors fixed: "Writing notes"→"Writing nodes", preview renders disambig/aliases, `backups`
      on `HealthResponse`) — [08-logs/m3.md](08-logs/m3.md) task 8. **Deferred to task-10 Accept:**
      the live capture→node→search flow vs the real stack. **Out of scope (M8/admin):** edge-
      consolidation + entity-merge UIs (endpoints exist, no web surface yet).
- [x] 9 — deploy/CI (`GRAPH_STORE_REPO`, `/srv/graph-store` mount, pubkey-print step, defaults.env) —
      done 2026-07-14; deploy layer renamed vault→graph-store across `defaults.env`/`.env.example`
      (`GRAPH_STORE_PATH=/srv/graph-store` + `GRAPH_STORE_REPO`), `docker-compose.yml` (store mount +
      `graph_store_deploy_key` bind), `docker-entrypoint.sh` (`KEY_SRC`/sshCommand), `Dockerfile`
      (`safe.directory /srv/graph-store` — was a functional miss), and `provision.sh`
      (`GRAPH_STORE_DIR/REPO/DEPLOY_KEY`, two-pass hardening guard intact); new deploy-workflow step
      prints the VPS graph-store deploy **PUBLIC** key into the Actions log (best-effort; private key
      never leaves the box, ADR-016). App bootstrap (`ensure_ready`) still owns remote-wire + `push -u`;
      no VPS git steps added (ADR-031 §6). YAML + `sh -n`/`bash -n` clean; **independent review APPROVE —
      no must-fix** (1 informational: local untracked `server/.env` VAULT_PATH, fixed) —
      [08-logs/m3.md](08-logs/m3.md) task 9. Commit `e97671b`, **not pushed**.
      **GitHub-side prep already done (2026-07-14): `PSB-graph` created + VPS deploy key pasted with
      write access** — so the first `push -u` should work first try; the pubkey-print step is a
      convenience, not a blocker.
- [x] 10 — **live M3 Accept** — ALL criteria GREEN (2026-07-14); only the `PSB-vault` archive
      (user action) remains to close M3. (per accept draft above + threshold tuning + cutover: verify
      capture→node→PSB-graph push, then user archives PSB-vault). **Pre-Accept build (replanned in
      the task-10 session):** close the carried backend smokes and the `profile-embedding-in-search`
      gap. Real-DB SQL smoke of task-6/7a/7b stores + migration 006/007 = green (23 checks, local
      pgvector). **`profile-embedding-in-search` = a real unbuilt ADR-030 §4 requirement** (search was
      chunks-only; the stored profile vector was never queried) — mechanism pinned by
      **[ADR-037](adr/037-profile-embedding-in-search-m3.md)**: a second per-profile vector leg
      (`node_profiles.embedding`) unioned best-per-node with the chunk leg, all tiers, no weighting,
      `SearchResultItem` unchanged, reindex-decoupled; **migration 007** adds the profile HNSW index.
      Decision (task-10 session): **go straight to prod** for the live Accept (no local dry-run).
      **Cutover done + Accept STARTED (2026-07-14):** code pushed, CI deploy green, prod on the graph
      schema (migrations 005/006/007 applied), `/api/v1/health` all-green. Deploy-key gap fixed on the
      VPS (the graph-store deploy key was missing → pushes soft-failed; reused the freed `vault_deploy
      _key` as `graph_store_deploy_key`, force-recreated the api container → per-capture push works,
      `PSB-graph` populated). Live captures then surfaced **organizer-quality defects** (dangling edges,
      person over-extraction, entity split, diacritic mangling) → **replanned to task 11**; the Accept
      resumes after task 11 lands. `inbox/` clarified = model-failure fallback only (gibberish → an
      `unclear` memory, working as designed).
- [x] 11 — **organizer-quality + data-survival pass** (replanned 2026-07-14, grilled; ADR-038…042).
      Built + reviewed + deployed + prod-reprocessed + Accept criteria verified (2026-07-14); M3 Accept
      complete pending the `PSB-vault` archive.
      **All must-fix; M3 is accepted only after this + the remaining task-10 criteria pass.**
      **Built 2026-07-14** (all five + `idea` reclassified content-only per the kickoff grill:
      `entity_like_types` realizes the ADRs' `entity_types` = person/place/topic/event/project).
      374 tests (+30), ruff clean; real-DB smoke green (29 checks incl. token-overlap SQL) + a
      rolled-back reset-SQL smoke (11 checks, dev data intact). Detail: [08-logs/m3.md](08-logs/m3.md)
      task 11. Independent review done (APPROVE-WITH-MINORS, no must-fix; 375 tests). **Local
      end-to-end reprocess dry-run DONE 2026-07-14 — GREEN** (throwaway pgvector + local Ollama +
      live claude-max: seeded 4 defect-repro captures → `reprocess-all` wiped + replayed 4/4, 0
      failed; before/after asserted Horia+Horia-Fenwick dedup, no dangling edges, diacritics folded,
      raw preserved). Surfaced + fixed a **prod-dormant** `claude-max` UTF-8 subprocess-encoding bug
      (`b0148a4`, Windows-only mojibake; pins `encoding="utf-8"`), independently reviewed
      (APPROVE-WITH-MINORS, no must-fix); applied the review minor `errors="replace"` + first
      `ClaudeMaxProvider` tests (`f8a0e1b`, 378 tests). **Pushed + deployed + prod reprocessed
      2026-07-14** (`f8a0e1b`; CI+deploy green, no migration): `POST /admin/reprocess` succeeded 9/9
      (28 nodes, 2 inbox, 0 failed, push=True); **verified live on the prod DB** — Mădălina
      over-extraction+split healed to one folded hub, Horia accreted, 0 unfolded diacritics, 0
      dangling edges, raw preserved.
      **Remaining task-10 Accept criteria — ALL GREEN (2026-07-14, live verification, no code):**
      **reindex parity EXACT** (snapshot-guarded `TRUNCATE nodes CASCADE` → `POST /admin/reindex`
      rebuilt a byte-identical index from the `PSB-graph` store — canonical/derived/node-id
      fingerprints all matched, `EXCEPT` set-diff 0/0; proves Rule-1/ADR-001 store-is-truth);
      **profile-in-search** (ADR-037 — found `node_profiles` empty post-reprocess, ran the VPS
      `profile-refresh` CLI → 17 profiles + HNSW; the exact two-leg search surfaces person hubs via
      the profile leg; PWA "Madalina" → the person hub with profile snippet); **vocab-proposal→
      consolidation** (seeded an `edge_rel` proposal → user Approve → `app_settings` forward-live +
      feed-visible `vocab-consolidation` run linked to the review id + item resolved; synthetic seed
      reverted); **`ENTITY_MATCH_MIN_CONF` = keep 0.8** (user decision — conservative, zero false
      merges on the current sample; revisit as captures grow). Detail: [08-logs/m3.md](08-logs/m3.md)
      "Task 10/11 — remaining M3 live Accept criteria". Follow-up logged: reprocess/reindex leave
      `node_profiles` empty until the nightly job (silent profile-in-search gap). **`PSB-vault`
      archived by the user (2026-07-14) → M3 CLOSED** — all acceptance criteria pass; graph-native
      stack live at `braindan.cc`. **Next: M4 (chat) — planning session (`/grilling` first).**
      - **Dangling edges** ([ADR-038](adr/038-reorganize-preserves-shared-entity-hubs.md)): reorganize
        `remove_nodes` becomes type-aware — removes only content nodes (`memory`/`conversation`/
        `insight`/`idea`), never entity hubs (shared substrate); orphan hubs tolerated (later GC).
      - **Over-extraction** ([ADR-039](adr/039-entity-types-are-mention-only.md)): organizer prompt +
        a structural guard coercing any entity-typed content node → `memory` (body/mentions kept).
      - **Entity resolution** ([ADR-040](adr/040-token-overlap-retrieval-and-alias-accretion.md)):
        token-overlap candidate retrieval (low-entropy guard) + alias accretion on link; conf-floor +
        LLM + review unchanged (never guess).
      - **Diacritics** ([ADR-041](adr/041-diacritic-folding-derived-content.md)): fold NFKD→ASCII on
        **all** derived fields (slug/title/aliases/disambig/tags/body) at the `NodeWriter` chokepoint +
        matching; raw kept un-folded (never-lose).
      - **Reprocess op** ([ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md)): reusable
        `reprocess-all-from-raw` admin op (confirm-gated, visible) — resets capture-derived state,
        replays raw chronologically through the fixed pipeline; preserves approved vocab (+ best-effort
        merges), rebuilds the rest; raw + git history kept. Standing mechanism for **P10**.
      - **Sequencing:** build all five + unit tests → independent review → **local dry-run of the
        reprocess op** → deploy + reprocess prod (heals the 4 captures) → finish the remaining task-10
        Accept criteria (reindex parity, profile-in-search, vocab-proposal→consolidation, threshold
        tuning) → user archives `PSB-vault`.

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

**M4 KICKOFF GRILL — BUILD-READY (2026-07-14, grilled decision-by-decision; recorded per [09](09-session-protocol.md)).**
The addendum's "re-check at kickoff" resolved to a **lean spine + explicit deferrals** (Accept criteria
need none of the deferred items):
- **Retrieval (build):** (b) **hybrid vector+FTS fused by RRF** (k=60) — the FTS leg mirrors the ADR-037
  vector union: `tsvector` over **`chunks` ⊍ `node_profiles`** (**migration 008**: two generated `tsvector`
  cols + GIN, `'english'`); (d) **recency prior** = bounded multiplicative nudge (floor ~0.9, config
  half-life ~180d, applied to the fused list pre-cut); (e) **temporal** = `since`/`until` window + **simple
  node-date `as_of`** (`occurred_start ≤ as_of`) on `/search` (params only, not auto-extracted from chat);
  (c) **English condensation**. FTS-noise guard = emergent self-suppression (English corpus) + skip on
  zero-lexeme tsquery — **no language-detect dependency**.
- **Deferred (NOT M4):** (a) **1-hop edge-neighbor injection + entity-seeded expansion** → **Backlog**
  (reconciles the addendum vs the §5 "backlog" line in favor of backlog); (f) **identity capsule** →
  **M5** (its `build_context` level-0 depends on it — M5 owns it, not a silent gap); (g) **challenge
  mode** → **Backlog**.
- **Chat behavior:** condensation on **turn ≥2** condenses last **N=15** (config) → standalone **English**
  query, runs on **`conspect`** and **inherits the conspect group's effort** (the [04-pipelines §5](04-pipelines.md)
  "low effort" note is corrected to this — ADR-025 is per-group, not per-call-site). Thin retrieval →
  answers **general questions uncited**; "not in your memories" for personal questions with no hits;
  **prompt-driven + `min_score` floor, no classifier**. Retrieved context is **fenced as data-not-
  instructions** (injection hygiene now, before connector/MCP content shares the path). Titling runs
  **best-effort, non-blocking, after** the first exchange.
- **Model routing = three groups (both `chat` + `conspect` wired now).** `ModelRoutingService` governs
  `chat`, `conspect`, **and a new `quick` tier** ([ADR-043](adr/043-quick-routing-tier-m4.md), extends
  ADR-025 2→3); all **6 `conspect` call sites** rewired through the service (mechanical, guarded by
  ingestion tests + independent review); effort per-call to `claude-max` (ADR-025 §4). **`quick`** =
  Claude **Sonnet 4.6** (config, low effort) / Nebius fallback, provider id `claude-max-sonnet`, **only
  M4 caller = session titling** (LLM-generated); all three UI-editable in Settings → Models.
- **Web:** **banner = fallback/model transparency** (`fallback_used`); **"Not from your memories"** = a
  subtle per-message chip derived from **empty `sources`**.
- **Accept (live chain-and-record):** **(A) required** — Settings-driven Nebius-primary drive recorded on
  `chat_messages.model`; **(B) true-fallback (claude-max forced down → `fallback_used=true`) optional**.
  The 21 registry unit tests stay green.

**Accept:** questions over known graph content answered with correct `[n]` node citations on
both Claude and Nebius; "not in your memories" verified; sessions persist; **plus the deferred
M0 clause** (Settings-driven Nebius-primary drive recorded in `chat_messages.model` + the 21
registry fallback unit tests).

- [x] 1 model routing engine — `ModelRoutingService` over **3 groups** (`chat`/`conspect`/`quick`, ADR-043) + per-call effort; `claude-max-sonnet` provider instance; **rewire the 6 `conspect` call sites** through the service — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 1)
- [x] 2 retrieval — **migration 008** (FTS `tsvector` on `chunks` ⊍ `node_profiles` + GIN); hybrid vector+FTS **RRF** (k=60) + recency prior + `since`/`until`/`as_of` on `/search` — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 2)
- [x] 3 chat service (condensation-in-English → hybrid retrieval → **fenced** grounded prompt → cited-only parse → persistence; non-blocking `quick` titling) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 3)
- [x] 4 chat routers (`POST /chat`, `GET /chat/models`, `GET /chat/sessions[/{id}]`) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 4)
- [x] 5 settings routers (enriched `GET /settings` + `PUT /settings/models`, all 3 groups) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 5)
- [x] 6 web chat screen (list/thread, picker + plane chips, reveal, source cards, fallback banner, "not from your memories" chip) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 6)
- [x] 7 web Settings → Models panel (3 group controls: active/fallback dropdowns + effort selector where supported) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 7)
- [x] 8 live M4 Accept — local dry-run + prod live drive; **all criteria green** (dual-model `[n]` citations Claude+Nebius, "not in your memories", persistence, **clause (A)** Nebius-primary recorded `chat_messages.model=nebius`, **clause (B)** true fallback shown live, 21 registry tests green). **Surfaced + fixed a real defect:** the committed `NEBIUS_CHAT_MODEL` was a HF-style id Nebius never served (retired 3.1) → every live Nebius call fell back to claude-max; corrected to `meta-llama/Llama-3.3-70B-Instruct` (commit `419ece4`) — **DONE 2026-07-14** ([08-logs/m4.md](08-logs/m4.md) task 8)

**→ M4 (chat) ACCEPT COMPLETE (2026-07-14)** — grounded cited chat over the graph is live at `braindan.cc` on both Claude and Nebius; model routing + Settings panel + fallback transparency all verified live. Next: **M5 (MCP server)** — planning session (`/grilling` first) per [09](09-session-protocol.md).

### M4 follow-up — provider observability (GRILLED TO BUILD-READY 2026-07-15 — [ADR-044](adr/044-provider-observability-surface.md))

The M4 Accept surfaced that a provider can **silently fall back with no visible reason** (the Nebius
model-id defect — [08-logs/m4.md](08-logs/m4.md) task 8); this closes that **vision P8 / rule-7** gap.
Grilled decision-by-decision (2026-07-15, recorded per [09](09-session-protocol.md)):
**in-memory** per-provider status on the registry (no migration, no persistence — a chat/STT fallback is
a degradation signal, not a durable job failure; the failing mode is persistent so memory repopulates on
the next call) captured at **every** provider call site (chat + STT + embedding); a **sticky** `last_error`
+ `last_success_at` + `consecutive_failures`; a new **session-gated `GET /admin/providers`** (`/health`
left untouched) whose `reachable` field is a **live `Provider.health()` probe** (config-reachability, *not*
a success guarantee — the seam that would have shown Nebius green throughout); and a **read-only Settings
Providers card** (dot + last-error, ADR-006 thin client) so it's a 30-second glance on the phone. Full
rationale + rejections in [ADR-044](adr/044-provider-observability-surface.md).

**Accept:** `ProviderStatusTracker` unit tests green; a **local forced-failure** (reproduce the Nebius
bad-model-id) drives `last_error`(sticky) + `consecutive_failures` up and, on recovery, `last_success_at`
stamped + counter reset — all via `GET /admin/providers`; **prod** endpoint enumerates all providers with
`reachable` + `last_success_at` populated for anything exercised since boot; the Settings card renders
read-only on device; independent review clean on both tasks.

- [x] 1 server (2026-07-15) — `ProviderStatusTracker` collaborator (in-memory, sticky `last_error` + `last_success_at` + `consecutive_failures`, injectable clock) wired at all 3 registry call sites (chat/STT/**embedding** — the previously-unrecorded no-fallback leg, ADR-044's key blind spot; recorded then re-raised, nothing swallowed) + session-gated `GET /admin/providers` (live concurrent `health()` probe via `asyncio.gather`, defensive against a raising probe) + `capabilities` from configuration (`can_chat`/`can_transcribe`/`can_embed`, not the class hierarchy) + response models. `/health` untouched, no migration. 476 tests green (+16), ruff clean; **independent review APPROVE-WITH-MINORS — no must-fix** (added the one worthwhile minor: an HTTP-level endpoint test covering the `last_error` present/null mapping + session gating). Commit `9dad941`, **not pushed**.
- [x] 2 web + live Accept — **DONE → M4 follow-up 2 (provider observability) COMPLETE** (2026-07-15). Read-only Settings **Providers** card: a thin TanStack read (`useProviders`, 15s poll) over `GET /admin/providers` with per-provider status dot (green `consecutive_failures == 0` / amber `> 0`), sticky `last_error` line (muted once recovered), capability chips, an "unreachable" tag, and last-success time; wire types (`ProviderStatusItem` etc.) exact-match the server Pydantic models; `relativeTime` extracted to a shared `ui/` helper (dedup, rule 10). No server code (ADR-006). tsc/eslint/build green; a **real-browser walkthrough vs a throwaway mock API** drove every render state **and a forced-failure→recovery transition through the endpoint shape** (amber→green, `consecutive_failures` cleared, `last_error` stays sticky+muted — console clean on the Settings surface). **Independent review APPROVE-WITH-MINORS — no must-fix** (3 minors all fixed: shared `relativeTime`, exhaustive `Record<ProviderCapability,…>`, muted recovered-error line). Commit `a82500b`. **DEPLOYED to prod (2026-07-15):** pushed the two M4-follow-up commits (`9dad941` + `a82500b`) → CI green (server/web/secrets/deploy all ✓, no migration) → `braindan.cc` live, `/api/v1/health` all-true. `GET /admin/providers` confirmed **deployed + correctly session-gated** (`401 Not authenticated` unauth); PWA serves 200. **Live prod Accept VERIFIED (2026-07-15)** in the browser pane against the real endpoint (existing prod session — agent never handled the login secret): Settings → **Providers** enumerates **all 6 registered providers** (claude-max/claude-max-sonnet=Chat, nebius=Chat, openai/groq=Speech, ollama=Embedding) with correct labels + config-derived capabilities + green dots, and the panel has **zero interactive controls** (read-only confirmed). All show "No successful call yet" — the correct post-redeploy zero-state (in-memory status resets on boot; `last_success_at`/`last_error` populate as providers get exercised). The *local* forced-failure→recovery through the real endpoint was already covered by task 1's HTTP-level test + the task-2 mock walkthrough.

### M4 follow-up 3 — provider/model/effort separation (GRILLED TO BUILD-READY 2026-07-15 — [ADR-045](adr/045-provider-model-effort-separation.md))

The system conflated **provider** with **model**: to route Opus for `chat`/`conspect` and Sonnet for
`quick`, the registry ran **two fake `ClaudeMaxProvider` ids** (`claude-max`/`claude-max-sonnet`) over
the *one* `claude` CLI — so the ADR-044 Providers card shows two `claude` rows for one credential, and
the raw `claude-max` id leaks into the UI. Grilled decision-by-decision (2026-07-15, per
[09](09-session-protocol.md)) to make **provider / model / effort** first-class:
the **routable unit becomes a model id** (provider is an attribute of the model — Option A); a **model id
is the raw vendor string** (`claude-opus-4-8`, …); `claude` collapses to **one provider serving Opus +
Sonnet** via the CLI's per-call `--model`; the concept is fixed across **all** providers internally while
the editable picker stays **chat-only**; config declares Claude's models as **named scalars**
(`CLAUDE_OPUS_MODEL`/`CLAUDE_SONNET_MODEL`/`CLAUDE_EFFORT`); saved routing is **migrated** (idempotent
Alembic, old id→vendor string) while historical `chat_messages.model` is **left as-is** with
legacy-tolerant labels (vision P10 — migrate config, don't rewrite audit); the API/web field
**`effort_by_provider`→`effort_by_model`**; the Providers card renders **one row per provider** (friendly
name, no raw id). **Supersedes** the routing-unit of [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)
+ the `claude-max-sonnet` mechanism of [ADR-043](adr/043-quick-routing-tier-m4.md); **refines**
[ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md). Doc reconciliations recorded
(this note + 02/03/04/06 + README). **Paused before implementation** per [09](09-session-protocol.md).
**No code this session.**

**Accept:** the registry exposes **5 providers** (`claude` serving Opus+Sonnet, `nebius`, `groq`, `openai`,
`ollama`); Settings → Models picks **model ids** (Opus / Sonnet / Llama 3.3, provider derived) with
`effort_by_model`; a group's saved routing survives the rename via the migration (a pre-seeded
`claude-max` override resolves to `claude-opus-4-8`, not a silent reset); the Providers card shows **one
"Claude" row**; chat stays grounded + cited on Opus/Sonnet/Llama; no `claude-max`/`claude_max` string
remains in code, config, or UI; independent review clean on every task.

- [x] 1 server core — `ClaudeProvider` (serves N models); registry chat-model **catalog** (`{id, provider, label, supports_effort, effort_levels}`) + model→provider index; `ModelRoutingService` keyed by model id; config named scalars (`CLAUDE_OPUS_MODEL`/`CLAUDE_SONNET_MODEL`/`CLAUDE_EFFORT`); `*_chain` seeds → model strings; `effort_by_provider`→`effort_by_model`. Update all tests. **DONE 2026-07-15** ([08-logs/m4.md](08-logs/m4.md) "follow-up 3 · Task 1"), commit `7c69449`, **not pushed**; 478 tests green, ruff clean, **independent review APPROVE — no must-fix**.
- [x] 2 migration — idempotent Alembic **revision 009** rewriting saved `app_settings.model_routing` (`claude-max`→`claude-opus-4-8`, `claude-max-sonnet`→`claude-sonnet-4-6`, `nebius`→`meta-llama/Llama-3.3-70B-Instruct` **plus the `effort_by_provider`→`effort_by_model` key rename**, in active/fallback + the effort object's keys; ordered text substitution over a closed token set; no-op on absent/empty/already-migrated rows; downgrade reverses it) + legacy-tolerant `friendly_model_label` (retired ids fold to their vendor string; `chat_messages.model` history untouched). **DONE 2026-07-15** ([08-logs/m4.md](08-logs/m4.md) "follow-up 3 · Task 2"), commits `6133296`/`74de20d`, **not pushed**; 481 tests + ruff green, **migration SQL verified on real Postgres 16** (remap, key rename, idempotency, downgrade round-trip, empty/absent no-op, scoping), **independent review APPROVE — no must-fix** (2 minors applied).
- [x] 3 server API — `/settings`, `/chat/models`, `/admin/providers` response shapes (model-id semantics, `effort_by_model`, provider-name label, provider-only rows). **DONE 2026-07-15** ([08-logs/m4.md](08-logs/m4.md) "follow-up 3 · Task 3"), commit `a28ab04`, **not pushed**; the sole residual wire gap after tasks 1/2 was `GET /settings` model options not carrying `provider` (03-api §Settings) — added `provider` to `RoutingModelItem`, sourced from the registry catalog (the serving provider id, derived — ADR-045 §1); `/chat/models` (`{id,label,effort}`) + `/admin/providers` (one provider-labelled row) already matched their contracts (03-api lines 52/107) after task 1, unchanged. 481 tests + ruff green, **independent review APPROVE — no must-fix**.
- [x] 4 web — types (`effort_by_model`, provider label), ModelsPanel (model-id picks), chat composer picker, ProvidersPanel (provider-only, drop id). **DONE 2026-07-15** ([08-logs/m4.md](08-logs/m4.md) "follow-up 3 · Task 4"), commit `fc193c0`, **not pushed**; wire types renamed `effort_by_provider`→`effort_by_model` (`GroupRoutingModel`/`ModelRoutingUpdate`) + `provider` added to `RoutingModelItem`; `ModelsPanel` reads/seeds/dirty-tracks/PUTs `effort_by_model` (picks were already model-id-valued with server-friendly labels); `ProvidersPanel` drops the rendered raw provider id (friendly name only, one row per provider — ADR-045 §6; `id` kept as React key); chat composer picker unchanged under `{id,label,effort}` (comment refreshed). tsc/eslint/vite green; **real-browser walkthrough vs a throwaway mock API** drove all three surfaces (Models seeds `effort_by_model` + a save round-trips it, Providers renders the 5 friendly rows with no id, the composer shows model-id options with friendly labels and its default follows the saved Chat routing forward-live — console clean); **independent review APPROVE — no must-fix**.
- [x] 5 live Accept — **DONE 2026-07-15** ([08-logs/m4.md](08-logs/m4.md) "follow-up 3 · Task 5"). Pushed the 5 commits `7c69449`→`fc193c0`; CI run `29397900778` all green; **migration 009 applied** on prod (`alembic upgrade head` under `set -e`); `/api/v1/health` all-true. Accept criteria met — **5 providers / one "Claude" row**, model-id picks + `effort_by_model`, saved `model_routing` migrated in place (not reset; prefix hazard handled), chat grounded on Opus/Sonnet/Llama, **no `claude-max`/`claude_max` residual** (grep clean except the deliberate legacy-fold map + migration test input), CLAUDE.md rules intact. Live PWA surfaces user-confirmed against the existing prod session (agent never handled the login secret). **Independent review APPROVE — no must-fix** (fresh agent re-derived all 8 criteria from ADR-045 + 03-api and checked the `a82500b..fc193c0` diff). **M4 follow-up 3 COMPLETE.**

## M5 — MCP server ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md) · **GRILLED TO BUILD-READY 2026-07-15, [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md)**)

**Scope.** A remote MCP server exposing the service layer, reachable from external LLMs. The
M5 grill (decision-by-decision, [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md)) turned
"smallest milestone" into a real multi-task build, because the connection surfaces the user
wants — the **mobile Claude app + claude.ai web (custom connectors)** — require an **OAuth 2.1
flow**, not a static bearer. The same OAuth build unlocks Claude Desktop + ChatGPT; Claude Code
CLI is deferred.
- **Transport/deploy:** official `mcp` SDK, **Streamable HTTP** under `/mcp` on the existing
  `api` app, single origin `braindan.cc` (Caddy root routes; Cloudflare un-cached); one
  container (ADR-003).
- **Auth:** self-hosted **OAuth 2.1 AS** (`authlib`) — `.well-known` discovery, open DCR
  (inert alone), `/authorize` **password + explicit-consent gate** (reuses Argon2id login,
  CSRF + rate-limited, PKCE), **opaque HMAC-hashed DB tokens** (~1h access + long-lived sliding
  refresh → no daily re-approve), **revoke-all** switch, single full scope. Resolves the old
  "MCP token distribution" open question (it's the add-connector flow).
- **Tools** (thin over the service layer, **markdown-rendered** at the boundary, IDs verbatim,
  cross-model): `search`, `get_node`, `traverse` (new `GraphService.neighbors` one-hop
  primitive — rel filter, both dirs, **cursor-paginated**; M7's map endpoint reuses it),
  `build_context` (get_node + neighbors, **depth ≤ 2**, fanout caps; **identity capsule as L0**),
  `list_planes`/`list_types`, `capture` (`source: mcp`, **burst-queued**, fast ack). Hub edge
  cap + `traverse` pointer. MCP `initialize` **`instructions`** usage capsule + rich tool
  descriptions + read/write annotations + a **research-via-MCP Prompt** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)
  #6: query known → find gaps → research → `capture` distillate w/ source refs).
- **Identity capsule** (ADR-033 #1, refined — insights barely exist at M5): **broadened source**
  (top entity-profile hubs + recent memories + insights when present), **300-token** distill on
  `conspect` → `app_settings` blob, **nightly refresh** + on-demand trigger; served as
  `build_context` L0 **and** `identity://me` resource; **wired into the M4 chat system prompt in
  M5** (in-app chat finally consumes it).
- **Observability:** MCP captures → `agent_runs` (source-tagged, visible in activity); reads
  unlogged.

**Accept:** the **Accept gate is a real Claude connector** (mobile app / claude.ai web) live
against `braindan.cc/mcp`: OAuth approve → a `capture` lands as organized nodes; `search`+
`get_node`+`traverse`+`build_context` (capsule L0) answer a question about known graph content;
**revoke-all locks it out**; MCP capture visible in activity. Verification: fakes for logic +
an **MCP-SDK-client protocol + scripted-OAuth integration harness** (pre-push gate) + an
**OAuth-focused independent security review**. **ChatGPT is a fast-follow verification before
M6** (may need thin `search`/`fetch` aliases; its quirks must not block M5's close).

- [x] **Task 1** — traverse primitive (`GraphService.neighbors`) + `build_context` core (service layer; M7-reused; fakes + real-PG edge-SQL smoke). **DONE 2026-07-15** — `PgNeighborStore.neighbors` (both-dir union, rel/direction filters, keyset pagination on `(origin,rel,dir,node_id)`, tombstone-excluded both ends) + `GraphService.neighbors`/`build_context` (opaque cursor, depth ≤2 + fanout cap + cycle guard, `NodeReader` seam over search; L0 capsule deferred to task 2). 16 unit tests + real-PG smoke; independent review APPROVE, no must-fix (3 minors fixed) — [08-logs/m5.md](08-logs/m5.md). Commits `ff4a729`/`bb33ae7`, **not pushed**.
- [x] **Task 2** — identity capsule (broadened-source 300-token distiller → `app_settings` blob + nightly refresh + on-demand trigger) **+ wire into the M4 chat system prompt**. **DONE 2026-07-15** — new `app/identity/` package: `PgCapsuleSourceStore` (top profile hubs by canonical graph degree + recent memories + recent insights, tombstone-excluded) + `PgIdentityCapsuleStore` (`{text, generated_at, source_refs}` blob in `app_settings`, rule-1 rebuildable) + `IdentityCapsuleService` (one `conspect` distill of the fenced blended source; best-effort — no source / LLM down / empty keeps the last capsule; single-flight nightly job **04:35** + on-demand `POST /admin/identity-capsule/refresh` + CLI). Served as `build_context` **L0** (cheap `IdentityCapsuleReader` seam, never distilled inline, best-effort) **and** prepended to the M4 chat system prompt (fenced data-not-instructions). Config knobs (rule 9). **524 tests green** (+25 unit) + **5 real-PG smoke checks** (`scripts/smoke_db.py`: degree-ranked hubs, recent ordering, tombstone exclusion, blob round-trip); ruff clean; **independent review APPROVE-WITH-MINORS — no must-fix** (1 typing regression fixed; 3 logged) — [08-logs/m5.md](08-logs/m5.md). Commits `2ea5834`/`cb3c25a`, **not pushed**. The `identity://me` MCP resource stays task 4.
- [x] **Task 3** — OAuth 2.1 AS (`authlib`: `.well-known`, DCR, `/authorize` gate, `/token`+PKCE+refresh, opaque HMAC DB tokens, revoke-all) + migration (client/token tables). **DONE 2026-07-15** — new `app/oauth/` package (errors · store · metadata · consent · service) + root-mounted `app/routers/oauth.py` (discovery/`/register`/`GET+POST /authorize`/`/token`) + session-gated `POST /admin/mcp/revoke-all` + **migration 010** (`mcp_oauth_clients`/`mcp_auth_codes`/`mcp_tokens`). authlib supplies only the security-critical crypto (PKCE S256, secure token gen, RFC 8414 metadata schema); the flow orchestration is hand-rolled over asyncpg (rule 5, mirrors the session-token discipline). Opaque HMAC-hashed codes+tokens (`mcp_token_hmac_secret`), atomic single-use codes w/ replay-revoke, refresh **rotation** w/ reuse detection, PKCE-required + CSRF (double-submit) + login-rate-limited password/consent gate w/ PWA-session short-circuit, RFC 8707 resource binding, revoke-all switch. `validate_access_token` seam ready for task 4. **576 tests green** (+52) + **13 real-PG smoke checks** (single-use consume, revoke-all count, FK cascade, `invalidate_all_codes`, `revoke_token` rowcount) + a **scripted HTTP E2E** (discovery→DCR→authorize→token→validate→refresh→revoke, real ASGI + real `PgOAuthStore` + local pgvector); migration 010 up/down round-trip verified on the dev DB; ruff clean. **`authlib>=1.3` added to `pyproject`/lock.** **Independent security review APPROVE-WITH-MINORS** — 3 findings fixed (atomic refresh rotation w/ race-proof reuse-detection, revoke-all now invalidates pending codes too, dropped un-wired `client_secret_basic`); 2 logged (pre-existing XFF rate-limit key → task-5 deploy decision; sync Argon2 nit) — [08-logs/m5.md](08-logs/m5.md). Commits `42ea108`(+review-fix) pending; **not pushed**. `MCP_TOKEN_HMAC_SECRET`/`PUBLIC_BASE_URL` provisioning + Caddy root routes are **task 5**.
- [x] **Task 4** — MCP server + tools (SDK Streamable HTTP under `/mcp`; six tools; markdown renderers; `instructions`+descriptions+annotations+research Prompt; `capture` source-tag + burst queue; capsule L0 + `identity://me`; resource-server token validation) + protocol integration harness. **DONE 2026-07-15** — new `app/mcp/` package (`render` markdown serializers · `text` instructions+descriptions+research prompt · `tokens` OAuth `TokenVerifier` bridge · `server` FastMCP factory), the official `mcp` SDK **Streamable HTTP** mounted at the ROOT so `/mcp` resolves exactly (SDK Route, no trailing-slash redirect), gated by the task-3 OAuth resource server (`validate_access_token` → `AccessToken`; 401 carries `WWW-Authenticate` → `/.well-known/oauth-protected-resource/mcp`). **7 tools** thin over the service layer, **markdown-rendered** (IDs verbatim, hub edge cap + `traverse` pointer, `readOnlyHint` on reads / write marker on `capture`), `initialize` **instructions** capsule + rich descriptions + `identity://me` resource + **research** prompt (ADR-033 #6). `capture` → **migration 011** `captures.source` column threaded to node frontmatter `source: mcp` (web falls back to kind) + `create_mcp_capture` **burst semaphore** (`mcp_capture_max_inflight`, fast ack). Session manager run in the app lifespan. **599 tests green** (+23: 11 render + 11 in-memory MCP-protocol harness + 1 capture-source) + a **real Streamable-HTTP MCP-client HTTP E2E** (uvicorn + the real app + local pgvector: unauth `/mcp`→401+WWW-Authenticate, OAuth→token, `initialize`→instructions, list 7 tools, call tools over the wire, read `identity://me`, list research prompt — all PASS); migration 011 up/down verified; ruff clean. `mcp>=1.28` added. **Independent review APPROVE-WITH-MINORS — no must-fix** (2026-07-15; fresh agent re-derived scope from ADR-046 + ADR-028 + 03-api §MCP + CLAUDE.md, verified SDK mount/no-redirect, lifespan single-run teardown, fail-closed OAuth gate, depth clamp, never-lose capture, additive migration 011, injection hygiene): 6 minors logged, 1 fixed in-pass (stale 03-api `traverse` signature → `direction?=both`); highest-value follow-up = an ASGI-level 401/scope regression test on the auth gate (committed tests bypass HTTP auth) — [08-logs/m5.md](08-logs/m5.md) task 4. Commit `170883b`, **not pushed**. Deploy (Caddy `/mcp` route + provisioning) is **task 5**.
- [x] **Task 5** — deploy + infra (Caddy root routes, Cloudflare passthrough, `MCP_TOKEN_HMAC_SECRET` provisioning, compose/env; push → migration applies). **CONFIG DONE 2026-07-15** — Caddy `@mcp_oauth` handle proxies `/mcp`·`/mcp/*`·`/.well-known/oauth-*`·`/authorize`·`/token`·`/register` → `api:8000` (`/mcp` matched trailing-slash-free; after `/api/*`, before the PWA catch-all); `PUBLIC_BASE_URL=https://braindan.cc` in defaults.env; `MCP_TOKEN_HMAC_SECRET` CI-injected (env+printf, mirrors `SESSION_SECRET`); both documented in `.env.example`; provision.sh/compose unchanged. **Independent review APPROVE — no must-fix** (env-name→Pydantic bindings, matcher coverage, handle ordering, secret classification all verified end-to-end; 2 cosmetic/pre-existing minors) — [08-logs/m5.md](08-logs/m5.md) task 5. Commit `ffd71f4`, **not pushed**. **⚠ Deploy is gated on two human steps** (agent cannot do either): (1) add `MCP_TOKEN_HMAC_SECRET` to GitHub `production` secrets — **if unset, CI writes an empty value that overrides the code default → empty-key HMAC on an internet-facing surface**; (2) add a Cloudflare Cache Rule bypassing cache on `/mcp*` + `/.well-known/oauth-*`. **DEPLOYED + live-verified 2026-07-15** (through `e0507e5`): both preconditions set by the user, 9 commits pushed, CI green, migrations 010/011 applied. `/.well-known/oauth-authorization-server` serves real AS metadata (PKCE S256), `/mcp` unauth → **401 + `WWW-Authenticate`**, protected-resource metadata JSON live, health all-green, PWA intact. **Two deploy bugs found + fixed live** (Caddy not reloaded on `up`; then the root cause — single-file bind-mount inode staleness across `git pull` rename → **`up -d --force-recreate --no-deps caddy`** now in the deploy step, recorded in 07-infra §CI/CD). **MCP + OAuth surface live at `braindan.cc`.**
- [x] **Task 6** — live M5 Accept (Claude connector) + OAuth-focused independent security review → **ChatGPT fast-follow (before M6)**. **DONE → M5 CLOSED (2026-07-16):** ChatGPT fast-follow PASSED — verified the current connector contract (Developer Mode lets ChatGPT call all tools incl. `capture`; Deep Research would need read-only `search`/`fetch` aliases, not the ingest path), re-checked server readiness (protected-resource metadata + open DCR accepting ChatGPT's `redirect_uri`), and the **user connected the custom connector at `braindan.cc/mcp` in Developer Mode → OAuth approve → 7 tools listed** with **no code change** (the issuer trailing-slash nit didn't trip ChatGPT). M5 Accept user-accepted (Claude connector `search` grounded + capture-side spot-checked; `revoke-all` lockout not live-demonstrated, user moved on); OAuth security review APPROVE + hardening deployed. Read-only Deep-Research `search`/`fetch` aliases remain a pre-approved on-demand add (ADR-046) if wanted later. — [08-logs/m5.md](08-logs/m5.md) task 6. **Original in-progress note (2026-07-15):** real Claude connector live at `braindan.cc/mcp` — OAuth approve → connect works, **`search` verified grounded over the graph**. Diagnosed+fixed a live connect bug: FastMCP DNS-rebinding protection allowed only localhost → authenticated `POST /mcp` got **421** behind the edge; fix passes `TransportSecuritySettings` from `public_base_url` (`1ed0ee0`, deployed, +regression test, 600 tests green) — [08-logs/m5.md](08-logs/m5.md) task 6. **OAuth security review DONE — APPROVE-WITH-MINORS, no must-fix** (gate PASSES); **hardening batch #1–#5 DEPLOYED + live-verified 2026-07-15** (`1ed5a68`+`2dc8f92`, CI `29445868984` green; boot guard passed → `/health` all-true proves both prod secrets real; `X-Frame-Options`/CSP `frame-ancestors` live on `/mcp`+`/authorize`; unauth `/mcp`→401 intact). **Remaining (user-driven live Accept — needs the real Claude connector):** `capture`→node + `get_node`/`traverse`/`build_context` + activity-visible capture + **revoke-all** lockout (run last, destructive); then **ChatGPT fast-follow** before M6. Follow-up: external "who am I" needs a manual `identity://me` resource-attach + a populated capsule (nightly 04:35 / on-demand refresh) — backlog design item.

## M5.5 — Scheduling (pipeline orchestrator, [ADR-047](adr/047-pipeline-scheduling-primitive.md))

**GRILLED TO BUILD-READY 2026-07-16** (M6 kickoff grill — a scheduling architecture change
surfaced and was carved out ahead of M6's features so a scheduling regression can't masquerade
as a chat-distiller bug). **Orchestration only — no job changes what it does.**
**✅ M5.5 CLOSED (2026-07-16)** — all 3 tasks done, live Accept passed on `braindan.cc` (one nightly
start drove the whole roster in order under a parent `agent_runs` run with per-step children). Details
below + [08-logs/m5.5.md](08-logs/m5.5.md).

**Scope.** Replace the per-job staggered nightly crons ([ADR-010](adr/010-agent-window-3-5am.md))
with **pipelines as the sole schedulable unit** (ADR-047): a config-defined pipeline = a name +
one cron + an ordered list of **steps**, each with an `on_fail` policy (`continue`|`halt`); the
runner executes steps **sequentially, each starting only when the previous completes** (one start
time, no minute-tuning, one step's RAM at a time). Migrate every existing nightly job off its own
cron into a **`nightly`** pipeline (daily steps in dependency order) and a **`weekly`** pipeline
(Sunday integrity-drill). A pipeline run opens a **parent `agent_runs`** row; each step keeps its
**own child run** (new `agent_runs.parent_run_id`). Jobs stay independently invokable
(CLI + `POST /agents/{name}/run`, invariant 4); the scheduler registers **one cron per pipeline**.
Richer visualization deferred to **M8**.

**Accept.** Every existing nightly job runs as a **step in a scheduled pipeline**, not its own
cron; `nightly` + `weekly` each have exactly **one start time** and run steps **sequentially on
completion regardless of duration**; a `continue` step's failure doesn't stop the pipeline and a
`halt` step's does; each step still opens its own `agent_runs` under a parent pipeline run; any
step is still runnable standalone (CLI + API) and a manual run mid-pipeline is serialized safely
by the existing single-flight locks; **no behavior change** to any job — the DB-wipe → `reindex`
durability drill still passes.

- [x] **Task 1** — pipeline runner + config model (`nightly`/`weekly` pipeline defs: name, cron,
      ordered steps, per-step `on_fail`; rule 9) + `agent_runs.parent_run_id` migration + parent/
      child run linkage. Sequential-on-completion; per-step `on_fail` honored; single-flight
      preserved. Unit-tested (fake steps: ordering, `continue`-vs-`halt`, parent/child runs).
      **DONE (2026-07-16):** orchestration **mechanism only** — the runner + config model + DB
      linkage, deliberately **not wired into the scheduler and no job migrated** (task 2). Migration
      012 (`agent_runs.parent_run_id`, nullable self-ref FK `ON DELETE SET NULL` + index — a bare
      job run stays parentless, unchanged); **parent/child linkage via a task-local contextvar**
      (`child_run_scope`) so a step's `run_scheduled` links to the pipeline parent with **no change
      to what any job does** (ADR-047 §5/consequences); `app/services/pipeline.py` `PipelineRunner`
      runs steps **sequentially-on-completion**, honours per-step `on_fail` (`continue`|`halt`,
      status **inferred from each step's child run** — a non-cleanly-done child reads as a failure so
      `halt` reliably aborts), closes an orphaned `running` child on a raising step (rule 7), records
      the per-step sequence in the parent run, and never crashes the scheduler. Config
      `pipeline_defs()` = the migrated ADR-010 roster in dependency order, **`continue`-dominant**
      (§4); step names map to the existing scheduler job ids (task-2 wiring is a lookup). **619 unit
      tests** (16 new, fake steps) **+ 79 real-PG smoke** (6 new: parent_run_id INSERT/FK/contextvar
      reset) green, ruff clean, migration 012 up/down round-trip verified on local pg. **Independent
      review APPROVE — no must-fix** (4 minors applied: robust `halt` inference, orphan-child close,
      `SKIPPED` constant, skipped count). Commit `c0a3bd6`, **not pushed**. Task-2 notes: omit
      disabled/optional jobs from the step-func map; register the pipeline cron with
      `max_instances=1`/`coalesce`. — [08-logs/m5.5.md](08-logs/m5.5.md) task 1.
- [x] **Task 2** — migrate the existing roster off individual crons into the two pipelines
      (scheduler registers one cron per pipeline; CLI/`run-now` entrypoints unchanged); retire
      the per-job cron settings. Verify the durability drill + a manual `run-now` mid-pipeline.
      **DONE (2026-07-16):** `BackupScheduler` → **`PipelineScheduler`** — registers **one cron per
      pipeline** (a step-name→job map + one `PipelineRunner` per `pipeline_defs()`; `max_instances=1`/
      `coalesce`/misfire so a long night can't overlap the next, ADR-047 §3/§7). An **unwired optional
      step is dropped with a log** (prod wires all four unconditionally, so all 8 nightly steps always
      run); **CLI + `POST /admin/reindex` untouched** (invariant 4 / §6). **Retired the nine per-job
      `*_cron` settings** (zero remaining refs); nightly roster maps **1:1** onto the old stagger in
      order; weekly cron byte-identical to the retired `integrity_drill_cron` (no schedule
      regression). **No job body changed** → the DB-wipe→reindex drill is preserved by construction
      (diff = scheduler/config/main/tests only; live drill re-run = task 3). **619 unit tests + 79
      real-PG smoke green**, ruff clean; a **real-PG integration** drove the full wiring (one parent
      `nightly` run, row-opening steps linked children in order, `store-sweep` unchanged/no row,
      pipeline completes). **Independent review APPROVE — no must-fix**; 2 follow-ups logged (Sunday
      nightly/weekly RAM overlap = deliberate ADR-047 §3 residual; `run_store_sweep` opens no row so
      shows as a `skipped` step — both out of orchestration-only scope). Commit `8a57611`, **not
      pushed**. — [08-logs/m5.5.md](08-logs/m5.5.md) task 2.
- [x] **Task 3** — live M5.5 Accept (deploy; confirm one nightly start drives the whole roster in
      order, per-step runs visible, durability drill green) → independent review → pause.
      **DONE → M5.5 CLOSED (2026-07-16):** pushed `c0a3bd6`/`8a57611`/`e77a694` (CI green, migration
      012 applied on prod, `/health` all-true). Added a **`python -m app.cli pipeline {nightly|weekly}`
      run-now verb** (ADR-047 §6 enabler) that prints parent run id + per-step status + child run id.
      The user ran `pipeline nightly` on the VPS: **one start drove all 8 steps in dependency order**
      under parent run `939306a4` — 7 succeeded / 0 failed / 1 skipped (`store-sweep` = the known
      no-row job, it ran `pushed=True`), each row-opening step linking a child run; every job did its
      normal work (**no behavior change**); the run **also confirms migration 012 on prod** (the
      `parent_run_id` INSERT succeeded). All Accept criteria met. **Independent review APPROVE — no
      must-fix**; one fidelity minor fixed (`2f6c8fb`, committed local: run-now now uses effective
      vocabulary so it scans the same entities as the cron). Follow-ups logged (give `store-sweep` a
      run row → M8; deploy `2f6c8fb` next push; Sunday RAM-overlap residual). — [08-logs/m5.5.md](08-logs/m5.5.md) task 3.

## M6 — Chat-distiller + review queue ✅ CLOSED (accepted 2026-07-16) ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md) · build decisions [ADR-048](adr/048-m6-chat-distiller-build-decisions.md))

**GRILLED TO BUILD-READY 2026-07-16** (decision-by-decision; [ADR-048](adr/048-m6-chat-distiller-build-decisions.md)).
Scope = **core + addendum (a)–(d)**; **contradiction sweep (e) deferred**. Runs inside the
[ADR-047](adr/047-pipeline-scheduling-primitive.md) pipelines (M5.5).

**Scope.** The stance-gated **chat-distiller** (single-pass multi-candidate distill on `conspect`,
salience gate, `endorsed`/`unclear`/`rejected`, hedge/affect→review) whose **endorsement
materializes a `captures` row** (`source=chat`, replayed by reprocess → **P10 holds**) through the
organizer; a **`chat_distill_state`** watermark (idle-eligibility + idempotent delta re-distill);
`stance-candidate` + `dedup-proposal` **review_queue** kinds (payloads + **maybe re-openable** +
**kind-aware reprocess** preservation); **batch** review actions + coarse-LLM **salience** ordering
+ weekly **maybe digest**; a nightly all-source **dedup sweep** (merge via an **extracted
merge-core** shared with entity-merge / keep / link; augment deferred); a nightly **inbox drainer**
(`reorganize_capture` with the richer registry); **one-tap remove** (git-rm + DB-delete + capture
**tombstone** `removed_at`); `POST /chat/sessions/{id}/remember` (sync distill / async organize);
web **Review** extensions + Chat **"Remember now"** + a chat-scoped **"recently auto-recorded"**
list. Full rationale in [ADR-048](adr/048-m6-chat-distiller-build-decisions.md).

**Accept.**
- A substantive chat session yields an `insight`/`conversation` node **overnight** via the pipeline,
  backed by a `captures` row (`source=chat`), auto-endorsed, shown in the "recently auto-recorded"
  list and **removable in one tap** (git-rm + DB-delete + capture-tombstone); a subsequent
  `reprocess-all` does **not** resurrect it.
- A **pure-retrieval** session is skipped-and-logged; a **stance-unclear** candidate waits in Review
  and ingests **only on agree**; a **maybe** parks and stays **re-openable**.
- **`POST …/remember`** distills on demand (sync `{recorded,to-review}` summary, async organize),
  idempotent with the nightly run via the watermark, same gate.
- **Dedup**: a near-duplicate files a `dedup-proposal`; **merge** collapses two content nodes via the
  shared merge-core (loser tombstoned `merged_into` survivor); **keep**/**link** also available.
- **Inbox drainer** re-organizes an `inbox/` capture that now resolves, removing it from `inbox/`.
- **`reprocess-all`** preserves `stance-candidate` items + rebuilds endorsed chat nodes from their
  captures (**P10 end-to-end**); `dedup-proposal`/`entity-ambiguity`/`vocab-proposal` are truncated
  (re-derivable).

**Task list ([ADR-048](adr/048-m6-chat-distiller-build-decisions.md)):**
- [x] **Task 1 DONE (2026-07-16)** — `chat_distill_state` migration + the **chat-distiller job**:
      idle-eligibility (`max(chat_messages.created_at)` + `chat_distill_idle_hours`), single-pass
      multi-candidate `conspect` distill (fenced input, structured output, hedge/affect→unclear),
      **endorsement → `captures` row** (source=chat, one row/candidate, `created_at`=anchoring-message
      time) → organizer; unclear → `stance-candidate` review item; rejected → run log; watermark
      advance (delta-only re-distill, idempotent) + `agent_runs`. **Not yet scheduled** (added as a
      pipeline step in Task 8). Details in [08-logs/m6.md](08-logs/m6.md) task 1.
- [x] **Task 2 DONE (2026-07-16)** — review_queue M6 kinds: `PgReviewQueue.resolve` **maybe-reopen**
      fix (`DECIDABLE_STATUSES = pending ∪ maybe`; `resolved`/`discarded` terminal), `stance-candidate`
      resolution (`verdict` agree/disagree/maybe) — **agree = the exact auto-endorse `create_chat_capture`
      path** (one ingest, rule 2b/10), disagree→discarded, maybe→parked+re-openable; conversation-time
      `anchor_at` recorded in the payload at file time so agree stamps the capture with the anchoring
      message time (ADR-048 §1); **kind-aware reprocess reset** (`DELETE … WHERE kind <> 'stance-candidate'`
      — preserve stance, truncate the rest; refines ADR-042 §2). 646 tests + smoke 91/91 & 12/12 green,
      ruff clean, **independent review APPROVE-WITH-MINORS — no must-fix** (2 minors fixed). Web Review
      surface for these kinds is task 7. Details in [08-logs/m6.md](08-logs/m6.md) task 2.
- [x] **Task 3 DONE (2026-07-16)** — `POST /chat/sessions/{id}/remember` + `POST /review/batch`.
      **remember**: `ChatDistillerService.remember(session_id)` runs the **same** `_distill_session`
      pass (same salience + stance gate, **no force-endorse**) on the delta-after-watermark, advances
      the **same** `chat_distill_state` watermark (idempotent with the nightly + double-remember via
      the deterministic capture id), opens a visible `agent_runs` row; endorsed captures organize in
      the background (fast ack). A new `PgChatDistillStore.session_state` (by-id, no idle filter,
      LEFT-JOIN) separates **unknown session → 404** from **nothing new → `{skipped}`**; returns
      `200 {endorsed,to_review}` else `{skipped:reason}`; `422` malformed id. **batch**: `POST
      /review/batch {ids[],action}` → `{results:[{id,ok,error?}]}`, best-effort per item reusing the
      single-item `resolve` (action passed as both `choice`+`verdict`; each kind reads its own field —
      rule 10), declared before `/{review_id}`, config-capped (`review_batch_max`, rule 9 → `422`).
      665 tests (+19) + smoke 94/94 (+3 `session_state`), ruff clean, **independent review
      APPROVE-WITH-MINORS — no must-fix** (batch cap + smoke-branch minors applied). Details in
      [08-logs/m6.md](08-logs/m6.md) task 3. **Committed locally, not pushed.**
- [x] **Task 4 DONE (2026-07-16)** — **one-tap remove** for chat-distilled nodes: **migration 014**
      (`captures.removed_at` tombstone + the **`chat_auto_recorded`** audit registry) + the remove op
      (`AutoRecordedService.remove`: keep-hubs `remove_nodes` → `delete_nodes` the content paths →
      `captures.removed_at` tombstone → commit; **replay-excluded** in `capture_ids_chronological`/
      `counts` **and** the reorganize/follow-up/§10-drainer path) + **`GET /chat/auto-recorded`** +
      **`POST /chat/auto-recorded/{id}/remove`** (204; 404 unknown/removed/not-auto-endorsed; 422
      malformed); the distiller's **endorsed** branch records `capture_id`+`salience` in the registry
      (agree-from-review does **not** → the list is auto-endorsed only, ADR-048 §11). 681 tests (+16)
      + real-PG smoke 103/103 (+10), migration 014 up/down round-trip verified, ruff clean.
      **Independent review APPROVE-WITH-MINORS** — 1 must-fix (a tombstoned capture could be
      resurrected via `reorganize_capture`/the §10 drainer — **fixed**: `_replace_notes_via_reorganize`
      skips a removed capture) + 1 correctness minor (DB-delete decoupled from the unlink result so a
      crash-retry self-heals — **fixed**) + 1 type nit (**fixed**). Details in
      [08-logs/m6.md](08-logs/m6.md) task 4. **Committed locally, not pushed.**
- [x] **Task 5 DONE (2026-07-16)** — **dedup sweep + extracted merge-core**
      ([ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md)): **`MergeCore.fold`** extracted
      (retarget→tombstone→reindex→force-commit) — entity-merge = core + alias-union (behaviour-identical),
      content-merge = core alone; **`DedupSweepService`** files `dedup-proposal` items for content-node
      near-dups passing a strict-AND SQL gate (HNSW top-K cosine ⋀ shared entity-hub edge ⋀ occurred-overlap,
      undated never excludes), driver bounded by a last-success `agent_runs` watermark (**no migration**),
      canonicalized+deduped pairs, **re-file guard** (`proposal_exists` any status), `default_survivor`
      (degree/older), capped per run. Resolution `{action: merge|keep|link, survivor?}` — **merge** folds via
      the core, **keep** discards, **link** writes a **canonical `similar`** edge (survives the derived
      recompute). Config knobs + `dedup-sweep` CLI verb (pipeline wiring = task 8); `dedup-proposal`
      truncate-on-reprocess (task-2 kind-aware reset). **707 tests** (+26) + **real-PG smoke 114/114** (+11:
      the un-fakeable candidate SQL — cosine/shared-hub/occurred/undated gates + re-file guard + degree),
      ruff clean; a CLI end-to-end on dev data + **independent review APPROVE — no must-fix** (5 minors logged;
      the "undated never excludes" smoke gap closed). Commit `fd18c7b`, **not pushed**. **augment deferred.**
      Details in [08-logs/m6.md](08-logs/m6.md) task 5.
- [x] **Task 6 DONE (2026-07-16)** — **inbox drainer** job (nightly, bounded, own step): find
      `inbox/`-materialized captures (`node_paths` under `inbox/`, `removed_at IS NULL`, oldest-first,
      capped by `inbox_drain_max_per_run`) → drive the **existing** `reorganize_capture` **inline**
      (`CapturePipeline.reorganize_capture_now`) through the single writer — replaced only on success,
      still-failing kept in `inbox/` and re-qualified next run (status-agnostic query). Own
      `inbox-drain` `agent_runs` row (found/reorganized/resolved/still_inbox/errored/truncated),
      best-effort per capture, never raises. `PgCaptureStore.list_inbox_materialized` (`unnest … LIKE
      inbox/%` ⋀ `removed_at IS NULL` — the §11 double guard with the core's `removed_at` skip, so a
      one-tap-removed capture can't resurrect). No new review kind (residual ambiguity → normal
      `entity-ambiguity`). `inbox-drain` CLI verb (`backup_now` flush); pipeline wiring = task 8. **715
      tests** (+8) + real-PG smoke **117/117** (+3, the un-fakeable `list_inbox_materialized` SQL), ruff
      clean; **independent review APPROVE-WITH-MINORS — no must-fix** (3 minors logged). Commit
      `af14de5`, **not pushed**. Details in [08-logs/m6.md](08-logs/m6.md) task 6.
- [x] **Task 7 DONE (2026-07-16)** — web: **Review** extensions + Chat distiller surfaces (ADR-048
      §12). **Review** (`ReviewScreen`): `stance-candidate` card (proposed memory + entity chips +
      why-unclear + excerpt → agree/disagree/maybe) and `dedup-proposal` card (two lazy `useNode`
      previews + **survivor radio** defaulting to `default_survivor` → merge/keep/link; merge sends
      `{action,survivor}`, keep/link `{action}` only); **salience ordering** (high-first stable sort,
      no-salience → med); **single-kind multi-select** batch bar (stance agree/disagree/maybe · dedup
      keep; batch merge deliberately omitted — survivor is per-item) over `POST /review/batch` with a
      per-item result summary; **parked/maybe** section (`GET /review?status=maybe`, collapsible,
      per-card "parked Nd ago" aging); **count badge** in the header **and** on the AppShell nav tab
      (shares the `['review','pending']` cache — no double fetch). **Chat** (`ChatScreen`): per-session
      **"Remember now"** → `POST …/remember` with an inline `{endorsed,to_review}`/skip summary; a
      chat-scoped **"recently auto-recorded"** panel (`AutoRecordedList`, `GET /chat/auto-recorded`)
      with salience + relative time, an "still organizing…" state (empty `node_paths`), and
      **optimistic one-tap remove** (`POST …/{capture_id}/remove` → 204, rollback on error). New
      `api/client` methods + wire types; `useReview`/`useChat` hooks; resolve/batch invalidate both
      review lists + `['types']`/`['node']`. `tsc --noEmit` + `eslint --max-warnings 0` + `vite build`
      green; cache-seeded browser smoke verified all four card kinds render, salience order, and the
      single-kind batch-bar switch. **Independent review APPROVE-WITH-MINORS — no must-fix** (fixed
      the setState-inside-updater in `toggleSelect`; logged follow-ups: eager dedup node fetches,
      `SalienceBadge`/`SaliencePill` duplication, batch-maybe-on-parked no-op, `batchNote` no
      auto-clear). Commit `c603cf5`, **not pushed**. Details in [08-logs/m6.md](08-logs/m6.md) task 7.
- [x] **Task 8 DONE → M6 (chat-distiller + review queue) ACCEPTED (2026-07-16)** — **maybe-digest**
      weekly job + the M6 jobs wired into the [ADR-047](adr/047-pipeline-scheduling-primitive.md)
      pipelines. **maybe-digest** (ADR-048 §8): a new `MaybeDigestService` emits a **feed-visible
      `agent_run`** summarizing the parked `maybe` items (`{total, by_kind, oldest_age_days}`; no push
      — M10) over a new `PgReviewQueue.maybe_kind_stats` aggregate (`GROUP BY status='maybe'`, rule 10);
      `maybe-digest` CLI verb. **Wiring**: `config.pipeline_defs()` nightly roster = chat-distiller →
      data-sync → db-backup → inbox-drain → reindex → profile-refresh → entity-backfill →
      identity-capsule-refresh → dedup-sweep → store-sweep → store-backup (04-pipelines §Scheduling
      order); weekly = integrity-drill → maybe-digest; `PipelineScheduler` gains the four optional
      steps (dropped-with-log if unwired, ADR-047 §5), wired in both `main.py` + the run-now CLI (the
      distiller/inbox-drain share one capture pipeline; run-now drains + flushes only when a
      capture-touching step ran). **719 unit tests** (+4) + **real-PG smoke 121/121** (+4), ruff clean;
      the `maybe-digest` CLI verb ran clean against local pg. **Independent review APPROVE-WITH-MINORS —
      no must-fix** (2 minors fixed: conditional run-now drain, import tidy; 3 logged: distiller
      fast-ack intra-night lag, age-from-filed-time, unused `settings`). Commits `0bf5312`/`faf1afd`.
      Details in [08-logs/m6.md](08-logs/m6.md) task 8.
      **DEPLOYED + Accept review APPROVE (2026-07-16).** The M6 range was pushed; the first push
      (`faf1afd`) **failed CI** on `ruff check` (E501 in migration 013's docstring), gating the
      deploy. Fixed + **hardened `.githooks/pre-commit` into a lint/format gate** (ruff + eslint on
      staged files, so a lint error can't reach `main` again) — commit `16eb2bd`; whole-repo ruff
      clean + 719 tests green, re-pushed. CI green → deploy ran `alembic upgrade head` (the full range
      carries **migrations 013/014** from tasks 1/4 — additive, up/down verified); `/api/v1/health`
      all-true and the M6 route surface is **live** (`chat/auto-recorded` 404→401; all three M6 write
      routes 401). **Independent M6 Accept review APPROVE — no must-fix** (all 6 Accept criteria +
      task-8 wiring mapped to `file:line`; invariants 2b/2/6/7/5/9 hold; minors cosmetic/deferred).
      **Remaining to CLOSE (user-driven — VPS shell + authenticated PWA):** a VPS `pipeline
      nightly|weekly` run-now + the PWA behavioral loop (auto-endorse → recently-recorded → one-tap
      remove; pure-retrieval skip; stance-unclear agree-only; maybe park; P10 reprocess).

**M6 follow-up (step-status fidelity) — GRILLED TO BUILD-READY (2026-07-16 →
[ADR-050](adr/050-pipeline-step-status-is-the-jobs-own-run.md)).** The live M6 Accept `pipeline
nightly` run-now surfaced **`chat-distiller`** and **`inbox-drain`** reported **failed** — traced (grilled
decision-by-decision) to a status-fidelity defect, **not** data loss: a capture that degrades to the
`inbox/` fallback closes its organize run `failed` (rule 7), and because `child_run_scope` flattens
that nested `agent="capture"` run into the enclosing step, `_step_status` (any non-`{succeeded,skipped}`
child → step failed) marked the step failed even though the distiller's/drainer's **own** runs closed
`succeeded` and P10 held. Fix (ADR-050): **a step's status = its own job run** (the child whose
`agent == step.name`); nested spawned runs stay parented + feed-visible but don't gate the step; the
inbox-fallback run keeps `status=failed` (fix is step-rollup only); `raised → failed` and `halt` on the
step's own failure unchanged; `store-sweep` (no own run) still `skipped`. Server-only, **no migration**.
- [x] **Task 1 DONE → M6 CLOSED (2026-07-16)** —
      `pipeline._step_status` keys on `agent == step.name` (own-run status; nested `capture` runs
      non-gating but visible; own `child_run_id` reported). **721 tests** (+2 ADR-050 regression: nested
      `capture` failed → step succeeded + no halt; own failed/raised → step failed + halt aborts; nested
      run stays parented/visible), whole-repo ruff + format clean. **Independent review APPROVE — no
      must-fix** (own-run gate can't mask a genuine failure nor be fooled by nested/own ordering; all 8
      ADR-047 behavior tests hold; `step.name == agent` invariant verified across every wired step; 2
      non-blocking minors logged). Commit `1c36e06` (on the `8e1c376` ruff-format sweep); **pushed**
      (`16eb2bd..1c36e06`, CI deploying — server-only, **no migration**); prod `/health` all-true through
      the deploy window. Details in [08-logs/m6.md](08-logs/m6.md) "M6 follow-up". **Live Accept PASSED
      (2026-07-16):** the user re-ran `pipeline nightly` on prod → parent `1f5099ef`, **11 steps, 10
      succeeded, 0 failed, 1 skipped** — **`inbox-drain` → succeeded** while still reporting "0/2 resolved"
      (its 2 unresolved captures' nested `capture` runs stayed visible-as-failed, the exact case that read
      `failed` pre-fix), **`chat-distiller` → succeeded**, only `store-sweep` skipped (by design). The user
      exercised part of the PWA behavioral loop and confirmed the remaining surfaces present, and
      **accepted M6 as closed** → **M6 CLOSED**.
- **Out of scope (logged, separate):** the organizer's inbox-fallback rate on chat-distilled
  claim-text (2/4 this run — the drainer retries); the `claude` Max CLI 300s hang before Nebius
  fallback ([ADR-044](adr/044-provider-observability-surface.md) Providers card).

**M6 addendum — RATIFIED 2026-07-13, then RESOLVED at the M6 kickoff grill (2026-07-16 →
[ADR-048](adr/048-m6-chat-distiller-build-decisions.md)).** The kickoff took **(a)–(d) into scope
and deferred (e)**; it also **overrode two of the ADR-032 extensions** for the M6 context:
**`augment` is deferred** (plain merge/keep/link ship first), and **salience for review triage is a
coarse LLM tag**, not the graph-degree/pins/edge-conf formula (which was written for *node*
re-split and doesn't fit a not-yet-a-node candidate; **no pin feature exists**). Original text kept
for provenance:**
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

**Chat/retrieval:** **graph-aware retrieval lite** — flat **1-hop canonical-edge neighbor injection
(`{rel,title,type}`) + entity-seeded expansion** (deferred from M4; the addendum-(a) work, PPR-swappable
behind one function) → seeded **Personalized PageRank** expansion (SPRIG GraphRRF — replaces the flat
union behind the same function; needs hub down-weighting) → agentic traversal (chat model gets
`search`/`get_node`/`traverse` tools; only if M5 MCP usage proves one-shot fails —
[ADR-032](adr/032-prior-art-adoptions.md)) · **challenge mode** (a chat preset arguing against an idea
from the user's own cited history — ADR-033 #4, deferred from M4) · **bitemporal `as_of`** (edge
`since`/`until` validity reconstruction; M4 shipped only simple node-date `as_of`) · cross-encoder rerank
(rejected for the hot path; revisit only if RRF ordering demonstrably fails) · session rename/delete ·
true token streaming (streaming provider interface + SSE).
**Graph:** node editing in web · undo a manual ingestion (soft-delete via `git rm`,
`captures.node_paths`) · entity extraction beyond person/idea.
**Data enrichment & correction** (raised 2026-07-16 during the M6 task-5 grill; own planning
session before build): a family of user-driven flows to enrich/correct what the organizer
extracted, framed as night-time "sleep-cycle" work + review-queue items. First concrete instance —
an **`occurred-enrichment` review kind**: surface undated / coarsely-dated nodes and ask the user to
tag the event time in **natural language** ("summer 2019", "March 2024", "last Tuesday ~6pm"),
LLM/date-lib parsed into an `occurred` range at the right *granularity*; **this also directly
upgrades the dedup occurred-signal** ([ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md) §3,
which is weak precisely because `occurred` is sparse). ⚠ Sub-day precision needs a schema/contract
decision — `nodes.occurred_start`/`occurred_end` are `date`, not `timestamptz` (grill at kickoff).
The broader theme: correcting mis-extracted facts, editing node body/title, fixing
dates/planes/tags/edges, re-stancing a memory, and entity mis-split/merge fixes — the correction
counterpart to the M6 auto-record/remove trust loop.
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
