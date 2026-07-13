# Data Model

**Version:** 3.0 · **Status:** Approved 2026-07-13 (3.0 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)/[027](adr/027-typed-vocabulary-governance.md):
typed-node **graph store** replaces the note vault (fresh start — old vault archived); `nodes`/`edges`
replace `notes`/`note_links`; wikilinks, `sb:related` blocks and folder-=-plane are deleted. The
concrete migration (005) is authored at M3 implementation; **exact DDL details are finalized at the
M3 build-ready grilling**. 2.x history: M3 chat/routing keys, M2 embeddings 768 + relatedness graph,
M1 capture columns — see git history.)
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [002](adr/002-supabase-pgvector-for-index.md) · [005](adr/005-planes-and-atomic-notes.md) (surviving half) · [022](adr/022-embeddings-self-hosted-nomic.md) · [025](adr/025-ui-editable-model-routing-and-per-task-effort.md) · **[026](adr/026-graph-native-storage-obsidian-removed.md) · [027](adr/027-typed-vocabulary-governance.md) · [029](adr/029-conversational-ingestion-stance-gate-review-queue.md)**

## 1. Graph store layout

```
<GRAPH_STORE_PATH>/                # on the VPS; git repo pushed to private GitHub
├── memory/                        # node types = top-level folders (config-driven)
│   └── 2026-07-13--dinner-with-alex--018f3c2e.md
├── person/
│   └── alex-marsh--018f4a11.md
├── idea/
├── conversation/
├── insight/
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
type: memory                    # memory | person | idea | conversation | insight | …
created: 2026-07-13T21:40:00+02:00
source: voice                   # voice | text | slack | chat | mcp | agent | …
source_ref: "slack:C0123/p1720771234.5678"   # opaque locator (optional)
plane: personal                 # primary plane (attribute, not folder)
planes: [personal, friends]     # full membership; superset of `plane`
tags: [dinner, catching-up]
edges:                          # canonical typed edges, target = node id
  - {rel: involves, to: 018f4a11-…}
  - {rel: part_of,  to: 018f6c33-…}
---
(Markdown prose body — the memory itself)
```

Rules:
- **Edge vocabulary** (starter: `involves`, `about`, `part_of`, `led_to`, `follows`) is governed
  like node types — LLM proposes, user approves, consolidation retrofits
  ([ADR-027](adr/027-typed-vocabulary-governance.md)). Edges are directional, written on the
  source node. **Similarity is never written into files** — derived `similar` edges live only
  in the DB.
- One source may produce **several atomic nodes** (split per plane/topic); siblings share
  `source_ref` and are connected by typed edges (no wikilinks, no `related:` field).
