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
routing + effort are UI-editable per group ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)) —
a group's chain is a chain of **model ids** ([ADR-045](adr/045-provider-model-effort-separation.md): provider ≠
model; `claude` is one provider serving Opus + Sonnet, resolved to its provider + vendor string per call);
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
   │   + a vocab-proposal review item. ENTITY TYPES ARE MENTION-ONLY (ADR-039): a
   │   person/place/topic/event/project node is coerced to `memory` (body/mentions kept)
   │   — prompt rule + deterministic guard; content types = memory/idea/insight/conversation
   │ • ENTITY RESOLUTION (ADR-030/032/040): mentions → alias-index candidates — the EXACT
   │   leg (normalized title/alias) PLUS a TOKEN-OVERLAP leg (a hub sharing a significant
   │   token, so "Horia Fenwick" surfaces the "Horia" hub; low-entropy guard: short/stop
   │   tokens never fan out). A single EXACT hit auto-links, NO LLM; a fuzzy candidate →
   │   LLM with structured candidates; < ENTITY_MATCH_MIN_CONF → edge PENDING +
   │   entity-ambiguity review item (never a fuzzy auto-link). On a confirmed link under a
   │   new surface form → ALIAS ACCRETION (append to the hub's aliases, folded, feed-
   │   visible). Intra-capture dedup; new entities minted with aliases/disambig; may close
   │   a superseded edge with `until` (invalidate, never delete)
   │ • all derived text diacritic-FOLDED to ASCII at the writer (ADR-041); raw kept
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
re-organize replaces the capture's **content** nodes (soft-delete via `git rm`) but **never its
entity hubs** — the removal is type-aware ([ADR-038](adr/038-reorganize-preserves-shared-entity-hubs.md):
hubs are shared substrate; the fresh pass re-links to the live hub, orphans tolerated for a later GC).

**Reprocess-all-from-raw** (`POST /admin/reprocess`, [ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md),
the vision-P10 data-survival op). Confirm-gated + single-flight; runs in the background with an
`agent_runs` row. **Reset** the derived state — every node file, the DB index
(`nodes`/`chunks`/`edges`/`node_profiles`), the alias index, the `review_queue`, and
`captures.node_paths` — while **preserving** raw `captures`, the store git history, and approved
vocabulary (`app_settings`); standing merges are **reported, not dropped**. Then **replay every
capture's raw input chronologically** (combined text where a follow-up was answered) through the
current pipeline — each replay is a normal organize→resolve→write→index, so it inherits every fix
and alias accretion rebuilds deterministically — recompute derived edges, **rebuild the derived
profiles the reset truncated** (a profile-refresh over the replayed graph, so the profile search leg
[ADR-037] is live immediately rather than empty until the nightly job), and force one commit+push.
The run reports per-heal totals (`coerced`/`accreted`/`profiles_refreshed`) for auditability.
Idempotent; raw is never touched, so a bad reprocess is recovered by fixing code and re-running.

## 2. Ingestion pipeline (connectors, scheduled)

Contract + per-connector specs: [05-connectors.md](05-connectors.md). Unchanged shape: fetch
since cursor → shared distiller → organizer-written nodes → advance cursor after materialization.
Two pivot changes: **(a)** distillation output is typed nodes + edges through the organizer;
**(b)** people-conversation distillation adopts the **stance gate** — the user's commitments/
agreements/decisions are the anchor; stance-unclear candidates go to the **review queue**
instead of being guessed ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).
Default lookback: **6 months**, per-connector UI override.

## 3. Chat-distillation pipeline (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md) · build decisions [ADR-048](adr/048-m6-chat-distiller-build-decisions.md))

