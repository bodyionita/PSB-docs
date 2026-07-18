# Data Model

**Version:** 3.2 · **Status:** Approved 2026-07-13 (3.2 = prior-art adoptions
[ADR-032](adr/032-prior-art-adoptions.md): edge `until`, observation-style profiles.
3.1 = **M3 grilled to build-ready**
([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[031](adr/031-m3-organizer-and-contract-extensions.md)):
aliases/disambig + `occurred` + edge `{conf,since}` + `organizer_version` in the contract; 9 node
types / 6 edge rels; `review_queue` pulled into M3, kind-generic; migration 005 DDL fixed.
3.0 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)/[027](adr/027-typed-vocabulary-governance.md):
typed-node **graph store** replaces the note vault (fresh start — old vault archived); `nodes`/`edges`
replace `notes`/`note_links`; wikilinks, `sb:related` blocks and folder-=-plane are deleted. The
concrete migration (005) is authored at M3 implementation; **exact DDL details are finalized at the
M3 build-ready grilling**. 2.x history: M3 chat/routing keys, M2 embeddings 768 + relatedness graph,
M1 capture columns — see git history.)
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [002](adr/002-supabase-pgvector-for-index.md) · [005](adr/005-planes-and-atomic-notes.md) (surviving half) · [022](adr/022-embeddings-self-hosted-nomic.md) · [025](adr/025-ui-editable-model-routing-and-per-task-effort.md) · **[026](adr/026-graph-native-storage-obsidian-removed.md) · [027](adr/027-typed-vocabulary-governance.md) · [029](adr/029-conversational-ingestion-stance-gate-review-queue.md)**

## 1. Graph store layout

```
<GRAPH_STORE_PATH>/                # on the VPS; git repo = PSB-graph (private GitHub)
├── memory/                        # node types = top-level folders (config-driven)
│   └── 2026-07-13--dinner-with-alex--018f3c2e.md
├── person/
│   └── alex-marsh--018f4a11.md
├── idea/ · conversation/ · insight/
├── place/ · event/ · project/ · topic/    # seeded at M3 (ADR-031)
└── inbox/                         # system folder: organizer-fallback nodes, until re-organized
```

- **Folder = node type** ([ADR-026](adr/026-graph-native-storage-obsidian-removed.md)). The type
  vocabulary is config + approved additions ([ADR-027](adr/027-typed-vocabulary-governance.md));
  adding a type is config/approval, not code. `inbox/` is the one system folder — nodes the
  organizer couldn't classify land there (never guessed) and move out on re-organize.
- **Planes are attributes, not folders** — `plane`/`planes[]` frontmatter is membership truth
  (the surviving half of [ADR-005](adr/005-planes-and-atomic-notes.md)).
- **Identity = frontmatter `id`** (uuid). **Filename = `slug--shortid.md`** (memory nodes prefix
  the date for readability) — a readable projection, rename-safe; the indexer keys on `id`,
  never on path.
- The indexer indexes every `*.md` except `STORE_IGNORE` prefixes (default: `.trash`, `.git`,
  `templates`).

## 2. Node frontmatter contract

Pipeline-written nodes always carry:

```yaml
---
id: 018f3c2e-…                  # THE identity (capture id / ingest item id)
type: memory                    # one of NODE_TYPES (9 seeded — ADR-031)
created: 2026-07-13T21:40:00+02:00
occurred: 2025-07               # OPTIONAL event time, partial ISO (2025 | 2025-07 | 2025-07-10)
occurred_end: 2025-08           # OPTIONAL range end; time axes use occurred ?? created
source: voice                   # voice | text | slack | chat | mcp | agent | …
source_ref: "slack:C0123/p1720771234.5678"   # opaque locator (optional)
plane: personal                 # primary plane (attribute, not folder)
planes: [personal, friends]     # full membership; superset of `plane`
tags: [dinner, catching-up]
organizer_version: v3           # which organizer generation wrote this (retrofit targeting)
edges:                          # canonical typed edges, target = node id (ADR-030/031/032)
  - {rel: involves, to: 018f4a11-…, since: 2025-07-10}
  - {rel: part_of,  to: 018f6c33-…}          # conf omitted ⇒ 1.0; <threshold ⇒ review, no edge
  - {rel: at, to: 018f7d44-…, until: 2025-08-02}   # `until` closes a superseded relation —
---                                                # invalidate, never delete (ADR-032)
(Markdown prose body — the memory itself)
```

