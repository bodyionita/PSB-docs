# Pipelines & Scheduling

**Version:** 3.1 · **Status:** Approved 2026-07-13 (3.1 = **M3 grilled**
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
   │ • ENTITY RESOLUTION (ADR-030): mentions → alias-index candidates (GIN over
   │   nodes.aliases) → only matching candidates injected as structured fields;
   │   confident → edge (conf/since); < ENTITY_MATCH_MIN_CONF → edge PENDING +
   │   entity-ambiguity review item; new entities minted with aliases/disambig
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
  (DB-side, embedded); **backfill scan** — new/renamed entities re-checked against recent
  unlinked/`inbox/` nodes (≥ threshold auto-edge + feed flag, below → review item).

## 5. Chat / search pipeline (M4 — the grilled chat plan carried, retargeted to nodes)

```
[search]  query → embed (search_query:) → cosine top_k (planes/types filters) → node-grouped results
[traverse] get_node/neighbors — served to the web, the map (M7) and MCP (M5) by one GraphService

[chat]                                             persist user msg BEFORE any model call
   │ turn 1? embed raw msg; else CONDENSE last N turns → standalone query (conspect chain,
   │ low effort; all-down ⇒ degrade to raw msg)
   ▼
   node-grouped top_k (CHAT_CONTEXT_TOP_K, optional planes) → numbered context [1..k]
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
- **Every job**: idempotent, manually triggerable (`POST /agents/{name}/run`), category-tagged,
  **live-observable while running** (status + logs), schedule + next-run visible (`GET /agents`).
  The scheduler decides *when*, never *what*. No cron-only ghosts (01 invariant 4).