```
nightly `chat-distiller` step — sessions idle since chat_distill_idle_hours
   │  (eligible: max(chat_messages.created_at) < now − idle; watermark in chat_distill_state)
   │  (manual per-session trigger: POST /chat/sessions/{id}/remember — sync distill, async organize)
   ▼
SINGLE-PASS MULTI-CANDIDATE DISTILL (conspect, night effort; input FENCED)
   over the session's NEW turns (after the watermark) → a LIST of candidates, each
   {candidate_text, stance, salience(high|med|low), evidence_excerpt, referenced_entity_names}.
   Segmentation is emergent (list-of-candidates, not a session summary); pure-retrieval → 0 candidates.
   ▼
SALIENCE GATE — zero candidates? → session skipped (logged)
   ▼
STANCE (anchored on the USER's messages; hedge/sarcasm/affect → unclear, never guessed)
   │ endorsed  → materialize a `captures` row (raw=candidate_text, source=chat,
   │             source_ref=session-id, created_at=anchoring-msg time) → ORGANIZER
   │             (types/links normally; conversation/insight + about/involves/led_to)
   │             → recorded in the "recently auto-recorded" list, one-tap remove
   │             → REPLAYED BY reprocess-all ⇒ P10 holds (survives bugfixes)
   │ unclear   → review_queue kind=stance-candidate (agree/disagree/maybe;
   │             agree = the SAME captures→organizer path; maybe re-openable)
   │ rejected  → run-log detail only (never a node, never a review item)
   ▼
advance watermark past materialized work (delta-only, idempotent re-distill)
finish agent_runs ("3 sessions read, 1 recorded, 1 to review, 1 skipped")
```