Entity-hub nodes (`person`, `place`, `topic`, `event`, `project` — the `ENTITY_LIKE_TYPES` set)
additionally carry ([ADR-030](adr/030-entity-substrate-and-lifecycle.md)):

```yaml
aliases: [alex, alexandru, "my brother"]   # resolver-maintained surface forms (accreted on link)
disambig: "younger brother, b.1994"        # one line, resolves same-name collisions
```

> **`idea` is a content type, not an entity** ([ADR-039](adr/039-entity-types-are-mention-only.md),
> M3 task 11): the resolver mints hubs only for `person`/`place`/`topic`/`event`/`project`, and the
> organizer may never emit any of those as a content node (a person/place/… is expressed **only** as
> a mention on a content node; a structural guard coerces a mis-typed one to `memory`). Content nodes
> are `memory`/`conversation`/`insight`/`idea`.

> **M8.2 addendum (exact shapes land with the build):** content nodes gain
> **`interiority: internal | external | mixed`** ([ADR-055](adr/055-interiority-inner-voice-first-class.md) —
> frontmatter + a `nodes` column; the organizer also *extracts* inner-voice content into its own
> `internal` nodes, edge-linked to their event node), and body prose may carry inline **date tokens**
> `[[t:START[/END][|label]]]` ([ADR-056](adr/056-temporal-correctness-date-tokens.md) — partial-ISO,
> ranges, optional time-of-day, absolute labels; resolved deterministically against the capture's
> stored anchor, never by the LLM; renderers expand, the indexer expands **before embedding**, raw
> captures untouched). `occurred`/`occurred_end` stay date-granular — tokens own sub-day.
> **Alias accretion** ([ADR-040](adr/040-token-overlap-retrieval-and-alias-accretion.md)): when a
> mention links to a hub under a surface form not already in its `aliases` (confirmed by the resolver
> LLM or a human review pick), that form is appended — so the exact short-circuit covers it next
> time. Short/low-entropy forms are never accreted.

Their files are **thin hubs** — the readable "who/what is X now" profile is **derived**
(regenerated nightly from the 1-hop neighborhood, DB-side, embedded for search, served by
`GET /nodes/{id}`), never LLM-accreted into the file. Tombstones after a merge keep only
`merged_into: <survivor-id>`.

Rules:
- **Edge vocabulary** (seeded: `involves`, `about`, `part_of`, `led_to`, `follows`, `at` —
  [ADR-031](adr/031-m3-organizer-and-contract-extensions.md)) is governed like node types — LLM
  proposes, user approves, consolidation retrofits
  ([ADR-027](adr/027-typed-vocabulary-governance.md)). Edges are directional, written on the
  source node; `part_of` targets event/project/conversation; `at` targets place. **Similarity
  is never written into files** — derived `similar` edges live only in the DB.
- One source may produce **several atomic nodes** (split per plane/topic); siblings share
  `source_ref` and are connected by typed edges (no wikilinks, no `related:` field).
