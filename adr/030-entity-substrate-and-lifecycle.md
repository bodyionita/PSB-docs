# ADR-030: Entity substrate & lifecycle — aliases, bounded resolution, thin hubs + derived profiles, merge/backfill

**Status:** Accepted · 2026-07-13 (M3 grilling) · Implements the entity half of the
[ADR-026](026-graph-native-storage-obsidian-removed.md) pivot; **extends**
[ADR-027](027-typed-vocabulary-governance.md)/[ADR-029](029-conversational-ingestion-stance-gate-review-queue.md)
(review queue pulled forward + made kind-generic) · updates [02](../02-data-model.md), [03](../03-api.md),
[04](../04-pipelines.md), [08 §M3](../08-implementation-plan.md).

## Context

The graph's value hinges on "Alex" / "Alexandru" / "my brother" collapsing onto one `person`
node — with no silent wrong links, no unbounded prompt growth, and a story for how entity
knowledge stays current as facts change. The 2026-07-13 deep re-review flagged these as the
highest-leverage M3-contract decisions (cheap now, LLM re-walks later).

## Decisions

1. **Alias substrate.** Entity-like nodes (`person`, `place`, `topic`, `idea`, `event`,
   `project`) carry `aliases[]` + a one-line `disambig` in frontmatter. The DB alias index is
   a **GIN over `nodes.aliases`** (no separate table). The organizer maintains aliases as it
   meets new surface forms.
2. **Retrieval-bounded resolution.** At organize time, the capture's mentions are matched
   against the alias index; only the **matching candidates** are injected into the prompt, as
   structured fields (`{id, name, aliases, disambig, type}`) — never the whole registry, never
   node bodies. Prompt cost is O(mentions), not O(corpus).
3. **Ambiguity → review, never guess.** Below `ENTITY_MATCH_MIN_CONF` (config, default 0.8,
   live-tuned at Accept) the organizer does not link: the memory node writes immediately with
   that edge **pending**, and a **review item** is filed. The **`review_queue` is pulled
   forward from M6 into M3** and is **kind-generic** — one table, one lifecycle
   (`pending → resolved/discarded/maybe`) for every human-decision item: `entity-ambiguity`
   (M3), `vocab-proposal` (M3 — ADR-027 proposals are a queue kind, surfaced in Settings),
   `stance-candidate` (M6), `dedup-proposal` (M6+). Items must be **decidable in place**:
   mention shown in its capture excerpt, candidates with name/disambig/aliases + tap-through
   to the node preview; actions *pick candidate / new entity / maybe*. Resolution materializes
   the pending edge (file + DB).
4. **Thin hub + derived profile.** Entity files stay thin and stable (identity, aliases,
   disambig, edges, optional hand-written seed line). The readable "who/what is X *now*"
   **profile is derived**: regenerated nightly for entities whose neighborhood changed, from
   the 1-hop memory neighborhood, stored DB-side, served by `GET /nodes/{id}`/the map, and
   **embedded** (searching "Alex" hits one dense, current summary). Accepted trade-off,
   explicit: the canonical file alone is not self-describing — legibility lives in the derived
   layer, consistent with derived-content-stays-out-of-files (ADR-026). Currency comes from
   edge **`since`** dates (no `supersedes` edge type unless governance later demands one).
5. **Merge primitive.** `merge(loser → survivor)`, surfaced as **propose → apply** (ADR-024
   shape) in Admin: DB reverse index finds inbound edges → source files rewritten atomically
   under the store lock → `aliases` unioned (loser's name kept as alias) → loser replaced by a
   **permanent tombstone** node (`merged_into: <survivor-id>`; old ids/source_refs keep
   resolving; indexer treats it as a redirect, hidden from search/map) → touched nodes
   reindexed. **Applied immediately on confirm** after a forced commit+push (git history is
   the checkpoint — ADR-014); no agent-window wait.
6. **Backfill.** When an entity is minted or gains an alias, a nightly scan re-checks recent
   unlinked/`inbox/` nodes for alias matches: above threshold → edge auto-added (feed-flagged),
   below → review item.

## Consequences

- M3 ships: alias/disambig frontmatter, GIN index, candidate-bounded organizer prompt,
  review_queue table + minimal admin Review list (polished UX still M6), profile-refresh job,
  merge propose/apply endpoints + tombstones, backfill scan job.
- ADR-029's queue lands early with a different first tenant; its M6 scope becomes UI polish +
  the stance kinds. ADR-027's "proposal storage" question is closed: a queue kind.
- ❌ Rejected: guess-linking at low confidence (silent corruption); whole-registry prompt
  injection (O(corpus) cost + injection surface); LLM-accreted entity bodies (hot-path
  read-modify-write, churn, prose conflict resolution); rendering profiles back into files
  (the `sb:related` trap again); deferred/window-only merge application (no safety gain over
  git history, worse UX).
