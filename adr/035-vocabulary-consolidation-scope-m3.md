# ADR-035: Vocabulary retro-consolidation scope for M3 — edges apply, nodes propose-only

**Status:** Accepted · 2026-07-14 · Refines [ADR-027](027-typed-vocabulary-governance.md) §3
(the propose→approve→consolidate governance) for the M3 build · scopes
[08 §M3](../08-implementation-plan.md) task 7 (vocabulary surface). Decided in the M3 task-7
implementation session when the retro-consolidation mechanics turned out to be under-specified
(a decision the docs didn't answer — recorded before coding, per
[09](../09-session-protocol.md)).

## Context

[ADR-027](027-typed-vocabulary-governance.md) §3 says approving a new node/edge type "queues a
**consolidation job** (ADR-024's propose→apply shape): it re-walks the existing graph, proposes
re-typings/new edges that should use the new type, and applies them on confirm." That is the
*intent*; it does not pin the **mechanics** of re-typing, and the two vocabulary axes are not
mechanically symmetric:

- **Edge re-typing is cheap and safe.** An edge is a frontmatter line on its source node
  (`- rel: … / to: …`). Re-typing / adding edges is the same class of operation as ADR-024 tag
  consolidation: rewrite a frontmatter line, reindex, commit — atomic, git-revertible, no identity
  change. The existing `NodeWriter` (`append_edges`/`retarget_edges`) already covers the write half.
- **Node re-typing is heavy and identity-adjacent.** **Folder = node type**
  ([ADR-026](026-graph-native-storage-obsidian-removed.md), [02 §1](../02-data-model.md)), so
  re-typing a node is a **cross-folder file move + filename re-slug + `captures.node_paths` update +
  frontmatter `type:` change**, not a line rewrite. No such machinery exists (`NodeWriter` has
  append/retarget/tombstone/remove and move-by-path in the indexer, but no *retype*). The node's
  uuid is stable across a move, so `edges`/`nodes` heal by id on reindex — but the operation still
  touches the "folder=type" invariant, filename identity, and the capture→node_paths back-reference,
  and deserves its own design pass rather than being improvised inside task 7.

The M3 Accept requires only that "a vocabulary proposal appears in Settings, approval runs
consolidation" — it does not require whole-graph node re-typing to ship in M3.

## Decision

**M3 vocabulary retro-consolidation ships the safe half and proposes (not applies) the heavy half.**

1. **Approval makes the type live immediately.** Approving a `node_type` / `entity_type` / `edge_rel`
   proposal writes the addition to `app_settings`; the **effective vocabulary** = config seeds ∪
   approved additions is what the organizer, `validate_organizer_output`, `GET /types`, and the
   entity-substrate readers consume from that point on. Growth is forward-live at once; the graph
   never bifurcates into pre-/post-approval eras for *new* captures.
2. **Edges: propose → apply (confirm-gated).** Approving an `edge_rel` queues the consolidation job,
   which re-walks the graph and **proposes** new/re-typed edges that should use the new rel (LLM,
   ADR-024 propose shape); on the user's confirm it **applies** them — frontmatter-line rewrites +
   reindex + commit, exactly the tag-consolidation safety envelope (git-tracked, revertible,
   feed-visible).
3. **Nodes: propose-only in M3.** Approving a `node_type` / `entity_type` queues the job, which
   re-walks and **surfaces candidate existing nodes to re-type** (feed / Review-visible) — but the
   **apply** half (the folder-move / re-slug / node_paths machinery) is **deferred to its own ADR +
   task**. Node re-typing is not performed automatically in M3.
4. **One choke point.** Both `PUT /settings/vocabulary` (Settings panel) and `POST /review/{id}`
   (the kind-generic Review queue, `vocab-proposal` kind) route through a single `VocabularyService`
   (rule 5; ADR-027 §4 "governance enforceable at one choke point"). The task-4 behaviour — approve
   only *queued a `vocab-consolidation` marker run*, mutation deferred — is replaced here by: mutate
   effective vocab → open the consolidation job.

## Rationale

- Ships **all of ADR-027's governance value** (propose → approve → forward-live → retro-consolidate)
  and the entire edge half of retrofitting, while not improvising an identity-affecting node-move
  subsystem days before the M3 Accept.
- "Propose-only for nodes" is still faithful to ADR-027's *organic* framing: the system tells the
  user which existing nodes it would re-type; only the automated **write** is deferred, and the user
  can re-type by hand in the meantime (or wait for the follow-up task).
- Keeps the "folder=type" invariant and node filename/`node_paths` identity untouched until they get
  a proper design pass.

## Consequences

- **Follow-up (recorded, deferred):** *node re-typing consolidation* — the folder-move + filename
  re-slug + `node_paths` update + reindex machinery, plus the apply half of the node-type
  consolidation walk. Its own ADR + task (post-M3 / M6-era graph-hygiene work, alongside the
  dedup-proposal drain). Until it lands, an approved node type is forward-live and its retro-walk is
  propose-only.
- `app_settings` gains vocabulary keys (approved additions per axis). Effective vocabulary is read
  through a provider, not `settings.node_types` directly, at the organizer / `GET /types` /
  entity-substrate call sites.
- The consolidation job (`agent="vocab-consolidation"`) joins the agent-jobs roster — manually
  triggerable, live-observable (08 M8), replacing task 4's SKIPPED marker with a real run.
- ❌ Rejected: **full node re-typing now** (unbudgeted identity-machinery risk before Accept);
  **forward-only, no retro-walk** (drops the edge half that ADR-027 §3 + the Accept call for).
