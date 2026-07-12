# Data Model

**Version:** 2.2 · **Status:** Approved 2026-07-12 (2.2 = M1 replan: migration 003 =
`capture_interactions` view, `agent="capture"` runs [ADR-021]; 2.1 = M1 migration 002: capture
follow-up columns; new `agent_runs` agent names)
**Key ADRs:** [001](adr/001-vault-on-vps-with-git-backup.md) · [002](adr/002-supabase-pgvector-for-index.md) · [005](adr/005-planes-and-atomic-notes.md) · [020](adr/020-stt-fallback-chain-groq-primary.md) · [021](adr/021-capture-interactions-agent-runs-logging.md)

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
stays plain asyncpg (no ORM). M0 ships revision 001 = the full schema below; **M1 adds
revision 002** (`captures.follow_up_question` / `follow_up_answer` — [ADR-019](adr/019-conversational-capture-minimal-in-m1.md))
and **revision 003** (the `capture_interactions` view — [ADR-021](adr/021-capture-interactions-agent-runs-logging.md);
a view only, no table/column change).

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
| id uuid pk · kind (`voice`\|`text`) · status (`received→transcribing→organizing→written→indexed` \| `failed`) · raw_text · audio_path · note_paths text[] · follow_up_question text null · follow_up_answer text null · error · created_at · updated_at |

`follow_up_question` / `follow_up_answer` are added by **migration 002** (M1,
[ADR-019](adr/019-conversational-capture-minimal-in-m1.md)): a single "dig deeper" nudge
generated after a successful organize; answering re-organizes and **replaces** the capture's
notes. Question-present + answer-absent = "nudge pending" (no separate status).

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
| agent | text | `capture` (M1, [ADR-021](adr/021-capture-interactions-agent-runs-logging.md)), `slack-ingest`, `daily-summary`, `weekly-review`, `reindex`, `vault-backup`, `integrity-drill`, `db-backup`, `data-sync` (last three added M1, [ADR-014](adr/014-vault-history-durability.md)) |
| status | text | `running` \| `success` \| `partial` \| `failed` |
| started_at / finished_at | timestamptz | |
| model_used | text null | resolved model after fallback |
| fallback_used | boolean | true ⇒ primary (Claude) was unavailable/limited |
| summary | text | human-readable one-liner for the feed |
| details | jsonb | per-item events: notes written, items skipped, errors |
| error | text null | |

**`capture_interactions`** (view, migration 003, [ADR-021](adr/021-capture-interactions-agent-runs-logging.md)) —
flattens `agent_runs` rows where `agent='capture'` into readable columns for the Supabase
dashboard / MCP and the future M4 activity feed: `capture_id, kind, stt_provider, stt_fallback,
organize_model, fallback_used, status, error, started_at, duration_ms`. Read-only projection over
`agent_runs.details`; no data of its own.

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

Durability tiers ([ADR-014](adr/014-vault-history-durability.md)): the **vault is the only
never-lose tier** (memory content); operational state gets a cheap second independent copy
and may restore-to-last-nightly. All memory content reaches the vault regardless (every
capture ends as a note — organized, or an Inbox fallback), so total DB loss never loses memory.

| Loss | Recovery | Tier |
|---|---|---|
| Derived tables (`notes`,`chunks`) | `POST /admin/reindex` from vault | rebuildable |
| Operational state (`agent_runs`, `chat_*`, `captures`, cursors, settings) | Supabase backup, or nightly `pg_dump` in R2 (second independent copy) → restore-to-last-nightly | operational (not never-lose) |
| Whole database | as above; worst case reindex restores search/chat, losing at most the current day's operational log | operational |
| Vault on VPS | `git clone` from GitHub **or** `git clone` the latest **R2 WORM bundle** (`git bundle --all`, object-locked) | **never-lose** |
| Vault history rewritten (force-push/rebase) on GitHub | Restore from R2 WORM bundle; server heals GitHub by merge-push (never resets to remote) | **never-lose** |
| GitHub account/repo lost or taken down | R2 WORM bundle holds full history; re-create remote and push | **never-lose** |
| Un-transcribed audio (`/srv/data`) | Nightly-synced to R2 (no longer VPS-disk-only) | input-safety |
| Whole VPS | Reprovision (scripted, [07-infrastructure.md](07-infrastructure.md)) + clone vault (GitHub/R2) + `pg_dump` restore + reindex | mixed |

Recovery is **verified**, not assumed: a weekly integrity drill (`git bundle verify` +
fingerprint check — HEAD sha, monotonic commit count, file count — on both R2 and GitHub)
runs on the VPS and degrades `/health` on failure ([ADR-014](adr/014-vault-history-durability.md) §6).