- **English-only store** (organizer translates; raw input preserved verbatim in
  `captures.raw_text`; conversational UI mirrors the user's language).
- **Tag slug rules kept on their own merits** (ex-"Obsidian tags"): English, lower-case, single
  word or hyphenated, no spaces, chars `a–z 0–9 _ - /`, must contain a letter; enforced by
  `_slugify_tag`.
- **Zero diacritics in the store** ([ADR-041](adr/041-diacritic-folding-derived-content.md), M3 task
  11): every derived field — filename slug, title, aliases, disambig, tags, **and body prose** — is
  NFKD-folded to ASCII at the single `NodeWriter` write chokepoint (`"Mădălina"` → `madalina`,
  never `m-d-lina`). Matching (`normalize_alias`, the alias index, tag slugs) folds too, so ASCII
  and raw-diacritic forms compare equal. The **raw capture is never folded** — it is the never-lose
  source of truth and what `reprocess-all-from-raw` replays.
- Hand-created nodes may omit any field; every field is optional at read time. Title = H1 if
  present, else filename stem. Missing `type` = `memory`.

## 3. Database schema (Supabase Postgres + pgvector)

Embedding dimension **768** (self-hosted `nomic-embed-text-v1.5` via Ollama,
[ADR-022](adr/022-embeddings-self-hosted-nomic.md)); `embedding_dim`/`embedding_model` are
settings — provider change = deliberate migration + reindex.

**Migrations:** Alembic, explicit SQL, no ORM ([ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md)).
**M3 (pivot) ships migration 005** ([ADR-031](adr/031-m3-organizer-and-contract-extensions.md)):
drops the note-model derived tables; creates `nodes`/`edges`/`review_queue`; renames
`captures.note_paths → node_paths` (fresh start — no data migrated, old vault archived).
New config: `GRAPH_STORE_PATH`, `GRAPH_STORE_REPO`, `NODE_TYPES` (9), `EDGE_RELS` (6),
`ENTITY_MATCH_MIN_CONF` (0.8, live-tuned), MCP burst-queue + profile-refresh settings.
**Migration 006** (task 6): `node_profiles` (derived entity profiles — tier, text, observations,
`neighborhood_hash`, `embedding`, fk → nodes cascade). **Migration 007** (task 10,
[ADR-037](adr/037-profile-embedding-in-search-m3.md)): HNSW `vector_cosine_ops` index on
`node_profiles.embedding` so the profile leg of search is ANN-indexed like `chunks`.
**Migration 008** (M4 retrieval): generated `tsvector` columns for the hybrid FTS leg —
`chunks.tsv GENERATED ALWAYS AS (to_tsvector('english', text)) STORED` + a matching
`node_profiles.tsv` over the profile text (mirrors the ADR-037 vector union so RRF fuses the same
node universe), each with a **GIN** index. Generated + store-derived, so `POST /admin/reindex`
restores them for free (Rule 1 clean); the `'english'` config matches the asserted English corpus.
**Migration 009** (M4 follow-up 3, [ADR-045](adr/045-provider-model-effort-separation.md) §4): a one-row
**data** migration of the saved `app_settings.model_routing` jsonb — remaps the old provider ids to model-id
vendor strings (`claude-max`→`claude-opus-4-8`, `claude-max-sonnet`→`claude-sonnet-4-6`,
`nebius`→`meta-llama/Llama-3.3-70B-Instruct`) in `active`/`fallback` + the effort object's keys, and renames the
key `effort_by_provider`→`effort_by_model`. Idempotent ordered text-substitution (plain SQL, ADR-011); a
`WHERE … ~ '<old-tokens>'` guard makes it a **no-op** on an absent/empty/already-migrated row. Preserves a
deliberate routing choice across the rename (vision P10); historical `chat_messages.model` audit rows are **left
untouched** (label resolution tolerates the retired ids instead).

### Derived graph index (rebuildable from the graph store)

**`nodes`** — one row per indexed node file
| column | type | notes |
|---|---|---|
| id | uuid pk | = frontmatter `id` (identity; paths are projections) |
| store_path | text unique | `/`-separated, store-relative |
| type | text | node type (folder) |
| title | text | |
| plane / planes | text / text[] | attribute membership (fallback: `{plane}`) |
| tags | text[] | |
| aliases | text[] | entity surface forms, **GIN index** — this *is* the alias index ([ADR-030](adr/030-entity-substrate-and-lifecycle.md)) |
| disambig | text null | one-line same-name disambiguator |
| occurred_start / occurred_end | date null | partial-ISO `occurred` expanded to a range ([ADR-031](adr/031-m3-organizer-and-contract-extensions.md)); time axes use `occurred ?? created` |
| organizer_version | text null | which organizer generation wrote the node (retrofit targeting) |
| merged_into | uuid null | tombstone marker after `merge` — node hidden from search/map, id keeps resolving |
| source / source_ref | text null | |
| content_hash | text | sha256 of the **whole file** (no exclusions — the `sb:related` machinery is gone); unchanged ⇒ skip reindex |
| embedding | vector(768) null | mean-pool of chunk embeddings; powers similar-edge k-NN. HNSW, cosine |
| node_created_at | timestamptz null | frontmatter `created`, else mtime |
| indexed_at | timestamptz | |

*Derived entity **profiles** ([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[032](adr/032-prior-art-adoptions.md)) —
regenerated nightly from the 1-hop neighborhood as **categorized observation lines** (`[role]`,
`[location]`, `[last-seen]`, …), not prose blobs — live DB-side in `node_profiles` (embedded,
rebuildable). The profile `embedding` is a **search retrieval source**
([ADR-037](adr/037-profile-embedding-in-search-m3.md)): search unions a per-profile vector leg with
the chunk leg (same `search_document:` space), so searching an entity's name surfaces its own hub
node with the summary as snippet — the ADR-030 §4 intent. Profiles are DB-only derived state, not in
the store: a plain reindex preserves them, a full DB wipe leaves profile-search degraded until
`profile-refresh` reruns (rule 1 restores from the store; profiles aren't in it).*

**`chunks`** — as before, `note_id` → **`node_id`** (fk → nodes, cascade); unique
`(node_id, chunk_index)`; content + `vector(768)`.

**`edges`** — the graph, both origins in one table
| column | type | notes |
|---|---|---|
| src_id / dst_id | uuid fk → nodes, cascade | directional |
| rel | text | `involves`, `about`, `at`, … for canonical; `similar` for derived |
| origin | text | `canonical` (materialized from frontmatter at index time) \| `derived` (recomputed nightly from embeddings, top-K over cosine ≥ floor — the surviving half of [ADR-023](adr/023-semantic-relatedness-graph.md)) |
| score | real null | one column, origin disambiguates: **confidence** for canonical (null ⇒ 1.0), **cosine** for derived ([ADR-031](adr/031-m3-organizer-and-contract-extensions.md)) |
| since / until | date null | canonical validity window ([ADR-030](adr/030-entity-substrate-and-lifecycle.md)/[032](adr/032-prior-art-adoptions.md)) — `until` closes a superseded relation; invalidate, never delete |
| | | pk `(src_id, dst_id, rel, origin)`. Both origins rebuildable: canonical from files, derived from vectors — the whole table is derived-tier |

### Operational state (not rebuildable — why the DB is managed/backed up)

**`captures`** — unchanged except `note_paths → node_paths`; follow-up columns
([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)) and interaction logging
([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)) carry over. **M5 adds a
`source` column** ([ADR-046](adr/046-m5-mcp-server-oauth-connectors.md) — `web` default \|
`mcp` \| `chat` (M6) \| **`instagram` (M9.5)** \| later `slack`; distinct from `kind` =
text/voice **\| `image` (M9, [ADR-057](adr/057-multimodal-media-ingestion-substrate.md) §6 —
raw file kept under `/srv/data/media/capture/` (the media `source`=`capture` layout — ADR-057 §3's
`captures/` sketch reconciled to `<source>` here, as with the `media` table name), vision
description derived; the fenced description becomes the capture's `raw_text`)**), threaded to
the node frontmatter `source:` + `agent_runs` so MCP-driven captures are activity-visible. **M6 task 1 adds
a nullable `source_ref` column** ([ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §1, mirroring
`nodes.source_ref`): an **endorsed** chat candidate materializes a `captures` row (`source=chat`) whose
`source_ref` is the originating **chat-session id**, so the chat→capture→node chain is traceable for the
M6 audit/remove surfaces without embedding node ids in chat state (NULL for the web/voice/MCP captures).
**M6 task 4 adds a `removed_at` column** ([ADR-048](adr/048-m6-chat-distiller-build-decisions.md)): one-tap
remove of a chat-distilled node tombstones its capture so `reprocess-all`/replay skips it (the node file
is git-rm'd — history kept — and DB rows deleted); a non-null `removed_at` is replay-excluded.

**`connector_cursors`** — unchanged (`connector` pk, `cursor` jsonb, `updated_at`). Used by
API-fetcher connectors (Instagram daily if the spike passes; Slack at M12). **The
chat-distiller does NOT use it** — it needs *per-session* watermarks, so it uses the dedicated
**`chat_distill_state`** table (above, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md)),
not a single connector cursor.

**`connector_threads` / `connector_messages` / `connector_media`** (**M9/M9.5**,
[ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md) §3 +
[ADR-057](adr/057-multimodal-media-ingestion-substrate.md) §3 — exact columns pinned at the
build task, migration numbered then): the source-generic **conversation substrate**.
`connector_threads` — `(source, thread_id)` unique, participants (+ the manifest's
`name_override`), per-thread import state. `connector_messages` — **connector raw** (the
replay source for re-sessionization/re-distillation): `(source, thread_id, message_id)` unique
**upsert key** (idempotent import), sender, `sent_at` (ms precision), repaired text, reactions,
share/link payload, edit marker. **`media`** (**physical table name pinned at M9 T2**, migration 017; ADR-057 §3 sketched it as
`connector_media`, but it is **source-generic** — it serves ad-hoc PWA photo captures *now* (M9)
and connector media at M9.5 — so the physical table is `media`) — one row per media item: `kind`
(`photo`/`voice`/`video`), `source` (`capture` for ad-hoc | `instagram` at M9.5 — also the top of
the `/srv/data/media/<source>/…` layout), a **nullable `capture_id` fk → `captures`**
(`ON DELETE CASCADE` — the M9 ad-hoc-capture link) with a **nullable `message_id` fk →
`connector_messages` added at M9.5** (the connector link — "media fk wiring" in M9.5 T1),
`file_path` **relative to the media root** (**null for video** — summary-only, ADR-057 §2),
`thumb_path`, `mime_type`, **derivation `status`** (`pending`/`derived`/`unavailable` — the
resumability + targeted-re-derive hinge) + `derived_text` (photo description / voice transcript /
video summary) + `model_used` + `attempts` + `error`. Messages/threads are **operational raw** (not rebuildable —
backed up with the DB; media *files* additionally R2-synced per ADR-014); derived text is
derived-tier (recomputable from kept raw, except video summaries — the recorded ADR-057 §2
exception). **Session distill state** (watermark per `(source, thread, session)` — the ADR-048
pattern generalized) + candidate dedup keys land alongside at the build task.

**`chat_auto_recorded`** (**M6 task 4**, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md)
§11/§12) — the registry behind the chat-scoped **"recently auto-recorded"** audit list
(`GET /chat/auto-recorded`). One row per **auto-endorsed** distiller candidate: `capture_id` (pk,
fk → `captures` `ON DELETE CASCADE`) + the coarse `salience` tag + `recorded_at`. **Its existence is
what marks a memory as auto-recorded** — the distinction the ADR draws for one-tap remove: the audit
list and the remove affordance are **auto-endorsed only**; an *agree-from-review* memory materializes
the same `source=chat` capture but writes **no** row here, so it stays off this surface (user-vetted;
general node removal is backlog). Provenance, **not** derived index — a `reprocess-all` replays
captures through the organizer, not chat sessions, so it never re-mints these rows (they survive,
like the preserved `stance-candidate` items). The audit list joins `captures` (for `node_paths` +
`source_ref` + `raw_text` snippet, filtered `removed_at IS NULL`) and `nodes` (for the primary
content node's title, entity hubs skipped). M8's general Activity feed absorbs this later.

**`agent_runs`** — **M5.5 added `parent_run_id`** (nullable self-fk,
[ADR-047](adr/047-pipeline-scheduling-primitive.md)): a pipeline run is a parent row, each step a
child linked by `parent_run_id`. **M8 adds `trigger`** (`scheduled`|`manual`, default `scheduled` —
[ADR-053](adr/053-m8-ops-console-observability-build-decisions.md) §5): set through an ambient
`_trigger` contextvar the manual endpoint sets around the call (no job-body change), so the Activity
feed can file a hand-run job under **manual actions** vs a scheduled run under **agents/jobs** by
*origin* not table. Agent names grow with the roadmap (`chat-distiller`, `dedup-sweep`,
`inbox-drainer`, `maybe-digest`, `graph-health`, `reflection`, …); **M8 also gives `store-sweep` its
own run row** (was a phantom `skipped` step, M5.5-task-3 follow-up). Schedule/next-run is *derived*
from the pipeline registry, exposed by `GET /pipelines` (03-api).

**`agent_run_logs`** (**M8**, [ADR-053](adr/053-m8-ops-console-observability-build-decisions.md)
§1/§2) — the live-log-tail store, backing `GET /activity/runs/{id}/logs`:
| id bigserial pk · run_id uuid fk→`agent_runs`(id) · seq int (per-run ordinal) · ts timestamptz · level text · message text |
An `app.*`/`INFO`+ logging handler tags records by the active run (a `_current_run_id` contextvar —
the ADR-047 §5 ambient pattern, no job-body change) into a **bounded per-run in-memory buffer**
(non-blocking; stdlib logging is sync, rule 8), which an async flusher persists here on a **~1s
cadence + on finish**. Buffer overflow drops-oldest and records an elision marker (rule 7). The
namespace/level filter structurally keeps library-DEBUG secret leakage out of this UI-rendered store
(rule 11). **Rebuildable op-state, not graph truth** (rule 1) — dropped and re-derived like the rest
of the index; a completed run's logs are fully durable after the on-finish flush.

**`review_queue`** (**M3**, [ADR-030](adr/030-entity-substrate-and-lifecycle.md) — pulled forward
from M6 and made **kind-generic**): every human-decision item the system files, one lifecycle.
| id uuid pk · kind (`entity-ambiguity` M3 \| `vocab-proposal` M3 \| `stance-candidate` M6 \| `dedup-proposal` M6) · payload jsonb (candidates / proposed content) · excerpt · source · source_ref · status (`pending`\|`resolved`\|`discarded`\|`maybe`, no expiry) · resolution jsonb null · created_at · resolved_at |

Items are decidable in place (mention in capture excerpt, candidates with name/disambig/aliases +
node-preview link). **Vocabulary proposals ([ADR-027](adr/027-typed-vocabulary-governance.md)) are
a queue kind** — no separate table; approved vocabulary lives in config + `app_settings`.
**M6 kinds ([ADR-048](adr/048-m6-chat-distiller-build-decisions.md)):** `stance-candidate` payload =
`{candidate_text, referenced_entity_names[], salience(high|med|low), why_unclear, anchor_at}` with
`source=chat`, `source_ref=session-id` — **names + text, never node ids** (survives a reprocess that
rebuilds the graph); `anchor_at` is the anchoring-message timestamp (ISO-8601, a *time* not a node
id) the distiller records so **agree** stamps the capture with conversation time, matching
auto-endorse. **agree** materializes a `captures` row = the auto-endorse path. `dedup-proposal`
(M6 task 5, [ADR-049](adr/049-dedup-sweep-merge-core-build-decisions.md)) payload =
`{node_a, node_b, signals:{cosine, shared_entity_ids, shared_entity_titles, occurred_overlap},
default_survivor}` — canonical `least→greatest` ids (may reference node ids — it is
truncate-on-reprocess); resolution `{action: merge|keep|link, survivor?}` (merge folds via the
shared merge-core, link writes a canonical `similar` edge).
**`maybe` is re-openable** (M6 fixes the `resolve` guard: `pending`+`maybe` are still-decidable,
`resolved`/`discarded` terminal). **Kind-aware reprocess** (see below): `reprocess-all` preserves
`stance-candidate`, truncates the other kinds.

**`chat_distill_state`** (**M6**, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md)): the
chat-distiller watermark — one row per distilled session. | session_id uuid pk (fk `chat_sessions`)
· last_message_at timestamptz (watermark — the distiller processes only messages after it) ·
distilled_at timestamptz · run_id uuid null | Idle-eligibility is derived from
`max(chat_messages.created_at)`; delta-after-watermark makes re-distillation idempotent.

**`summaries`** — retired at the pivot; daily/weekly output becomes `insight` nodes produced by
the reflection agent (M10). (Table kept until M10 replaces it; no new writers.)

**`chat_sessions` / `chat_messages`** — unchanged ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)):
`chat_messages.model` records the resolved **model id** (the vendor string; [ADR-045](adr/045-provider-model-effort-separation.md) — legacy provider-id rows like `claude-max`/`nebius` are **left untouched** and stay label-tolerated, not rewritten); `sources` = cited **nodes** (renumbered).
Chat-distiller session state (idle-eligibility + delta watermark) lives in `chat_distill_state`
([ADR-048](adr/048-m6-chat-distiller-build-decisions.md)), not on `chat_sessions`.

**`auth_sessions`** — unchanged.

**MCP OAuth (M5, [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md) — supersedes the
static-bearer sketch of ADR-028 §5):** two new tables. **`mcp_oauth_clients`** — dynamically
registered clients (RFC 7591): `client_id` pk · metadata jsonb (redirect_uris, name) ·
`created_at`. **`mcp_tokens`** — `id` pk · `client_id` fk · `token_hash` (HMAC-SHA256, like
`auth_sessions`; plaintext only ever to the connector) · `kind` (`access`\|`refresh`) ·
`expires_at` (~1h access; long-lived sliding refresh) · `revoked_at` null · `created_at`. Auth
codes may be a third short-lived table or in-memory (PKCE-bound, single-use — impl detail at
build). **Revoke-all** = flag all rows. Single full-access scope in M5.

**`app_settings`** — unchanged shape; keys grow (`model_routing` — jsonb, groups `chat`/`conspect`/`quick` (**+ `vision` at M9**, [ADR-057](adr/057-multimodal-media-ingestion-substrate.md) §4) each `{active, fallback, effort_by_model}` where `active`/`fallback` + the `effort_by_model` keys are **model ids** (vendor strings; [ADR-045](adr/045-provider-model-effort-separation.md) — was `effort_by_provider`/provider ids, **migrated in place** by an idempotent Alembic revision, vision P10), [ADR-025] + [ADR-043](adr/043-quick-routing-tier-m4.md); config seeds the default when unset), vocabulary,
connector lookback overrides — default **6 months** per connector, UI-overridable; **`identity_capsule`**
— M5 [ADR-046](adr/046-m5-mcp-server-oauth-connectors.md)/[ADR-033](adr/033-external-inspirations-obsidian-second-brain.md)
#1, a derived blob `{text (~300 tok), generated_at, source_refs}` distilled nightly from entity-profile
hubs + recent memories + insights, served as `build_context` L0 + the `identity://me` resource + the M4
chat system prompt; rebuildable, regenerated never hand-kept).

## 4. Chunking policy

Split on headings, then paragraphs; target `CHUNK_SIZE` (1200 chars), overlap `CHUNK_OVERLAP`
(200) on hard splits. **The embed path strips frontmatter only** — a node's embedded identity is
its human prose; edges live in frontmatter, so "never embed links" is now structural (no
special-case stripping, the M2 `## Related`/`sb:related` rules are gone). `content_hash` covers
the whole file. Embedded text = `"search_document: {title}\n\n{chunk}"`; queries =
`"search_query: {q}"` — the asymmetric nomic prefixes stay **mandatory**
([ADR-022](adr/022-embeddings-self-hosted-nomic.md)). `nodes.embedding` = mean-pool of chunk
vectors.

## 5. Rebuild & recovery matrix

Durability tiers ([ADR-014](adr/014-vault-history-durability.md) — machinery unchanged by the
pivot): the **graph store is the only never-lose tier**; operational state gets a second
independent copy and may restore-to-last-nightly. Every capture ends as a node (organized or
`inbox/` fallback), so total DB loss never loses memory.

| Loss | Recovery | Tier |
|---|---|---|
| Derived tables (`nodes`,`chunks`,`edges`) | `POST /admin/reindex` from the store | rebuildable |
| A format/organizer-quality change left old nodes stale | `POST /admin/reprocess` ([ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md)): reset derived state (node files + `nodes`/`chunks`/`edges`/`node_profiles`, `captures.node_paths`) and the **capture-derived `review_queue` kinds only** (`entity-ambiguity`/`vocab-proposal`/`dedup-proposal` — **kind-aware**, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §7; `stance-candidate` items are **preserved**), replay every capture's raw chronologically (incl. chat-endorsed captures → **P10**); **raw + approved vocabulary + `removed_at`-tombstoned captures' exclusion preserved**, standing merges reported | derived (from raw) |
| Operational state (`agent_runs`, `chat_*`, `captures`, `review_queue`, cursors, settings) | Supabase backup, or nightly `pg_dump` in R2 → restore-to-last-nightly | operational |
| Whole database | as above; worst case reindex restores search/traverse/chat, losing at most the day's operational log + pending review items (re-derivable from persisted sessions) | operational |
| Graph store on VPS | `git clone` from GitHub **or** the latest R2 WORM bundle | **never-lose** |
| Store history rewritten on GitHub | Restore from R2 WORM bundle; server heals GitHub by merge-push | **never-lose** |
| GitHub account/repo lost | R2 WORM bundle holds full history; re-create remote and push | **never-lose** |
| Un-transcribed audio (`/srv/data`) | nightly-synced to R2 | input-safety |
| Whole VPS | reprovision ([07](07-infrastructure.md)) + clone store + `pg_dump` restore + reindex | mixed |

Recovery is verified: the weekly integrity drill (bundle verify + fingerprint check on R2 and
GitHub) degrades `/health` on failure ([ADR-014](adr/014-vault-history-durability.md) §6).
