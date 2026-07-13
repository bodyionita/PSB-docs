# ADR-011: Alembic for migrations, authored as plain SQL (no ORM)

**Status:** Accepted · 2026-07-12
**Terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md), 2026-07-13):** derived
tables are now `nodes`/`chunks`/`edges`; "vault" → graph store. The Alembic / plain-SQL / no-ORM
decision stands unchanged.
**Supersedes:** the "schema changes only as numbered `.sql` files" clause of
[templates/CLAUDE.md](../templates/CLAUDE.md) rule 5 and the migration-mechanism
implication of [02-data-model.md](../02-data-model.md). The **no-ORM** and
**plain-SQL-via-asyncpg** rules stand unchanged.

## Context
M0 ships the full schema ([02-data-model.md](../02-data-model.md)) and every later
milestone adds tables/indexes. The original plan applied numbered `.sql` files via a tiny
in-house runner (forward-only, no version graph, no down path). The user wants a proper,
maintainable migration tool with `upgrade`/`downgrade` and a standard workflow.

Two concerns were explicitly separated during grilling:
1. **Migration tooling** — how schema changes are versioned and applied.
2. **Data access** — how application code reads/writes rows.

The system's centerpiece is pgvector similarity search, which is exactly where an ORM adds
friction and no benefit; the schema is small (~11 tables) and mostly flat; asyncpg is the
fastest PG driver and keeps the data layer thin and transparent. So the ORM's cost lands on
the feature that matters most while its wins (autogenerate, models-as-source-of-truth) are
muted here.

## Decision
Adopt **Alembic for migrations only**, with **no ORM**:

- Migrations are Alembic revision files, but their bodies are **explicit SQL** via
  `op.execute(...)` / `op.create_table(...)` — **no SQLAlchemy models**, therefore
  **no `--autogenerate`** (revisions are hand-authored).
- Application data access stays **plain SQL over asyncpg** (unchanged; ADR-002, CLAUDE.md).
  SQLAlchemy is a migration-time dependency, never imported by services/routers.
- `upgrade()` and `downgrade()` are both written. For the **derived index** tables
  (`notes`, `chunks`) downgrades are best-effort only — the real recovery path is
  `POST /admin/reindex` from the vault, not a schema rollback ([02-data-model.md](../02-data-model.md) §5).
- **Application:** migrations are applied **explicitly** via `alembic upgrade head` in
  CI and `deploy/provision.sh` — never inside the FastAPI request/boot path in production.
  The service on startup only **checks and warns** if the DB is behind `head`. Local dev
  runs the same explicit one-liner (a documented convenience may auto-apply in dev only).
- pgvector: the `vector` extension and `vector(1536)` columns are created via raw SQL in
  the migrations; no `pgvector.sqlalchemy` needed since there are no models.

## Consequences
- ✅ Real version graph, `upgrade`/`downgrade`, and a mainstream, maintainable tool.
- ✅ Query layer stays plain, fast asyncpg; the "vault is truth, index is disposable"
  model is untouched.
- ✅ Migrations are reviewable as plain SQL — no hidden ORM diffing.
- ⚠️ No autogenerate: every schema change is a hand-written revision (acceptable at this
  schema size; keeps migrations honest).
- ⚠️ Alembic pulls in SQLAlchemy as a dependency; it must never leak past `migrations/`.
- ❌ Rejected: full SQLAlchemy ORM + Alembic (pivots the whole data layer, muted benefit,
  friction on vector search); forward-only in-house runner (no down path, non-standard).
