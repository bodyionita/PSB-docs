# ADR-037: profile-embedding-in-search — retrieval mechanism (M3)

**Status:** Accepted · 2026-07-14 · Implements the *"embedded — searching 'Alex' hits one dense,
current summary"* clause of [ADR-030 §4](030-entity-substrate-and-lifecycle.md) for
[08 §M3](../08-implementation-plan.md) task 10 · scopes against [ADR-032](032-prior-art-adoptions.md)
(RRF hybrid retrieval = M4). Decided in the M3 task-10 (Accept) implementation session, when the
real-DB smoke confirmed the stored profile vector is never queried — an unbuilt contract
requirement — and recorded before coding, per [09](../09-session-protocol.md).

## Context

[ADR-030 §4](030-entity-substrate-and-lifecycle.md) decided the derived entity **profile** is
**embedded** so that searching an entity's name surfaces one dense, current summary — the whole
reason a *thin* canonical hub is legible. The M3 profile-refresh job (task 6) computes and stores
that vector in `node_profiles.embedding` (with the same `search_document:` nomic prefix the indexer
uses for chunks). But M3 search (`search_chunks`) ranks **chunks only** — the profile vector is
written and never read. An entity is a thin hub with (usually) no body chunks, so today searching
"Alex" can only hit memory chunks that *mention* Alex, never the Alex node with its summary. This is
the gap flagged "open before Accept (profile-embedding-in-search)" on task 6.

The fix is a retrieval-layer change with several non-obvious choices worth pinning (all-tiers
eligibility, no leg weighting, reindex decoupling) so M4's RRF work knows M3 did a deliberate raw
union. RRF/FTS rank-fusion is explicitly **M4 chat retrieval** ([04-pipelines](../04-pipelines.md) §
"M4 retrieval"), out of scope here.

## Decision

**M3 search gains a second vector-retrieval leg over `node_profiles.embedding`, unioned with the
chunk leg in one query.**

1. **Union of two legs, best-per-node.** `search_chunks` becomes `UNION ALL` of (a) the existing
   per-chunk scan over `chunks.embedding` and (b) a per-profile scan over `node_profiles.embedding`,
   both against the shared `search_document:` / `search_query:` nomic space (ADR-022). `DISTINCT ON
   (node_id) ORDER BY node_id, distance` collapses to the single best-scoring row per node across
   both legs (free dedup when a node matches on both), then `WHERE score >= min_score ORDER BY score
   DESC LIMIT top_k` as today. `score = 1 - cosine_distance` — identical metric on both legs, so
   chunk-vs-profile scores are directly comparable.
2. **Eligibility guards.** Profile leg: `embedding IS NOT NULL` (the job stores null on an embedder
   outage) and `merged_into IS NULL` (tombstones excluded, like the chunk leg). Plane/type filters
   apply in both legs via the join to `nodes`. **All tiers are searchable** (stub/snapshot/full) —
   the long tail of low-degree entities is the majority, and excluding stubs would make most entity
   nodes un-findable by their own name, defeating the ADR-030 §4 intent; `min_score` + `top_k` bound
   any stub noise. Only entity-like nodes carry profiles, so the profile leg surfaces exactly the
   entity hubs.
3. **No leg weighting; raw union.** Scores are compared as-is — no boost/normalization on the
   profile leg. Rank-based fusion (RRF, k=60) is the M4 retrieval operator (ADR-032 §5); introducing
   a weight now would be unspecified tuning. A profile is a centroid-ish dense vector so its scores
   may skew, accepted for M3.
4. **`SearchResultItem` unchanged; no marker.** A profile-won hit maps onto the existing shape:
   entity `node_id`/`type`/`title`/`plane`/`planes`/`tags`, `snippet` = the profile text (trimmed to
   `search_snippet_max_chars`, same as a chunk snippet), `score` = its cosine. No `via`/`is_profile`
   field — the snippet already reads as a summary, and a marker is an API/OpenAPI/web change for
   marginal value (deferred until a surface needs it). No web change: profile hits render as entity
   cards through the task-8 type-icon UI.
5. **Reindex stays decoupled from profiles.** Reindex does **not** rebuild profiles. A plain
   `POST /admin/reindex` preserves them (node ids are file-sourced and stable, so the `ON CONFLICT
   (id)` upsert keeps the `node_profiles` FK intact). After a **full DB wipe**, profile-search is
   degraded (empty) until the nightly `profile-refresh` job (or a manual `python -m app.cli
   profile-refresh`) repopulates — profiles are DB-only derived state, not in the store, so this
   respects rule 1 ("reindex restores search *from the store*") and ADR-030 §4 / ADR-034 (nightly,
   LLM-frugal). Coupling reindex to an LLM profile rebuild would blow spend and cross the rule-1
   boundary. The task-10 Accept's DB-wipe/reindex **parity** criterion stays scoped to chunks +
   canonical/derived edges; profile-search is demonstrated separately by running `profile-refresh`
   after the seed captures.
6. **Schema — migration 007.** Add `hnsw (embedding vector_cosine_ops)` on `node_profiles.embedding`,
   mirroring `chunks_embedding_hnsw`, so the profile leg is ANN-indexed rather than a sequential scan.

## Rationale

- One query, one metric, one `min_score`/`top_k` — the profile leg is additive and reuses the whole
  existing collapse/rank/limit path; `DISTINCT ON` gives cross-leg dedup for free.
- Reads `node_profiles` directly rather than materializing profile text as a sentinel `chunks` row —
  which would be **wiped on every reindex** (chunks are rebuilt from store files; profiles are DB-only
  derived) and would break both rule 1 and the Accept parity check.
- Keeps the M3 retrieval contract simple and the M4 RRF door open: a deliberate raw union now, rank
  fusion later, no weighting cruft to unwind.

## Consequences

- `search_chunks` SQL grows the profile `UNION ALL` leg; `SearchStore`/`SearchService` signatures and
  `SearchResultItem` are unchanged. `02-data-model` (migration 007 + index), `03-api` (§Search:
  profiles are a retrieval source, result shape unchanged), and `04-pipelines` (search leg) updated in
  the same change set; `08-logs/m3.md` task-10 records the build.
- **Migration 007** (`node_profiles` HNSW index) — the first migration past 006; plain SQL, no ORM
  (ADR-011).
- **Testing:** unit tests cover the service via the fake store; the real-DB SQL smoke's prior
  "[EXPECTED-GAP] profile NOT reachable" assertion **flips** to prove a profile-only entity (no
  chunks) now surfaces via `search_chunks` — the regression anchor for this ADR.
- No new config: `min_score`/`top_k`/`snippet_max_chars` are shared; **no** profile-leg kill-switch
  (deliberately omitted — this is a corrective to meet the contract, not an experiment).
- ❌ Rejected: **profile-as-sentinel-chunk** (reindex-wiped, rule-1 violation); **stub tier excluded**
  (hides the majority of entities by name); **leg weighting / RRF now** (M4 scope); **a `via` marker
  field** (contract + web change, marginal value); **reindex triggers profile rebuild** (LLM-spend +
  rule-1 boundary).
