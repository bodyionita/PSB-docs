# Architecture

**Version:** 3.0 · **Status:** Approved 2026-07-13 (3.0 = **mind-graph pivot** — Obsidian removed,
typed-node **graph store** replaces the note vault, MCP promoted to a peer surface, conversational
ingestion + review queue, jobs-observability contract —
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md) · [027](adr/027-typed-vocabulary-governance.md) ·
[028](adr/028-one-service-layer-mcp-peer-surface.md) · [029](adr/029-conversational-ingestion-stance-gate-review-queue.md);
2.1 = M2 refresh; 2.0 = cloud-service pivot)
**Key ADRs:** [001 store-on-VPS+git](adr/001-vault-on-vps-with-git-backup.md) · [003 single-service](adr/003-single-service-on-vps.md) · [004 provider-registry](adr/004-provider-registry-claude-primary-nebius-fallback.md) · [006 monorepo-decoupled](adr/006-monorepo-with-strict-server-web-decoupling.md) · [008 connectors-on-vps](adr/008-connectors-run-on-vps.md) · [013 web-on-VPS-single-origin](adr/013-web-stays-on-vps-single-origin.md) · [022 self-hosted-embeddings](adr/022-embeddings-self-hosted-nomic.md) · [026–029 the graph pivot](adr/026-graph-native-storage-obsidian-removed.md)

## High-level view

```
   Phone / Desktop                               Hetzner VPS (always on)
┌──────────────────────────┐          ┌──────────────────────────────────────────┐
│  Web PWA (React+Vite)    │          │  Caddy (TLS, serves PWA, proxies /api)   │
│  capture voice/text      │  HTTPS   │      │                                   │
│  chat + model picker     │─────────▶│  FastAPI service (single process)        │
│  review queue · map      │ (behind  │  ├─ API layer          (REST routers)    │
│  activity · settings     │ Cloud-   │  ├─ MCP server         (peer surface) ◀──┼── other LLMs (Claude
└──────────────────────────┘  flare)  │  │      └── both are thin skins over:    │   anywhere, bearer token)
                                      │  ├─ Services (capture, organizer, search,│
                                      │  │   graph/traverse, chat, distillers)   │
                                      │  ├─ Provider registry  (Claude Max SDK ──┼──▶ Anthropic (Max sub)
                                      │  │   primary → Nebius fallback;          ├──▶ Nebius AI (fallback LLM)
                                      │  │   STT chain Groq→OpenAI)              ├──▶ Groq / OpenAI (Whisper STT)
                                      │  ├─ ollama sidecar (nomic embeddings)    │  (self-hosted, no external call)
                                      │  ├─ Scheduler (03:00–05:00 agent window) │
                                      │  └─ Ingestion agents (chat-distiller,    │
                                      │      Instagram, Slack, …) ───────────────┼──▶ source APIs / export imports
                                      │      │                                   │
                                      │      ▼                                   │
┌────────────────┐   git push        ┌┴──────────────────┐                       │
│ Private GitHub │◀─────────────────│  GRAPH STORE       │                       │
│ repo (backup + │                  │  (typed-node .md,  │                       │
│ history)       │                  │  SOURCE OF TRUTH,  │                       │
└────────────────┘                  │  on VPS disk)      │                       │
                                    └────────┬───────────┘                       │
                                             │ index / rescan                    │
                                             ▼                                   │
                                    ┌────────────────────┐                       │
                                    │ Supabase Postgres  │◀──────────────────────┘
                                    │ + pgvector         │   operational state
                                    │ (DERIVED nodes/    │   (captures, runs, chat,
                                    │  edges index)      │    review queue, settings)
                                    └────────────────────┘
```

## Components

### 1. Web PWA (`second-brain/web`)
React + Vite + TypeScript single-page app, installable PWA. Premium design with first-class
animations (see [06-web-app.md](06-web-app.md)). Talks **only** to the HTTP API — this contract
is the future repo-split seam ([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)).
Its exclusives are the **human affordances**: voice capture (Romanian-capable STT), the visual
map, review queue, settings ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)).

### 2. API service (`second-brain/server`)
Single FastAPI process ([ADR-003](adr/003-single-service-on-vps.md)) hosting:

- **Surfaces (thin, logic-free):** REST routers (auth, capture, chat, search, graph, review,
  activity, settings, admin, health) **and the MCP server** (`search`, `get_node`, `traverse`,
  `list_planes`/`list_types`, `capture` — bearer-token auth,
  [ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)). Validation + delegation only;
  no duplication between surfaces.
- **Services / pipelines** — all business logic (capture, **organizer** — the single writer of
  graph structure for every surface — indexing, search/traverse, chat, distillers, analysis —
  see [04-pipelines.md](04-pipelines.md)).
- **Provider registry** — named LLM/STT/embedding providers + per-task routing with fallback
  chains ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md), UI-editable
  per group [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)). Primary mind:
  Claude via Agent SDK (Max). STT: Groq→OpenAI ([ADR-020](adr/020-stt-fallback-chain-groq-primary.md));
  embeddings self-hosted nomic via `ollama` ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)).
- **Scheduler + agents** — APScheduler, 03:00–05:00 window ([ADR-010](adr/010-agent-window-3-5am.md));
  chat-distiller ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)),
  connectors (Instagram [ADR-058](adr/058-instagram-dm-connector-and-conversation-substrate.md),
  Slack at M12, …), consolidation, reflection. **Jobs-observability contract:** every job
  is manually triggerable from the UI, live-observable while running (status + logs), and its
  schedule (cadence + next run) is inspectable — no cron-only ghosts.

### 3. Graph store (on VPS disk)
Canonical memory: **typed nodes as Markdown + YAML frontmatter** (folder = type, id = identity,
typed `edges` in frontmatter — [ADR-026](adr/026-graph-native-storage-obsidian-removed.md),
format contract in [02-data-model.md](02-data-model.md)). Auto-committed and pushed to a private
GitHub repo after every write batch ([ADR-001](adr/001-vault-on-vps-with-git-backup.md)) —
backup and full history, hardened by [ADR-014](adr/014-vault-history-durability.md) (atomic
writes, ff-only push, R2 WORM bundles, integrity drills — all unchanged by the pivot).

### 4. Supabase Postgres + pgvector
Two roles, one database ([ADR-002](adr/002-supabase-pgvector-for-index.md)):
- **Derived graph index** (rebuildable from the store): `nodes`, `chunks` (+embeddings), and
  `edges` — canonical edges materialized from frontmatter **+** derived `similar` edges
  (recomputed nightly, never written into files).
- **Operational state** (not derivable): `captures`, `connector_cursors`, `agent_runs`,
  `chat_*`, `review_queue`, `auth_sessions`, `app_settings`.

## Invariants

1. Node content flows **graph store → index**, never index → store (pipeline-generated nodes
   are written to the store first, then indexed). Derived similarity lives in the DB only.
2. Dropping all derived tables + `POST /admin/reindex` restores search/traverse/chat over the store.
3. Every model call goes through the provider registry; nothing else imports vendor SDKs.
4. Every background action is persisted and visible — **live while running** (status + logs),
   manually triggerable, schedule inspectable; activity covers automatic and manual events.
5. Raw inputs (audio files, chat sessions, raw connector payload digests) survive pipeline failures.
6. `web/` and `server/` share nothing but the HTTP contract in [03-api.md](03-api.md).
7. **The organizer is the single writer of graph structure** — every surface (UI, MCP,
   connectors, chat distillation) funnels through it; vocabulary is governed
   ([ADR-027](adr/027-typed-vocabulary-governance.md)), stance is gated
   ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)).

## Trust & security model

- Single user. Password login → httpOnly session cookie ([ADR-007](adr/007-auth-password-session-cloudflare.md));
  **MCP uses a separate revocable bearer token** ([ADR-028](adr/028-one-service-layer-mcp-peer-surface.md)).
- VPS behind Cloudflare DNS proxy (TLS at edge + origin, IP hidden). Cloudflare Access is a
  planned optional second wall.
- Secrets (Slack token, API keys, DB URL, MCP token) live only on the VPS (env file, not in
  git). The Claude Max OAuth session lives on the VPS in the `claude` CLI credential store.