**Durability (P10).** An endorsed chat memory is a `captures` row, so it is indistinguishable
downstream and `reprocess-all` rebuilds it for free; `reprocess-all` is **kind-aware** — it
preserves `stance-candidate` review items (chat sessions aren't replayed by capture-replay) and
truncates the capture/graph-derived kinds (`entity-ambiguity`/`vocab-proposal`/`dedup-proposal`,
re-derivable). **Remove** = git-rm node file(s) + DB-delete + **tombstone the capture**
(`removed_at`) so replay/reprocess can't resurrect it.

## 3b. Dedup sweep + inbox drainer (M6 sleep-cycle jobs, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md))

- **`dedup-sweep`** (nightly, all-source, after reindex so embeddings exist —
  [ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md)): recently-ingested **content** nodes
  (`indexed_at ≥` the last-success watermark, no migration) whose HNSW top-K neighbour clears a
  **strict AND** of **high cosine (`nodes.embedding`) + a shared canonical edge to a common entity
  hub + occurred-overlap** (a null `occurred_start` never excludes) → a `dedup-proposal` review item
  `{node_a, node_b, signals, default_survivor}` (higher canonical-degree / older). A **re-file guard**
  skips any pair that already has a `dedup-proposal` in any status (a merged pair self-excludes via
  its tombstone). Actions: **keep** (dismiss) / **link** (a **canonical `similar`** edge — persists
  where the derived one is recomputed away) / **merge** (fold — an **extracted merge-core** shared
  with entity-merge: retarget inbound edges → tombstone loser `merged_into: survivor` → reindex →
  force-commit; content-merge = core only, no alias union). `augment` ("same event, new fact")
  deferred. Pipeline wiring is M6 task 8.
- **`inbox-drainer`** (nightly, before reindex, bounded): captures materialized as an `inbox/`
  fallback (`captures.node_paths` in `inbox/`) → `reorganize_capture` with the now-richer entity
  registry; replaced only on success, still-failing stays in `inbox/`. Residual ambiguity files
  the normal `entity-ambiguity` items.

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
   │ turn 1? embed raw msg; else CONDENSE last N=15 turns (config) → standalone query IN
   │ ENGLISH (conspect chain — inherits the conspect group's effort, ADR-025 is per-group not
   │ per-call-site; all-down ⇒ degrade to raw msg — ADR-032: the corpus is English; a
   │ Romanian tsquery matches nothing (FTS self-suppresses), vectors stay cross-lingual)
   ▼
   M4 retrieval (M4 kickoff grill — lean spine): vector top_k ⊍ tsvector FTS (over chunks ⊍
   node_profiles, migration 008) fused by RRF (rank-based, k=60; FTS self-suppresses on the
   English corpus + skips on a zero-lexeme tsquery — no language-detect dep) + mild recency
   prior on occurred ?? created (bounded multiplicative, floor ~0.9, config half-life) →
   numbered context [1..k]. (Graph-aware expansion — 1-hop edge-neighbor injection + entity-
   seeded, PPR-swappable — is DEFERRED to backlog; see 08 §Backlog.)
   ▼
   FENCE each numbered context item as data-not-instructions (injection hygiene, before
   connector/MCP content shares this path); chat model (Settings chat group; per-conversation
   picker overrides active model)
   ▼
   answer with [n] → keep ONLY cited nodes, renumber [1..m] (out-of-range [n] dropped, never errors)
   ▼
   persist assistant msg (model = model_used, sources = cited nodes)
```

Prompt rules: hybrid, grounding-biased; **general questions answered uncited** (empty `sources`);
"that's not in your memories" for personal questions with no hits (prompt-driven + `min_score`
floor, no classifier); reply in the user's language. Non-streaming + client-side reveal; the web
surfaces a **fallback/model banner** (`fallback_used`) and a **"not from your memories" chip** (empty
`sources`). Session **titling** runs on the **`quick`** tier ([ADR-043](adr/043-quick-routing-tier-m4.md)),
best-effort + non-blocking after the first exchange. **Backlog:** graph-aware context (one-hop
canonical-edge expansion of retrieved nodes), then agentic traversal.

## 6. Reflection & life-manager agents (M10/M11 — scope recorded, grilled at their milestones)

- **Reflection agent (M10):** absorbs the old daily-summary/weekly-review — one agent, multiple
  timescales (1d/1w/1m/1y): what went well, what to work on, improvements. Output = `insight`
  nodes through the organizer; morning **push notification** + on-demand asks.
- **Life-manager agent (M11):** schedule/tasks/goals across planes — full grilling session
  before build (node types? calendar integration? advisor vs state-manager?).

## Scheduling policy ([ADR-047](adr/047-pipeline-scheduling-primitive.md) pipelines + [ADR-010](adr/010-agent-window-3-5am.md) window + the jobs-observability contract)

**The pipeline is the only schedulable unit (M5.5, [ADR-047](adr/047-pipeline-scheduling-primitive.md)).**
A **pipeline** = a name + one cron + an ordered list of **steps**, each with an `on_fail` policy
(`continue`|`halt`). The runner runs steps **sequentially, each starting only when the previous
completes** — one start time, dependency order guaranteed regardless of step duration, one step's
RAM at a time. A bare job is **never** cron-scheduled; even single-step work is wrapped in a
pipeline. Cadence maps to a pipeline:
- **`nightly`** (one start, e.g. 03:00): chat-distiller → data-sync → db-backup → inbox-drain →
  reindex → profile-refresh → backfill → identity-capsule → dedup-sweep → store-sweep → bundle.
- **`weekly`** (Sunday): integrity-drill; maybe-digest (M6).
A pipeline run opens a **parent `agent_runs`** row; each step keeps its **own child run**
(`agent_runs.parent_run_id`). Jobs stay independently invokable (CLI + `POST /agents/{name}/run`,
invariant 4); the scheduler registers **one cron per pipeline**. Ordering rationale: the distiller
writes nodes before reindex; inbox-drain enriches entities before the entity jobs; dedup needs
post-reindex embeddings. Richer visualization → M8.

**A step's status is its own job's run, not the transitive scope
([ADR-050](adr/050-pipeline-step-status-is-the-jobs-own-run.md)).** A step that funnels through the
organizer (the chat-distiller's endorsed captures, the inbox-drainer's reorganize — rule 2b) spawns
nested `agent="capture"` runs; a capture that degrades to the `inbox/` fallback closes its run
`failed` (rule 7, ADR-021) even though the data is safe. Those nested runs stay parented to the
pipeline parent and visible in the feed, but the step's pass/fail is read from **its own** run (the
one whose `agent == step.name`) — so a benign inbox fallback of one capture never reads as a failed
step and never trips an `on_fail=halt`. A step that raises still fails (halt still aborts on it); a
step that opens no own run (`store-sweep`) still reads `skipped`.

- Heavy agent work runs inside the **03:00–05:00 Europe/Bucharest** window (ADR-010), now enforced
  by *sequencing from one start*, not by hand-tuned stagger; store backup sweep near the end +
  debounced after every write batch.
  **`identity-capsule-refresh` joins the roster at M5** ([ADR-046](adr/046-m5-mcp-server-oauth-connectors.md)/[ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)
  #1 — runs after profile-refresh so it distills over fresh hubs; `conspect` tier, on-demand
  triggerable like every job; `build_context` serves the last-generated blob, never inline).
- **Day/night effort split ([ADR-032](adr/032-prior-art-adoptions.md), via ADR-025 routing):**
  the sync capture-organize defaults to lower effort; nightly consolidation/profile/reflection
  jobs default to higher effort — cheap by day, strong by night.
- **Every job**: idempotent, manually triggerable (`POST /agents/{name}/run`), category-tagged,
  **live-observable while running** (status + logs), schedule + next-run visible (`GET /agents`).
  The scheduler decides *when*, never *what*. No cron-only ghosts (01 invariant 4).
