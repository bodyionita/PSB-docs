# ADR-024: Tag vocabulary reuse (forward) + manual consolidation tool

**Status:** Accepted · 2026-07-13 (M2 planning)
**Terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md)/[027](027-typed-vocabulary-governance.md), 2026-07-13):**
survives the pivot — read "notes" as **nodes**; the "valid Obsidian tags" slug rules are kept **on
their own merits** (nothing Obsidian remains); the propose→apply consolidation shape here is
**generalized** by ADR-027 into node/edge-type vocabulary governance.
**Relates to:** [005 planes & atomic notes](005-planes-and-atomic-notes.md) · [019 conversational capture](019-conversational-capture-minimal-in-m1.md) · [02-data-model §2 (tags)](../02-data-model.md) · updates [03-api](../03-api.md), [04-pipelines](../04-pipelines.md), [08-implementation-plan §M2](../08-implementation-plan.md)

## Context

Live use surfaced **tag fragmentation**: the organizer coins tags free-form (then `_slugify_tag`
normalizes case/spacing — [02 §2](../02-data-model.md)), so semantically-identical concepts drift
into variants (`second-brain` / `secondbrain` / `second-brain-app`). Variants split what should be
one facet, so "everything tagged X" misses closely-related notes.

The fix has a natural home in M2: M2 builds the queryable index, so the **canonical tag vocabulary**
(distinct `notes.tags` across the vault, with frequency) exists for the first time and can be fed
back into the models that assign tags.

## Decision

**1. Forward-only reuse — inject the live vocabulary into the taggers.** The organizer (capture,
[ADR-019](019-conversational-capture-minimal-in-m1.md)) and the future distiller (M4) receive the
current tag vocabulary (distinct `notes.tags`, frequency-ordered, capped) in their prompt with the
rule: **"Prefer an existing tag when one fits; only coin a new tag if nothing here matches."** New
captures/distills converge onto the existing vocabulary instead of spawning variants.
- Vocabulary source = `notes.tags` aggregation (available from M2's index). Slugging
  (`_slugify_tag`) still runs on whatever the model returns, so the invariant in [02 §2](../02-data-model.md)
  (valid Obsidian tags) is unchanged.
- This is a **prompt/behavioral** change to the organizer; no schema change.

**2. Existing cruft is handled by a *manual* consolidation tool, not a nightly job.**
`POST /admin/tags/consolidate` is a two-step, human-in-the-loop maintenance action:
- **Propose:** compute merge candidates over the current tag set (embedding-similarity and/or an
  LLM pass) → return a merge plan (`{ canonical, variants[] }[]`). No writes.
- **Apply:** on the user's confirmation of a plan, rewrite affected notes' frontmatter tags
  (canonical replaces variants), then reindex the touched notes. Reuses the vault-write + reindex
  path of `POST /admin/captures/{id}/reorganize` (never-lose: a git commit, revertible).
- Surfaced as a **button in the web Admin tab** (Propose → review → Apply).

**3. Not wired into the nightly job.** Auto-consolidation would silently rewrite the user's tags as
the vocabulary shifts. Forward reuse is the high-value 80%; the manual tool cleans existing cruft
on demand. Ongoing/automatic consolidation is a noted future option if drift proves real.

## Consequences
- ✅ New notes stop fragmenting the vocabulary at the source (organizer sees what already exists).
- ✅ Existing variants are fixable when the user chooses, with review — no surprise vault rewrites.
- ✅ Reuses existing seams (index aggregation for the vocabulary; the reorganize vault-write/reindex
  path for apply); no new durability surface.
- ⚙️ The organizer prompt grows by an injected tag list (capped/frequency-ordered to bound size);
  a very large vocabulary is truncated to the top-N by frequency.
- ⚙️ `POST /admin/tags/consolidate` mutates human vault notes on Apply — same never-lose treatment
  as reorganize (git-tracked, revertible), gated behind explicit user confirmation of the plan.
- ↩️ Automatic/nightly consolidation deferred; merge-proposal quality (embeddings vs LLM) is an
  implementation choice, tunable, not fixed here.
