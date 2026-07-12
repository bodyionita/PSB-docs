# Data Model

**Version:** 2.0 · **Status:** Approved 2026-07-12
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [002](adr/002-supabase-pgvector-for-index.md) · [005](adr/005-planes-and-atomic-notes.md)

## 1. Vault layout

```
<VAULT_PATH>/                      # on the VPS; git repo pushed to private GitHub
├── Inbox/                         # classifier said "don't know" or capture fallback
├── Professional/
├── Personal/
├── Family/
├── Friends/
├── Health/
├── Ideas/
│   └── 2026-07-12 Second brain connector idea.md
├── Summaries/
│   ├── Daily/2026-07-12.md
│   └── Weekly/2026-W28.md
└── .obsidian/ …                   # ignored by the indexer
```

- Top-level folders = **planes**, defined in config (`PLANES=` list) — adding a plane is
  config, not code. `Inbox/` and `Summaries/` are system folders, always present.
- The indexer indexes every `*.md` except `VAULT_IGNORE` prefixes (default:
  `.obsidian`, `.trash`, `.git`, `templates`).

## 2. Note frontmatter contract

Pipeline-written notes always carry:

```yaml
---
id: 018f3c2e-…                  # capture id / ingest item id
created: 2026-07-12T09:14:03+02:00
source: voice | text | slack | summary-daily | summary-weekly   # extensible per connector
source_ref: "slack:C0123/p1720771234.5678"   # opaque locator within the source (optional)
plane: professional             # primary plane == folder
planes: [professional, friends] # full membership; superset of `plane`
tags: [standup, planning]
related: ["Friends/2026-07-12 Dinner plans with Alex.md"]  # sibling notes from same source
---
```

Rules ([ADR-005](adr/005-planes-and-atomic-notes.md)):
- **Primary plane = folder** (one physical home); **membership truth = `planes:`** —
  analysis/queries filter on frontmatter, never on folder.
- One source may produce **several atomic notes** (split per plane); siblings share
  `source_ref` and cross-link via `related` + `[[wikilinks]]` in the body.
- User-created notes may have no frontmatter; every field is optional at read time.
  Title = H1 if present, else filename stem.

## 3. Database schema (Supabase Postgres + pgvector)

Embedding dimension **1536** (`text-embedding-3-small`); model change = deliberate
migration + full reindex ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md)).

**Migrations:** managed by **Alembic**, authored as **explicit SQL** (`op.execute` /
`op.create_table`) — **no ORM, no autogenerate** ([ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md)).
Applied explicitly via `alembic upgrade head` in CI / `provision.sh`, never in the request
path. The `vector` extension and `vector(1536)` columns are created in raw SQL. Query code
stays plain asyncpg (no ORM). M0 ships revision 001 = the full schema below.

### Derived index (rebuildable from vault)

**`notes`** — one row per indexed vault file
| column | type | notes |
|---|---|---|
| id | uuid pk | |
| vault_path | text unique | `/`-separated, vault-relative |
| title | text | |
| plane | text | primary plane (folder) |
| planes | text[] | membership from frontmatter (fallback: `{plane}`) |
| tags | text[] | |
| source | text null | frontmatter `source` |
| source_ref | text null | |
| content_hash | text | sha256; unchanged ⇒ skip reindex |
| note_created_at | timestamptz null | frontmatter `created`, else file mtime |
| indexed_at | timestamptz | |

**`chunks`**
| column | type | notes |
|---|---|---|
| id | uuid pk | |
| note_id | uuid fk → notes on delete cascade | |
| chunk_index | int | unique (note_id, chunk_index) |
| content | text | retrieval cache; canonical text is the file |
| embedding | vector(1536) | HNSW, cosine |

### Operational state (not rebuildable — this is why the DB is managed/backed up)

**`captures`** — user capture pipeline state
| column | type |
|---|---|
| id uuid pk · kind (`voice`\|`text`) · status (`received→transcribing→organizing→written→indexed` \| `failed`) · raw_text · audio_path · note_paths text[] · error · created_at · updated_at |

**`connector_cursors`** — incremental sync position per connector
| column | type | notes |
|---|---|---|
| connector | text pk | e.g. `slack` |
| cursor | jsonb | connector-defined (e.g. per-channel `latest_ts`) |
| updated_at | timestamptz | |

**`agent_runs`** — powers the activity feed (high-level entry + expandable details)
| column | type | notes |
|---|---|---|
| id | uuid pk | |
| agent | text | `slack-ingest`, `daily-summary`, `weekly-review`, `reindex`, `vault-backup` |
| status | text | `running` \| `success` \| `partial` \| `failed` |
| started_at / finished_at | timestamptz | |
| model_used | text null | resolved model after fallback |
| fallback_used | boolean | true ⇒ primary (Claude) was unavailable/limited |
| summary | text | human-readable one-liner for the feed |
| details | jsonb | per-item events: notes written, items skipped, errors |
| error | text null | |

**`summaries`** — daily/weekly analysis registry
| period (`daily`\|`weekly`) + period_start date, unique · content · note_path · created_at |

**`chat_sessions`** / **`chat_messages`**
| sessions: id uuid pk · title · created_at · last_model text |
| messages: id uuid pk · session_id fk · role (`user`\|`assistant`) · content · model text null · sources jsonb null · created_at |

**`auth_sessions`** — login sessions (httpOnly cookie → hashed token)
| id uuid pk · token_hash text unique · user_agent · created_at · last_seen_at · expires_at · revoked boolean |

**`app_settings`** — UI-editable runtime settings (agent model chain, etc.)
| key text pk · value jsonb · updated_at |

## 4. Chunking policy

Split on headings, then paragraphs; target `CHUNK_SIZE` (1200 chars), overlap
`CHUNK_OVERLAP` (200) on hard splits. Frontmatter stripped; `"{title}\n\n{chunk}"` is what
gets embedded.

## 5. Rebuild & recovery matrix

| Loss | Recovery |
|---|---|
| Derived tables | `POST /admin/reindex` from vault |
| Whole database | Supabase backup; worst case: reindex restores search/chat (capture/run/chat history lost) |
| Vault on VPS | `git clone` from private GitHub repo |
| Whole VPS | Reprovision (scripted, [07-infrastructure.md](07-infrastructure.md)) + git clone + reindex |
