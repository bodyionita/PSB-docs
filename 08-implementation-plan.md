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

## M5 — MCP server ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md))

**Scope.** Token-authenticated MCP server on the VPS exposing the service layer: `search`
(+temporal filters), `get_node`, `traverse` (center+depth+rel, **cursor-paginated**),
`build_context` (get_node+traverse in one round-trip — [ADR-032](adr/032-prior-art-adoptions.md)),
`list_planes`/`list_types`, `capture` (full organizer pipeline, `source: mcp`, burst-queued).
`build_context` level-0 serves the **identity capsule** ([ADR-033](adr/033-external-inspirations-obsidian-second-brain.md))
— **building the capsule itself (the derived ~200-token "who I am" preamble + its sleep-cycle refresh)
is deferred here from M4** (M4 kickoff grill); M4 chat consumes it once M5 lands it, and it also feeds
the M4 chat system prompt at that point. No logic of its own; smallest milestone, biggest compounding effect (external LLMs start
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
