# ADR-002: Supabase Postgres + pgvector for index and operational state

**Status:** Accepted · 2026-07-12

## Context
The system needs vector search (chunk embeddings) plus relational operational state
(capture status, connector cursors, agent runs, chat history, auth sessions). With a VPS
available, self-hosting Postgres was reconsidered.

## Decision
Keep Supabase (managed Postgres + pgvector, HNSW/cosine), accessed via `asyncpg` with
plain SQL (no ORM). Embedding dimension fixed at 1536 (`text-embedding-3-small`).

## Consequences
- ✅ Zero RAM footprint on the 4GB VPS; managed backups exactly where non-rebuildable
  operational state lives; one database for vectors + relational.
- ✅ Migration to self-hosted later = `pg_dump` + change `DATABASE_URL`.
- ⚠️ Pipelines need network to Supabase; raw capture input is persisted locally first, so
  capture never fails on DB latency.
- ⚠️ Embedding model change = migration (`alter … type vector(N)`) + full re-embed via
  reindex — acceptable, the index is disposable.
- ❌ Rejected: Postgres-in-Docker on the VPS (RAM pressure + self-managed backups of
  data we can't rebuild); dedicated vector DBs (second datastore, no gain at this scale).
