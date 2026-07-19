# Status history (archived from README.md — 2026-07-18)

> The append-only, session-by-session status log that used to live at the top of the README.
> **Append-only:** each session's superseded README snapshot entry moves here verbatim.
> Authoritative build state: [08-implementation-plan.md](../08-implementation-plan.md) + the per-milestone logs in this folder.

**Status (2026-07-13).** **THE MIND-GRAPH PIVOT approved (2026-07-13, grilled decision-by-decision)**
— the vision is reframed as a **typed mind graph**: Obsidian removed entirely, the note vault
becomes a **graph store** of typed nodes (`memory`/`person`/`idea`/`conversation`/`insight`) with
typed edges, governed vocabulary growth, MCP as a peer surface (query + store), conversational
ingestion with a stance gate + review queue, a visual map, and an ops-console/observability
contract — **[ADR-026](../adr/026-graph-native-storage-obsidian-removed.md) ·
[027](../adr/027-typed-vocabulary-governance.md) · [028](../adr/028-one-service-layer-mcp-peer-surface.md) ·
[029](../adr/029-conversational-ingestion-stance-gate-review-queue.md)**; new roadmap **M3–M11** in
[08](../08-implementation-plan.md). **M0 / M1 / M2 ACCEPT COMPLETE** on the pre-pivot note model —
deployed live at `https://braindan.cc` (capture → organize → index/search, full ADR-014
durability); that system stays live until M3 lands (fresh start: the old vault is archived, no
data migration). The previously grilled chat plan ([ADR-025](../adr/025-ui-editable-model-routing-and-per-task-effort.md))
is carried intact to **M4**, retargeted to nodes.
**ADR-033 (2026-07-13): `obsidian-second-brain` review adopted in full** — identity capsule,
contradiction sweep, freshness stamps + staleness interviews, reflection enrichments (emerge
taxonomy/challenge/belief timeline), graph-health, research-via-MCP pattern, **Telegram capture
promoted into M9** (pull-forward eligible); storage-model ideas explicitly rejected (safeguarded
in the ADR). **ADR-034: round-2 repo review** — **evidence-tiered profiles** (stub/snapshot/full
by graph degree) into the M3 profile job; ecosystem references saved; the rest skipped.
**Prior-art research pass adopted in full (2026-07-13, [ADR-032](../adr/032-prior-art-adoptions.md))** —
field survey validated the design (several areas ahead of SOTA); adoptions: edge `until`,
exact-alias short-circuit, observation-style profiles, day/night effort, RRF hybrid retrieval +
guards, MCP pagination/`build_context`, plus M6/M7/backlog refinements and explicit rejections.
**M3 GRILLED TO BUILD-READY (2026-07-13 — [ADR-030](../adr/030-entity-substrate-and-lifecycle.md)/[031](../adr/031-m3-organizer-and-contract-extensions.md)):**
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
minors fixed) — [08-logs/m3.md](../08-logs/m3.md).
**Task 7a done (2026-07-14):** vocabulary governance core — `PgVocabularyStore` over `app_settings`,
an `EffectiveVocabulary` provider (config seeds ∪ approved additions) threaded into every writer so an
approved type is **forward-live**, `GET /types`, `PUT /settings/vocabulary`, the vocab-proposal branch
delegated to one `VocabularyService` (shared with `POST /review/{id}`), and a feed-visible
`vocab-consolidation` run on approve (replaces task 4's SKIPPED marker). Scope split by
**[ADR-035](../adr/035-vocabulary-consolidation-scope-m3.md)** (edges apply / nodes propose-only). 318
tests green (+24), ruff clean, **independent review APPROVE — no must-fix** (2 minors fixed) —
[08-logs/m3.md](../08-logs/m3.md) task 7a; commits `dd3c5be`/`410b5d2`.
**Task 7b done (2026-07-14):** edge retro-consolidation apply — `POST /admin/vocab/consolidate`, an
on-demand two-step (ADR-024 shape) that **re-types existing edges** onto a newly-approved rel:
`NodeWriter.retype_edge` (rel-only frontmatter rewrite, `conf`/`since`/`until` preserved, field-boundary
match, dedup-safe), a bounded recency-ordered edge inventory (`PgEdgeConsolidationStore`), and a
`vocab-consolidation` background apply (rewrite → reindex → force-commit, skip-and-continue). Scope
pinned by **[ADR-036](../adr/036-edge-retro-consolidation-walk-retypings-only-m3.md)** (re-typings only;
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
**independent review APPROVE-WITH-MINORS — no must-fix** (3 minors fixed) — [08-logs/m3.md](../08-logs/m3.md)
task 8. Out of scope (M8/admin, endpoints exist but no web surface): edge-consolidation + entity-merge UIs.
**Task 9 done (2026-07-14):** deploy/CI retarget vault→graph store — `GRAPH_STORE_PATH=/srv/graph-store`
+ `GRAPH_STORE_REPO` in `defaults.env`/`.env.example`, `/srv/graph-store` mount + `graph_store_deploy_key`
bind in compose, matching `docker-entrypoint.sh`/`Dockerfile` (`safe.directory` — a functional miss)
/`provision.sh` (two-pass hardening guard intact), and a deploy-workflow step printing the VPS deploy
**public** key into the Actions log (best-effort; private key never leaves the box). App bootstrap still
owns remote-wire + `push -u` (ADR-031 §6); no VPS git steps. YAML + `sh -n` clean; **independent review
APPROVE — no must-fix** — [08-logs/m3.md](../08-logs/m3.md) task 9; commit `e97671b`.
**Task 10 in progress (2026-07-14): live M3 Accept.** Green baseline (344 tests) + carried backend
smokes closed against **real local pgvector**: migration 006 applied; a real-DB SQL smoke
(`server/scripts/smoke_db.py`) drives the actual task-6/7a/7b `Pg*` stores — **23/23 green**. The
smoke surfaced a real gap → replanned: **`profile-embedding-in-search` was an unbuilt
[ADR-030](../adr/030-entity-substrate-and-lifecycle.md) §4 requirement** (search was chunks-only; the
stored profile vector never queried). Mechanism pinned by **[ADR-037](../adr/037-profile-embedding-in-search-m3.md)**
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
lifted to **vision P10**. Grilled decision-by-decision → **[ADR-038](../adr/038-reorganize-preserves-shared-entity-hubs.md)**
(hubs shared, never deleted by reorganize) · **[039](../adr/039-entity-types-are-mention-only.md)** (entity
types mention-only + coercion guard) · **[040](../adr/040-token-overlap-retrieval-and-alias-accretion.md)**
(token-overlap retrieval + alias accretion) · **[041](../adr/041-diacritic-folding-derived-content.md)**
(fold diacritics on all derived content, raw kept) · **[042](../adr/042-reprocess-all-from-raw-and-data-survival.md)**
(reusable `reprocess-all-from-raw` op + the data-survival principle). New **task 11** in [08](../08-implementation-plan.md)
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
Follow-up logged: reprocess/reindex leave `node_profiles` empty until the nightly job. **`PSB-vault`
archived by the user (2026-07-14) → M3 CLOSED** — all acceptance criteria pass; the graph-native stack is
live at `braindan.cc` ([08-logs/m3.md](../08-logs/m3.md) "Task 10/11").
**M4 (chat) GRILLED TO BUILD-READY (2026-07-14 — planning session, decision-by-decision).** Resolved to
a **lean spine + explicit deferrals**: build the chat loop + **hybrid vector+FTS RRF** retrieval
(migration 008, FTS mirrors the ADR-037 chunks⊍profiles union) + recency prior + `since`/`until`/simple
`as_of` + English condensation + **fenced** grounded prompt + cited-only `[n]`; **3 UI-editable routing
groups** (`chat`/`conspect`/`quick`) with all 6 conspect call-sites rewired + per-call effort; new
**[ADR-043](../adr/043-quick-routing-tier-m4.md)** adds the **`quick` tier** (Sonnet-4.6-low / Nebius) for
trivial tasks (M4's only caller = LLM session titling). **Deferred:** graph-aware expansion (1-hop
injection + entity-seeded) → backlog, **identity capsule → M5** (build_context owns it), challenge mode →
backlog. Doc reconciliations recorded (08 M4 kickoff note + task list; 04-pipelines condensation-effort +
retrieval; 03/02/06 for 3 groups + migration 008). **Paused before implementation** per
[09](../09-session-protocol.md) — 8 M4 tasks open in [08](../08-implementation-plan.md). **No code this session.**
**M4 task 1 DONE (2026-07-14):** model routing engine — `ModelRoutingService` over the 3 groups
(`chat`/`conspect`/`quick`, [ADR-043](../adr/043-quick-routing-tier-m4.md)) reading config seeds overlaid
with `app_settings.model_routing` (cache-bust on save, rule-7 degrade to seed); **per-call effort** through
the provider boundary (`claude-max` honors `--effort`, Nebius ignores); a distinct **`claude-max-sonnet`**
provider id (same CLI, config model, `quick` seed = sonnet@low / nebius); and **all 6 `conspect` call sites
rewired** through the service (organize, nudge, entity resolution, profile gen, tag + edge consolidation).
No migration (routing is `app_settings` jsonb). ruff clean, **396 tests green** (+25); **independent review
APPROVE — no must-fix** (3 minors fixed) — [08-logs/m4.md](../08-logs/m4.md) task 1. **Code committed locally,
not pushed.** Next: M4 task 2 (retrieval — migration 008 + hybrid RRF).
**M4 task 2 DONE (2026-07-14):** retrieval — **migration 008** (a generated `tsvector` on `chunks` **and**
`node_profiles` + GIN, mirroring the ADR-037 vector union) and a rewritten hybrid `search_chunks`:
vector (cosine) ⊍ FTS (`websearch_to_tsquery('english', …)`) legs, each best-per-node + candidate-pool-
bounded, fused by **RRF** (rank-based, `k=60` — never blends cosine with `ts_rank`, [ADR-032](../adr/032-prior-art-adoptions.md)
§5), then a **mild recency prior** (bounded `[floor,1]` multiplicative on `occurred ?? created`, §7). The FTS
leg **self-suppresses** on non-English / zero-lexeme queries (no language-detect dep). New temporal filters on
`/search`: `since`/`until` occurred-window + simple node-date `as_of`. Config knobs (`rrf_k`/`candidates`/
`recency_*`, Rule-9); `SearchResultItem` shape unchanged. ruff clean, **400 tests green** (+4) + **12 new
real-PG16 smoke checks** (hybrid SQL isn't fake-testable); **independent review APPROVE-WITH-MINORS — no
must-fix** (3 minors fixed: half-life div-by-zero guard, UTC-pinned recency date, deterministic rank tiebreak;
1 logged for task 3 — tune the chat `min_score` floor to the RRF scale) — [08-logs/m4.md](../08-logs/m4.md) task 2.
**Code committed locally, not pushed.** Next: M4 task 3 (chat service).
**M4 task 3 DONE (2026-07-14):** chat service (service layer only; routers = task 4) — new `app/chat/`
package: `ChatService.send` **persists the user turn before any model call** (never-lose) → turn-1 raw /
**turn-≥2 English condensation on `conspect`** (degrade-to-raw when down) → **hybrid RRF retrieval** via the
existing `SearchService` → **fenced** grounded prompt (data-not-instructions) → **cited-only `[n]` renumber**
(`citations.py`, out-of-range dropped never-errors) → persist assistant turn (`model` incl. fallback +
`sources` jsonb, `PgChatStore`) → **best-effort non-blocking `quick`-tier titling** after the first exchange
(drained on shutdown). **MINOR-1 resolved:** the chat "not in your memories" floor (`chat_retrieval_min_score`
= 0.01) is floored on the **fused RRF×recency** scale, **not** cosine — `SearchService.search` gained a
keyword-only `min_score` override (default keeps `/search` unchanged). ruff clean, **423 tests green** (+23)
+ **7 real-PG chat-store smoke checks**; **independent review APPROVE — no must-fix** (1 minor fixed —
`_clean_title` tests; 2 logged) — [08-logs/m4.md](../08-logs/m4.md) task 3; commits `6406bd9`/`0559799`.
**Code committed locally, not pushed.** Next: M4 task 4 (chat routers).
**M4 task 4 DONE (2026-07-14):** chat routers (`app/routers/chat.py`, thin over the task-3 `ChatService`) —
**`POST /chat`** (`RegistryExhausted`→**503** with the user turn still persisted, `ChatSessionNotFound`→**404**;
picker/`planes`/`top_k` forwarded, `model_used`/`fallback_used` surfaced), **`GET /chat/models`** (a new
`ModelRoutingService.chat_catalog` over `registry.chat_models()`; `default` = the Chat group's active model),
**`GET /chat/sessions[/{id}]`** (read paths added to `ChatService`, bounded by a new `chat_sessions_list_limit`).
Added a provider **`can_chat`**+`label` seam (the OpenAI-compatible class also backs STT/embedding — so
`chat_models()` filters on real capability, not the `ChatProvider` class) and **boundary uuid validation** on
`session_id` (body + path → 422 on malformed, so the uuid column never 500s). ruff clean, **439 tests green**
(+16); **independent review APPROVE after fixes** — 2 must-fix (STT/embedding providers leaking into the picker;
malformed-session 500), **both resolved + regression-tested** — [08-logs/m4.md](../08-logs/m4.md) task 4; commit
`5059570`. **Code committed locally, not pushed.** Next: M4 task 5 (settings routers).
**M4 task 5 DONE (2026-07-14):** settings routers — **`GET /settings`** (all 3 routing groups `chat`/`conspect`/
`quick` with effective saved-over-seed routing + the registry's pickable models, each carrying `supports_effort`
+ **provider-sourced `effort_levels`** — no hardcoded lists in web) and **`PUT /settings/models`** (saves one
group, busts the routing cache **forward-live**; unknown active/fallback id, effort on a non-effort model, bad
level, or bad group → **422** before any `app_settings` write). Added a `ModelRoutingService.save_group`/
`all_settings` + `GroupSettings`/`UnknownModel`, a `ChatProvider.effort_levels` seam, and extended
`ChatModelOption` (defaulted, so `GET /chat/models` is unchanged). ruff clean, **449 tests green** (+12);
**independent review APPROVE — no must-fix** (2 minors: flagged test edges added; a deliberate ADR-025-accepted
display-verbatim choice logged) — [08-logs/m4.md](../08-logs/m4.md) task 5; commit `bb6857a`. **Code committed
locally, not pushed.** Next: M4 task 6 (web chat screen).
**M4 task 6 DONE (2026-07-14):** web chat screen — the PWA's Chat tab, a thin TanStack-Query client over the
task-4/5 API (03-api §Chat; no server code, ADR-006): implicit-session chat loop with **list/open/new** (a
`seededFor`-guarded one-time thread seed so a send's optimistic turns — incl. a just-created session — survive
the detail refetch), a composer with a **per-conversation model picker** (`GET /chat/models`, resets to the
Chat-group default per thread) + **plane chips** + auto-grow textarea + **Enter-to-send**, a **reduced-motion-
gated client-side reveal** (word+`[n]`-badge stagger, fresh turns only) whose citation clicks expand +
scroll to **cited-only source cards**, a **"not from your memories"** chip on empty `sources`, and an
**"answered by `<model>`"** banner on `fallback_used`. Titles land via a **bounded session-list poll**. Search's
node preview was extracted into a shared **`ui/NodePreview`** + **`ui/nodeDetail`** primitive so the chat source
cards reuse it (rule 10). tsc/eslint/vite green; a **real-browser walkthrough vs a throwaway mock API** drove
every surface (grounded/ungrounded, citation→preview expand, model switch, seeding, titling — console clean);
**independent review APPROVE-WITH-MINORS — no must-fix** (2 fixed: citation-adjacent space drop, reduced-motion
on the thinking dots/entrances; 1 deferred — the sessions-list item carries no first-message text, a server-
contract follow-up) — [08-logs/m4.md](../08-logs/m4.md) task 6; commits `fea9f9d`/`85be897`. **Code committed
locally, not pushed.** Next: M4 task 7 (web Settings → Models panel).
**M4 task 7 DONE (2026-07-14):** web Settings → Models panel — the PWA's Settings **Models** section, a thin
TanStack-Query client over the task-5 API (03-api §Settings; no server code, ADR-006), replacing the `ComingSoon`
placeholder: one card per **routing group** (`chat`/`conspect`/`quick`) with an **active** + **fallback** dropdown
(fallback carries a **None** = `""`) and a **segmented effort selector rendered only for the selected
effort-supporting models** in `{active, fallback}` — **all choices + effort levels registry-sourced** from
`GET /settings`, **nothing hardcoded** (ADR-006 / ADR-025 §6). Save `PUT`s **one group** with a **normalized**
`effort_by_provider` (a valid level for exactly the effort-capable selected models, stale keys dropped → never a
server 422), is **dirty-gated**, and is **forward-live** (invalidates `['settings']` **and** `['chat','models']`
so the composer default follows); a `signature`-ref re-seed applies server changes without clobbering in-progress
edits. tsc/eslint/vite green; a **real-browser walkthrough vs a throwaway mock API** drove every surface (3 groups
seed-correct, a second effort control on a dual-Claude group, a Chat@high save persisted + reflected forward-live,
a Nebius active dropping its effort control + saving an empty payload with a None fallback — console clean);
**independent review APPROVE — no must-fix** (1 minor fixed: reduced-motion on the "Saved" slide; 1 deferred —
cosmetic "Saved" re-show on manual revert) — [08-logs/m4.md](../08-logs/m4.md) task 7. **Code committed locally, not
pushed.** Next: M4 task 8 (live M4 Accept).
**M4 task 8 DONE → M4 (chat) ACCEPT COMPLETE (2026-07-14).** Pushed the 9 M4 commits to prod (CI green, migration
008 applied, `/api/v1/health` all-true) and ran the live Accept at `braindan.cc`: grounded `[n]`-cited chat on
**both Claude** (`claude-opus-4-8`, 4 source cards) **and Nebius**, "not in your memories" on ungrounded questions,
sessions persist with LLM titles, **clause (A)** Settings-driven **Nebius-primary drive recorded** on
`chat_messages.model=nebius` (verified via authenticated API — the mobile PWA's bottom-nav tap was flaky), **clause
(B)** true fallback shown live, 21 registry tests green. A **local in-process dry-run** (real app + local pgvector,
auth gate overridden) preceded the push and verified the Claude leg, persistence, not-in-memories, fallback
mechanism, and 503-on-exhaustion — no code defects. The prod Accept **surfaced + fixed a real config defect**: the
committed `NEBIUS_CHAT_MODEL` was `meta-llama/Meta-Llama-3.1-70B-Instruct` — a HuggingFace-style id **Nebius never
served** (it uses `Llama-3.1-...`, since retired for **Llama 3.3**), so **every** live Nebius call silently fell
back to claude-max (never caught pre-M4: M0/M3 made no live Nebius call, the 21 tests use fakes). Corrected to
**`meta-llama/Llama-3.3-70B-Instruct`** in all four committed spots (commit `419ece4`), redeployed, and re-verified
Nebius answers grounded + recorded. Prod Chat routing restored to the seed default (Claude/opus primary, Nebius
fallback). Grounded cited chat over the graph is **live at `braindan.cc`** — [08-logs/m4.md](../08-logs/m4.md) task 8.
**Follow-ups (out of M4):** UI "Claude Max"→"Claude"+effort-label relabel (needs a planning pass); optional
provider-observability surface (a provider silently fell back with no visible reason — rule 7 / vision P8).
**Code pushed through `419ece4`.** Next: **M5 (MCP server)** — planning session (`/grilling` first).
**M4 follow-up 1 (relabel) DONE (2026-07-15):** friendly model names ("Claude Opus 4.8"/"Claude Sonnet 4.6"/
"Llama 3.3 70B") + effort labels on the chat picker + per-message "answered by" caption; server-derived labels
(rule 9), 460 tests green, independent review APPROVE, **pushed to prod** (`ba07eb7`/`753eff5`) — [08-logs/m4.md](../08-logs/m4.md).
**M4 follow-up 2 (provider observability) GRILLED TO BUILD-READY (2026-07-15 — [ADR-044](../adr/044-provider-observability-surface.md)):**
in-memory per-provider status on the registry (no migration; sticky `last_error` + `last_success_at` +
`consecutive_failures`, recorded at chat/STT/embedding call sites) behind a session-gated **`GET /admin/providers`**
(live `Provider.health()` reachability probe — *not* a success guarantee; `/health` untouched) + a read-only
Settings **Providers** card. Closes the P8/rule-7 silent-fallback gap. **2 tasks** open in [08 §M4 follow-up](../08-implementation-plan.md)
(server; web + live Accept). **Paused before implementation — no code this session.** Next: build task 1, or **M5** planning.
**M4 follow-up 2 · Task 1 (server) DONE (2026-07-15):** in-memory `ProviderStatusTracker` on the registry
(sticky `last_error` + `last_success_at` + `consecutive_failures`, injectable clock) wired at **all three**
call sites — chat, STT, and the previously-unrecorded no-fallback **embedding** leg (ADR-044's key blind
spot; records then re-raises, nothing swallowed) — plus session-gated **`GET /admin/providers`** (live
concurrent `health()` probe, defensive against a raising probe) with `capabilities` derived from
configuration (`can_chat`/`can_transcribe`/`can_embed`), not the class hierarchy. `/health` untouched, no
migration. 476 tests green (+16), ruff clean, **independent review APPROVE-WITH-MINORS — no must-fix** (the
one worthwhile minor, an HTTP-level endpoint test, was added) — [08-logs/m4.md](../08-logs/m4.md). **Code
committed locally (`9dad941`), not pushed.** Next: **Task 2** (web Providers card + live Accept), or **M5** planning.
**M4 follow-up 2 · Task 2 (web) DONE + mock-verified; live prod Accept pending a prod push (2026-07-15):**
read-only Settings **Providers** card — a thin TanStack read (`useProviders`, 15s poll) over `GET /admin/providers`
with a status dot (green `consecutive_failures == 0` / amber `> 0`), the sticky `last_error` line (muted once
recovered), capability chips, an "unreachable" tag, and last-success time; wire types exact-match the server
Pydantic models; `relativeTime` hoisted to a shared `ui/` helper. No server code (ADR-006). tsc/eslint/build
green; a **real-browser walkthrough vs a throwaway mock API** drove every render state **and a forced-failure→
recovery transition** (amber→green, failures cleared, `last_error` stays sticky+muted). **Independent review
APPROVE-WITH-MINORS — no must-fix** (3 minors all fixed). Commit `a82500b`. **DEPLOYED to prod + live Accept VERIFIED
→ Task 2 DONE → M4 follow-up 2 (provider observability) COMPLETE (2026-07-15):** pushed `9dad941` + `a82500b`
(CI run `29390674935` all green); the real `braindan.cc` Settings → **Providers** card (read in the browser pane
against the live endpoint, existing prod session — agent never handled the login secret) **enumerates all 6
registered providers read-only** (claude-max/-sonnet + nebius = Chat, openai/groq = Speech, ollama = Embedding),
green dots, config-derived capabilities, **0 interactive controls**; all "No successful call yet" (correct
post-redeploy zero-state). **Both M4 follow-ups now complete; graph-native chat + observability live at
`braindan.cc`** — [08-logs/m4.md](../08-logs/m4.md).
**M4 follow-up 3 (provider/model/effort separation) GRILLED TO BUILD-READY (2026-07-15 — [ADR-045](../adr/045-provider-model-effort-separation.md)):**
the code conflated **provider** with **model** (two fake `claude-max`/`claude-max-sonnet` ids over one
`claude` CLI → duplicate Providers-card rows + a `claude-max` id leaking into the UI). Grilled
decision-by-decision to make **provider/model/effort** first-class: the **routable unit is a model id**
(provider derived; Option A), a **model id = the raw vendor string**, `claude` collapses to **one provider
serving Opus+Sonnet** via per-call `--model`, concept fixed across all providers while the picker stays
**chat-only**, config uses **named scalars** (`CLAUDE_OPUS_MODEL`/`CLAUDE_SONNET_MODEL`/`CLAUDE_EFFORT`),
saved routing **migrated in place** (idempotent Alembic) while historical `chat_messages.model` is **left +
legacy-tolerated** (P10), `effort_by_provider`→**`effort_by_model`**, and the Providers card renders **one
row per provider** (friendly name, no raw id). **Supersedes** the routing-unit of [ADR-025](../adr/025-ui-editable-model-routing-and-per-task-effort.md)
+ the `claude-max-sonnet` mechanism of [ADR-043](../adr/043-quick-routing-tier-m4.md); **refines** [ADR-004](../adr/004-provider-registry-claude-primary-nebius-fallback.md).
**5 tasks** open in [08 §M4 follow-up 3](../08-implementation-plan.md) (server core · migration · server API ·
web · live Accept). **Paused before implementation** per [09](../09-session-protocol.md) — **no code this session.**
Next: build task 1 (server core), or **M5 (MCP server)** planning (`/grilling` first).
**M4 follow-up 3 · Task 1 (server core) DONE (2026-07-15):** the internal provider ≠ model engine — the two fake
single-model provider ids that shared the one `claude` CLI collapse into **one `claude` provider serving Opus +
Sonnet** via per-call `--model`; the routable unit is now a **model id** (raw vendor string, provider derived).
Config named scalars (`CLAUDE_OPUS_MODEL`/`CLAUDE_SONNET_MODEL`/`CLAUDE_EFFORT`) + model-string seed chains;
`ClaudeMaxProvider`→**`ClaudeProvider`** (`claude_max.py`→`claude.py`); registry gains a chat-model **catalog** +
**model→provider index** (routes by model id, runtime status keyed by **provider id** — a within-`claude` fallback
is one provider event, ADR-045 §6); **`effort_by_provider`→`effort_by_model`** through the service, jsonb, and wire;
`build_registry` = **5 providers**. Model labels are derived per model id (`labels.py`, rule 9); a new
`provider_label` carries the friendly provider name for the Providers card. **Deliberate behavior change (ADR-045
§5):** the `quick` group's effort seed is now `claude_effort` (medium), not the old per-tier low — the tier's
cheapness comes from the Sonnet **model** now; retunable in Settings. 478 tests green (+ conftest that isolates the
Claude Code shell's colliding `CLAUDE_EFFORT`; no prod risk), ruff clean, **independent review APPROVE — no
must-fix** ([08-logs/m4.md](../08-logs/m4.md) "follow-up 3 · Task 1"); commit `7c69449`, **not pushed**. **⚠ Do not
deploy before task 2 (migration)** — pre-existing saved `model_routing` would degrade to seed until it lands.
Next: **Task 2** (idempotent Alembic migration of saved routing + legacy-tolerant `chat_messages.model` labels).
**M4 follow-up 3 · Task 2 DONE (2026-07-15):** the P10-load-bearing migration — **Alembic revision 009**, an
idempotent plain-SQL data migration ([ADR-045](../adr/045-provider-model-effort-separation.md) §4) that rewrites the
saved `app_settings.model_routing` row from old provider ids to model-id vendor strings (`claude-max`→
`claude-opus-4-8`, `claude-max-sonnet`→`claude-sonnet-4-6`, `nebius`→`meta-llama/Llama-3.3-70B-Instruct`) in
active/fallback + the effort object's keys **plus the `effort_by_provider`→`effort_by_model` key rename** — an
ordered text-substitution over a closed token set (the `-sonnet` prefix hazard handled), no-op on absent/empty/
already-migrated rows, downgrade reverses it. Legacy-tolerant `friendly_model_label` folds the retired ids to their
vendor string so historical `chat_messages.model` rows (**left untouched** — rewriting audit would falsify it) still
render a friendly name. 481 tests + ruff green; the un-fakeable migration SQL **verified against a throwaway
Postgres 16** (remap, key rename, idempotency, downgrade round-trip, empty/absent no-op, key-scoping); **independent
review APPROVE — no must-fix** (2 minors applied) — [08-logs/m4.md](../08-logs/m4.md) "follow-up 3 · Task 2"; commits
`6133296`/`74de20d`, **not pushed**. The task-1 deploy-ordering guard is now satisfiable (deploy is still **task 5**,
after task 3 API + task 4 web). Next: **Task 3** (server API response shapes — `/settings`, `/chat/models`,
`/admin/providers`: model-id semantics, `effort_by_model`, provider-name label, provider-only rows).
**M4 follow-up 3 · Task 3 DONE (2026-07-15):** server API response shapes — comparing all three endpoints' wire
against the binding [03-api](../03-api.md) contract, tasks 1/2 had already landed model-id semantics, `effort_by_model`,
the friendly `provider_label`, and the 5-provider collapse, leaving **one genuine gap**: `GET /settings` model
options didn't carry the contract-required `provider` (03-api §Settings). Added **`provider` to `RoutingModelItem`**,
sourced from the registry catalog (the *serving* provider id, derived — [ADR-045](../adr/045-provider-model-effort-separation.md)
§1; both Claude models → one `claude`). `/chat/models` (`{id,label,effort}`, line 52) and `/admin/providers` (one
provider-labelled row, line 107) were **verified against the contract as already-correct — no change** ("no raw id"
is qualified as *in the UI*, so dropping the card's id is task 4). 481 tests + ruff green; **independent review
APPROVE — no must-fix** (reviewer re-derived scope from 03-api+ADR-045, confirmed the two no-change endpoints) —
[08-logs/m4.md](../08-logs/m4.md) "follow-up 3 · Task 3"; commit `a28ab04`, **not pushed**. Next: **Task 4** (web —
types, ModelsPanel model-id picks, chat composer picker, ProvidersPanel provider-only/drop id).
**M4 follow-up 3 · Task 4 DONE (2026-07-15):** the web half of ADR-045 (4 PWA files, no server code) — wire types
renamed `effort_by_provider`→**`effort_by_model`** (`GroupRoutingModel`/`ModelRoutingUpdate`) + **`provider` added to
`RoutingModelItem`** (03-api §Settings), exact-matching the task-1/3 server Pydantic; **`ModelsPanel`** reads/seeds/
dirty-tracks/PUTs `effort_by_model` (the dropdowns were already model-id-valued with server-friendly labels, so no raw
id shows); **`ProvidersPanel`** **drops the rendered raw provider id** (friendly name only, one row per provider —
ADR-045 §6 "no raw id in the UI"; `id` kept as the React key); the **chat composer picker** is unchanged under the
label-only `{id,label,effort}` contract (only a stale comment refreshed). tsc/eslint/vite green; a **real-browser
walkthrough vs a throwaway mock API** drove all three surfaces — Models seeds `effort_by_model` + a save round-trips it,
Providers renders the **5 friendly rows** (one "Claude") with **no id**, the composer shows model-id options with
friendly labels and its default follows the saved Chat routing **forward-live** (console clean); **independent review
APPROVE — no must-fix**. Commit `fc193c0`, **not pushed**. Next: **Task 5** (live M4 follow-up 3 Accept — deploy the
5 commits `7c69449`→`fc193c0`; migration 009 applies; saved routing preserved; card shows 5 rows / one "Claude"; chat
still grounded → independent review → pause).
**M4 follow-up 3 · Task 5 DONE → M4 follow-up 3 (provider/model/effort separation) COMPLETE (2026-07-15).**
Pushed the 5 commits `7c69449`→`fc193c0`; CI run `29397900778` all green; **migration 009 applied** on prod
(`alembic upgrade head` under `set -e`); `/api/v1/health` all-true. All Accept criteria met — the registry exposes
**5 providers with one `claude`** serving Opus+Sonnet, the **Providers card shows one "Claude" row** (no raw id),
Settings → Models picks **model ids** with `effort_by_model`, saved `model_routing` was **migrated in place** (not
silently reset — prefix hazard handled), chat stays **grounded + cited on Opus/Sonnet/Llama**, and **no `claude-max`/
`claude_max` string remains** in code/config/UI except the deliberate legacy-fold map + migration test input. Live
PWA surfaces user-confirmed against the existing prod session (agent never handled the login secret); **independent
review APPROVE — no must-fix** (a fresh agent re-derived all 8 criteria from ADR-045 + 03-api and checked the
`a82500b..fc193c0` diff). Provider is now first-class and distinct from model; grounded chat + provider observability
live at `braindan.cc` — [08-logs/m4.md](../08-logs/m4.md) "follow-up 3 · Task 5". **Code pushed through `fc193c0`.**
**M5 (MCP server) GRILLED TO BUILD-READY (2026-07-15 — [ADR-046](../adr/046-m5-mcp-server-oauth-connectors.md), decision-by-decision).**
The "smallest milestone" grew a real spine: the connection surfaces the user wants — **mobile Claude app + claude.ai web
(custom connectors)** — require an **OAuth 2.1 flow**, not a static bearer, so M5 stands up a **self-hosted OAuth 2.1
authorization server** (`authlib`: `.well-known` discovery, open DCR, `/authorize` password+consent gate w/ PKCE, opaque
HMAC-hashed DB tokens ~1h + long-lived sliding refresh, **revoke-all**, single scope) — the same build unlocks Claude
Desktop + ChatGPT (Claude Code CLI deferred). Remote **`mcp` SDK Streamable HTTP under `/mcp`** on the existing `api`
container, single origin (Caddy root routes, Cloudflare un-cached). Tools are **thin over the service layer, markdown-rendered**
(IDs verbatim, cross-model; hub edge cap + `traverse` pointer): `search`/`get_node`/`traverse` (new `GraphService.neighbors`
one-hop primitive, cursor-paginated — M7 reuses it)/`build_context` (depth ≤ 2, **identity capsule L0**)/`list_planes|types`/
`capture` (`source: mcp`, burst-queued, fast ack). Plus an `initialize` **`instructions`** usage capsule + tool descriptions
+ annotations + a **research-via-MCP Prompt** (ADR-033 #6). The **identity capsule** (ADR-033 #1, refined — insights barely
exist at M5): broadened source (hubs + recent memories + insights), **300-token** nightly distill → `app_settings` blob,
served as `build_context` L0 + `identity://me` **and wired into the M4 chat system prompt in M5**. **Resolves the old "MCP
token distribution" open question** (it's the add-connector OAuth flow). Accept gate = a **real Claude connector** live
(revoke-all locks it out); **ChatGPT fast-follow before M6**. **6 tasks** open in [08 §M5](../08-implementation-plan.md).
**Paused before implementation** per [09](../09-session-protocol.md) — **no code this session.** Next: build **M5 task 1**
(traverse primitive + `build_context` core), or respawn.
**M5 task 1 DONE (2026-07-15):** the graph read side — `PgNeighborStore.neighbors` (one-hop, both-dir union, rel/direction
filters, **keyset pagination** on `(origin,rel,dir,node_id)`, tombstoned endpoints excluded both ends) + `GraphService`
`neighbors` (opaque base64 cursor, limit-clamped, `InvalidDirection`/`InvalidCursor`) and `build_context` (get_node via a
`NodeReader` seam over search + a bounded neighbor tree: **depth ≤2** = 03-api contract, per-node **fanout cap** + `truncated`
flag, path **cycle guard**; **L0 identity capsule deferred to task 2**). Thin over the `edges` table, no LLM call; shared by
MCP `traverse`/`build_context` (task 4) + the M7 map. Config knobs (rule 9), wired into `app.state` + a `get_graph_service`
dep. **499 tests green** (+16 unit) + real-PG edge-SQL smoke (un-fakeable union/keyset/tombstone; needs local pgvector up —
pre-push gate), ruff clean; **independent review APPROVE — no must-fix** (3 of 6 minors fixed) — [08-logs/m5.md](../08-logs/m5.md).
Commits `ff4a729`/`bb33ae7`, **not pushed.** Next: **M5 task 2** (identity capsule distiller → `build_context` L0 + `identity://me` + wire into the M4 chat system prompt).
**M5 task 2 DONE (2026-07-15):** identity capsule — new `app/identity/` package: `PgCapsuleSourceStore`
(top profile hubs by canonical graph degree + recent memories + recent insights, tombstone-excluded) +
`PgIdentityCapsuleStore` (`{text, generated_at, source_refs}` blob in `app_settings`, no migration, rule-1
rebuildable) + `IdentityCapsuleService` — one `conspect` distill of the **fenced** blended source into a
~300-token capsule; **best-effort (rule 7)** (no source / LLM down / empty **keeps the last capsule**);
**single-flight** nightly job **04:35** (after profile-refresh/backfill) + on-demand **`POST
/admin/identity-capsule/refresh`** (`202`/`409`) + CLI, each wrapped in `agent_runs`. Served as
`build_context` **L0** (cheap `IdentityCapsuleReader` seam, never distilled inline, best-effort) **and**
prepended to the M4 chat system prompt (fenced data-not-instructions) — in-app chat finally consumes it.
The `identity://me` MCP resource stays task 4. Config knobs (rule 9). Also **closed the task-1 pre-push
gate** (real local pgvector; fixed two smoke-harness fragilities — Windows UTF-8 + global-read isolation,
`8afbb25`). **524 unit tests green + smoke 63/63** (5 new real-PG capsule checks), ruff clean; **independent
review APPROVE-WITH-MINORS — no must-fix** (1 typing regression fixed `cb3c25a`; 3 logged) —
[08-logs/m5.md](../08-logs/m5.md) task 2. Commits `8afbb25`/`2ea5834`/`cb3c25a`, **not pushed.** Next: **M5
task 3** (OAuth 2.1 AS: `authlib` `.well-known`/DCR/`/authorize`+PKCE/`/token`+refresh, opaque HMAC DB
tokens, revoke-all + client/token migration).
**M5 task 3 DONE (2026-07-15):** the self-hosted **OAuth 2.1 authorization server** gating the MCP surface
([ADR-046](../adr/046-m5-mcp-server-oauth-connectors.md) §2) — new **`app/oauth/`** package (errors · store ·
metadata · consent · service) + a **root-mounted** `routers/oauth.py` (`/.well-known/oauth-*` discovery, open
DCR `/register`, the `GET+POST /authorize` password+consent gate, `/token`) + session-gated `POST
/admin/mcp/revoke-all` + **migration 010** (`mcp_oauth_clients`/`mcp_auth_codes`/`mcp_tokens`, all rebuildable
op-state). **authlib supplies only the security-critical crypto** (PKCE S256, secure token gen, RFC-8414
metadata schema); the flow is hand-rolled over asyncpg (rule 5), mirroring the session-token HMAC discipline.
Opaque HMAC-hashed codes+tokens; **atomic single-use** codes with replay-revoke; **refresh rotation** with
reuse detection; PKCE-required + **double-submit CSRF** + login-rate-limited password/consent gate with a
**PWA-session short-circuit**; RFC-8707 resource binding; the **revoke-all** switch; `validate_access_token`
seam ready for task 4. Config (`public_base_url`, `mcp_token_hmac_secret`, `mcp_oauth_scope`, TTLs). **576 unit
tests green** (+52) + **13 real-PG smoke checks** (single-use consume, revoke-all count, FK cascade,
`invalidate_all_codes`, `revoke_token` rowcount → 77/77) + a **scripted HTTP E2E** (real ASGI + real
`PgOAuthStore` + local pgvector: discovery→DCR→authorize→token→validate→refresh→revoke, all PASS); migration
010 up/down round-trip verified; ruff clean. `authlib>=1.3` added to `pyproject`/lock. **Independent security
review APPROVE-WITH-MINORS** — 3 findings fixed (atomic refresh rotation with race-proof reuse-detection;
revoke-all now invalidates pending auth codes too; dropped the un-wired `client_secret_basic`), 2 logged as
pre-existing (XFF rate-limit key → task-5 deploy decision; sync Argon2 nit). **Commit(s) pending, not pushed.**
`MCP_TOKEN_HMAC_SECRET`/`PUBLIC_BASE_URL` provisioning + Caddy root routes are **task 5**. Next: **M5 task 4** (MCP server: SDK
Streamable HTTP under `/mcp`, six tools + markdown renderers + `instructions`/prompt, `capture` source+burst,
capsule L0 + `identity://me`, resource-server token validation).
**M5 task 4 DONE (2026-07-15):** the remote **MCP server** ([ADR-046](../adr/046-m5-mcp-server-oauth-connectors.md) §1/§3/§4)
— new **`app/mcp/`** package (`render` markdown serializers · `text` instructions/descriptions/research prompt ·
`tokens` OAuth `TokenVerifier` bridge · `server` FastMCP factory), the official `mcp` SDK **Streamable HTTP**
mounted at the ROOT so `/mcp` resolves exactly (no trailing-slash redirect), **gated by the task-3 OAuth resource
server** (`validate_access_token`→`AccessToken`; unauth→401 + `WWW-Authenticate`→protected-resource metadata).
**7 tools** thin over the service layer, **markdown-rendered** (IDs verbatim, hub edge cap + `traverse` pointer,
`readOnlyHint` on reads / write marker on `capture`), `initialize` **instructions** capsule + rich descriptions +
`identity://me` resource + **research** prompt (ADR-033 #6). `capture` → **migration 011** `captures.source`
column stamped to node frontmatter `source: mcp` (web falls back to kind) + `create_mcp_capture` **burst
semaphore** (fast ack). Session manager run in the app lifespan. **599 tests green** (+23: render + an in-memory
MCP-protocol harness + capture-source) + a **real Streamable-HTTP MCP-client HTTP E2E** (uvicorn + real app + local
pgvector: unauth `/mcp`→401, OAuth→token, `initialize`→instructions, list 7 tools, call tools over the wire, read
`identity://me`, list research prompt — all PASS); migration 011 up/down verified; smoke 73/73; ruff clean. `mcp>=1.28`
added. **Commit(s) pending, not pushed;** independent review pending. Deploy (Caddy `/mcp` route + provisioning) is
**task 5**. Next: **M5 task 5** (deploy + infra: Caddy root routes, Cloudflare passthrough, secret provisioning,
compose/env; push → migrations 010/011 apply).
**M5 task 5 DONE + M5 task 6 IN PROGRESS (2026-07-15):** MCP + OAuth surface **deployed live at `braindan.cc`**
(Caddy root routes, migrations 010/011 applied, `MCP_TOKEN_HMAC_SECRET`/Cloudflare-cache-bypass set by the user);
a real Claude connector connects (a live 421-on-connect bug — FastMCP DNS-rebind allowed only localhost — was
fixed in `1ed0ee0`) and **`search` is verified grounded over the graph**. The **OAuth-focused independent
security review** ran — **APPROVE-WITH-MINORS, no must-fix** (no auth bypass / token forgery / replay / PKCE
downgrade / SQLi / XSS; gate PASSES); its **hardening batch #1–#5** (CF-Connecting-IP rate-limit key, production
secret boot-guard, clickjacking headers, loopback-only plaintext redirect, constant-time CSRF) is **built,
reviewed, and now DEPLOYED + live-verified** (`1ed5a68`+`2dc8f92`, CI `29445868984` green; `/health` all-true
confirms the boot guard passed → both prod secrets real; `X-Frame-Options`/CSP live on `/mcp`+`/authorize`).
The Claude connector Accept was user-accepted (`search` grounded + capture-side spot-checked; `revoke-all`
lockout not live-demonstrated, user moved on).
**M5 task 6 DONE → M5 (MCP server) CLOSED (2026-07-16):** the **ChatGPT fast-follow PASSED**. Confirmed the
current connector contract first (ChatGPT **Developer Mode** — Pro/Team/Enterprise/Edu — lets ChatGPT call
**all** tools incl. `capture`, so the **ingest** path works as-is; **Deep Research** would instead need read-only
`search`/`fetch` aliases and can't write), re-verified server readiness (protected-resource metadata + open DCR
accepting ChatGPT's `redirect_uri`), and the **user added the custom connector at `braindan.cc/mcp` in Developer
Mode → OAuth approve → connected, 7 tools listed — with no code change** (the known issuer trailing-slash nit
didn't trip ChatGPT). Read-only Deep-Research `search`/`fetch` aliases stay a pre-approved on-demand add (ADR-046)
only if the user later wants research-over-the-brain. Code pushed through `2dc8f92`. **The MCP + OAuth surface is
live and connector-verified on both Claude and ChatGPT.** — [08-logs/m5.md](../08-logs/m5.md) task 6.
**M6 GRILLED TO BUILD-READY + a scheduling milestone carved out (2026-07-16 — planning session,
decision-by-decision).** Scope resolved to **core + addendum (a)–(d); (e) contradiction sweep
deferred**. The grill surfaced a scheduling-architecture change and an important durability seam,
both recorded:
**(1) [ADR-047](../adr/047-pipeline-scheduling-primitive.md) — the pipeline is the scheduling
primitive.** Per-job staggered nightly crons (fragile when a step overruns; the reason for
RAM-stagger) are replaced by **pipelines** = one cron + an ordered list of steps run
**sequentially on completion**, per-step `on_fail` (`continue`|`halt`), parent+child `agent_runs`
(`parent_run_id`). Cadence → its own pipeline (`nightly`/`weekly`); a bare job is never scheduled,
even single-step work is a pipeline. Built as a small **M5.5** milestone (orchestration-only, all
existing jobs migrated) **before** M6's features, so a scheduling regression can't look like a
distiller bug. Supersedes ADR-010's stagger (the window stands).
**(2) [ADR-048](../adr/048-m6-chat-distiller-build-decisions.md) — M6 build decisions.** The load-bearing
call: an **endorsed chat memory materializes a `captures` row** (`source=chat`) → organizer, so it's
**replayed by `reprocess-all` ⇒ vision P10 holds** with zero chat-specific reprocess machinery.
Plus: single-pass multi-candidate distill on `conspect` (segmentation emergent, hedge/affect→review);
a per-session `chat_distill_state` **watermark** (idle-eligibility + idempotent delta re-distill);
**kind-aware reprocess** (preserve `stance-candidate`, truncate the rest); `maybe` **re-openable**;
coarse-LLM **salience** triage + **batch** actions + weekly **maybe digest**; a nightly all-source
**dedup sweep** (merge via an **extracted merge-core** shared with entity-merge; keep/link; augment
deferred); a nightly **inbox drainer**; **one-tap remove** = git-rm + DB-delete + **capture
tombstone** (`removed_at`, no resurrection); `remember` = sync distill / async organize; web Review
extensions + Chat "Remember now" + a chat-scoped "recently auto-recorded" list. Docs updated
(02/03/04/06/08 + ADR-047/048; the ratified M6 addendum marked resolved — `augment` + graph-degree
salience overridden for the M6 context). **Paused before implementation** per [09](../09-session-protocol.md)
— **no code this session.** Next: build **M5.5 task 1** (pipeline runner), or respawn.
**M5.5 task 1 DONE (2026-07-16):** the pipeline scheduling primitive's **orchestration mechanism** —
runner + config model + `agent_runs.parent_run_id` — built against [ADR-047](../adr/047-pipeline-scheduling-primitive.md),
**deliberately not wired into the scheduler and no job migrated** (task 2). **Migration 012**
(`agent_runs.parent_run_id`, nullable self-ref FK `ON DELETE SET NULL` + index — a bare job run stays
parentless, unchanged); **parent/child linkage via a task-local contextvar** (`child_run_scope`) so a
step's `run_scheduled` links to the pipeline parent with **no change to what any job does** (ADR-047
§5); `PipelineRunner` runs steps **sequentially-on-completion**, honours per-step `on_fail`
(`continue`|`halt`, status **inferred from each step's child run** — a non-cleanly-done child reads as
a failure so `halt` reliably aborts), closes an orphaned `running` child on a raising step (rule 7),
records the per-step sequence, never crashes the scheduler. Config `pipeline_defs()` = the migrated
ADR-010 roster in dependency order, **`continue`-dominant** (§4); step names map to the existing
scheduler job ids (task-2 wiring = a lookup). **619 unit tests** (16 new, fake steps) **+ 79 real-PG
smoke** (6 new) green, ruff clean, migration 012 up/down round-trip verified on local pg; **independent
review APPROVE — no must-fix** (4 minors applied). Commit `c0a3bd6`, **not pushed** —
[08-logs/m5.5.md](../08-logs/m5.5.md) task 1. Next: **M5.5 task 2** (migrate the roster off individual
crons into the two pipelines; scheduler registers one cron per pipeline; retire per-job cron settings;
verify durability drill + a manual run-now mid-pipeline).
**M5.5 task 2 DONE (2026-07-16):** the nightly roster is **migrated onto the pipelines** — the
scheduler now registers **one cron per pipeline**, not one per job. `BackupScheduler` →
**`PipelineScheduler`** (a step-name→job-coroutine map + one `PipelineRunner` per `pipeline_defs()`;
`max_instances=1`/`coalesce`/misfire so a long night can't overlap the next, [ADR-047](../adr/047-pipeline-scheduling-primitive.md)
§3/§7); an **unwired optional step is dropped with a log** (prod wires all four agent-window jobs
unconditionally → all 8 nightly steps always run). **Retired the nine per-job `*_cron` settings**
(zero remaining refs); the nightly roster maps **1:1** onto the old stagger **in order**; the weekly
cron is byte-identical to the retired `integrity_drill_cron` (no schedule regression). **CLI +
`POST /admin/reindex` standalone paths untouched** (invariant 4 / §6). **No job body changed** → the
DB-wipe→reindex durability drill is preserved by construction (diff = scheduler/config/main/tests
only; live re-run = task 3). **619 unit tests + 79 real-PG smoke green**, ruff clean; a **real-PG
integration** drove the full wiring end-to-end (one parent `nightly` run, row-opening steps linked
children in order, `store-sweep` unchanged/no row, pipeline completes). **Independent review APPROVE —
no must-fix** (2 follow-ups logged: Sunday nightly/weekly RAM overlap = deliberate ADR-047 §3 residual;
`run_store_sweep` opens no row so shows as a `skipped` step — both out of orchestration-only scope).
Commit `8a57611`, **not pushed** — [08-logs/m5.5.md](../08-logs/m5.5.md) task 2. Next: **M5.5 task 3**
(live M5.5 Accept — deploy; confirm one nightly start drives the whole roster in order, per-step runs
visible under the parent, durability drill green → independent review → pause).
**M5.5 task 3 DONE → M5.5 (scheduling / pipeline orchestrator) CLOSED (2026-07-16).** Pushed
`c0a3bd6`/`8a57611`/`e77a694` to prod (CI green, **migration 012 applied**, `/health` all-true). Added
a **`python -m app.cli pipeline {nightly|weekly}` run-now verb** ([ADR-047](../adr/047-pipeline-scheduling-primitive.md)
§6 enabler) printing parent run id + per-step status + child run id. The user ran `pipeline nightly` on
the VPS: **one start drove all 8 steps in dependency order** under a single parent run `939306a4` — 7
succeeded / 0 failed / 1 skipped (`store-sweep` = the known no-row job, ran `pushed=True`), each
row-opening step linking a distinct child run; every job did its normal work (**no behavior change**);
the run also **confirms migration 012 on prod** (the `parent_run_id` INSERT succeeded). **All Accept
criteria met; independent review APPROVE — no must-fix** (one fidelity minor fixed `2f6c8fb`, committed
local: run-now now uses effective vocabulary). The pipeline is the scheduling primitive — the whole
nightly roster runs sequentially from one start under a parent `agent_runs` run with per-step children,
live at `braindan.cc` — [08-logs/m5.5.md](../08-logs/m5.5.md) task 3. **Follow-ups:** `run_store_sweep`
gets its own run row (M8); deploy `2f6c8fb` on the next push; Sunday nightly/weekly RAM-overlap residual
(§3). Next: **M6 task 1** (chat-distiller — [ADR-048](../adr/048-m6-chat-distiller-build-decisions.md)),
whose four jobs are born as steps in these pipelines.
**M6 task 1 DONE (2026-07-16):** the stance-gated **chat-distiller** —
**migration 013** (`chat_distill_state` watermark table + nullable `captures.source_ref`; up/down/up
verified) + `PgChatDistillStore` (idle-eligibility `GROUP BY`/`LEFT JOIN`, oldest-first delta-after-
watermark, upsert) + **`ChatDistillerService`**: one `conspect` pass over a session's fenced delta →
`{candidate_text, stance, salience, evidence_excerpt, referenced_entity_names, why_unclear}` candidates,
stance-gated — **endorsed → a `captures` row** (`source=chat`, `source_ref=session-id`, `created_at`=
anchoring-message time, via the capture pipeline = single writer → organizer → **P10 holds**),
**unclear → a `stance-candidate` review item** (names+text, no node ids), **rejected → run-log only**;
unknown stance → `unclear` (never guessed), a hedge post-check downgrades hedged endorsements, within-run
content dedup, every run in `agent_runs`. Endorsement uses a **deterministic capture id** so a re-distill
can't duplicate (rule 6); the delta is **oldest-first bounded batches** so an oversized session defers its
remainder rather than silently skipping it. Extracted a shared **`build_capture_pipeline`** factory (rule
10, reused by reprocess); `chat-distill` CLI verb. **Not yet scheduled** (a `nightly` pipeline step in task
8). **639 tests** (+20), ruff clean, **real-PG smoke 86/86** (+7), migration round-trip verified; a **CLI
end-to-end on real dev data + LLM** distilled 6 idle sessions → 1 endorsed → capture (node dated to the
conversation day, not run time) → memory+event nodes → indexed, a **second run = 0/0** (idempotent).
**Independent review APPROVE after fixes** — 2 must-fix (cross-run duplicate materialization; silent delta
truncation), **both resolved + regression-tested**; 3 minors logged — [08-logs/m6.md](../08-logs/m6.md) task 1.
**Code committed locally, not pushed.**
**M6 task 2 DONE (2026-07-16):** review_queue M6 kinds ([ADR-048](../adr/048-m6-chat-distiller-build-decisions.md)
§7) — the **maybe-reopen** guard fix (`PgReviewQueue.resolve` now decidable = `pending` ∪ `maybe` via a
`DECIDABLE_STATUSES`/`status = ANY($)` guard, mirrored in the service + fake; `resolved`/`discarded` terminal),
**`stance-candidate` resolution** (`verdict` agree/disagree/maybe): **agree = the exact auto-endorse path** —
the same `CapturePipeline.create_chat_capture` the distiller's endorsed branch uses (one ingest, not two —
rule 2b/10; clean prose raw → organizer re-resolves, ADR-048 §2; deterministic-id idempotent), **disagree →
discarded** (logged, no node), **maybe → parked + re-openable**; **conversation-time anchoring** — the distiller
now records `anchor_at` (the anchoring-message time, same `_anchor_time` as endorsed) in the `stance-candidate`
payload so agree stamps the capture with conversation time not decision time (a timestamp, never a node id —
ADR-048 §1; recorded in 02/03, the immutable ADR left as-is); and a **kind-aware reprocess reset**
(`reset_derived_and_review`: `DELETE … WHERE kind <> 'stance-candidate'` — preserves parked human decisions,
truncates the re-derivable kinds; refines ADR-042 §2). `ReviewService` now built after the pipeline (injected
`chat_ingest`). Web Review surface for these kinds is task 7. **646 tests green** (+7), ruff clean, **real-PG
smoke 91/91** (+5: a new `check_review_queue_reopen` drives the un-fakeable `ANY()` guard) **& 12/12**
(+2: kind-aware DELETE preserves stance / clears the rest); **independent review APPROVE-WITH-MINORS — no
must-fix** (2 minors fixed: `review.py` list docstring, `smoke_reprocess` uses `KIND_STANCE_CANDIDATE`) —
[08-logs/m6.md](../08-logs/m6.md) task 2. **Code committed locally, not pushed.**
**M6 task 3 DONE (2026-07-16):** the two on-demand seams — **`POST /chat/sessions/{id}/remember`** +
**`POST /review/batch`** ([ADR-048](../adr/048-m6-chat-distiller-build-decisions.md) §6/§8), both thin over
existing services (no migration). **remember**: `ChatDistillerService.remember` runs the **same**
`_distill_session` pass (same salience + stance gate, **no force-endorse** — timing changes, not judgment),
advances the **same** `chat_distill_state` watermark (idempotent with the nightly + a double-remember via
the deterministic capture id), opens a visible `agent_runs` row; endorsed captures organize **in the
background** (fast ack). A new `PgChatDistillStore.session_state` (by-id, **no idle filter**, LEFT-JOIN)
separates **unknown session → 404** from **nothing-new → `{skipped}`**; returns `200 {endorsed,to_review}`
else `{skipped:reason}`, `422` on a malformed id. **batch**: `resolve_batch` applies one `action` to many
items **best-effort per item** (`{results:[{id,ok,error?}]}`, one bad item never aborts), **reusing the
single-item `resolve` unchanged** — the action is passed as both `choice`+`verdict` and each kind reads only
its own field (rule 10); the route is declared **before `/{review_id}`** (so `/review/batch` isn't a
malformed-uuid id) and **config-capped** (`review_batch_max`, rule 9 → `422`). `ChatDistillerService` is now
wired into `main.py`/`app.state` after the capture pipeline (its single writer, §1). **665 tests green**
(+19), ruff clean, **real-PG smoke 94/94** (+3 `session_state` LEFT-JOIN branches); both routes confirmed
in the app's OpenAPI. **Independent review APPROVE-WITH-MINORS — no must-fix** (2 minors applied: the
`review_batch_max` cap + a smoke check for the message-less `session_state` branch; a pre-existing task-1
watermark-vs-swallowed-review-hiccup observation logged) — [08-logs/m6.md](../08-logs/m6.md) task 3. **Code
committed locally, not pushed.** Next: **M6 task 4** (one-tap remove for chat-distilled nodes —
`captures.removed_at` migration + git-rm/DB-delete/capture-tombstone remove op + endpoint).
**M6 task 4 DONE (2026-07-16):** one-tap remove for chat-distilled nodes — **migration 014**
(`captures.removed_at` tombstone + the **`chat_auto_recorded`** audit registry) + **`AutoRecordedService`**
(new `app/chat/auto_recorded.py`): the chat-scoped **audit list** (`GET /chat/auto-recorded` — the
`chat_auto_recorded ⋈ captures ⋈ nodes` JOIN, content-node title with hubs skipped, `removed_at IS NULL`)
and the **remove op** (`POST …/remove` → **204**): git-rm the content nodes (**shared entity hubs
preserved**, ADR-038) → DB-delete the same content paths (`chunks`/`edges` cascade) → **tombstone** the
capture → commit; **replay-excluded** so `reprocess-all` can't resurrect it (`capture_ids_chronological`/
`counts` skip `removed_at`, **and** `_replace_notes_via_reorganize` skips a tombstoned capture — closing
the reorganize/§10-drainer vector). The distiller's **endorsed** branch records `capture_id`+`salience`
in the registry (best-effort, rule 7); **agree-from-review does not** → the list is auto-endorsed only
([ADR-048](../adr/048-m6-chat-distiller-build-decisions.md) §11/§12). **681 tests** (+16), ruff clean,
**real-PG smoke 103/103** (+10), migration 014 up/down verified; **independent review APPROVE-WITH-MINORS**
— 1 must-fix (reorganize-resurrection) + 2 minors (self-healing DB delete; type nit), **all resolved +
regression-tested** — [08-logs/m6.md](../08-logs/m6.md) task 4. **Code committed locally, not pushed.**
**M6 task 5 GRILLED/REPLANNED TO BUILD-READY (2026-07-16 — [ADR-049](../adr/049-dedup-sweep-merge-core-build-decisions.md)).**
An implementation session hit the one unrecorded decision in the dedup task — which edge a `link`
writes (the seed edge vocab has no associative rel) — and **switched to a replanning pass** per
[09](../09-session-protocol.md). Grilled decision-by-decision → **[ADR-049](../adr/049-dedup-sweep-merge-core-build-decisions.md)**:
**`link` = a canonical `similar` edge** (persists where the derived one is recomputed away; no new
governed rel); **extract the merge-core** shared by entity-merge (core + alias-union) and content-merge
(core alone, rule 10); strict-AND gate (high-cosine + shared-entity-hub edge + occurred-overlap, undated
never excludes) over **content nodes**, `indexed_at ≥` a last-success **watermark** (no migration);
a **re-file guard** (skip any pair with an existing `dedup-proposal`, any status — a merged pair
self-excludes via its tombstone); `default_survivor` = higher canonical-degree / older; resolution
`{action: merge|keep|link, survivor?}`. **Two backlog items logged** (08 §Backlog): an
**`occurred-enrichment` review kind** (NL date tagging → `occurred` range; also upgrades the dedup
occurred-signal) under a broader **"enriching & correcting ingested data"** theme. Docs updated
(02/03/04/08 + ADR-049). **Paused before implementation** per [09](../09-session-protocol.md) — **no code
this session.** Next: build **M6 task 5** against ADR-049, or respawn.
**M6 task 5 DONE (2026-07-16):** dedup sweep + the extracted **merge-core**
([ADR-049](../adr/049-dedup-sweep-merge-core-build-decisions.md)). **`MergeCore.fold`** (retarget inbound →
tombstone loser → reindex → force-commit) is now shared: **entity-merge** = core + alias-union
(behaviour-identical, `test_entity_merge` unchanged assertions), **content-merge** = core alone. The nightly
**`DedupSweepService`** files `dedup-proposal` items for content-node near-dups clearing a **strict-AND SQL
gate** (HNSW top-K cosine ⋀ shared canonical edge to a common entity hub ⋀ occurred-overlap — a null
`occurred_start` on either side **never excludes**), driver-bounded by a last-success `agent_runs`
**watermark** (**no migration**), pairs canonicalized+deduped, a **re-file guard** (`proposal_exists`, any
status) for idempotent re-scans, `default_survivor` (higher canonical degree / older), capped per run.
Resolution `{action: merge|keep|link, survivor?}` — **merge** folds via the core, **keep** discards, **link**
writes a **canonical `similar`** edge (frontmatter, so it survives the derived recompute). Config knobs +
`dedup-sweep` CLI verb (pipeline wiring = task 8); `dedup-proposal` truncate-on-reprocess (task-2 kind-aware
reset). **707 tests** (+26) + **real-PG smoke 114/114** (+11: the un-fakeable candidate SQL — cosine/
shared-hub/occurred/undated gates, re-file guard, degree), ruff clean; a CLI end-to-end on dev data +
**independent review APPROVE — no must-fix** (5 minors logged; the "undated never excludes" smoke gap closed).
Commit `fd18c7b`, **not pushed** — [08-logs/m6.md](../08-logs/m6.md) task 5. Next: **M6 task 6** (inbox drainer),
or respawn.
**M6 task 6 DONE (2026-07-16):** the nightly **inbox drainer** ([ADR-048](../adr/048-m6-chat-distiller-build-decisions.md) §10).
Re-runs the **existing** `reorganize_capture` **inline** (new `CapturePipeline.reorganize_capture_now`) over captures
still materialized as an `inbox/` fallback, so a now-richer entity registry can resolve a previously unorganizable
capture into real typed nodes — **replaced only on success**; a still-failing capture keeps its `inbox/` node and
re-qualifies next run. `InboxDrainService` mirrors the dedup-sweep template (own `inbox-drain` `agent_runs` row,
bounded by `inbox_drain_max_per_run` oldest-first, best-effort per capture, never raises); tallies resolved vs
still-inbox by re-fetch. New **`PgCaptureStore.list_inbox_materialized`** (`unnest(node_paths) LIKE inbox/%` ⋀
**`removed_at IS NULL`** — the **§11 double guard** with the core's `removed_at` skip, so a one-tap-removed capture
can't resurrect through the drainer) is **status-agnostic** (a `failed` still-inbox capture stays eligible). No new
review kind (residual ambiguity → normal `entity-ambiguity`). `inbox-drain` CLI verb (`backup_now` flush); pipeline
wiring = task 8. **715 tests** (+8) + **real-PG smoke 117/117** (+3), ruff clean; **independent review
APPROVE-WITH-MINORS — no must-fix** (3 minors logged: boundary `truncated` over-report; latent `inbox_folder` vs
hardcoded writer-folder coupling; a core partial-write counts as reorganized-not-errored). Commit `af14de5`,
**not pushed** — [08-logs/m6.md](../08-logs/m6.md) task 6.
**M6 task 7 DONE (2026-07-16):** web — **Review** extensions + Chat distiller surfaces (ADR-048 §12; thin PWA
over the tasks 1–6 endpoints, no server change, rule 4): `stance-candidate` (agree/disagree/maybe) +
`dedup-proposal` (survivor radio → merge/keep/link) cards, **salience ordering**, a **single-kind multi-select
batch bar** over `POST /review/batch`, a **parked/maybe** section with per-card aging + a header/nav **count
badge**; Chat **"Remember now"** → `POST …/remember` + a chat-scoped **"recently auto-recorded"** list with
**optimistic one-tap remove**. tsc/eslint/vite green; cache-seeded browser smoke; **independent review
APPROVE-WITH-MINORS — no must-fix** (1 fixed, 5 logged). Commit `c603cf5`, **not pushed** — [08-logs/m6.md](../08-logs/m6.md) task 7.
**M6 task 8 DEPLOYED + independent Accept review APPROVE (2026-07-16); live behavioral checks + VPS run-now pending the user — the last M6 gate.** The
**maybe-digest** weekly job + the M6 jobs wired into the [ADR-047](../adr/047-pipeline-scheduling-primitive.md)
pipelines (ADR-048 §8/§consequences). **maybe-digest**: a new `MaybeDigestService` emits a **feed-visible
`agent_run`** summarizing the parked `maybe` items (`{total, by_kind, oldest_age_days}`; **no push** — M10;
the Review UI already carries the badge + aging) over a new `PgReviewQueue.maybe_kind_stats` aggregate
(`GROUP BY status='maybe'`, rule 10); `maybe-digest` CLI verb. **Wiring**: the nightly roster now weaves the
four M6 sleep-cycle jobs into their dependency slots (04-pipelines §Scheduling order) — **chat-distiller** →
data-sync → db-backup → **inbox-drain** → reindex → profile-refresh → entity-backfill → identity-capsule →
**dedup-sweep** → store-sweep → store-backup; **weekly** = integrity-drill → **maybe-digest**;
`PipelineScheduler` gains the four optional steps (dropped-with-log if unwired, ADR-047 §5), wired in both
`main.py` + the run-now CLI (distiller/inbox-drain share one capture pipeline; run-now drains + flushes only
when a capture-touching step ran). **719 unit tests** (+4) + **real-PG smoke 121/121** (+4), ruff clean;
`maybe-digest` CLI verb clean against local pg. **Independent review APPROVE-WITH-MINORS — no must-fix**
(2 minors fixed, 3 logged). Commits `0bf5312`/`faf1afd`, **not pushed** — [08-logs/m6.md](../08-logs/m6.md) task 8.
**Live Accept (closes M6):** push → CI deploys → verify the full M6 loop live at `braindan.cc`
(overnight auto-endorse via the pipeline, dedup/inbox-drain nightly steps, the weekly maybe-digest feed row,
`reprocess-all` P10) + a VPS `pipeline nightly|weekly` run-now.
**Deployed + Accept review APPROVE (2026-07-16).** The M6 range (10 commits `2f6c8fb`→`faf1afd`, tasks 1–8 +
the carried M5.5 follow-up) was pushed; the first push **failed CI** on `ruff check` (E501 in migration 013's
docstring), gating the deploy. Fixed + **hardened `.githooks/pre-commit` from a secret-only guard into a
secret + lint/format gate** (ruff `check`+`format --check` on staged `server/**.py`, `eslint --max-warnings 0`
on staged web source — staged-scoped, fails closed if `uv`/`pnpm` missing; a lint error can't reach `main`
again) — commit `16eb2bd`; whole-repo ruff clean + 719 tests green, re-pushed. CI green → deploy ran
`alembic upgrade head` (the full range carries **migrations 013/014** from tasks 1/4 — additive, up/down
verified; task 8's own diff added none, hence the earlier "no migration" note); `/api/v1/health` all-true and
the M6 route surface is **live** (`chat/auto-recorded` 404→401; the three M6 write routes 401). **Independent
M6 Accept review APPROVE — no must-fix** (all 6 Accept criteria + task-8 wiring mapped to `file:line`;
invariants 2b/2(P10)/6/7/5/9 hold; minors cosmetic/deferred). **Remaining to CLOSE M6 (user-driven — needs the
VPS shell + the authenticated PWA):** a VPS `pipeline nightly|weekly` run-now (one start drives the whole
roster incl. the M6 steps) + the PWA behavioral loop (auto-endorse → recently-recorded → one-tap remove;
pure-retrieval skip; stance-unclear agree-only; maybe park + re-openable; optional P10 reprocess). Code pushed
through `16eb2bd`; deploy live. Next: **user runs the VPS run-now + PWA checks → M6 CLOSED**, or respawn.
**M6 follow-up (step-status fidelity) GRILLED TO BUILD-READY (2026-07-16 — [ADR-050](../adr/050-pipeline-step-status-is-the-jobs-own-run.md)).**
The live Accept `pipeline nightly` run-now surfaced `chat-distiller` + `inbox-drain` reported **failed** —
grilled to a status-fidelity defect, **not** data loss (P10 held): a capture degrading to the `inbox/`
fallback closes its `agent="capture"` organize run `failed` (rule 7), and `child_run_scope` flattens that
nested run into the enclosing step so `_step_status` (any non-`{succeeded,skipped}` child → failed) failed
the step even though the distiller's/drainer's **own** runs succeeded. Fix (ADR-050): **a step's status =
its own job run** (`agent == step.name`); nested spawned runs stay feed-visible but non-gating; inbox-fallback
keeps `status=failed` (step-rollup layer only); `raised → failed` + `halt`-on-own-failure unchanged; server-only,
**no migration**. Docs recorded (ADR-050 + 04/08). Out of scope (logged): the organizer's inbox-fallback rate on distilled
claim-text (2/4 this run; the drainer retries); the `claude` Max CLI 300s hang before Nebius fallback
(Providers card).
**M6 follow-up (ADR-050) DONE → M6 (chat-distiller + review queue) CLOSED (2026-07-16).**
`pipeline._step_status` now keys on the step's own run (`agent == step.name`); nested `capture` runs stay
feed-visible but non-gating; `raised → failed`, `halt`-on-own-failure, `store-sweep → skipped` preserved;
inbox-fallback run keeps `status=failed`. **721 tests** (+2 regression), ruff+format clean, **independent
review APPROVE — no must-fix** (invariant verified across every wired step); commit `1c36e06` (on the
`8e1c376` ruff-format sweep), pushed + deployed (server-only, no migration). **Live Accept PASSED:** the
user re-ran `pipeline nightly` on prod → parent `1f5099ef`, **11 steps, 10 succeeded, 0 failed, 1 skipped**
— **`inbox-drain` → succeeded** while still reporting "0/2 resolved" (its 2 unresolved captures' nested
`capture` runs stayed visible-as-failed, the exact case that read `failed` pre-fix), `chat-distiller →
succeeded`, only `store-sweep` skipped (by design). The user exercised part of the PWA behavioral loop +
confirmed the remaining surfaces and **accepted M6 as closed**. The M6 sleep-cycle (chat-distill →
inbox-drain → dedup-sweep; weekly maybe-digest) runs clean from one nightly start under a parent run with
per-step children, live at `braindan.cc` — [08-logs/m6.md](../08-logs/m6.md) "M6 follow-up".
**M7 (the map) GRILLED TO BUILD-READY (2026-07-16 — [ADR-051](../adr/051-m7-map-build-decisions.md); planning
session, decision-by-decision).** The map is a **re-center neighborhood explorer** (one focal node at a time,
breadcrumbs, never a whole-graph client layout — settled the "expand" vs "re-center" doc tension) over a new
**grouped-by-`(origin,rel)`-zone** `GET /nodes/{id}/neighbors` (per-zone caps `map_zone_fanout` + `total`/
`next_cursor`; reuses the M5 `GraphService.neighbors` primitive for "show more"; one new endpoint, no migration).
Render: **`react-force-graph-2d`** (2D only) with per-zone directional forces; **emoji=type / ring+size=hub-vs-
content / theme-independent plane color**; single-click re-center (plex fade) / hover peek / center-click →
`NodePreview` drawer; canonical solid+arrowhead / derived faint / **superseded (`until`) dashed+dimmed**. New
full-width **Map tab**; **canvas on phone too** (supersedes the "phone=list, no canvas" wording — list kept as
reduced-motion fallback + toggle); entry from Search cards + `NodePreview` edges; empty state = embedded search +
restore-last-centered. **4 tasks** in [08 §M7](../08-implementation-plan.md) (server endpoint · web canvas · web list
fallback+phone · live Accept). Backlog (ADR-051): **auto-center on top hubs** (new endpoint, user wants it *later*),
in-app reduced-motion override, multi-plane ring. Docs recorded (ADR-051 + 03/06/08). **Paused before
implementation** per [09](../09-session-protocol.md) — **no code this session. Next:** build M7 task 1, or respawn.
**M7 task 1 DONE (2026-07-16):** the grouped map endpoint **`GET /nodes/{id}/neighbors`** — one route,
two modes, thin over the M5 `GraphService.neighbors` primitive (no migration, no new store surface):
**no `rel`** → the grouped first page (one zone per **`rel`**, each capped at `map_zone_fanout` seed 8
with a per-zone `total` + `next_cursor`) via a new `PgNeighborStore.neighbor_zones` window query
(`ROW_NUMBER`/`COUNT` `PARTITION BY rel`, direction-scoped before the window, tombstone-excluded,
`until`-edges returned) + a light `center_header`; **`?rel=&cursor=`** → a single zone's flat "show
more" reusing the M5 rel-filtered keyset untouched. **Replanned mid-implementation → [ADR-052](../adr/052-map-zones-keyed-by-rel.md):**
an independent review caught that ADR-051 §2's `(origin,rel)` zone key vs the rel-only "show more"
cursor diverge for the sole dual-origin rel `similar` (canonical `link` + derived recompute) — a
short `/grilling` pass resolved it to **zones keyed by `rel`** (`similar` collapses to one zone;
per-neighbor `origin` carries solid/faint styling; zone-level `origin` dropped), shipping with the
dual-origin regression the gap had lacked. **737 unit + 129 real-PG smoke green, ruff clean;
independent review APPROVE-WITH-MINORS** (Finding 1 resolved; sole minor — a stale docstring —
fixed) — [08-logs/m7.md](../08-logs/m7.md). Commits `1cef778`/`98b9ce3`/`4a0d810`/`8e25e38`, **not
pushed.** Next: **M7 task 2** (web canvas map).
**M7 task 2 DONE (2026-07-16):** the web **canvas map** — a `react-force-graph-2d` (2D-only, ADR-032
#12) neighborhood explorer over the task-1 `GET /nodes/{id}/neighbors` (client-only, ADR-006 / rule
4 — no server code). New **`features/map/`**: a zoned force canvas (pinned focal, a custom per-zone
directional force settling "people one side, topics another"), **emoji=type** marks + **plane-
coloured halo / hub ring** (hub-vs-content from the governed `entity_like_types`; a new **theme-
independent** `ui/planeColors` palette — the theme accent is reserved for the focal node), edges
**canonical solid+arrow / derived faint / superseded (`until`) dashed+dimmed**, **single-click
re-center** with breadcrumbs (forward-history truncation), **hover peek**, **center-click →** the
shared **`NodePreview` drawer** (rule 10; its edges re-center), per-zone **"+N" show-more** (appends
without refetching the neighborhood), and an **empty state** (embedded search + restore-last-centered
from `localStorage`). Shell gains the **8th "Map" tab** with a per-tab **`wide`** breakout of the
640px column + a lifted one-shot `mapSeed`; entry from **"Explore in map"** on Search cards +
**navigable `NodePreview` edge rows** (Search + Chat, via an optional `onOpenNode` that keeps the
`ui/` primitive feature-agnostic). tsc/eslint/vite build green; a **real-browser walkthrough vs a
throwaway mock API** drove the empty-state search → pick → **center (grouped `neighbors` fires; canvas
mounts, caption + breadcrumbs correct; zero console errors)** — the one live defect (a size-0 host
race) found + fixed (callback-ref ResizeObserver). **Independent review APPROVE after fixes** — 2
must-fix (re-center canvas teardown → `keepPreviousData` plex swap; first-view force/fit unwired →
fg-readiness gate), both resolved + re-verified; minors applied (show-more in-flight guard, dimmed
arrowheads). Commits `93ac37f`/`624a644`, **not pushed** — [08-logs/m7.md](../08-logs/m7.md) task 2.
**Interactive canvas hops (node-click re-center, "+N", drawer) + the mobile viewport were not driven
in-browser** (react-force-graph hit-testing ignores synthetic events; the pane's input degraded) —
they're **task 3's** walkthrough scope. Next: **M7 task 3** (list fallback + reduced-motion + phone).
**M7 task 3 DONE (2026-07-16):** the web **map list fallback + reduced-motion + view toggle** — a
`features/map/MapList.tsx` grouped tappable-list renderer over the **same rel-keyed `effectiveZones`**
the canvas draws (extracted as one shared memo so the two views stay in lockstep), serving as both
the **`prefers-reduced-motion` fallback** and a manual **Canvas/List toggle**
([ADR-051](../adr/051-m7-map-build-decisions.md) §7; client-only, ADR-006 / rule 4 — no server code).
Focal header → the shared `NodePreview` drawer (rule 10); one `<section>` per rel zone with tappable
**tap-to-recenter** rows + per-zone **"Show N more"** (reusing the M5 rel-cursor); **inline edge
styling mirroring the canvas** (§6 — derived "similar" **no arrow**, superseded `until` dashed+dimmed,
canonical direction arrow). `MapScreen` gains the `useReducedMotion`-driven default (`view =
override ?? (reduced ? 'list' : 'canvas')`) + the header toggle + a `busyRels` render-state mirror of
the show-more guard. tsc/eslint/vite green; a **real-browser walkthrough vs a throwaway mock API**
drove the list end-to-end on **desktop and a 375px mobile viewport** (center, toggle, 3-hop
tap-to-recenter, breadcrumb truncation, show-more paging, drawer + drawer-edge re-center; phone
full-width, canvas mounts on phone, no horizontal overflow; console clean) — the interactive taps
task 2 deferred. **Two documented preview-pane limits (not product defects, same class task 2 hit):**
`requestAnimationFrame` is frozen in the pane (framer-motion enter/exit + the force sim don't visually
animate, screenshots time out — handlers verified via DOM `.click()` + observed state changes), and
reduced-motion can't be OS-emulated there (framer caches it at load) — the branch selects the same
list view driven above. **Independent review APPROVE-WITH-MINORS** — 1 must-fix (derived rows carried
a direction arrow; symmetric derived edges must carry none, §6 — gated canonical-only, matching the
canvas) fixed + re-verified; 1 minor logged (`isHub` helper duplication). Commits `78ec92f`/`469d504`,
**not pushed** — [08-logs/m7.md](../08-logs/m7.md) task 3. Next: **M7 task 4** (live Accept + docs
close-out — deploy the M7 range, live-accept at `braindan.cc`, record → M7 CLOSED).
**M7 task 4 DEPLOYED + live Accept verified (2026-07-16); user's final CLOSE nod pending — the last M7 gate.**
Pushed the 8-commit M7 range `1cef778`→`469d504` (tasks 1–3; **no migration**) after a green local gate
(ruff/format, tsc, eslint, 737 tests); CI green → deploy landed (the new `GET /nodes/{id}/neighbors`
flipped **404→401**, `/health` all-true; web bundle scp'd alongside). **Live-accepted the map in the
user's already-authenticated `braindan.cc` Chrome** (agent never handled the login secret; the real
browser's working canvas hit-testing/rAF drove the interactive taps tasks 2/3 had deferred):
**5 of 6 Accept criteria fully verified** (node identities redacted here — the graph holds real
personal data): ① one-click from a search hit to a `person` **constellation** (gold focal + hub ring,
emoji/plane-colour neighbors), ② **3-hop re-center** with tracking breadcrumbs + forward-history
truncation + hover peek, ③ **canonical** (arrow+`since`) vs **derived** (`similar`, no arrow) edge
styling in the List view + `NodePreview` drawer, ④ **per-zone "Show 4 more"** appended in place with
the sibling zone/focal untouched (no neighborhood refetch), ⑥ empty-state **embedded search** +
**"↩ Return to …"** restore-last-centered (after reload, from `localStorage`); plus center-click
**drawer** and the live `MapNeighborItem` contract (incl. `until`).
**2 documented residuals, neither a defect:** ③ **superseded (`until`) edge styling** has **no live
data instance** — a scan of 78 hubs (194 canonical + 582 derived edges, 157 with `since`) found **0
`until` edges** (additive graph; capability is contract- + smoke- + task-3-mock-verified); ⑤ the
**phone viewport** couldn't be re-narrowed on the managed prod-Chrome window (`resize_window` clamped
at ~1707px), but the phone canvas + tap-to-recenter + reduced-motion→list were verified at 375px in
task 3 on the **same deployed bundle** + review. Docs recorded ([08-logs/m7.md](../08-logs/m7.md) task 4);
code pushed through `469d504`; deploy live.
**M7 CLOSED (2026-07-16):** the **user confirmed the map on their actual phone** ("works very good"),
closing the phone-viewport residual; the only residual left is cosmetic (superseded `until` styling
has no live data instance yet — rendering already in place). All Accept criteria met — the re-center
neighborhood explorer (search→constellation, 3-hop re-center, canonical/derived edge styling, per-zone
show-more, phone canvas + list toggle + reduced-motion→list, restore-last-centered) is **live at
`braindan.cc`**.
**M8 (ops console & activity restructure) GRILLED TO BUILD-READY (2026-07-17 — [ADR-053](../adr/053-m8-ops-console-observability-build-decisions.md),
decision-by-decision).** The load-bearing find: **no per-run log capture exists** (`agent_runs` holds
only an end-of-run `summary`/`details`), so the **live log tail** is the one genuinely new subsystem;
everything else is projection/CRUD over existing tables + the live scheduler. Decisions: **live logs**
= an `app.*`/`INFO`+ logging handler tagged by a `_current_run_id` contextvar (ADR-047 §5 ambient
pattern → **no job-body churn**) → bounded per-run in-memory buffer (non-blocking, rule 8) → async
~1s + on-finish flush → durable **`agent_run_logs`** table; `GET /activity/runs/{id}/logs` **polls**
(`?after_seq=`, not stream); `app.*`+`INFO` keeps library-DEBUG secret leakage off the UI-rendered
store (rule 11). **Merged `GET /activity`** = a **UNION-of-views** over `agent_runs`/`captures`/
`review_queue` (no events table), keyset-paginated, **3 tabs** (agents/jobs · conversations · manual
actions), category by *origin* via a new **`agent_runs.trigger`** column. **Ops** = **`GET /agents`**
flat roster (0..N pipeline membership) + **`GET /pipelines`** as a first-class resource +
**`POST /agents|pipelines/{name}/run`**, single-flight via one in-process **JobRunner** guard the
scheduler shares; **one unified ordered console** (bare Run for zero-arg jobs incl. `graph-health`;
parameterized ops keep their `/admin/*` input controls); **pipeline editing deferred**.
**`graph-health`** = read-only reporter (7 checks → `agent_runs.details`), **nightly-tail**, config
thresholds, no auto-remediation (→M10). **`store-sweep`** gets its own run row (carried M5.5 follow-up).
**Web** = one **Activity** tab, **Feed/Ops** segmented sub-views (M2 Admin panel absorbed). **Tasks**
(08 §M8): **T1 foundation** (solo, owns the sole migration) → **Batch B {T2 feed · T3 agents/pipelines
· T4 graph-health}** run as a **≤3 parallel fan-out** (disjoint files, 0 migrations in-batch, no
intra-batch dep — user-approved trial of the 09 v1.7 provisional mode) → **T5 web** (solo) → **T6 live
Accept** (solo). Docs recorded (ADR-053 + 08/03/02/04/06). **Paused before implementation** per
[09](../09-session-protocol.md).
**M8 task 1 DONE (2026-07-17):** the observability foundation — **migration 015** (`agent_run_logs`
live-log-tail store + `agent_runs.trigger` scheduled|manual); an `app.*`/`INFO`+ **log-capture
handler** tagging records by the active run (a `_current_run_id` contextvar **stack** the `agent_runs`
store owns — task-safe + nested, **no job-body churn**) → a **bounded per-run buffer** (drop-oldest +
elision marker, rule 7) → an **async flusher** (~1s + on finish, then reap); the **JobRunner**
single-flight seam (scheduler + manual endpoints share it, threaded through the pipeline —
`scheduled_step` skips on a manual collision, `run_manual` 409s); the **`store-sweep` own run row**
(ADR-053 §10, kills the phantom `skipped` step); **`GET /activity/runs/{id}/logs`** (poll, `?after_seq=`
+ `running`). Namespace+`INFO` filter keeps library-`DEBUG` secret leakage off the UI-rendered store
(rule 11). **764 tests** (+43), ruff clean, **real-PG smoke 136/136** (+7), migration 015 up/down +
**e2e log capture** verified; **independent review APPROVE — no must-fix** (1 nit applied; 2 follow-ups
logged: the on-finish flush is scheduled-not-awaited so the **T5 web client must page past the tail cap
until an empty page**, not stop solely on `running==false`; the run-stack pop is coupled to `finish`).
Commit `4750f12`, **not pushed** — [08-logs/m8.md](../08-logs/m8.md) task 1. Next: **Batch B {T2,T3,T4}**
as the ≤3 parallel fan-out, or respawn.
**Batch B DONE (2026-07-17):** the ≤3 parallel fan-out (T2 feed `f28dc5d` · T3 agents/pipelines `c8e61f3`
· T4 graph-health `f51674b`) landed disjoint with **zero collisions** (T3 sole `main.py` editor; 0
in-batch migrations — Alembic head stays `015`). Per-task **fresh independent review**: T2/T3 **APPROVE
no must-fix**; T4 **1 must-fix resolved** (`graph_health_sample_offenders=0` silently zeroed 5 checks'
counts — decoupled count from sample via CTE, rule-7). Coordinator closed the one cross-task seam
(`main.py` graph-health DI wiring, in T3's commit) + ran the **merged-tree integration gate**: ruff clean,
**full suite 809 passed** (+45 over the 764 T1 baseline), import-clean, single migration head. First 09
v1.7 provisional ≤3 fan-out ran clean — no reversal warning. Code committed through `c8e61f3`, **not
pushed**. Next: **T5 web Activity screen** (solo), or respawn — [08-logs/m8.md](../08-logs/m8.md) "Batch B —
complete".
**M8 Task 5 DONE (2026-07-17, `5c7a97b`):** the web **Activity screen** — one tab, **Feed/Ops** segmented
(client-only, ADR-006; the M2 Admin tab **absorbed** → 7 tabs). **Feed** = the merged `GET /activity` as 3
categorized tabs (keyset infinite scroll, expand run detail, conversations **one-tap remove** folding in
the M6 auto-recorded control). **Ops** = pipelines (schedule/next-run/steps + whole-pipeline Run), the
runnable **agent roster** with a **persistent live log tail** (polls `…/logs`, **drains past run
completion** — ADR-053 §2), the **graph-health card** (latest run's `details.checks`), and the
**parameterized ops** that can't collapse to a bare Run (ADR-053 §8): tags-consolidate, reprocess
(confirm-gated), **entities/merge**, **edge vocab/consolidate** — zero-arg reindex/backup are the roster's
Run buttons. tsc/eslint/vite green; a **real-browser walkthrough vs a throwaway mock API** drove every
surface (a roster Reindex Run streamed its full log tail and the tail **persisted after SUCCEEDED**).
**Independent review APPROVE after fixes** — 3 must-fix resolved (reindex/backup were mis-rehomed as
`/admin/*` cards → §8 roster jobs, which surfaced + added the 2 missing parameterized ops; `RunningDot`
reduced-motion; roster tail unmounting before the drain) + 2 minors; minors logged. Commit `5c7a97b`,
**not pushed** — [08-logs/m8.md](../08-logs/m8.md) task 5. Next: **T6 · live M8 Accept** (solo, last), or respawn.
**M8 Task 6 DONE → M8 (ops console & activity restructure) CLOSED (2026-07-17).** Pushed the 5-commit
range `469d504..5c7a97b` (T1–T5) after a green local gate (ruff/format, 809 tests, tsc/eslint/build); CI
green → deploy landed in ~30s. `/api/v1/health` all-true ⇒ **migration 015 applied** (`alembic upgrade
head` runs under `set -e`; new `agent_run_logs` + `agent_runs.trigger`); the three M8 routes
`/api/v1/{activity,agents,pipelines}` flipped **404→401** (session-gated, live). **Live-accepted in the
user's authenticated `braindan.cc` browser** (agent read the live prod session, never handled the login
secret) — **all Accept criteria pass:** ① Ops lists **14 jobs** each with schedule + pipeline membership
+ **Run** + last-run status, and **2 pipelines** (`nightly` `0 3 * * *` with 12 ordered steps incl. the
new `store-sweep` + `graph-health` tail; `weekly` `30 4 * * sun`) with next-run + "Run pipeline" — the
two new jobs correctly "Never run"; ② manual **`reindex`** and **`graph-health`** Runs each **streamed a
real live log tail that persisted after SUCCEEDED** (the drain), `db-backup` = graceful empty-tail
placeholder; ③ **Feed → Manual actions** shows the manual `db-backup` (`pg_dump …→ db/pg_dump-…sql`) +
review verdicts (`Reviewed: entity-ambiguity/dedup-proposal/vocab-proposal`), **Agents & jobs** shows
scheduled runs with pipeline parent/`↳ step` nesting + the `nightly … 10 succeeded/0 failed/1 skipped`
rollup, **Conversations** shows the M6 auto-recorded list + one-tap Remove; keyset "Load older"; **no
console errors**. **Bonus:** the **graph-health card** populated live (4/7 checks flagged, all 7 read-only
checks with bounded offender samples) + the 4 parameterized Operations cards; Admin tab absorbed → 7 tabs.
**Independent review APPROVE-WITH-MINORS — no must-fix** (fresh agent re-derived the criteria from ADR-053
+ 08 + 03-api against the deployed diff; adversarial rule-11 secret-leakage / no-job-body-churn / keyset /
log-tail-drain / ADR-006 / contextvar-task-safety checks all clean; 5 minors logged — the one new
follow-up: a whole-pipeline **manual** trigger doesn't 409 against a concurrent **scheduled** run of the
same pipeline, data-safe + low-impact, agent-level guard is correct for `POST /agents/{name}/run`). The
ops console & activity restructure is **live at `braindan.cc`** — [08-logs/m8.md](../08-logs/m8.md) task 6.
**Code pushed through `5c7a97b`.** Next: **M9 (connectors: Slack stance-gated + Telegram capture)** —
planning session (`/grilling` first); Telegram capture may be pulled forward at the user's call.
**M8.1 + M8.2 GRILLED TO BUILD-READY; M9 re-sequenced after them (2026-07-17 — planning session,
decision-by-decision).** A post-M8 request batch was grilled the same day into two inserted milestones
(M5.5-style decimals) + a re-pinned capstone: **[M8.1 — UI & navigation consolidation](../08-implementation-plan.md)
([ADR-054](../adr/054-m8.1-ui-navigation-consolidation.md)):** exact-time **tap+hover** tooltip on every
relative timestamp (one `<TimeAgo>` primitive); pipeline runs collapse to **one feed row** with a
**recursive `children[]` tree** on run detail (depth in the data structure; early→late reading order);
**Search + Map merge into one "Explore" tab** (7→6; filter chips dropped — taxonomy stays product-wide,
manual pre-filter UI earns nothing at personal scale; chat plane chips stay); the Conversations feed
category becomes **Captures** (all sources, paginated, expandable, chat rows keep Remove; Capture-tab
Recents → 5 + link); a universal clickable **`NodeChip`** → `NodePreview` → map (graph-health samples
gain node ids). 5 tasks, batch-annotated. **[M8.2 — Data quality](../08-implementation-plan.md)
([ADR-055](../adr/055-interiority-inner-voice-first-class.md) · [ADR-056](../adr/056-temporal-correctness-date-tokens.md)):**
**interiority** — organizer stamps `internal|external|mixed` + **extracts inner-voice content into its
own nodes** (no new type; sole migration = `nodes.interiority`), consumed by a chat-retrieval boost knob,
a dedicated identity-capsule slice, and a Map/preview marker; **temporal correctness** — organizer prompt
carries the **stored capture anchor** (never wall-clock; P10-deterministic under reprocess), the LLM
emits **symbolic time-references only** with a deterministic Python resolver doing all date math
(**"LLMs classify, code computes" — new CLAUDE.md hard rule 12**, product-wide; fail → prose, never
guessed), resolved dates land as inline **`[[t:START[/END][|label]]]` tokens** (ranges + absolute labels;
recurrence fenced to M11) rendered live in web (never raw; exact-time tooltip), expanded **before
embedding** in the indexer, and governed by a binding **LLM-bound rendering contract** (every path to any
LLM ships token expansion + a recorded-at·occurred metadata header); **two-tier edits** (token edit =
mechanical/instant; anchor edit = background one-capture reorganize; ripple = nightly); `occurred_*`
stays `date` (tokens own sub-day); the backlog's **`occurred-enrichment` review kind absorbed** into
M8.2; one prod **reprocess-all** backfills both dimensions. 5 sequential tasks. **The v1 documentation
suite** (non-technical audience, hosted, visualized, incl. a full agent-prompt inventory) is **re-pinned
as the v1 capstone after M9/M10/M11** (user-confirmed: docs need the system feature-complete);
requirements captured in 08 §Backlog. Sequencing: **M8.1 → M8.2 → M9 → M10 → M11 → docs capstone**.
**Paused before implementation** per [09](../09-session-protocol.md) — **no code this session** (docs +
CLAUDE.md rule 12 only). Next: build **M8.1 task 1** (server), or respawn.
**M8.1 task 1 (server) DONE (2026-07-17):** the server half of the UI/nav consolidation
([ADR-054](../adr/054-m8.1-ui-navigation-consolidation.md) §2/§4/§5; **no migration**). The activity
feed's `agent_runs` branch now returns **parentless runs only** (a pipeline night collapses from 12
rows to 1); `GET /activity/runs/{id}` gained a **recursive `children[]` tree** (pure `build_run_tree`
+ a depth-bounded recursive CTE `children_tree`, siblings early→late) — real trees are one level deep
since [ADR-050](../adr/050-pipeline-step-status-is-the-jobs-own-run.md) parents spawned `capture` runs to
the pipeline root, but the render is genuinely recursive (depth-2 proven). The **`conversations` feed
category is renamed `captures`** and widened to **all** sources (row `kind` `chat_capture`→`capture`);
every row carries `status` + `source` (`COALESCE(source, kind)`), and the expand reuses
`GET /captures/{id}` (`CaptureView` gained `source`). graph-health offender ids were already present
(M8 T4) — recorded a **carve-out refining ADR-054 §5** (node-checks → `NodeChip`; `pending-review-
aging` → Review item, not a node). 816 unit + 149 real-PG smoke green, ruff clean; **independent
review APPROVE after 1 must-fix** (a 03-api over-claim on offender ids — corrected; the code was
right) — [08-logs/m8.1.md](../08-logs/m8.1.md) task 1. **Code committed locally, not pushed.** Next:
**M8.1 task 2** (web primitives — `<TimeAgo>` + `NodeChip`), or respawn.
**M8.1 task 2 REPLANNED (2026-07-17, grilled — no code).** A T2 implementation session hit a
contract-vs-data conflict and stopped to replan (09 rule 4): ADR-054 §5 wants **capture node chips**
clickable → `NodePreview`, but `NodePreview` needs a **frontmatter-uuid** node id (`GET /nodes/{id}`)
while captures expose only store **paths** (`CaptureView.node_paths`; [02](../02-data-model.md) §Identity —
"paths are projections"), so it's web-only-infeasible. Grilled decision-by-decision: **capture/Captures
chip clickability + the server node-id exposure defer to T4** (read-time `node_paths → nodes.id` join,
**no migration**); **T2 ships** `<TimeAgo>` (wraps `relativeTime` unchanged + custom tap+hover exact-time
tooltip; 9 sites + SettingsScreen), a single app-level **`NodePreview` bottom-sheet drawer** (`NodeChip`
→ `useNodePreview()` context in `AppShell`; drawer chrome owns header + "Explore in map"; `NodePreview`
unchanged, edges jump to Map), and wires **only uuid-bearing surfaces** — graph-health **node-check**
offenders → `NodeChip`, **`pending-review-aging`** offenders → a **new Review deep-link**
(`ReviewNavContext` → Review tab, scroll-to + transient highlight, pending ∪ maybe, silent-land on a
resolved id) as a separate review-chip (`NodeChip` stays node-uuid-only). **Refines ADR-054 §5** (no
edit / no new ADR — plan refinement + minor nav affordance) — [08-logs/m8.1.md](../08-logs/m8.1.md) "Replan
— T2 scope"; 08 §M8.1 T2/T4. **Paused before implementation.** Next: build **M8.1 task 2** to this plan,
or respawn.
**M8.1 task 2 DONE (2026-07-17):** web primitives (ADR-054 §1/§5 + the replan; **web-only, no server
code — ADR-006**) — **`<TimeAgo>`** wraps `relativeTime` **unchanged** + a custom **mouse-gated-hover /
tap** exact-time tooltip (`17 Jul 2026, 08:36`; `aria-label` carries relative + exact); swapped all 9
`relativeTime` sites + SettingsScreen's absolute stamp. **`NodeChip`** → one app-level **`NodePreview`
bottom-sheet drawer** via a `useNodePreview()` context in `AppShell` (drawer chrome owns the header +
"Explore in map"; `NodePreview` unchanged, edges jump to Map; pixel-rise entrance). Graph-health
**node-check** offenders → `NodeChip`; **`pending-review-aging`** → a new **Review deep-link**
(`ReviewNavContext` → Review tab, scroll-to + transient ring, pending ∪ maybe, **React-18-StrictMode-
safe** — seed not consumed in-child, `AppShell` clears it on manual nav). tsc/eslint/build green + a
**real-browser mock walkthrough** (drawer open / Explore-in-map / deep-link ring / TimeAgo tooltip;
caught + fixed 3 real bugs — timer-teardown, StrictMode double-mount, drawer percentage-`y`).
**Independent review APPROVE-WITH-MINORS** — 1 must-fix (TimeAgo iOS touch-tap → pointer-gated) + 2
minors applied (`ee229f0`); 1 follow-up logged (drawer focus-trap). Commits `20a8fba`/`ee229f0`, **not
pushed** — [08-logs/m8.1.md](../08-logs/m8.1.md) task 2. Next: **Batch C {T3 Explore, T4
Activity/Captures}** (T4 folds in capture/Captures chip clickability + the server node-id exposure), or
respawn.
**M8.1 Batch C DONE (2026-07-17):** the second 09 v1.7 provisional **≤3 parallel fan-out** (after M8
Batch B) — **T3 Explore `ee6c88d`** + **T4 Activity/Captures `ddbbb03`** — landed **disjoint, zero
collisions**, no reversal warning. **T3:** `SearchScreen`+`MapScreen` merged into
`features/map/ExploreScreen.tsx` (search-box landing → cards → constellation re-center; internal
search⇄map toggle; all prior behavior carried), `AppShell` **7→6 tabs**, filter chips dropped
(`planes`/`types` API params preserved). **T4:** run-subtree render (recursive `children[]`,
depth-indented early→late), **Captures** feed tab (all sources, expand, source badge, **Remove gated
to chat rows**), Recents→5 + "see all"; **server fold-in** = read-time `node_paths → nodes.id` join
as **`CaptureView.node_refs`** (`LEFT JOIN LATERAL`, no migration — **Alembic head still `015`**) so
capture chips open `NodePreview` via the T2 `NodeChip`. Per-task **fresh independent review**: both
**APPROVE-WITH-MINORS — no code must-fix** (T4's lone "must-fix" was docs-only: the `node_refs` 03-api
entry, applied by the coordinator). **Coordinator** closed the one cross-task seam (the `AppShell`
`ActivityNavContext` "see all" wire, mirroring `reviewNav`) + ran the **merged-tree integration gate**:
ruff+format clean, eslint clean, tsc/vite green, **pytest 822 passed** (+6), **real-PG smoke 156/156**
(+7 `node_refs` checks), head `015`. Docs recorded (03-api `node_refs` addendum · 06 Explore/Captures/
subtree flipped to built · 08 T3/T4 ticked · [08-logs/m8.1.md](../08-logs/m8.1.md) "Batch C — complete").
Follow-ups logged: the `features/map/`→`features/explore/` rename (blocked by `ChatScreen` imports —
post-batch cleanup), tracked `tsconfig.tsbuildinfo` hygiene, review minors. **Code committed locally
through `ddbbb03`, not pushed.** Next: **M8.1 Task 5 · live Accept** (solo) — deploy the T1–T4 range,
verify the Accept block at `braindan.cc` → independent review → **M8.1 CLOSED**.
**M8.1 Task 5 DONE → M8.1 (UI & navigation consolidation) CLOSED (2026-07-17).** Deployed the T1–T4 range
(`5c7a97b..02132a0`, **no migration — Alembic head `015`**; CI green, `/health` all-true, bundle
content-verified) and **verified all six Accept criteria live at `braindan.cc`** in a real authenticated
prod session (agent never handled the login secret): exact-time `<TimeAgo>` tooltip (viewport-clamped),
one collapsed feed row per pipeline run expanding to the recursive step tree (early→late), the single
**Explore** tab (6-tab bar, search→constellation, no Search tab), the **Captures** feed (all sources +
badges, Recents=5 + "See all"), universal **`NodeChip`**→NodePreview→"Explore in map", console clean. The
independent Accept-gate review surfaced **one must-fix** — [ADR-054](../adr/054-m8.1-ui-navigation-consolidation.md)
§5 / the Accept block name "review cards" as a clickable-NodeChip surface, but review-card node references
shipped as **decision controls only**. The user chose to **wire** them (keep §5 literal, not carve out):
commit **`8b25cf6`** makes each uuid-bearing **entity-ambiguity candidate** and **dedup-proposal node** a
`NodeChip`→NodePreview (candidate pick moved to an explicit "Use this"; the dedup survivor radio preserved
— a button-in-`<label>` that previews without toggling; stance name-only refs + vocab values correctly stay
static, NodeChip being **strictly node-uuid-only**). Fix reviewed **APPROVE-WITH-MINORS — no must-fix**,
deployed (`02132a0..8b25cf6`, prod bundle = deterministic local build `index-8fhD_X4z.js`), and verified
via a **real-browser walkthrough vs a throwaway mock API** (the prod Review queue was empty) — both cards
render with chips + intact decision controls, chips open the drawer, the dedup button-in-label leaves the
radio unchanged, console clean. **M8.1 CLOSED — all Accept criteria pass; the UI/nav consolidation is live
at `braindan.cc`** ([08-logs/m8.1.md](../08-logs/m8.1.md) "Task 5"). **Code pushed through `8b25cf6`.** Next:
**M8.2 (data quality)** — build **M8.2 task 1** (per [08 §M8.2](../08-implementation-plan.md)), or respawn.
**M8.2 task 1 DONE (2026-07-17):** the **temporal engine** — the pure-logic core of "LLMs classify,
code computes" ([ADR-056](../adr/056-temporal-correctness-date-tokens.md), CLAUDE.md rule 12; **no
migration/DB/LLM/wiring** — all consumers are T2–T5). New **`server/app/temporal/`** package:
**`symbolic`** (a fail-closed pydantic discriminated union of what the organizer LLM may emit —
explicit/relative/weekday/month/season *classifications*, never a computed date; `parse_symbolic`
never raises), **`resolver`** (deterministic resolution against the capture's **stored anchor** —
offsets, weekday walks, month/year snapping incl. yearless most-recent-past back-walk, N-hemisphere
season windows; unresolvable → `None`, never guessed), **`tokens`** (the `[[t:START[/END][|label]]]`
token + `PartialDate`: parse/serialize, locate-in-body, and the day-granular `occurred` floor/ceil
the DB stores — `occurred_*` stay `date`, tokens own sub-day), and **`render`** (tokens → live
relative phrase + absolute tooltip for web, absolute-only for the indexer, absolute+relative hint
for LLM-bound paths; never shown raw). **Stdlib-only, no `dateutil`** — so the T4 web mirror is
byte-identical (humanizer rounding pinned round-half-up). **57 unit tests, zero mocks** (both
ADR-056 Accept scenarios: "10 days ago"→"a year ago" next year; "last summer"→
`[[t:2025-06/2025-08|summer 2025]]`); **full suite 882 green, ruff clean**. **Independent review
CHANGES-REQUIRED → 1 must-fix** (yearless "29 Feb" against a leap anchor fell through to a *future*
date — rewrote snapping to a year-by-year back-walk → previous leap year or `None`) **+ 2 minors
(round-half-up pin, test adds), all resolved + regression-tested.** Commits `d30d6b6`/`2787701`,
**not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) task 1.
**M8.2 task 2 DONE (2026-07-17):** organizer v6 + migration 016 — the organizer now emits **symbolic
`time_refs` only** (never computed dates, rule 12); `validate_organizer_output` gained an `anchor`
param + a two-pass `arose_from` remap, resolving refs against the capture's **stored anchor**
(`created_local`, threaded through all 3 organize call sites into both prompt and validation) into
**day-granular `occurred`/`occurred_end` + inline `[[t:…]]` body tokens** — fail-closed (unresolvable
→ no token/date, phrase stays prose). Stamps **`interiority`** (`internal|external|mixed`; `external`
default, entity hubs unstamped) and **extracts inner-voice** into its own `internal` node linked to
the event node — **the M8.2 grill resolved the unspecified edge (user call): reuse the seeded
`led_to` rel, edge on the EVENT node → internal node, sibling by local index, NO new vocabulary**
(ADR-055 §2's "existing types" minimalism; recorded in [08](../08-implementation-plan.md) §M8.2 T2, no
new ADR). **Migration 016** adds the nullable `nodes.interiority` column (additive/reversible/no-
backfill, Rule-1 rebuildable); interiority persisted end-to-end (writer render, frontmatter parse,
index upsert). Node contract → `v4`, prompt → `organizer-v6`. **900 tests green** (+18), ruff clean;
migration up/down/up + the real `PgIndexStore` interiority upsert **verified on real PG**;
**independent review APPROVE-WITH-MINORS — no must-fix** (2 minors applied). Commits `afd959f`/`8e85a67`,
**not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) task 2.
**M8.2 task 3 · PART 1 of 2 DONE (2026-07-17):** the **read-side consumers** (user split the 6-part task
— the two data-critical write halves deferred to a fresh session for a focused pass). **(A)** the
**indexer expands `[[t:…]]` tokens to absolute before chunking/embedding** (ADR-056 §4 — vectors + the
FTS `tsvector` see prose, not token noise; `content_hash` stays on the untouched file). **(B) LLM-bound
rendering-contract sweep** — new `temporal.render.temporal_header`/`format_occurred`; **chat context**,
**MCP** `render_search_results`/`render_node`/`render_build_context`, and the **capsule source** all
ship token expansion + a per-item **temporal header** (`recorded … · occurred …`); `SearchHit`/
`NodePreview`/`NodeRow` carry `occurred_*`/`created_at`/`interiority`; **profile-gen + tag/edge
consolidation verified N/A** (no node bodies reach the LLM). **(C) chat interiority boost** (ADR-055
§3a) — `chat_interiority_boost` (1.2) multiplies the fused RRF×recency score for `internal` nodes in
the `scored` SQL, **chat-only** (`/search` neutral at 1.0). **(D) capsule internal slice** (ADR-055 §3b)
— `recent_internal` + a labeled "inner voice" section in the distiller source, counted + provenanced.
**908 unit green** (+8) + **real-PG smoke 156/156** (the changed search SQL runs on live pgvector), ruff
clean; commits `7f2271e`/`edc6e80`, **not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) "Task 3 · PART 1".
**M8.2 Task 3 · PART 2 of 2 DONE (2026-07-17):** the **write-side** half. **(E)** `occurred-enrichment`
review kind — a nightly DB-only flagger (`OccurredEnrichmentService`, idempotent incl. **dismiss-sticks**,
bounded) files a review item per **undated content node**; the NL `answer` is classified into a symbolic
time-ref (`NlTimeClassifier`, `conspect`, rule 12, fail-closed) → the T1 resolver (anchored to the
answer's own date) → the **mechanical** `occurred` apply. **(F)** two-tier edits — `NodeWriter`
`replace_body_token`/`set_occurred_frontmatter`/`edit_time_token` + `ResolvedTime.start/end_date_iso`;
**`PUT /nodes/{id}/date-token`** (mechanical: rewrite body token + move `occurred` iff event date +
re-embed) and **`PUT /captures/{id}/anchor`** (→ background one-capture reorganize, reprocess-safe).
Wired end-to-end (nightly step + scheduler/CLI/roster; `answer` on `POST /review/{id}`). **949 unit
green**, ruff clean; commits `5c4ccd7`/`2911f42`. **Independent review over the whole of Task 3 →
CHANGES-REQUIRED → 2 must-fix fixed** (day-precise event-date `occurred` never moved; a dismissed
enrichment item re-filed nightly) + 1 finding a **false positive** (capsule excerpt is chunk-sourced,
already index-expanded); commit `0417079`. Minors logged (a derived-node date edit is reverted by a
later `reprocess-all` — accepted ADR-042-class caveat, anchor edit is reprocess-safe; capsule internal
double-count; occurred-enrichment SQL wants a T5 real-PG smoke). 03-api pinned (both endpoints + the
`answer` field/kind). **Not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) "Task 3 · PART 2".
**M8.2 Task 3.5 DONE (2026-07-17) — a replan seam.** Starting Task 4 (web) surfaced a plan gap: T3's
consumer sweep wired `interiority` into the LLM-bound renders + chat boost + capsule slice but **never
exposed it on the two web read endpoints**, so the web couldn't render the ADR-055 §3c inner-voice
marker. Rather than fold server work into a "web" task (ADR-006), the user **stopped and replanned the
seam** — grilled, then split out as its own **server Task 3.5 before the web task** (numbering stays
3.5 / 4 / 5; **no new ADR** — contract wiring delivering the already-Accepted ADR-055 §3c). Built
(no migration — the column exists from migration 016): **`GET /nodes/{id}`** now carries `interiority`
(1-line pass-through from the `NodePreview` that already had it) and **`GET /nodes/{id}/neighbors`**
carries it on every neighbor (`MapNeighborItem`, grouped + `?rel=` page) **and** the `center`
(`NeighborCenter`), threaded through the graph-store SQL (`neighbors`/`neighbor_zones`/`center_header`)
+ the `NeighborEdge`/`NeighborHeader` dataclasses (trailing + defaulted → all call sites safe).
`SearchResultItem` deliberately unchanged. Marker semantics pinned for T4: `internal` = full / `mixed`
= subtle / `external`|null = none. **951 unit green** (+2) + **real-PG neighbor smoke 157/157** (added
assertion, not deferred), ruff clean; **independent review (`/code-review high`) — no findings**.
Commit `44fb4e1`, **not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) "Task 3.5".
**M8.2 Task 4 (web) DONE (2026-07-17):** the web half of interiority + temporal correctness — **pure
web, no server code** (ADR-006; consumes the T3.5 fields + the Task-3 edit endpoints). New
**`ui/dateToken.ts`** — a **byte-for-byte web mirror** of the server temporal render (`tokens.py`+
`render.py`; round-half-up pinned, stdlib-only) — feeds **`ui/TokenizedBody.tsx`**, which splits a node
body on its `[[t:…]]` tokens and renders each as a **live relative phrase** (recomputed at render) with a
tap/hover exact-date tooltip via a new shared **`ui/HoverTip.tsx`** (extracted from `<TimeAgo>`, which
became a thin wrapper — rule 10); tokens are **never shown raw**, and where the body is editable
(NodePreview) tapping one opens a **date/range picker → `PUT /nodes/{id}/date-token`** (client-side
fail-closed validation via the mirror; a server 400 surfaces inline). `NodePreview` also gained a
read-only occurred "when" line. **Anchor-edit affordance** on the capture detail (`FeedView`) — a
`datetime-local` editor → **`PUT /captures/{id}/anchor`** (background one-capture reorganize).
**Interiority marker** (ADR-055 §3c) — `ui/interiority.ts` (`internal` full / `mixed` subtle /
`external`|null none) + an "inner voice" pill on NodePreview + a subtle accent-2 **ring** on the Map mark
(threaded through `graphModel`/`MapCanvas`; `SearchResultItem` untouched). **`occurred-enrichment`
review card** — an NL date input → `POST /review/{id}` `{answer}` (maybe/skip; a 400 "couldn't
interpret" is shown verbatim to rephrase). Wire types gained `interiority` on NodeDetail/MapNeighbor/
NeighborCenter + the date-token/`answer` shapes. Gate green (tsc/eslint/vite); the temporal mirror was
**executed** and **both ADR-056 Accept scenarios reproduce exactly** ("10 days ago"→"a year ago" next
year; "last summer"→"summer 2025"); **independent review `/code-review high` — 1 correctness must-fix
fixed** (malformed-token slice off-by-one) + 1 simplification applied + 1 cosmetic logged. The standard
real-browser mock walkthrough was **not run this session** (no browser tooling) — the live interactive
drive is Task 5. Committed locally, **not pushed** — [08-logs/m8.2.md](../08-logs/m8.2.md) "Task 4". Next:
**M8.2 Task 5** — deploy the range (migration 016), prod `reprocess-all-from-raw` (backfills interiority
+ tokens), verify the Accept block live → independent review → M8.2 CLOSED.
**M8.2 Task 5 DONE → M8.2 (data quality: interiority + temporal correctness) CLOSED (2026-07-18).**
Pushed the 11-commit range `6d9a97b..ffd4ca9`; CI green (gitleaks + server pytest + web build), the
deploy step ran `alembic upgrade head` under `set -e` → **migration 016 (`nodes.interiority`) applied**,
`/api/v1/health` all-true. **Prod `reprocess-all-from-raw`** (web Ops **Reprocess everything** card,
confirm→apply): **41/41 captures re-ingested, 0 failed; 143→160 nodes** (the ~17 extra are inner-voice
`internal` nodes — **interiority backfilled**), **800 derived edges**, **84 profiles refreshed inline**
(the M3 "empty profiles" follow-up does not bite), 4 inbox fallbacks, 3 accreted, 0 coerced, push=True;
**2 standing merges NOT re-applied** (ADR-042 §4 caveat — surfaced to the user before the destructive
apply per the data-survival principle, reported in the run summary). Post-reprocess graph-health:
**tombstone-integrity=0 / alias-less=0** (ADR-038 held), orphans 4→1, stale-obs 22→8. **Live Accept**
(real Chrome vs the prod session — agent never handled the login secret): #1 a 2005-dated node renders
its `[[t:2005]]` token as the live phrase **"21 years ago"** + exact-date tooltip **"2005"**, never raw,
while the search snippet shows the index-expanded absolute (ADR-056 §4 dual contract); #4 chat grounded
*"…from 2005 [1]. That's about 21 years ago, from today's date in 2026."*; #5 tap-to-edit picker
opens/parses + live "Reads as" preview; #7 occurred-enrichment cards surface in Review + **fail-closed**
proven ("could not interpret …; try rephrasing"); #8 an inner-voice `internal` node linked via `led_to`
from its event node, "Inner voice" pill on NodePreview + accent ring on the Map. **#6 anchor-edit +
#7 occurred-enrichment SUCCESS writes were not live-demonstrated by user call** (both write real personal
dates) — covered by T3 unit tests + T4 client validation + verified UI. **Independent milestone-close
review (fresh agent over `6d9a97b..ffd4ca9` vs Accept + ADR-055/056 + hard rules): APPROVE-WITH-MINORS —
no must-fix** (rule 12 clean; ADR-042 data-survival clean; `dateToken.ts` mirror faithful/import-free;
reindex-rebuildable; idempotent); 3 minors logged (phrase-splice word-boundary nit; manual date
corrections don't survive reprocess — quieter than the merge caveat; cosmetic coarse occurred line).
**M8.2 CLOSED** — the interiority + temporal-correctness data-quality upgrade is live at `braindan.cc`
([08-logs/m8.2.md](../08-logs/m8.2.md) "Task 5"). Code pushed through `ffd4ca9`. Next: **M9 — Slack
(stance-gated) + Telegram capture** (Telegram pull-forward-eligible); needs a **planning session
(grill to build-ready)** first — 08 §M9 is a stub.

**Where we are (2026-07-18):** **M9 + M9.5 GRILLED TO BUILD-READY** (planning session,
decision-by-decision) — **[ADR-057](../adr/057-multimodal-media-ingestion-substrate.md)** (multi-modal
media substrate: media first-class raw, `vision` routing group Groq-primary, screenshot-attribution
contract, PWA photo capture; video = processed-form-only exception) ·
**[ADR-058](../adr/058-instagram-dm-connector-and-conversation-substrate.md)** (Instagram DM
connector: export-first, local prep tool + opt-in CSV triage, conversation substrate
`connector_threads`/`messages`/`media`, deterministic 6h-gap sessionization — no summary-chaining,
M6 distiller generalized with stance gate + backfill floor/cap protections, parallel-Claude
backfill campaign, targeted re-derive/re-distill, session-transcript traceability, entity-merge
UI, business-account API spike gating a webhook/poller daily fetcher) ·
**[ADR-059](../adr/059-roadmap-restructure-telegram-removed-slack-m12.md)** (roadmap: **Telegram
removed entirely**, **Slack → M12**). Contract docs 00–08 updated; task lists with
parallel-eligibility live in [08 §M9/§M9.5](../08-implementation-plan.md). **Paused before
implementation** per [09](../09-session-protocol.md) — **no code this session.**
**Next:** build **M9 T1+T2** (batch-A: vision routing group ∥ media substrate), or respawn.

**Where we are (2026-07-18):** **M9 batch-A BUILT** (implementation session) — **T1 vision routing
group** + **T2 media substrate** ([ADR-057](../adr/057-multimodal-media-ingestion-substrate.md)). The
4th UI-editable `vision` routing group (Groq VLM `llama-4-scout` primary → Nebius `Qwen2.5-VL-72B`
fallback, effort N/A; OpenAI-compatible provider gains `image_url` parts + N-models-per-endpoint),
the `media` table (migration 017; serves ad-hoc captures now via `capture_id`, connector media at
M9.5), `/srv/data/media/<source>/…` storage (R2-synced free), the resumable status-tracked
derivation stage (photo→`vision`, voice→STT; bounded retries → `unavailable` → placeholder;
targeted re-derive core), the §5 screenshot-attribution description prompt, and session-gated
`GET /media/{id}`. Both tasks passed a **fresh independent review** (PASS, no must-fix; two cheap
hardenings applied). Full suite **977 green**, ruff clean; commits `fff261d`/`b1d1aa5`/`dac5a72`/
`d592cc4` — **code not pushed** (user's call). Build-time pins recorded in
[08 §M9](../08-implementation-plan.md): vision model seeds + the **`media` table name** (was sketched
`connector_media` in ADR-057 §3; reconciled in [02](../02-data-model.md) + the M9.5 T1 bullet).
**Next:** **M9 T3** (image capture pipeline: `POST /capture/image` → describe → organize; wires the
derivation trigger — `depends-on: T1+T2`), then T4 (web) → T5 (live Accept incl. migration apply);
or respawn.

**Where we are (2026-07-18):** **M9 T3 BUILT** (implementation session) — **ad-hoc image capture**
end-to-end ([ADR-057](../adr/057-multimodal-media-ingestion-substrate.md) §3/§5/§6). `POST /capture/image`
(kind `image`) mirrors the voice leg: raw image kept under the media substrate → vision description
**derived** (`derive_until_settled` drives the per-invocation retries so a failure walks
retry→`unavailable`→placeholder **without a human** — closing the "T3/derivation trigger" follow-up)
→ **organized as fenced `<photo: …>`** text (the derived description is the organize/reprocess replay
source, like a voice transcript). The organizer prompt gained the **binding §5
screenshot-attribution rule** (`<photo: …>` content is shared material, never the person's words;
prompt bumped **v7**). New **`redescribe_image_capture`** seam closes the re-derive→graph loop
(re-derive → refresh fenced `raw_text` → reorganize, so a recovered description reaches the **node**,
not just `GET /media/{id}`); its HTTP trigger + live drill land at T5/M9.5. `CaptureView.media`
(read-time join) surfaces the photo + status badge off the capture; new `deriving` capture status.
**No migration** (captures.kind/status are plain text; the `media` table + fk exist from T2).
Independent review **PASS** — one must-fix (re-derive recovered only the media row, not the node)
caught, resolved by the recovery seam, **re-reviewed PASS**. Full suite **991 green**, ruff + format
clean; commit `0d63067` — **code not pushed** (user's call).
**Next:** **M9 T4** (web: capture-strip image affordance + thumbnail/status, photo on capture/node
via `GET /media/{id}`, Settings Vision group verified + the Claude-route guard; `depends-on: T3`),
then T5 (live Accept incl. migration apply + the re-derive drill); or respawn.

**Where we are (2026-07-18):** **M9 REPLANNED — ADR-060 recorded** (planning session, grilled
decision-by-decision). The ask — *see the images/media a node references, in-app and inline, + a
"see raw capture" affordance, voice included* — exceeded the approved T4 (whose "photo on the node"
accept line was unbuildable: media hung off captures with no node→capture reverse path), so it was
grilled + recorded, **not built**. **ADR-060 decisions:** first-class **`node_media`** link
(migration 018; content-nodes-only policy, `MergeCore` repoint, derived-tier — rebuilt by
organize/retry/reorganize/rederive/reprocess), **voice unification** (voice mints `media`, STT
through the T2 derivation engine, **symmetric placeholder-degrade** — `failed` = infra only;
kind-aware **`rederive_capture`**; legacy-voice **relocate+backfill op**), `GET /nodes/{id}.media[]` +
`media_kinds` list glyphs, the **surfacing UX package** (NodePreview media strip + lightbox + shared
capture-detail sheet, themed voice player, glyphs-only lists, nothing on the Map), client-side
**HEIC→JPEG** at capture, video = summary + **1–2 representative keyframe thumbnails** (M9.5
refinement of ADR-057 §2); MCP media exposure = backlog. **M9 restructured:** open T4/T5 superseded
by **T4 (server: substrate + voice) → T5 (web: surfacing) → T6 (live Accept: deploy + backfill +
both-kind re-derive drill)** — strictly sequential; contracts updated (02 / 03 / 04 / 06 / 08). **No
code this session.**
**Next:** build **M9 T4** (server: media–node substrate + voice unification; `depends-on: T3`),
then T5 (web) → T6 (live Accept); or respawn.

**Where we are (2026-07-18):** **M9 T4 BUILT** (implementation session) — server **media–node
substrate + voice unification** ([ADR-060](../adr/060-node-media-linkage-and-voice-unification.md)
§1–§6). The first-class **`node_media`** link (migration 018) makes a node's media visible:
derived-tier, rebuilt on **every** content-node write (organize/retry/reorganize/`rederive_capture`/
reprocess) keyed on the raw-truth `media_id`, **content-nodes-only** (§2), and **repointed
loser→survivor** by `MergeCore` so a merged survivor inherits the loser's media. **Voice unified onto
the T2 derivation engine**: `create_voice_capture` mints a `voice` `media` row under the uniform
`/srv/data/media/capture/…` layout, STT runs through `derive_until_settled`, the transcript mirrors
**plain** to `captures.raw_text` (the person's words, unlike the `<photo: …>` fence); **symmetric
placeholder-degrade** (§6) — a persistent STT failure walks retry → `unavailable` → the
`<voice note — transcript unavailable>` placeholder and organizes anyway (**never `failed`**;
`failed` = true infra only). `redescribe_image_capture` → kind-aware **`rederive_capture`**. Read
side: **`GET /nodes/{id}.media[]`** + **`media_kinds`** glyphs on search results & chat sources.
An **idempotent, degrading voice-media backfill op** (CLI `voice-media-backfill`) relocates legacy
voice audio → mints rows → links `node_media`; wired into `build_capture_pipeline` so a CLI
reprocess-all re-links too. Full suite **999 green**, ruff + format clean; **independent review PASS**
(no must-fix; two minors resolved). Commit `1a1528d` — **code not pushed** (user's call). Live
migration 018 apply + the backfill run are **T6**.
**Next:** build **M9 T5** (web: the surfacing package — capture-strip image affordance, NodePreview
media strip + lightbox + "see raw capture" sheet, themed voice player, list glyphs, HEIC→JPEG,
Settings Vision group + Claude-route warning; `depends-on: T4`), then T6 (live Accept); or respawn.

**Where we are (2026-07-18) — superseded by the T6-tooling snapshot.** **M9 T5 BUILT
(implementation session)** — the web **media surfacing package** ([ADR-060](../adr/060-node-media-linkage-and-voice-unification.md)
§7–§8 + the original T4 web scope). A shared **`ui/media/`** package: the **NodePreview media strip**
(lazy photo thumbnails, themed voice player, `pending` shimmer / `unavailable` broken tiles — never a
silent gap), a full-screen **lightbox** (framer-motion zoom, swipe/Esc dismiss, left/right nav across
a node's photos, reduced-motion aware), and the **"see raw capture" sheet** — a shared
**`CaptureDetailBody`** (source badge, status, media, raw text, NodeChips) that the Activity ›
Captures expanded row now renders too. The **capture screen** gains a photo affordance
(`POST /capture/image`) with **lazy client-side HEIC→JPEG** (`heic2any` dynamic-imported only on a
HEIC pick → synthetic `photo.jpg`). Search-result + chat-source cards get a **📷/🎙 glyph** off
`media_kinds`. **Settings → Models** renders the **Vision group** with the inline **Claude-route
warning**. **tsc + eslint clean, `vite build` green**; **independent review PASS** — one must-fix
(lightbox index reset on ancestor re-render) resolved + one minor simplification applied. Commit
`4adab51` — **code not pushed** (user's call). **Next:** build **M9 T6 — live M9 Accept**.

**Where we are (2026-07-18) — superseded by the M9.6-pivot snapshot (2026-07-19).** **M9 T6 tooling PREPARED; live drills PENDING** (implementation
session). T5 is BUILT (the web surfacing package, commit `4adab51`; see
[status-history](status-history.md)). **T6 is the live M9 Accept** — a deploy + real-device
milestone gated on the operator (push is the user's call; the phone captures are physical), so this
session prepared everything the single T6 deploy needs, then paused. Built: the re-derive drill's
**live trigger** — a new **`rederive-capture <capture_id>` CLI verb** (the `rederive_capture` seam had
no live path; the HTTP endpoint lands at M9.5) that re-runs the VLM/STT → refreshes `raw_text` →
reorganizes so a recovered description reaches the **node**; `build_capture_pipeline` gained opt-in
`wire_media_derivation` (default **off** — reprocess-all still replays stored text, no VLM/STT re-run).
Plus the **media-join SQL smoke** (`deploy/smoke/m9_media_join_smoke.sql`, the open T3 follow-up — the
real joins against the prod DB) and an **executable Accept runbook**
([m9-t6-live-accept-runbook.md](m9-t6-live-accept-runbook.md)) covering every Accept ¶
as operator steps + PASS checks (forced failure is config-only — point the Vision group at a bogus
model, reversible). Commit `2629053` — **code not pushed** (user's call). Suite **1001 green**, ruff
clean; **independent review PASS** (no must-fix; two minors applied). **T6 is NOT done** — it ticks
only when every Accept ¶ is verified **live**.
**Next:** run the live T6 Accept per the runbook — **push**
(user's call) → CI deploy (migrations 017+018 auto-apply) → **`voice-media-backfill`** → real-phone
photo/voice/screenshot drills → group-edit forward-live → the both-kinds failure→placeholder→
`rederive-capture` drill → merge-inherits-media → SQL smoke → final independent review; then tick T6.
Or respawn.

**Where we are (2026-07-19, superseded by the M9.6 build session).** **M9 T6 live Accept STARTED then paused for a planning pivot; M9.6 composite capture GRILLED TO BUILD-READY.** The M9 media stack (T1–T5) is shipped and **live on prod**: the prior session pushed it (`2629053`), CI deployed it (migrations **017+018** applied), the **`voice-media-backfill`** op ran, and **Accept ¶1** (real-phone photo → described/organized media-backed node, inline in NodePreview + lightbox + "see raw capture") + the **session gate** (`GET /api/v1/media/{id}` → **401** without a cookie) verified live. A capture-screen **preview-centering bug** found during ¶1 — the `Lightbox`/`CaptureDetailSheet` overlays are `position:fixed` but were trapped inside framer-motion-transformed capture rows — was fixed by **portaling both overlays to `<body>`** and **shipped** (`8579974`, deployed). Then the user requested **composite multi-part capture**, an architecture change, so per [09](../09-session-protocol.md) the session **switched to a grilling/planning pass** and recorded **[ADR-061](../adr/061-composite-multi-part-capture.md)** + a new **M9.6** milestone (server-side **draft** compose → **one blended organize**; capture = typed text + 0..N photos + ≤1 voice; **per-node media attribution**; `raw_text` stays the cached assembled replay source). **M9 T6 superseded by M9.6.** **Next was:** implement M9.6 strictly sequentially T1→T6. **No composite code written yet** (planning pause).

**Where we are (2026-07-19, superseded by the drill-prep + openRun support session):** **M9.6 composite capture — SHIPPED TO PROD (T1–T5 done + deployed); only the manual live Accept remains.**
A single-session build against the approved plan ([08 §M9.6](../08-implementation-plan.md) +
[ADR-061](../adr/061-composite-multi-part-capture.md)); the user directed all M9.6 tasks in one pass
(no per-task pause), agreeing to pushes + postponing manual/live drills to T6. **T1 (draft lifecycle
+ schema) is committed** (`e785554`, code repo — **not yet deployed**; M9.6 deploys once at T6):
migration **019** (`captures.text_body`, `media.part_ordinal`, `captures_single_active_draft` partial
unique index), the server draft surface (open/resume · part add/remove with ≤1-voice + ordinals ·
text edit · submit ≥1-part gate · discard · 7-day GC; orphan-sweep skips drafts), draft endpoints +
`DraftView`, and a **baseline sequential composite `_process`** so submit works end-to-end. Full
suite **1027 pass**, ruff clean. **T2** (`8ebd2c4`) concurrent-bounded derivation + cached
`[[part N · kind]]` marker assembly + composite `rederive`; **T3** (`d4f1df9`) organizer `parts:[…]`
contract (prompt v8) + per-node attribution; **T4** (`d942432`) CaptureView media→list + `text_body`
+ removed the one-shot endpoints; **independent `/code-review` (high)** on the T1–T4 server diff found
**4 issues, all fixed** (`ba9d465`): `captures.run_id` (migration **020**, live Activity deep-link
replacing an unindexed JSON scan) + two draft-race guards. **T5** (`6ada4f4`) shipped the **web
compose surface** — draft-backed text + multi-photo + record-voice-≤1 + per-part 'x' + Send +
resume/discard, `CaptureView.media` singular→list across the capture surfaces, and the "See
processing →" run deep-link (`tsc`+`eslint`+`vite build` green). **T6: DEPLOYED** — this session
pushed the code (`e785554..6ada4f4`), CI deployed to prod, and automated verification confirms
**migrations 019+020 applied** (`/health` ok, `db:true`) + the **endpoint fold is live** (`POST
/capture/{text,image}` → **404**, `POST /capture/draft`/`…/submit` → **401** gated). Review posture:
server T1–T4 got a high `/code-review` (4 fixed); web got typecheck+lint+build+self-review. See the
**T1–T6 progress notes** in [08 §M9.6](../08-implementation-plan.md). **Next (the user runs it):** the
**manual live Accept** — real-phone composite compose drill (multi-photo + voice + text → cross-
referencing nodes, per-node attribution, unattributed→capture-only), draft resume/discard,
Activity-run deep-link, `reprocess-all` byte-parity, the folded M9 T6 single-part drills, media-join
SQL smoke — then flip T6 done. One UI follow-up: wire `activityNav.openRun` in AppShell so the "See
processing →" link navigates (the field + affordance already ship).

**Where we are (2026-07-19, superseded by the drill-run + M9.7 replanning session):** **M9.6 composite capture — SHIPPED TO PROD (T1–T5 done + deployed); only the manual live Accept (T6) remains, and it's the user's to run.**
The full T1–T5 build + T6 deploy is recorded in [08 §M9.6](../08-implementation-plan.md) and
[status-history](status-history.md) (migrations **019+020** applied live, endpoint fold
verified). **This session was drill prep + the one UI follow-up** — no phone/VPS is drivable from a
fresh agent, so it did the automatable slice and left the manual Accept staged for the user:
- **`activityNav.openRun` wired in AppShell** (the "one UI hop still to land"): the capture "See
  processing →" link now pins that run's `RunDetail` atop Activity→Feed (pagination-proof, reuses the
  existing by-id fetch; ADR-061 §10). Committed to the code working tree, **not pushed** — rides the
  next deploy. `tsc`+`eslint`+`vite build` green; **independent review — no must-fix** (one minor
  dismiss-stickiness gap found + fixed).
- **media-join SQL smoke pre-run via Supabase MCP** — migrations 019+020 confirmed, integrity all
  green (no dangling `node_media`/`media`/`run_id`, 0 tombstone-stranded links, voice backfill
  complete). Caveat: no multi-part composite exists in prod yet (the one indexed composite is
  text-only), so **part-ordinals + per-node `parts:[…]` attribution are first exercised by the phone
  drill**.
- **Runnable drill script written:** [m9.6-accept-drill.md](m9.6-accept-drill.md) —
  step-by-step phone/VPS/SQL commands for the composite compose, resume/discard, deep-link,
  `reprocess-all` byte-parity, folded M9 T6 single-part drills, and SQL smoke.

**Next (the user runs it):** work the drill script → then flip **T6 done** + update this snapshot.
Optionally push the web change first so the deep-link click is live during the drill.

---

**Where we are (2026-07-19, superseded by the M9.7 implementation session — Batch A + B built):** **M9.6 T6 drill partially run — the composite core VERIFIED live;
two FAILs replanned into M9.7 (GRILLED TO BUILD-READY, [ADR-062](../adr/062-chat-screenshot-self-attribution.md)).**
The `openRun` deploy went live first (`6786bb6`), then the user worked the
[drill](m9.6-accept-drill.md): **§A all 7 + §B all 3 PASSED** (ordinals, blended organize,
**per-node attribution**, resume/discard — the first real multi-part composite in prod), §C.1/.3 +
§E.1/.3 passed; still to run: §A DB check, §D, §E.4–6, §F, §G. The two failures, root-caused in
code and grilled into **[08 §M9.7](../08-implementation-plan.md)**:
- **§C.2 — no live "inner running bits"** on the run deep-link: `agent_runs.details` land only at
  finish, the pipeline emits no per-part progress lines, and RunDetail renders neither the existing
  `RunLogTail` nor a parts block. → **M9.7 C**: finish ADR-061 §10 *by reuse* — milestone
  `logger.info` lines (stream via the M8 run-log tail, zero new schema) + RunDetail renders the
  live tail + a structured `derive.parts[]` block.
- **§E.2 — own-chat screenshot misattribution**: the user's own conversation organized as "P1 + an
  unnamed *sender*" (the ADR-057 §5 / organizer-v7 "never you" rule is wrong for the own-chat
  case; no identity signal exists at ingest). → **ADR-062**: vision emits disciplined per-message
  lines (side + sender + reply-quote insets), organizer v8 maps right→the user's own words (no
  self-entity, phantom-sender ban), default own-chat; **A/B** `llama-4-scout-17b` vs
  `qwen2.5-vl-72b` before any routing change; **migration** = rederive+reorganize existing photos.
- **Scope add — general capture remove** (`DELETE /captures/{id}` + double-confirmed UI),
  reversing ADR-048 §11's "backlog": entirely deletes a capture (nodes, media files, everywhere-
  visibility; hubs preserved, tombstone replay-excluded) — needed for A/B hygiene and as product.

**Next:** implement **M9.7** (implementation session, no grilling): **Batch A {T1 vision format,
T2 pipeline logging, T3 RunDetail tail+parts}** → **Batch B {T4 organizer v8, T5 remove server,
T6 remove web}** → **T7 live Accept** (deploy; A/B on the real screenshot; migration rederive;
re-drill §C.2/§E.2 + the remaining drill steps; flip **M9.6 T6 + M9.7** done together).

---

**Status (2026-07-19, M9.7 code-complete snapshot — superseded by the T7-done + M9.8 snapshot).**
**M9.7 code COMPLETE — Batch A + B built, reviewed, committed locally; only T7 (deploy + live
Accept) remains, and it's the user's.** The two M9.6-drill FAILs + the general-remove scope-add
([ADR-062](../adr/062-chat-screenshot-self-attribution.md), grilled to build-ready) were implemented
against the approved [08 §M9.7](../08-implementation-plan.md): **Batch A {T1,T2,T3}**
(`e91d1cb`/`1a8a809`/`ab173db`) — vision per-message screenshot lines, pipeline milestone logging
through the M8 run-log tail, openRun RunDetail live tail + `derive.parts[]`; **Batch B {T4,T5,T6}**
(`8ec8de6`/`dcefee7`/`f83268e`) — organizer v9 self-attribution, `DELETE /captures/{id}` remove,
double-confirm Remove UI. Every task independently reviewed (no must-fix); gate green (1037 pytest;
web tsc+eslint+build); no migrations; commits local on `main`, not pushed. **Next was T7:** deploy;
A/B `llama-4-scout-17b` vs `qwen2.5-vl-72b`; migration rederive; re-drill §C.2/§E.2; remove-drill;
flip M9.6 T6 + M9.7 done. *(What actually happened in T7: the Groq Scout primary was decommissioned
mid-drill → ADR-063 swap to `qwen/qwen3.6-27b` + `reasoning_effort=none`; all Accept items verified;
the duplicate-hub/merge-durability findings became M9.8/ADR-064. See the current README snapshot.)*

---

**Status (2026-07-19, M9.7 done + M9.8-grilled snapshot — superseded by the M9.8-T1-deployed + PII-redaction snapshot).**
**M9.7 + M9.6 T6 CLOSED — deployed, live Accept all verified.** Batch A/B (`e91d1cb`…`f83268e`)
shipped chat-screenshot self-attribution (organizer v9), live per-part capture observability (M8
run-log tail + `derive.parts[]`), and general capture remove (`DELETE /captures/{id}`, double-confirm
UI). T7's live drill surfaced a live-ops issue — Groq decommissioned the `llama-4-scout-17b` vision
primary (404 mid-drill) — fixed by [ADR-063](../adr/063-groq-vision-model-scout-decommissioned.md):
vision primary → `qwen/qwen3.6-27b` with `reasoning_effort=none` (Groq-scoped `extra_body`). T7
Accept verified (own-chat attribution under v9; live processing view; remove proven via Supabase;
migration confirmed). Gate green (1039 pytest).
**M9.8 GRILLED TO BUILD-READY ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md)):** the T7 wrap
found manual entity merges don't survive `reprocess-all` (keyed on node id), the merge UI is unusable
(paste two UUIDs), no entity-hub dedup, graph-health read-only → plan: durable merges (surface-form +
type), a shared visual name-typeahead picker, inline-actionable graph-health, a conservative entity-hub
dedup detector, manual orphan GC.
**M9.8 T1 LANDED (server foundation):** durable, replayable merges ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md) §1)
— `entity_merges` (migration 021) keyed on surface form + type; merge apply upserts, `reprocess-all`
re-applies after the raw rebuild (new `MergeReplayService`, wired into the in-app service + CLI);
unresolvable/ambiguous → skipped. Gate green (1049 pytest, +10). *(What happened next: T1 was pushed
+ deployed to prod; a PII history-rewrite of both public repos + a hashed pre-commit guard landed; the
deploy job was hardened from `git pull --ff-only` to fetch+reset for force-push resilience. See the
current README snapshot.)*

---

**Superseded README snapshot (moved verbatim at the M9.8 T5 pause, 2026-07-19):**

> **Where we are (2026-07-19):** **M9.8 T1 DONE + DEPLOYED TO PROD.** **Durable, replayable merges**
> ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md) §1, migration 021): each merge is recorded in
> `entity_merges` keyed on **surface form + type** (not node id); merge apply upserts the decision, and
> `reprocess-all` re-applies them after the raw rebuild (new `MergeReplayService`, wired into the in-app
> `ReprocessService` + the CLI) — matched by surface form, title-form ranked first so a survivor/loser
> sharing a short alias never cross; unresolvable/ambiguous → skipped (never-lose). Fixes ADR-042 §4's
> silent drop (a name-merge now survives a reprocess). Gate green (1049 pytest, +10); shipped via CI
> (migration 021 applied on prod, app live). Docs: 02 §3, 03 §Admin, 08 §M9.8 T1.
>
> **Next:** continue **M9.8** server foundation — **T4** (conservative entity-hub dedup detector) and
> **T5** (node-delete path); then the shared picker + merge surfaces (T2/T3), inline graph-health (T6);
> T7 live Accept.

---

**Superseded README snapshot (moved verbatim at the M9.8 T4 pause, 2026-07-19):**

> **Where we are (2026-07-19):** **M9.8 T5 DONE (committed, not yet pushed).** T1 remains DONE + DEPLOYED.
> **Node-delete path for orphan hubs** ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md) §5): a new
> `NodeDeleteService` + `POST /admin/nodes/{id}/delete` deletes a genuinely **zero-degree entity hub** via
> git-rm (`NodeWriter.remove_nodes`) + index prune (`NodeDeleteStore.delete_nodes`) + force-commit, under
> an `agent_runs` row (P8) → `202 {run_id}`. Synchronous validation: `404` unknown/tombstone; `400` a
> content node (routed to `DELETE /captures/{id}` capture-remove, so a reprocess can't replay the raw +
> resurrect it); `409` a still-referenced node (routed to Merge). Zero-degree = empty canonical
> `neighborhood` either direction (tombstoned endpoints excluded); a reprocess won't recreate an
> unreferenced hub, so the bare git-rm is never-lose-safe. Independent review: no must-fix. Gate green
> (1058 pytest, +10, ruff clean). Docs: 03 §Admin, 08 §M9.8 T5.
> *(M9.7 + M9.6 T6 + M9.8 T1 closed prior — see status history; M9.8 grilled to build-ready in ADR-064.
> T5 was built parallel-eligible with T1/T4 per the tracker.)*
>
> **Next:** finish the **M9.8** server foundation — **T4** (conservative entity-hub dedup detector →
> inline feed + Review for low-confidence; suppresses the genuinely-different dupe; the last server
> task, paused-for-review before build); then the shared picker + merge surfaces (T2/T3), then
> inline-actionable graph-health (T6); T7 live Accept.

---

**Superseded README snapshot (moved verbatim at the M9.8 T2 pause, 2026-07-19):**

> **Where we are (2026-07-19):** **M9.8 T4 DONE + PUSHED (deploying).** T1/T4/T5 all pushed to `main`
> (CI deploying; no migration — the detector is read-mostly, T5's delete is a gated admin endpoint).
> **The M9.8 server foundation is complete** (T1/T4/T5). **Conservative entity-hub dedup
> detector** ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md) §4): a new nightly `EntityDedupService`
> proposes duplicate **same-type** hubs gated by a **strict AND** — a **name gate** (surface-form
> containment OR high fuzzy match via stdlib `difflib`, low-entropy token guarded) **AND** a
> **shared-neighborhood gate** (≥ N common canonical neighbours). The AND leg suppresses the named false
> positive: **"Diana Wren"** (same first name, different neighbourhood → 0 shared) is never proposed. It
> powers **both** surfaces: **high-confidence** pairs land **inline** in the run's
> `agent_runs.details.high_confidence` (read off the latest `entity-dedup` run like the graph-health card
> → one-click Merge via the existing `POST /admin/entities/merge`, pre-filled from the higher-degree
> survivor); **lower-confidence** pairs file a new **`entity-dedup`** review kind whose **merge** folds the
> loser hub into the survivor **with the alias union** (shared `fold_entities`) **and records a durable
> `entity_merges` decision** (survives a reprocess, §1). **Never auto-merges** (rule 2); a **re-file
> guard** makes a re-scan idempotent. Wired into the nightly pipeline + roster + CLI (`entity-dedup`).
> Gate green (1077 pytest, +19, ruff clean). Docs: 02 §3, 03 §Review + API-addendum, 04 §3b + §Scheduling,
> 08 §M9.8 T4.
> *(M9.7 + M9.6 T6 + M9.8 T1/T5 closed prior — see [status history](status-history.md); M9.8
> grilled to build-ready in [ADR-064](../adr/064-durable-merges-visual-dedup-gc.md). T1/T4/T5 were the
> parallel-eligible server foundation per the tracker.)*
>
> **Repo hygiene (2026-07-19):** a **PII history-rewrite** (`git filter-repo` + force-push) of **both
> public repos** replaced every real contact's full name with a fabricated stand-in (real first name
> kept, so code examples still work: e.g. Horia Fenwick, Diana Vance, Madalina Fairfax); a **hashed
> pre-commit guard** (`.githooks/pre-commit` → `pii_scan.py`, wire with `git config core.hooksPath
> .githooks`) blocks re-introduction. The CI **deploy** step was hardened from `git pull --ff-only` to
> **`git fetch + reset --hard origin/main`** so a force-push no longer wedges the VPS deploy (07-infra).
> ⚠ GitHub may still serve pre-rewrite commits by SHA until GC — verify no forks.
>
> **Next:** the M9.8 **web** work — the shared **visual entity picker** (T2, name-typeahead → id) and the
> **merge surfaces** (T3, profile "Merge into…" + AdminOps upgrade over T2), then **inline-actionable
> graph-health** (T6, per-section Merge/Delete/Keep wired to T3's picker + T5 delete + capture-remove; dupe
> candidates from T4's inline feed). Then **T7 live Accept**.

---

**Superseded README snapshot (moved verbatim at the M9.8 T3 pause, 2026-07-19):**

> **Where we are (2026-07-19):** **M9.8 T2 DONE (committed to `main`, not yet pushed).** The M9.8
> server foundation (T1/T4/T5) is complete + deployed; **T2 begins the web work** — the shared
> **name-typeahead entity picker** ([ADR-064](../adr/064-durable-merges-visual-dedup-gc.md) §2, the reusable
> component every merge surface will use). Discovery: the picker's documented backend
> `GET /entities?q=&type=&limit=` (03-api §Search, [ADR-058](../adr/058-instagram-dm-connector-and-conversation-substrate.md)
> §11) was **specced but never built** — so T2 was slightly wider than the tracker's "web-only" note: it
> adds a thin **server** endpoint (`browse_entities` → new `EntityBrowseService` → the existing
> `EntityStore.list_entities`, ranked in Python by a pure **diacritic-folded name/alias matcher**
> `rank_entity_matches`: exact-title > title-prefix > exact-alias > title-contains > alias-contains, then
> alphabetical; empty `q` = alphabetical browse) that reuses `normalize_alias` so **"madalina"** finds a
> hub written **"Mădălina"** (ADR-041). Read-only, no model — `/search` stays the query-shaped semantic
> surface. **Web:** a reusable `<EntityPicker>` (`ui/`) — controlled name-typeahead that resolves a typed
> name to an entity **id** (no UUIDs), debounced `GET /entities`, keyboard nav, selected-chip + clear;
> `useEntitySearch` hook split out. **Wiring into the merge surfaces is T3.** Self-review (no independent
> agent this pass): no must-fix; read-only, gate green (**1090 pytest**, +14; **tsc + eslint + ruff
> clean**). Docs: 03 §Search & graph (endpoint confirmed live), 08 §M9.8 T2.
> *(M9.7 + M9.6 T6 + M9.8 T1/T4/T5 closed prior — see [status history](status-history.md); M9.8
> grilled to build-ready in [ADR-064](../adr/064-durable-merges-visual-dedup-gc.md).)*
>
> **Repo hygiene (2026-07-19):** a **PII history-rewrite** (`git filter-repo` + force-push) of **both
> public repos** replaced every real contact's full name with a fabricated stand-in (real first name
> kept, so code examples still work: e.g. Horia Fenwick, Diana Vance, Madalina Fairfax); a **hashed
> pre-commit guard** (`.githooks/pre-commit` → `pii_scan.py`, wire with `git config core.hooksPath
> .githooks`) blocks re-introduction. The CI **deploy** step was hardened from `git pull --ff-only` to
> **`git fetch + reset --hard origin/main`** so a force-push no longer wedges the VPS deploy (07-infra).
> ⚠ GitHub may still serve pre-rewrite commits by SHA until GC — verify no forks.
>
> **Next:** **T3 — merge surfaces** (web, `depends-on: T2`): drop the T2 `<EntityPicker>` into the
> **AdminOps** "Merge entities" card (replace its two raw id boxes with two pickers — loser/survivor, the
> survivor picker `excludeId`-ing the loser) and add a **"Merge into…"** affordance on the entity/profile
> view, both feeding the unchanged two-step propose→apply (inbound-edge preview → confirm). Then
> **inline-actionable graph-health** (T6, per-section Merge/Delete/Keep wired to T3's picker + T5 delete +
> capture-remove; dupe candidates from T4's inline feed). Then **T7 live Accept**.
