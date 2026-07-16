# ADR-049: Dedup sweep + shared merge-core — build decisions (M6 task 5)

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision) · **Refines**
[ADR-048](048-m6-chat-distiller-build-decisions.md) §9 (the dedup-sweep scope) and
[ADR-030](030-entity-substrate-and-lifecycle.md) §5 (the entity-merge primitive whose core
this extracts); builds on [ADR-026](026-graph-native-storage-obsidian-removed.md) (derived
`similar` edges are DB-only), [ADR-042](042-reprocess-all-from-raw-and-data-survival.md)
(kind-aware reprocess reset) and [ADR-047](047-pipeline-scheduling-primitive.md) (the nightly
pipeline the sweep runs in — wiring is M6 task 8, not task 5).

## Context

M6 task 5 is the nightly, all-source **dedup sweep** (ADR-048 §9): recently-ingested content
nodes that read as near-duplicates file a `dedup-proposal` review item the user resolves with
**merge / keep / link**. The build hit points ADR-048 §9 left open — chiefly which edge a
`link` writes (no associative rel exists in the seed vocabulary), how a *nightly* job stays
idempotent once a pair is decided, and how "recently-ingested" and "occurred-overlap" are
computed. Grilled decision-by-decision; these are the build-ready decisions. **No new
migration** (the sweep is watermarked off `agent_runs`, the proposal reuses `review_queue`).

## Decisions

### 1. Extract the merge-core; content-merge = the core alone (rule 10)
The M3 entity-merge (ADR-030 §5) and the M6 content-merge share one mechanism, so it is
**extracted**, not reimplemented: a new `MergeCore.fold(loser, survivor, reason,
survivor_extra_paths=())` = **retarget** the loser's inbound canonical edges → survivor across
their source files → **tombstone** the loser (`merged_into: survivor`) → **reindex** the touched
files → **force commit+push**. The store is truth (rule 1): the core rewrites *files* and lets
the indexer re-materialize the `edges`/tombstone into the DB; per-file failures are
skip-and-continue (rule 7); a partial fold is git-revertible and safe to re-drive.
- **Entity-merge** composes the core with its **alias-union** on top: it computes the union,
  `set_aliases(survivor)`, then calls `fold(..., survivor_extra_paths=[survivor.path])` so the
  rewritten survivor file is reindexed in the same pass. Behaviour-identical to the pre-extraction
  service (the alias write simply moves ahead of the retarget; both are disjoint frontmatter
  regions, same end state), and its richer summary/`agent_runs` details are composed from the core
  result + the alias info.
- **Content-merge** (dedup) is `fold(...)` **alone** — no alias union (content nodes are not the
  alias substrate). Survivor keeps its own type; the loser is tombstoned.

### 2. `link` writes a **canonical `similar`** edge
`keep` = "not a dup" (dismiss). `link` = "related but distinct — keep both, connect them." The
seed `edge_rels` (`involves, about, part_of, led_to, follows, at`) has no generic associative
relation, and no `related` concept exists. Rather than invent one, `link` writes a **canonical**
edge `rel: similar` (`origin=canonical`) from `node_a → node_b` (the canonical `least→greatest`
ordering; one edge — the neighbor read unions both directions, so no reverse row is needed).
- **Why `similar`, not a new `related` rel:** it reuses the one relatedness concept the system
  already has (the derived layer already computes `similar` = "close in embedding space"); a
  human `link` is *exactly that, confirmed*. The `(src, dst, rel, origin)` PK lets the canonical
  and derived `similar` coexist, and **nothing in the code assumes `rel='similar'` implies
  `origin='derived'`** (verified — the derived recompute only ever writes/deletes
  `origin='derived'`).
- **It persists where derived doesn't:** the nightly recompute wipes + rebuilds `origin='derived'`
  edges, so a derived `similar` between the pair can drop below the top-K floor and vanish; a
  **canonical** `similar` lives in frontmatter (rule 1) and survives forever — the right meaning
  for "the user deliberately linked these."
- **Governance:** `similar` is outside the `edge_rels` seed, but vocabulary governance gates the
  *organizer's* guesses (ADR-027), not an explicit human review action, so no `vocab-proposal` is
  involved. ❌ Rejected: a new governed `related` rel (expands the governed set + an ADR/config
  change for one review action); a `dedup_link_rel` config knob (defers the semantics with a weak
  default).

### 3. The filing gate — a strict AND of three signals
For each recently-ingested content node, a `dedup-proposal` is filed for a candidate `m` iff
**all three** hold (ADR-048 §9): **cosine(`nodes.embedding`) ≥ `dedup_min_cosine`** *and* a
**shared canonical edge to a common entity hub** *and* **occurred-overlap**. Candidates come from
an HNSW **top-K LATERAL** per recent node (mirrors `DerivedEdgeGraph.compute_similar`), so the
work is index-bounded.
- **Content nodes only.** Both ends are restricted to content types (`node_types` minus
  `entity_like_types` = `memory, conversation, insight, idea`). Entity-hub dedup is the separate
  entity-merge flow (it must keep the alias-union); folding hubs here would bypass it.
