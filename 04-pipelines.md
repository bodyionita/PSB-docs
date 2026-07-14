# Pipelines & Scheduling

**Version:** 3.2 · **Status:** Approved 2026-07-13 (3.2 = prior-art adoptions
[ADR-032](adr/032-prior-art-adoptions.md): resolution short-circuit + entropy guard + intra-capture
dedup; RRF/English-condensation/recency/expansion-guards in chat retrieval; day/night effort
defaults. 3.1 = **M3 grilled**
([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[031](adr/031-m3-organizer-and-contract-extensions.md)):
sync-full organize stays + MCP burst queue; entity resolution flow fixed; profile-refresh +
backfill join the nightly roster. 3.0 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–[029](adr/029-conversational-ingestion-stance-gate-review-queue.md):
organizer emits typed nodes + entity resolution + typed edges; indexing loses the render/strip
machinery and gains canonical-edge materialization; chat retargeted to nodes; **chat-distiller**
pipeline added; summaries absorbed into the reflection agent. Per-pipeline build detail is grilled
at each milestone. 2.x history in git.)

Pipelines are decoupled: each fails, retries and evolves independently. Every step transition is
persisted (`captures.status`, `agent_runs`) — nothing silent (vision P8). All LLM calls resolve
through the provider registry ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md));
routing + effort are UI-editable per group ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md));
STT walks Groq→OpenAI ([ADR-020](adr/020-stt-fallback-chain-groq-primary.md)).

**The organizer is the single writer of graph structure** (01 invariant 7): every pipeline that
creates memory — capture, connectors, chat distillation, MCP — converges on it.

## 1. Capture pipeline (user-initiated, immediate; surfaces: UI voice/text, MCP `capture`)

```
POST /capture/{voice,text}  |  MCP capture(text)
   │ persist raw input + captures row, return 202 instantly
   │ open agent_runs row (agent="capture", running)          ← ADR-021
   ▼
[voice] TRANSCRIBE (STT chain groq→openai)          status=transcribing
   ▼
ORGANIZE — LLM, JSON out (synchronous-full, ADR-031; MCP surface burst-queued)   status=organizing
   │ { nodes: [ { title, type, occurred?, plane, planes[], tags[], body, edges[] } ] }
   │ • typing against the 9-type vocabulary (ADR-027/031); no fit → type=memory
   │   + a vocab-proposal review item
   │ • ENTITY RESOLUTION (ADR-030/032): mentions → alias-index candidates (GIN over
   │   nodes.aliases) → single EXACT alias hit auto-links, NO LLM round-trip;
   │   multi-candidate/fuzzy → LLM with structured candidates; < ENTITY_MATCH_MIN_CONF
   │   → edge PENDING + entity-ambiguity review item; short/low-entropy aliases never
   │   fuzzy auto-link (exact or review); intra-capture dedup pass (one new entity,
   │   not two); new entities minted with aliases/disambig; may close a superseded
   │   edge with `until` (invalidate, never delete)
   │ • occurred extracted only when the text implies a time (partial ISO), never fabricated
   │ • typed edges (involves/about/part_of/led_to/follows/at) targeting node ids
   │ • may SPLIT into multiple atomic nodes; tag-vocabulary reuse (ADR-024, bounded)
   │ • injection hygiene (ADR-031): delimited untrusted text, no node bodies in prompt
   │ • "don't know" → inbox/ node, never guessed; organizer_version stamped
   ▼
WRITE NODES to the graph store (frontmatter contract, 02 §2)   status=written
   ▼
INDEX each node + enqueue store git backup          status=indexed
   │ close agent_runs (model_used, fallback_used, details incl. entity
   │ resolutions + edges created)
```

Failure ⇒ `status=failed` + retry from first incomplete step. Organizer failure fallback: single
node in `inbox/`, title = first 8 words — a capture is never lost to a model error. The follow-up
nudge ([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)) and interaction logging
([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)) carry over unchanged; Pass-2
re-organize replaces the capture's nodes (soft-delete via `git rm`).

## 2. Ingestion pipeline (connectors, scheduled)

Contract + per-connector specs: [05-connectors.md](05-connectors.md). Unchanged shape: fetch
since cursor → shared distiller → organizer-written nodes → advance cursor after materialization.
Two pivot changes: **(a)** distillation output is typed nodes + edges through the organizer;
**(b)** people-conversation distillation adopts the **stance gate** — the user's commitments/
agreements/decisions are the anchor; stance-unclear candidates go to the **review queue**
instead of being guessed ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).
Default lookback: **6 months**, per-connector UI override.

## 3. Chat-distillation pipeline (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md))

```
nightly (agent window) — cursor over ended/idle chat sessions   agent="chat-distill"
   │  (manual per-session trigger: POST /chat/sessions/{id}/remember)
   ▼
SALIENCE GATE — new info / decision / reflection / intention?  no → skip (logged)
   ▼
STANCE-FIRST DISTILL — anchor on the user's messages
   │ endorsed  → organizer ingests (conversation/insight nodes, edges to the
   │             nodes discussed; source=chat, source_ref=session id)
   │             → flagged in the feed with one-tap remove
   │ rejected  → dropped, kept in run log
   │ unclear   → review_queue (agree/disagree/maybe; agree → organizer)
   ▼
finish agent_runs ("3 sessions read, 1 insight recorded, 2 skipped")
```

## 4. Indexing pipeline

Triggers: capture/ingestion writes · nightly `reindex` · `POST /admin/reindex`.

