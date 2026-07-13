# ADR-026: Graph-native storage; Obsidian removed from the architecture

**Status:** Accepted · 2026-07-13 · Partially supersedes [ADR-001](001-vault-on-vps-with-git-backup.md)
(the Obsidian-bridge half), [ADR-005](005-planes-and-atomic-notes.md) (the folder-=-plane half),
[ADR-023](023-semantic-relatedness-graph.md) (the render-into-files half). Companion:
[ADR-027](027-typed-vocabulary-governance.md) (vocabulary), [ADR-028](028-one-service-layer-mcp-peer-surface.md)
(surfaces), [ADR-029](029-conversational-ingestion-stance-gate-review-queue.md) (chat ingestion).

## Context

The system's purpose was re-stated (2026-07-13, grilled decision-by-decision): ingest a life —
thoughts, memories, conversations with people across sources, ideas — and hold it **organized,
categorised, and mapped with meaningful relations**: a mind graph, explorable conversationally
and visually, accessible to the user's own app *and* to other LLMs via MCP.

Obsidian turned out to be load-bearing nowhere and constraining everywhere: the untyped
note-with-tags atom, wikilinks, the `sb:related` rendered block (plus its churn-gating and
content-hash exclusion rules — source of two M2-Accept bugs), "valid Obsidian tags", folder
semantics, `.obsidian/` ignores, obsidian-git merge discipline. None of it serves the mind-graph
purpose; all of it shapes the data model. **Decision: remove Obsidian entirely and design the
storage format for the graph.**

## Decision

1. **Files + git remain canonical; the DB remains derived.** The never-lose substrate and all
   [ADR-014](014-vault-history-durability.md) machinery (atomic writes, ff-only push, WORM
   bundles, integrity drills) survive unchanged. Only the *format* changes, not the substrate.
2. **The atom is a typed node.** Starter vocabulary: `memory` · `person` · `idea` ·
   `conversation` · `insight` ([ADR-027](027-typed-vocabulary-governance.md)). Entities —
   people first — are first-class nodes that memories link to.
3. **Canonical typed edges** (starter: `involves`, `about`, `part_of`, `led_to`, `follows`)
   are written by the organizer into the **source node's frontmatter**, targeting node `id`s.
4. **Similarity edges are derived-only** — computed into the DB, **never rendered into files**.
   The `sb:related` block, its churn-gating, hash-exclusion rules, and the co-capture
   `## Related` strip logic are deleted with no replacement.
5. **Node file format:** Markdown + YAML frontmatter (prose stays prose; structure lives in
   frontmatter). **Folder = node type**; plus one system folder `inbox/` for organizer-fallback
   nodes. **Identity = `id`** (uuid, frontmatter); **filename = slug + short-id** — readable,
   rename-safe, collision-proof. Planes survive as attributes only (`plane`, `planes[]`),
   used for filtering/scoping/coloring — never as folders.
6. **Fresh start.** The existing vault (4 test memories) is **archived** (GitHub repo archived,
   kept read-only); the new store boots empty via the bootstrap. **No migration code is written.**
7. **Full rename.** The concepts are **nodes / edges / the graph**; the on-disk store is
   **the graph store**. Docs, DB schema, API paths, code identifiers and config
   (`GRAPH_STORE_PATH`, `/srv/graph-store`) all follow. "Note" and "vault" survive only in
   superseded ADRs and historical logs.

### Node file shape (canonical example)

```
graph-store/
├── memory/2026-07-13--dinner-with-alex--018f3c2e.md
├── person/alex-marsh--018f4a11.md
├── idea/second-brain-connector--018f5b02.md
├── conversation/…
├── insight/…
└── inbox/                       # organizer-fallback nodes until re-organized
```

```yaml
---
id: 018f3c2e-…            # THE identity; filename is a readable projection
type: memory
created: 2026-07-13T21:40:00+02:00
source: voice             # voice|text|slack|chat|mcp|…
source_ref: "…"
plane: personal
planes: [personal, friends]
tags: [dinner, catching-up]
edges:
  - {rel: involves, to: 018f4a11-…}
  - {rel: part_of,  to: 018f6c33-…}
---
(Markdown prose body — the memory itself)
```