- **Occurred-overlap with open bounds.** Effective window = `[occurred_start,
  COALESCE(occurred_end, occurred_start)]`; a **null `occurred_start` on either side never
  excludes** (unknown time can't rule the pair out). The occurred signal therefore only *rejects*
  a pair when both nodes are dated and their windows are disjoint (e.g. "met Ana in 2019" vs "…in
  2024"); an undated near-dup still reaches review on cosine + shared-entity, where the human
  decides. Excluding all undated nodes would silently drop most real dups (undated memories are
  common — vision P10 spirit: don't lose signal).

### 4. Recently-ingested = a last-success watermark (no migration)
The sweep examines content nodes with `indexed_at ≥ watermark`, where `watermark` = the last
**successful** `dedup-sweep` run's `started_at` (read from `agent_runs`, the backfill idiom),
falling back to `now − dedup_window_days` on the first run / a run-store hiccup. No dedicated
watermark table — and the re-file guard (§5) makes the run-overlap re-scan harmless. The candidate
`m` may be any content node (older too — a new node often dups an old one); only the *driver* `n`
is recency-bounded. Pairs are canonicalized `(least(id), greatest(id))` and de-duplicated, and the
run is capped at `dedup_max_pairs_per_run`.

### 5. Re-file guard — a decided pair is never re-proposed
A **merged** pair self-excludes (the loser is tombstoned → `merged_into` filters it out). A
**kept** or **linked** pair stays live, high-cosine, and time-overlapping, so a naive nightly sweep
would re-file it forever. The guard: **skip any candidate pair that already has a `dedup-proposal`
row in *any* status** (`pending`/`maybe`/`resolved`/`discarded`) — matched on the canonical
`payload->>'node_a'` / `payload->>'node_b'`. Consistent consequence (correct, not a bug):
`reprocess-all` truncates `dedup-proposal` rows (kind-aware reset, ADR-042/ADR-048 §7), wiping the
guard history, so a reprocess re-proposes surviving dups — they are re-derivable, so that is fine.

### 6. Default survivor + resolution
The review item carries a **`default_survivor`** computed at file time: **higher canonical
degree** (in+out `origin='canonical'` edges — derived edges are transient noise), tie-broken by
**older `node_created_at`** (fallback `indexed_at`) — keep the original, fold the newer duplicate
into it. Storing it in the payload makes the survivor the UI shows identical to the one a
batch-merge (which sends no explicit survivor) applies.
- **Resolution** (`POST /review/{id}` `{action: merge|keep|link, survivor?}`, and the batch path):
  **merge** → content-merge via `MergeCore` (survivor = request `survivor` else
  `payload.default_survivor`; loser = the other) → status **`resolved`**; **keep** → **`discarded`**
  (dismissed, no action); **link** → canonical `similar` edge (§2) → **`resolved`**. Cross-content-
  type merges are allowed (a near-dup the organizer typed `memory` vs `insight` is still a dup; the
  human confirmed it) — the core imposes no type-equality constraint.

### 7. Payload + `augment` deferred
`dedup-proposal` payload = `{node_a, node_b, signals, default_survivor}`; `signals` =
`{cosine, shared_entity_ids, shared_entity_titles, occurred_overlap}`. Unlike `stance-candidate`,
this payload **may reference node ids** (ADR-048 §9) — a reprocess truncates the kind, so no id is
stranded. `augment` ("same event, new fact" — an LLM prose-reconciliation) stays **deferred**
(ADR-048 §9); merge/keep/link ship first.

## Consequences

- **No migration.** New code only: `entities/merge_core.py` (extracted core); `app/dedup/`
  (`store.py` candidate SQL + refile guard + degree, `sweep.py` `DedupSweepService`); a
  `dedup-proposal` branch in `ReviewService.resolve` (+ `survivor` param) and the `POST
  /review/{id}` router (`action`/`survivor` fields); `MergeService` refactored onto the core;
  config knobs (`dedup_min_cosine` 0.90, `dedup_candidate_k` 10, `dedup_window_days` 1,
  `dedup_max_pairs_per_run` 200 — rule 9, retunable); CLI verb `dedup-sweep` (pipeline wiring =
  task 8).
- **Contract docs updated:** [02-data-model](../02-data-model.md) (`dedup-proposal` payload),
  [03-api](../03-api.md) (resolution `{action, survivor?}` + payload), [04-pipelines](../04-pipelines.md)
  §3b (canonical-`similar` link, refile guard, watermark, occurred semantics),
  [08-implementation-plan](../08-implementation-plan.md) (task 5 → this ADR).
- **Not a P10 risk:** merge/keep/link are user-confirmed; a merge is a git-revertible tombstone
  (ADR-014), a link is additive, keep writes nothing. Reprocess re-derives the whole kind.