```
file → read → sha256 whole file ── unchanged? skip
     → parse frontmatter (id/type/plane/planes/tags/edges/…)
     → strip frontmatter → chunk (02 §4) → batch-embed ("search_document: …")
     → per-node TRANSACTION: delete old chunks + upsert node + insert chunks
       + nodes.embedding = mean-pool + MATERIALIZE canonical edges (frontmatter → edges table)
```

- Self-hosted nomic, single provider ([ADR-022](adr/022-embeddings-self-hosted-nomic.md));
  skip-and-continue on embed failure → run `partial`; deletions reconciled (id-keyed — a moved
  file is a path update, not delete+insert); fully idempotent.
- **Derived `similar` edges** recomputed nightly (+ on `/admin/reindex`): top-K cosine over
  `nodes.embedding` above floor → `edges (origin=derived)`. **DB-only — no file rendering, no
  commit step, no churn-gating** (deleted by [ADR-026](adr/026-graph-native-storage-obsidian-removed.md)).
- The nightly `reindex` job: `git pull` store → full pass → recompute derived edges. Single-flight
  with `/admin/reindex`.
- **Nightly entity jobs (M3, [ADR-030](adr/030-entity-substrate-and-lifecycle.md)):**
  **profile-refresh** — regenerate derived profiles for entities whose 1-hop neighborhood changed
  (DB-side, embedded — the profile embedding is a **search retrieval source**, so a full DB wipe
  leaves profile-search degraded until this reruns; [ADR-037](adr/037-profile-embedding-in-search-m3.md));
  **backfill scan** — new/renamed entities re-checked against recent
  unlinked/`inbox/` nodes (≥ threshold auto-edge + feed flag, below → review item).
- **Vocabulary consolidation (M3, on-demand — [ADR-027](adr/027-typed-vocabulary-governance.md) §3 /
  [ADR-035](adr/035-vocabulary-consolidation-scope-m3.md)):** approving a type proposal (via `PUT
  /settings/vocabulary` or the Review queue) writes the addition to `app_settings` (effective vocab =
  seeds ∪ approved, forward-live) and opens the `vocab-consolidation` job. On-demand retro-walk
  (`POST /admin/vocab/consolidate`, ADR-024 propose→apply): for a new **edge rel** proposes + (on
  confirm) applies **re-typings of existing edges** — frontmatter `rel:` rewrites, bounded edge
  inventory ([ADR-036](adr/036-edge-retro-consolidation-walk-retypings-only-m3.md); inventing
  brand-new edges from node bodies is a deferred follow-up); for a new **node type** surfaces
  candidate re-typings **propose-only** in M3 (the folder-move/re-slug apply machinery is a deferred
  follow-up, ADR-035).

## 5. Chat / search pipeline (M4 — the grilled chat plan carried, retargeted to nodes)

```
[search]  query → embed (search_query:) → cosine top_k over chunks ⊍ node_profiles.embedding
          (best-per-node union, ADR-037; planes/types filters) → node-grouped results
[traverse] get_node/neighbors — served to the web, the map (M7) and MCP (M5) by one GraphService

[chat]                                             persist user msg BEFORE any model call
   │ turn 1? embed raw msg; else CONDENSE last N turns → standalone query IN ENGLISH
   │ (conspect chain, low effort; all-down ⇒ degrade to raw msg — ADR-032: the corpus is
   │ English; a Romanian tsquery matches nothing, vectors stay cross-lingual)
   ▼
   M4 retrieval (ratified + ADR-032): vector top_k ⊍ tsvector FTS fused by RRF (rank-based,
   k=60; degenerate-signal suppression; FTS weight→0 on non-English raw queries) + mild
   recency prior on occurred ?? created + 1-hop edge-neighbor {rel,title,type} injection
   (config-capped) + entity-seeded expansion (falls back to vector/FTS seeds, never a hard
   gate; expansion function PPR-swappable) → numbered context [1..k]
   ▼
   chat model (Settings chat group; per-conversation picker overrides active model)
   ▼
   answer with [n] → keep ONLY cited nodes, renumber [1..m]
   ▼
   persist assistant msg (model = model_used, sources = cited nodes)
```

Prompt rules: hybrid, grounding-biased; "that's not in your memories" when context is silent;
reply in the user's language. Non-streaming + client-side reveal. **Backlog:** graph-aware
context (one-hop canonical-edge expansion of retrieved nodes), then agentic traversal.

## 6. Reflection & life-manager agents (M10/M11 — scope recorded, grilled at their milestones)

- **Reflection agent (M10):** absorbs the old daily-summary/weekly-review — one agent, multiple
  timescales (1d/1w/1m/1y): what went well, what to work on, improvements. Output = `insight`
  nodes through the organizer; morning **push notification** + on-demand asks.
- **Life-manager agent (M11):** schedule/tasks/goals across planes — full grilling session
  before build (node types? calendar integration? advisor vs state-manager?).

## Scheduling policy ([ADR-010](adr/010-agent-window-3-5am.md) + the jobs-observability contract)

- Heavy agent work runs **03:00–05:00 Europe/Bucharest**, staggered: connectors → reindex →
  distillers → reflection; store backup sweep 04:55 + debounced after every write batch.
- **Day/night effort split ([ADR-032](adr/032-prior-art-adoptions.md), via ADR-025 routing):**
  the sync capture-organize defaults to lower effort; nightly consolidation/profile/reflection
  jobs default to higher effort — cheap by day, strong by night.
- **Every job**: idempotent, manually triggerable (`POST /agents/{name}/run`), category-tagged,
  **live-observable while running** (status + logs), schedule + next-run visible (`GET /agents`).
  The scheduler decides *when*, never *what*. No cron-only ghosts (01 invariant 4).