## Rationale

- Files+git canonical (A over DB-canonical): keeps the proven, already-live never-lose story;
  nothing about a richer graph requires DB truth — edges materialize into Postgres for query
  speed exactly as the relatedness graph already did.
- Typed nodes over untyped notes: "chats with people across sources" only becomes a *map* when
  the person is a node all sources converge on.
- Edges-in-frontmatter over rendered link blocks: one write path, one truth, no feedback-loop
  hazards; the file *is* the graph fragment.
- Fresh start over migration: 4 test memories don't justify a migration tool.
- Full rename: a single-user API with zero external consumers; the MCP is born speaking the
  new language. Carrying "note = node, vault = graph store" translation forever costs more.

## Consequences

- Migration 005 rebuilds the derived schema (`nodes`, `chunks`, `edges` — see
  [02-data-model.md](../02-data-model.md)); the M3 pivot milestone retargets organizer,
  indexer, search, and API naming ([08-implementation-plan.md](../08-implementation-plan.md)).
- A new GitHub repo backs the graph store (name = open item, README §5); the old `PSB-vault`
  repo is archived, its R2 WORM bundles retained.
- Chunking/embedding survive conceptually (per-node chunks, nomic prefixes, ADR-022), minus
  all strip-the-rendered-block special cases — the embedded identity of a node is frontmatter-
  stripped prose only, which is now trivially true.
- ❌ Rejected: Postgres-canonical truth (discards working durability for power derivable
  anyway); split truth files+DB (two never-lose stores to reconcile); keeping Obsidian
  compatibility "just in case" (it constrains the format and serves no stated purpose);
  migrating the 4 test notes (cost without value).

## Appendix — ruthless-cleanup inventory (every Obsidian artifact and its fate)

| Artifact | Where | Fate |
|---|---|---|
| Wikilinks `[[…]]` in note bodies | organizer output, co-capture siblings | **Deleted** — siblings become `edges` frontmatter entries |
| `sb:related` rendered block + churn-gating + render step | indexer/nightly job, ADR-023 | **Deleted** — similarity edges live in DB only |
| `content_hash` exclusion of the `sb:related` block | indexer, 02 §3 | **Deleted** — hash = whole file again |
| Co-capture `## Related` section + embed-strip rule | organizer, 02 §4 | **Deleted** — replaced by edges; embed path strips frontmatter only |
| `related:` frontmatter field | note contract | **Replaced** by `edges:` (typed, id-targeted) |
| Folder = plane; `Inbox/`, `Summaries/` plane folders | 02 §1, ADR-005 | **Replaced** — folder = type (+ `inbox/` system folder); planes are attributes; summaries become `insight` nodes |
| Date-title filenames + numeric-suffix collision logic | note writer | **Replaced** — slug + short-id, id is identity |
| "Valid Obsidian tags" framing | 02 §2, ADR-024 | **Reworded** — same slug rules kept on their own merits (`_slugify_tag` unchanged) |
| `.obsidian/` in `VAULT_IGNORE` + `.gitignore` | 02 §1, 07, ADR-014 §3 | **Deleted** from the new store's ignores |
| obsidian-git merge-only guidance | ADR-014 §4, 07 | **Generalized** — any external git client must be merge-only; no Obsidian-specific text |
| `*.md eol=lf` `.gitattributes` (CRLF churn fix) | ADR-014 amendment | **Kept** — correct regardless of Obsidian |
| "Explorable in Obsidian" vision line; Obsidian in diagrams | 00, 01 | **Deleted** — exploration = the in-app map (M7) + any editor on a clone |
| `vault` / `note` vocabulary (docs, schema, API, code, config) | everywhere | **Renamed** — graph store / node / edges (`GRAPH_STORE_PATH`, `/srv/graph-store`, `nodes`/`edges` tables, `GET /nodes/{id}`) |
| Local `ObisidanVault/` dev folder | workspace layout, README | **Replaced** by a local dev graph-store scratch clone |