- **English-only store** (organizer translates; raw input preserved verbatim in
  `captures.raw_text`; conversational UI mirrors the user's language).
- **Tag slug rules kept on their own merits** (ex-"Obsidian tags"): English, lower-case, single
  word or hyphenated, no spaces, chars `a–z 0–9 _ - /`, must contain a letter; enforced by
  `_slugify_tag`.
- Hand-created nodes may omit any field; every field is optional at read time. Title = H1 if
  present, else filename stem. Missing `type` = `memory`.

## 3. Database schema (Supabase Postgres + pgvector)

Embedding dimension **768** (self-hosted `nomic-embed-text-v1.5` via Ollama,
[ADR-022](adr/022-embeddings-self-hosted-nomic.md)); `embedding_dim`/`embedding_model` are
settings — provider change = deliberate migration + reindex.

**Migrations:** Alembic, explicit SQL, no ORM ([ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md)).
**M3 (pivot) ships migration 005**: drops the note-model derived tables, creates `nodes`/`edges`,
renames `captures.note_paths → node_paths` (fresh start — no data migrated, old vault archived).

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
| source / source_ref | text null | |
| content_hash | text | sha256 of the **whole file** (no exclusions — the `sb:related` machinery is gone); unchanged ⇒ skip reindex |
| embedding | vector(768) null | mean-pool of chunk embeddings; powers similar-edge k-NN. HNSW, cosine |
| node_created_at | timestamptz null | frontmatter `created`, else mtime |
| indexed_at | timestamptz | |

**`chunks`** — as before, `note_id` → **`node_id`** (fk → nodes, cascade); unique
`(node_id, chunk_index)`; content + `vector(768)`.

**`edges`** — the graph, both origins in one table
| column | type | notes |
|---|---|---|
| src_id / dst_id | uuid fk → nodes, cascade | directional |
| rel | text | `involves`, `about`, … for canonical; `similar` for derived |
| origin | text | `canonical` (materialized from frontmatter at index time) \| `derived` (recomputed nightly from embeddings, top-K over cosine ≥ floor — the surviving half of [ADR-023](adr/023-semantic-relatedness-graph.md)) |
| score | real null | derived edges only |
| | | pk `(src_id, dst_id, rel, origin)`. Both origins rebuildable: canonical from files, derived from vectors — the whole table is derived-tier |

### Operational state (not rebuildable — why the DB is managed/backed up)

**`captures`** — unchanged except `note_paths → node_paths`; follow-up columns
([ADR-019](adr/019-conversational-capture-minimal-in-m1.md)) and interaction logging
([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)) carry over.

**`connector_cursors`** — unchanged (`connector` pk, `cursor` jsonb, `updated_at`). The
chat-distiller registers here like any connector ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).

**`agent_runs`** — unchanged shape; agent names grow with the roadmap (`chat-distill`,
`consolidate`, `reflection`, …). Plus (M8 ops console) a **schedule registry** surface: each
registered job's cadence + next-run time is queryable (implementation detail at M8 grilling).

**`review_queue`** (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)) —
stance-unclear distillation candidates: proposed memory + conversation excerpt + source/
source_ref + status (`pending` | `agreed` | `discarded`), no expiry. Exact DDL at the M6 grilling.

**Vocabulary proposals** ([ADR-027](adr/027-typed-vocabulary-governance.md)) — pending node/edge
type proposals (name, definition, examples, status). Table vs `app_settings` decided at the M3
grilling; approved vocabulary lives in config + `app_settings`.

**`summaries`** — retired at the pivot; daily/weekly output becomes `insight` nodes produced by
the reflection agent (M10). (Table kept until M10 replaces it; no new writers.)

**`chat_sessions` / `chat_messages`** — unchanged ([ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)):
`chat_messages.model` records the resolved model; `sources` = cited **nodes** (renumbered).
Chat-distiller cursor tracks session activity for nightly distillation.

**`auth_sessions`** — unchanged. **MCP bearer token**: stored as a hash in env/settings
(single-user), revocable independently ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)).

**`app_settings`** — unchanged shape; keys grow (`model_routing` [ADR-025], vocabulary,
connector lookback overrides — default **6 months** per connector, UI-overridable).

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
| Operational state (`agent_runs`, `chat_*`, `captures`, `review_queue`, cursors, settings) | Supabase backup, or nightly `pg_dump` in R2 → restore-to-last-nightly | operational |
| Whole database | as above; worst case reindex restores search/traverse/chat, losing at most the day's operational log + pending review items (re-derivable from persisted sessions) | operational |
| Graph store on VPS | `git clone` from GitHub **or** the latest R2 WORM bundle | **never-lose** |
| Store history rewritten on GitHub | Restore from R2 WORM bundle; server heals GitHub by merge-push | **never-lose** |
| GitHub account/repo lost | R2 WORM bundle holds full history; re-create remote and push | **never-lose** |
| Un-transcribed audio (`/srv/data`) | nightly-synced to R2 | input-safety |
| Whole VPS | reprovision ([07](07-infrastructure.md)) + clone store + `pg_dump` restore + reindex | mixed |

Recovery is verified: the weekly integrity drill (bundle verify + fingerprint check on R2 and
GitHub) degrades `/health` on failure ([ADR-014](adr/014-vault-history-durability.md) §6).
