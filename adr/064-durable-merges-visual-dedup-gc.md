# ADR-064 — Graph hygiene: durable entity merges + visual/actionable dedup & GC

**Status:** Accepted 2026-07-19 (grilled) · **Extends** [ADR-030](030-entity-resolution-and-merge.md) §5
(entity merge) · **Supersedes** [ADR-042](042-reprocess-all-from-raw-and-data-survival.md) §4's
"standing merges reported-but-dropped" · **Relates to** [ADR-049](049-dedup-sweep-merge-core-build-decisions.md)
(content dedup sweep), [ADR-053](053-m8-ops-console-observability-build-decisions.md) §9 (graph-health) ·
**Feeds** 08 §M9.8 (build).

## Context

Investigating duplicate "Diana" / "Diana Vance" person hubs surfaced a cluster of related gaps:

* **Merges don't survive `reprocess-all`.** A manual entity merge (ADR-030 §5) writes a tombstone
  (`merged_into`) keyed on the loser's **node id**. `reprocess-all` rebuilds entities from raw
  captures and mints **fresh ids**, so the merge can't be re-applied *by id* — the code already
  admits this (`reprocess.py`: "standing merges the rebuild can't re-apply by id", ADR-042 §4) and
  merely **warns** ("re-merge manually"). The 2026-07-17 reprocess dropped 2 standing merges; the
  Diana merge was one, so the duplicate reappeared silently.
* **The merge UX is unusable.** `EntityMergeCard` (AdminOps) takes two **raw node UUIDs** pasted by
  hand, and nothing in the app surfaces a node's id from its name — so a merge dead-ends at
  "preview → nothing." The backend propose/apply is healthy; the entry is the problem.
* **No entity-hub dedup.** The dedup sweep (ADR-049) is **content-only** (entity hubs are used as a
  shared-entity *signal*, never proposed as merge candidates), so duplicate hubs are never surfaced.
* **Entity resolution missed it at ingest.** "Diana Vance" was created new instead of aliasing onto
  "Diana" — a first-name↔full-name miss. (Deferred — see §Deferred.)
* **Graph-health is read-only.** It *detects* orphan-nodes (12), alias-less entities, etc., but only
  links into Review; there are no per-section actions, and there is **no path to delete an orphan
  hub** (capture-remove is keyed on captures; merge needs a survivor).

## Decision

**§1 — Durable, replayable merges (the core fix).** A merge is recorded as a durable decision keyed
on **stable identity — the loser's surface forms (name + aliases) + node type — not its id.**
`reprocess-all` re-applies these after a rebuild: any re-created entity of the recorded type whose
surface form matches the recorded loser is **re-folded** into the survivor (matched by *its* surface
forms). This turns ADR-042 §4's "reported but dropped" into "reported **and** re-applied," using the
same durability posture as removed-capture tombstones (`removed_at`). Merges become idempotent across
any number of rebuilds.

**§2 — One shared visual entity picker (no ids).** A single reusable **name-typeahead** entity
picker (search by name/alias → resolves to id internally) is the only merge entry the user ever
touches. It is exposed on three surfaces: (a) a **"Merge into…"** affordance on each entity/profile
view; (b) **one-click Merge** on graph-health duplicate candidates (§4); (c) the AdminOps card,
upgraded from id-boxes to the same typeahead as the power-user path. The two-step propose→apply
(inbound-edge preview → confirm, ADR-030 §5) is unchanged underneath.

**§3 — Inline-hybrid actionable graph-health.** Graph-health gains **inline actions on each section**
for decisive single-step operations (delete an orphan, merge an obvious dupe via the §2 picker),
calling the **same** underlying services (merge engine, capture-remove, the new node-delete). Items
needing judgment (low-confidence pairs) are **filed to Review** (which already owns the
accept/skip/merge machinery) rather than acted inline — so action logic is never forked.

**§4 — Conservative entity-hub dedup detector (shared).** One detector proposes likely-duplicate
hubs, gated by **name/alias containment or high fuzzy match AND shared-neighborhood overlap**
(Diana & Diana Vance both wire into the same content; **"Diana Wren" — a genuinely different person —
does not**, and must be suppressed). It powers **both** surfaces: high-confidence pairs appear
**inline** on graph-health with a one-click Merge (via §2's picker, pre-filled); lower-confidence
pairs are **filed to Review**. **Never auto-merge** — always human-approved.

**§5 — Orphan GC, manual + kind-aware.** Each graph-health orphan (zero-degree node) offers three
actions — **Delete**, **Keep** (dismiss/whitelist so it stops nagging — e.g. intentionally-kept
"Father"/"Mother" hubs), **Merge** (an orphan hub that's actually a dupe → the §2 picker). **Never
auto-delete** (rule 2 / data-survival). Delete is **routed by node kind** so nothing resurrects:
* an orphan **content** node (from a capture) → **tombstone its capture** (reuse capture-remove), or
  `reprocess-all` replays the raw and recreates it;
* an orphan **hub** (not a capture) → **node-level git-rm + index prune** (the net-new "delete node"
  path); reprocess won't recreate an unreferenced hub. Offered **only** for genuinely zero-degree
  hubs — a still-referenced hub isn't an orphan and is routed to Merge instead.

**§6 — Deferred.** Improving **entity resolution at ingest** so first-name↔full-name duplicates never
form (Diana Vance == Diana) is a separate, later roadmap item — resolution is never perfect and is a
deep rabbit hole; §1's durable merges + §2–4's easy detection/merge cover the present pain. Legacy
dupes are cleaned via §2–5 regardless.

## Consequences

* A merge — done from anywhere via the visual picker — **sticks across `reprocess-all`**; the Diana
  case (and future dupes) stays merged.
* Graph-health becomes a working console: detect → act (merge/delete/keep) inline for the clear
  cases, Review for the ambiguous, one shared merge engine + picker underneath.
* New surface area: a durable merge-decision store + reprocess replay step (§1), a node-delete path
  for orphan hubs (§5), an entity-hub dedup detector (§4), and the shared picker + inline actions
  (§2/§3). Task breakdown + parallel-batch annotations live in **08 §M9.8**.
* Byte-parity (ADR-042) is preserved: merges/deletes remain file-first + git-revertible; the durable
  merge record is replayed *after* the raw rebuild, not baked into raw.
* Detector false-positives (Diana Wren) are contained by the human-in-the-loop gate — §4 never
  auto-merges, and low-confidence goes to Review.
