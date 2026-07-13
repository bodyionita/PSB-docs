# ADR-031: M3 contract extensions — pipeline shape, `occurred`, seeded vocabulary, edge metadata, hygiene, zero-touch cutover

**Status:** Accepted · 2026-07-13 (M3 grilling) · Companion to [ADR-030](030-entity-substrate-and-lifecycle.md);
finalizes the [ADR-026](026-graph-native-storage-obsidian-removed.md)/[027](027-typed-vocabulary-governance.md)
node/edge contract for build · updates [02](../02-data-model.md), [03](../03-api.md), [04](../04-pipelines.md),
[07](../07-infrastructure.md), [08 §M3](../08-implementation-plan.md).

## Decisions

1. **Pipeline shape: synchronous-full organize stays (option A).** Every capture gets the full
   organize in-line, as today — the Claude-Max-window concern was re-examined and downgraded:
   connectors are already nightly batch jobs by construction, so sync volume = UI + MCP
   captures, both human-initiated. Guard: **MCP `capture` gets a burst queue** (beyond N
   in-flight, captures wait — config). Tiered organizing (cheap sync pass + nightly batch) is
   the **documented evolution path** if window pressure ever materializes. Effort levels in
   practice top out at `high`.
2. **Event time.** Optional frontmatter **`occurred`** (partial ISO — `2025`, `2025-07`,
   `2025-07-10`) + optional **`occurred_end`** for ranges; precision is implicit in the
   partial date. The indexer expands to a `[occurred_start, occurred_end]` DB range. Every
   time axis (reflection windows, map timelines) runs on **`occurred ?? created`**. The
   organizer extracts it only when the text implies a time — **never fabricated**; absent
   means unknown.
3. **Seeded vocabulary.** Node types (9): `memory · person · idea · conversation · insight ·
   place · event · project · topic`. Edge rels (6): `involves · about · part_of · led_to ·
   follows · at` (memory→place). Guidance: *event = bounded happening; project = ongoing
   directed effort; idea = actionable proposal; topic = recurring theme.* `part_of` targets
   event/project/conversation. **`about` stays unsplit** (targets topic/idea) — governance
   (ADR-027) splits it later if it bloats. Entity machinery (aliases/profiles, ADR-030) is
   generic across `person/place/topic/idea/event/project`.
4. **Edge metadata + organizer versioning.** Frontmatter edges are `{rel, to, conf?, since?}`
   — `conf` omitted ⇒ 1.0; `since` = currency date (typically the memory's
   `occurred ?? created`). In the DB `edges` table one numeric column **`score`** serves both
   origins (confidence for `canonical`, cosine for `derived`). Every node carries
   **`organizer_version`** in frontmatter — future quality-retrofits target "everything below
   vN" instead of whole-graph re-walks.
5. **Injection hygiene (organizer-contract law).** (a) every prompt injection is
   retrieval-bounded, never O(corpus) — incl. the ADR-024 tag list once it grows (bounded to
   most-used + embedding-similar, config cap); (b) all ingested text (capture/Slack/MCP/chat)
   sits behind hard delimiters and is treated as material, never instructions; (c) node bodies
   are never injected into the organizer/resolver — structured candidate fields only; (d) MCP
   text gets zero special trust.
6. **Bootstrap & zero-touch cutover.** New private GitHub repo **`PSB-graph`**;
   `GRAPH_STORE_REPO` config in `defaults.env`. **App-level bootstrap** (idempotent, M1
   pattern): missing/empty `/srv/graph-store` → `git init` + remote add + skeleton (9 type
   folders + `inbox/` + `.gitkeep`s, `.gitattributes` `*.md eol=lf`, OS-cruft-only
   `.gitignore`) + initial commit + `push -u`. The deploy workflow **prints the VPS's existing
   public deploy key into the Actions log** once. **The user's entire manual surface is GitHub
   UI:** create `PSB-graph` → paste the deploy key (write access) → after the M3 Accept,
   archive `PSB-vault` (read-only; its R2 bundles retained, new bundles land as
   `graph-store-*` in the same WORM bucket). **No SSH, no VPS steps.**

## Consequences

- Migration **005**: drop note-model derived tables; create `nodes` (+`type`, `aliases` GIN,
  `disambig`, `occurred_start/end`, `organizer_version`, `merged_into`), `chunks(node_id)`,
  `edges(src_id,dst_id,rel,origin,score,since)` pk(src,dst,rel,origin), `review_queue(kind,…)`;
  rename `captures.note_paths → node_paths`. Config: `GRAPH_STORE_PATH`, `GRAPH_STORE_REPO`,
  `NODE_TYPES`, `EDGE_RELS`, `ENTITY_MATCH_MIN_CONF`, MCP burst + profile-refresh settings.
- The M4 (graph-aware retrieval lite + FTS leg) and M6 (segmentation, queue ergonomics,
  dedup-via-queue, inbox drainer, sleep-cycle framing) addenda were **ratified 2026-07-13**,
  to be **re-checked at each milestone's kickoff** before build (08 records this).
- ❌ Rejected: tiered organizing now (complexity ahead of evidence); a granularity enum next
  to `occurred` (redundant with partial ISO); seeding `concerns`/`mentions` as an `about`
  split (premature); a separate vocab-proposals table (queue kind instead, ADR-030); any
  cutover step requiring the user on the VPS.
