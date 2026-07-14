# ADR-036: M3 edge retro-consolidation walk = re-typings of existing edges only

**Status:** Accepted · 2026-07-14 · Refines [ADR-035](035-vocabulary-consolidation-scope-m3.md)
(which fixed *edges apply / nodes propose-only*) by pinning the **edge-walk mechanics** for
[08 §M3](../08-implementation-plan.md) task 7b · generalizes the on-demand shape of
[ADR-024](024-tag-vocabulary-reuse-and-consolidation.md) §2. Decided in the M3 task-7b
implementation session when the walk's *input/output* turned out to be under-specified — the two
contract docs disagreed — and recorded before coding, per [09](../09-session-protocol.md).

## Context

ADR-035 settled that the M3 `vocab-consolidation` job **applies** edge retro-consolidation (safe
frontmatter rewrites) while nodes stay propose-only. It did **not** pin what the edge re-walk feeds
the LLM or what it emits, and the two docs describing it diverge — both authored in the same commit
(`a528eba`), so this is loose phrasing of one intent, not a later refinement of an earlier one:

- [ADR-035 §2](035-vocabulary-consolidation-scope-m3.md): the walk "proposes **new/re-typed edges**
  that should use the new rel."
- [04-pipelines.md](../04-pipelines.md): the walk "proposes + (on confirm) applies **edge
  re-typings** (frontmatter rewrites)" — no mention of inventing new edges.

The gap is not cosmetic. The two readings are very different amounts of machinery:

- **Re-type an existing edge** = change one frontmatter line's `rel:` on the edge's source node
  (`{rel: …, to: …}`). Bounded, cheap, byte-level — the exact ADR-024 tag-consolidation envelope,
  and the `NodeWriter` already has the write-side twin (`retarget_edges` rewrites `to:`; this adds
  `retype_edge` rewriting `rel:`).
- **Invent a brand-new edge** = re-read node **bodies** to discover relations that were never
  captured, i.e. a partial re-run of the organizer's structural job. Heavy, expensive, and exactly
  the class of "improvise machinery days before Accept" that ADR-035's rationale pushes to a
  follow-up. The `new` word was carried over verbatim from ADR-027 §3's "re-typings/new edges" and
  was never a deliberate M3 commitment.

## Decision

**The M3 edge retro-consolidation walk proposes re-typings of existing canonical edges only.**

1. **Candidate set = existing canonical edges, bounded.** On propose for an approved `edge_rel`, the
   job reads the current canonical edges from the index (source node id + title + short body/context,
   current `rel`, target id + title), **capped by a config setting** (an edge analogue of
   `tags_consolidate_max_vocabulary`, frequency/recency-ordered so a large graph is bounded — the
   ADR-024 "top-N to bound the prompt" treatment). It asks the LLM which of these edges would be
   **more accurately typed as the newly approved rel**, and returns a plan of
   `{src_id, to, from_rel → to_rel}` re-typings. No node bodies are scanned for *missing* edges.
2. **On-demand two-step, mirroring ADR-024 tags.** `POST /admin/vocab/consolidate` is a manual,
   human-in-the-loop action: **propose** (`apply=false`, input = the target `edge_rel`) → `200
   {plan_id, retypings[]}`, no writes; **apply** (`apply=true` + the reviewed plan) → `202 {run_id}`,
   rewriting the affected edges' `rel:` frontmatter + reindexing the touched files + force-commit in
   the background (`agent="vocab-consolidation"`). The plan is stateless server-side (apply carries
   it), like tag consolidation. The **7a approve-time run is unchanged** — it stays the feed-visible
   "type is now live" marker; this retro-walk is the separate, user-invoked cleanup, exactly as
   ADR-024 keeps tag consolidation deliberately manual (not auto-fired on every approval).
3. **Nodes unchanged from ADR-035.** A `node_type`/`entity_type` walk stays propose-only (surfaces
   candidate re-typings); its apply machinery remains the deferred ADR-035 follow-up.

## Rationale

- Keeps the edge half inside the proven ADR-024 safety envelope (bounded prompt, confirm-gated,
  atomic + git-revertible frontmatter rewrites) and ships all of ADR-035's *edges-apply* value.
- Resolves the ADR-035 §2 / 04-pipelines wording split in favour of the reading that ADR-035's own
  rationale already implies (safe frontmatter rewrites, no identity/body machinery before Accept).
- On-demand (not auto-on-approve) matches ADR-024's explicit stance: silent whole-graph rewrites on
  every approval are exactly what the human-in-the-loop tool exists to avoid.

## Consequences

- **Follow-up (recorded, deferred):** *new-edge invention on consolidation* — an organize-style
  body re-read that proposes edges the graph never captured but which should use a rel. Its own task
  (post-M3, alongside the ADR-035 node-retype apply + the M6 dedup-proposal drain). Until it lands,
  the walk only re-types edges that already exist.
- New config: an edge-inventory cap for the propose prompt (`VOCAB_CONSOLIDATE_MAX_EDGES`-style,
  settings module only — CLAUDE.md rule 9). New API row `POST /admin/vocab/consolidate` in
  [03-api](../03-api.md); 04-pipelines edge-walk wording reconciled to point here.
- `NodeWriter.retype_edge` joins the writer surface (rewrite an edge line's `rel`, keyed by `to` +
  old `rel`; atomic, idempotent, dedups against an identical existing edge — the `retarget_edges`
  treatment). It is edge-only; no folder/filename/`node_paths` identity is touched (that is the
  node-retype follow-up).
- ❌ Rejected: **new-edge invention now** (unbudgeted body-scan machinery before Accept, against
  ADR-035's rationale); **auto-fire the walk on approve** (diverges from ADR-024's on-demand,
  human-confirmed stance and couples a heavy LLM pass to every approval).
