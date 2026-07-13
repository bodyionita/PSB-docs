# ADR-032: Prior-art adoptions — research-validated refinements to the graph design

**Status:** Accepted · 2026-07-13 (post-M3-grilling research pass; user: "adopt all") · Refines
[ADR-030](030-entity-substrate-and-lifecycle.md)/[031](031-m3-organizer-and-contract-extensions.md)
without changing their decisions · updates [02](../02-data-model.md), [03](../03-api.md),
[04](../04-pipelines.md), [08](../08-implementation-plan.md).

## Context

Two web-research agents surveyed the state of the art (LLM memory systems: mem0, Zep/Graphiti,
Letta, LangMem, Basic Memory; graph-RAG: GraphRAG/LazyGraphRAG, LightRAG, HippoRAG 2, SPRIG;
PKM models: Tana/Roam/Logseq; retrieval practice: Postgres hybrid search; graph-exploration UX:
Obsidian/TheBrain/InfraNodus/Nomic Atlas/Map-of-GitHub; rendering libraries). **Validation
headline:** the decided design matches or leads the field — git-canonical durability (no surveyed
system has it), node bi-temporality, never-guess entity resolution, vocabulary governance, the
stance gate (ahead of the provenance literature), thin-hub+derived-profile, the sleep cycle, and
the SPRIG paper demonstrates our exact retrieval stack (RRF + seeded PPR) in a 4 GB CPU-only
envelope. Adoptions below are sharpenings, not redesigns.

## Adopted — M3 (contract window)

1. **Edge `until`.** Frontmatter edges become `{rel, to, conf?, since?, until?}`; `edges.until
   date null` column in migration 005. The organizer may *close* a relationship a memory clearly
   supersedes ("quit Google") — invalidate, never delete (Zep/Graphiti bi-temporal edges; the
   field's most consistently shipped temporal feature). Populated conservatively; lighter than
   the rejected `supersedes` edge type.
2. **Deterministic resolution short-circuit** (Zep/Graphiti IR-front-end): a single
   exact/normalized alias-index hit auto-links with **no LLM round-trip** (high `conf`, `since`
   set); only multi-candidate or fuzzy-only cases reach the LLM / review queue. Plus an
   **intra-capture dedup pass** (same new entity mentioned twice in one capture mints one node)
   and an **entropy guard**: short/low-entropy aliases ("Al", "IT", "mom") never fuzzy
   auto-link — exact match or review.
3. **Derived profiles are categorized observations**, not prose blobs: `[role]`, `[location]`,
   `[last-seen]`-style lines extracted from the 1-hop neighborhood (Basic Memory observations /
   Tana fields) — scannable in the map, chunk-friendly to embed, diffable nightly. Derived
   layer only; canonical prose rules unchanged.
4. **Cheap-day / strong-night routing defaults** (Letta "sleep-time compute", as ADR-025
   configuration, not code): sync capture-organize runs at lower effort; nightly
   consolidation/profile/reflection jobs run at higher effort.

## Adopted — M4 (refines the ratified retrieval addendum)

5. **RRF is the union operator** (rank-based, k=60) for the FTS ⊍ vector legs — never blend raw
   cosine with `ts_rank` (incommensurate scales); with degenerate-signal suppression (a
   near-flat leg is down-weighted) and **FTS weight → 0 for non-English raw queries** (vectors
   stay cross-lingual).
6. **Condensation renders the query in English** (one prompt line) — the corpus is
   English-by-construction; a Romanian `tsquery` matches nothing.
7. **Mild recency prior** on `occurred ?? created` in ranking (SQL-only decay).
8. **Expansion guards:** config cap on injected 1-hop neighbors (injection-hygiene consistent);
   entity-seeding **falls back to vector/FTS seeds** — never a hard gate (SPRIG: ~42% of queries
   may lack an extractable entity); the expansion function is designed **PPR-swappable** (seeded
   Personalized PageRank replaces the flat 1-hop union later without API change).
9. **Temporal filters** — optional `as_of` / date-window params on `POST /search` (M4) and the
   MCP `search`/`traverse` tools (M5), running on the `occurred` range columns.

## Adopted — at their milestones

10. **M5:** MCP `traverse` = center + depth + rel filter + **cursor pagination** (mirrors
    `GET /nodes/{id}/neighbors`); optional `build_context(seed, depth)` convenience bundling
    get_node+traverse in one round-trip (Basic Memory pattern).
11. **M6:** dedup review items gain an **`augment`** verb (append new detail to the existing
    node — many near-dupes are "same event, new fact"; Mem0's UPDATE, human-gated); nightly
    **re-split proposals** for bloated/multi-plane nodes (Basic Memory "defrag"); queue/reflection
    **salience = graph degree + user pins + edge conf** (no LLM scoring).
12. **M7:** `react-force-graph` **2D canvas build only** (bundle-lean, phone-safe); plex-style
    animated re-center (TheBrain); **rel-based zones** (people one side, topics another) over
    pure physics; per-hop fanout cap with "show more".

## Backlog (recorded with evidence)

**Continents architecture** — nightly server-side layout (UMAP/community detection over
`nodes.embedding`) served as static tiles, clusters LLM-named once/night (Map-of-GitHub /
Nomic Atlas pattern; never live client GPU layout) · **PPR upgrade** (SPRIG GraphRRF,
+12.5% Recall@10 over BM25, CPU-only; requires hub down-weighting — top-degree nodes dominate
walks) · **structural-gap prompts** for the M10 reflection agent ("much about X and Y, nothing
linking them" — InfraNodus) · **serendipity resurfacing** (M10: "On This Day" on the `occurred`
range + a weighted random-walk digest over edges with a "why this" path; tune the similarity
floor first) · **per-type field templates** (Tana supertags — with M11's task/goal types, via
ADR-027 governance).

## Explicitly rejected (do not revisit without new evidence)

- ❌ **Auto-UPDATE/DELETE of memories** (Mem0): unaudited data loss; our review-queue +
  git-soft-delete stands.
- ❌ **GraphRAG global community summaries**: 20–100× embedding cost, re-summarize-on-ingest;
  `topic` nodes + planes + nightly derived edges are the cheap substitute.
- ❌ **LLM-in-the-retrieval-loop / cross-encoder rerank on the hot path**: 2 vCPU contention
  for marginal gain over RRF; revisit only if live tuning shows RRF ordering fails.
- ❌ **Live whole-graph force layout** (the Obsidian hairball, "useless past ~200 nodes"):
  aerial view = precomputed tiles or nothing.
- ❌ **Per-node LLM importance scoring / importance-triggered reflection** (generative-agents):
  noisy + a call per memory; fixed windows + derived salience suffice.
- ❌ **In-app agentic traversal now**: 3–10× cost, 2–5× latency; M5 MCP lets external LLMs run
  the loop first — build in-app only if one-shot demonstrably fails (already the backlog order).

## Consequences

Migration 005 gains one column (`edges.until`); the organizer contract gains the short-circuit/
dedup/entropy rules; ADR-025 routing gets day/night effort defaults; the M4 addendum gains its
operator (RRF) and guards. Full source list lives in the research session (2026-07-13); key
refs: Zep/Graphiti (arxiv 2501.13956), SPRIG (arxiv 2602.23372), HippoRAG 2 (2502.14802),
LightRAG (EMNLP 2025), Letta sleep-time compute, Basic Memory, Map of GitHub (anvaka),
micelclaw/ParadeDB Postgres RRF.
