# Architecture

**Version:** 2.1 · **Status:** Approved 2026-07-13 (2.1 = M2 refresh — self-hosted nomic embeddings
via an `ollama` sidecar [ADR-022], OpenAI/Groq now STT-only, and the derived `note_links` relatedness
graph [ADR-023]; 2.0 = cloud-service pivot)
**Key ADRs:** [001 vault-on-VPS+git](adr/001-vault-on-vps-with-git-backup.md) · [003 single-service](adr/003-single-service-on-vps.md) · [004 provider-registry](adr/004-provider-registry-claude-primary-nebius-fallback.md) · [006 monorepo-decoupled](adr/006-monorepo-with-strict-server-web-decoupling.md) · [008 connectors-on-vps](adr/008-connectors-run-on-vps.md) · [013 web-on-VPS-single-origin](adr/013-web-stays-on-vps-single-origin.md) · [022 self-hosted-embeddings](adr/022-embeddings-self-hosted-nomic.md) · [023 relatedness-graph](adr/023-semantic-relatedness-graph.md)

## High-level view

```
        Phone / Desktop                          Hetzner VPS (always on)
┌──────────────────────────┐          ┌──────────────────────────────────────────┐
│  Web PWA (React+Vite)    │          │  Caddy (TLS, serves PWA, proxies /api)   │
│  capture voice/text      │  HTTPS   │      │                                   │
│  chat + model picker     │─────────▶│  FastAPI service (single process)        │
│  activity feed           │ (behind  │  ├─ API layer          (routers)         │
│  settings (agent models) │ Cloud-   │  ├─ Pipelines/services (business logic)  │
└──────────────────────────┘  flare)  │  ├─ Provider registry  (Claude Max SDK ──┼──▶ Anthropic (Max sub)
                                      │  │   primary → Nebius fallback;          ├──▶ Nebius AI (fallback LLM)
   Obsidian (optional,                │  │   STT chain Groq→OpenAI)              ├──▶ Groq / OpenAI (Whisper STT)
                                      │  ├─ ollama sidecar (nomic embeddings)    │  (self-hosted, no external call)
   occasional exploration)            │  ├─ Scheduler (03:00–05:00 agent window) │
        ▲                             │  └─ Ingestion agents (Slack, …)  ────────┼──▶ Slack API
        │ obsidian-git                │      │                                   │
        │                             │      ▼                                   │
┌───────┴────────┐   git push        ┌┴──────────────────┐                       │
│ Private GitHub │◀─────────────────│  Vault (Markdown,  │                       │
│ repo (backup + │                  │  SOURCE OF TRUTH,  │                       │
│ history)       │                  │  on VPS disk)      │                       │
└────────────────┘                  └────────┬───────────┘                       │
                                             │ index / rescan                    │
                                             ▼                                   │
                                    ┌────────────────────┐                       │
                                    │ Supabase Postgres  │◀──────────────────────┘
                                    │ + pgvector         │   operational state
                                    │ (DERIVED INDEX +   │   (cursors, runs,
                                    │  operational state)│    chat, sessions)
                                    └────────────────────┘
```

## Components

### 1. Web PWA (`second-brain/web`)
React + Vite + TypeScript single-page app, installable PWA. Premium design with
first-class animations (see [06-web-app.md](06-web-app.md)). Talks **only** to the HTTP
API — this contract is the future repo-split seam ([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)).

### 2. API service (`second-brain/server`)
Single FastAPI process hosting four layers ([ADR-003](adr/003-single-service-on-vps.md)):

- **API layer** — routers: auth, capture, chat, search, activity, settings, admin, health.
  Validation + delegation only.
- **Services / pipelines** — all business logic (capture, ingestion, indexing, chat,
  analysis — see [04-pipelines.md](04-pipelines.md)).
- **Provider registry** — named LLM/STT/embedding providers + per-task routing with
  fallback chains ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md)).
  Primary mind: Claude via Agent SDK on the Max subscription; automatic fallback: Nebius.
  **STT** walks a Groq→OpenAI Whisper chain ([ADR-020](adr/020-stt-fallback-chain-groq-primary.md));
  **embeddings** are **self-hosted** — `nomic-embed-text` on a local `ollama` sidecar (768-dim,
  single provider, no external call), the Nebius cold-swap kept in reserve
  ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)).
- **Scheduler + agents** — APScheduler firing ingestion connectors and analysis jobs in
  the **03:00–05:00 Europe/Bucharest window** ([ADR-010](adr/010-agent-window-3-5am.md)),
  staggered; every run recorded in `agent_runs` and surfaced in the activity feed.

### 3. Vault (on VPS disk)
Canonical Markdown memory. Auto-committed and pushed to a private GitHub repo after every
write batch ([ADR-001](adr/001-vault-on-vps-with-git-backup.md)) — backup, full history,
and the bridge to optional Obsidian exploration via obsidian-git.

### 4. Supabase Postgres + pgvector
Two roles, one database ([ADR-002](adr/002-supabase-pgvector-for-index.md)):
- **Derived index** (rebuildable from vault): `notes`, `chunks` (+embeddings), and `note_links`
  — the materialized semantic relatedness graph rendered back into notes as an Obsidian-visible
  `## Related notes` block ([ADR-023](adr/023-semantic-relatedness-graph.md)).
- **Operational state** (not derivable): `captures`, `connector_cursors`, `agent_runs`,
  `chat_*`, `summaries`, `auth_sessions`, `app_settings`.

## Invariants

1. Note content flows **vault → index**, never index → vault (pipeline-generated notes are
   written to the vault first, then indexed).
2. Dropping all derived tables + `POST /admin/reindex` restores search/chat over the vault.
3. Every model call goes through the provider registry; nothing else imports vendor SDKs.
4. Every background action (agent run, job, fallback event) is persisted and visible in
   the activity feed — nothing silent, nothing lost.
5. Raw inputs (audio files, raw connector payload digests) survive pipeline failures.
6. `web/` and `server/` share nothing but the HTTP contract in [03-api.md](03-api.md).

## Trust & security model

- Single user. Password login → httpOnly session cookie ([ADR-007](adr/007-auth-password-session-cloudflare.md)).
- VPS behind Cloudflare DNS proxy (TLS at edge + origin, IP hidden). Cloudflare Access is
  a planned optional second wall.
- Secrets (Slack token, API keys, DB URL) live only on the VPS (env file, not in git).
  The Claude Max OAuth session lives on the VPS in the `claude` CLI credential store.
