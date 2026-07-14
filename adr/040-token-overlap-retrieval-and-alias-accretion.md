# ADR-040: token-overlap candidate retrieval + alias accretion (M3)

**Status:** Accepted · 2026-07-14 · Pulls forward the fuzzy-matching + alias-accretion follow-ups
deferred at [ADR-030](030-entity-substrate-and-lifecycle.md) / [ADR-032](032-prior-art-adoptions.md)
(and noted in 08-logs/m3.md tasks 2/3/6), after the M3 task-10 Accept showed a real resolution miss.
Decided in the M3 organizer-quality replanning session; recorded before coding.

## Context

The Accept split one person into two hubs: "Horia" (from the relationship capture) and "Horia
Stanciu" (from the standup). Cause: candidate retrieval for a mention is **exact/normalized-alias
only** (`PgAliasStore` — `normalize_alias` = lower + whitespace-collapse). "Horia Fenwick" never
retrieves the "Horia" hub as a candidate, so the resolver LLM never gets to compare them and just
mints a second person. Two capabilities, both previously deferred, are missing:

- **Broader candidate retrieval** so variant surface forms surface the existing hub.
- **Alias accretion** so linking a variant records it as a new alias — closing the loop for next time.

The exact-alias short-circuit (ADR-032) already serves "same surface form mentioned twice → one
node"; this ADR handles **different surface forms of the same entity**.

## Decision

**Add conservative token-overlap candidate retrieval and alias accretion; the confidence floor +
LLM disambiguation + review queue still decide — nothing auto-guesses.**

1. **Token-overlap retrieval leg.** Candidate generation gains a second leg alongside exact/
   normalized-alias: same-type hubs sharing a **significant token** (or a token prefix) with the
   mention (e.g. "Horia Fenwick" → hubs whose aliases contain "horia"). Bounded by
   `entity_candidate_max` and ordered by overlap strength. Retrieval only — it merely *surfaces*
   candidates for the resolver.
2. **Low-entropy guard (ADR-032 carried).** Short or low-entropy tokens never drive fuzzy retrieval
   (a config `entity_min_token_len` / stop-token list), so "Ana"/"the"/initials don't fan out to
   everything. Single-token, low-entropy mentions fall back to exact-only.
3. **Decision unchanged — never guess.** Retrieved candidates go through the existing gate
   ([resolver](../04-pipelines.md)): `conf ≥ ENTITY_MATCH_MIN_CONF` + LLM picks a candidate → link;
   `conf ≥ floor` + "new" → mint; otherwise → `entity-ambiguity` review item with the edge left
   **pending** (ADR-030 §3). Broadening candidates only changes *what the LLM sees*, not the
   safety semantics — ambiguity still routes to review.
4. **Alias accretion on link.** When a mention **links** to a hub under a surface form not already in
   its `aliases`, that surface form is **accreted** as a new alias — frontmatter rewrite (store is
   truth) + alias-index upsert, idempotent (rule 6), feed-visible in the run. Applies on the capture
   link path and the review-resolution link path. Short/low-entropy surface forms are **not**
   accreted (same guard), to avoid polluting a hub with generic tokens.
5. **Diacritic-insensitive.** Retrieval + accretion operate on folded text
   ([ADR-041](041-diacritic-folding-derived-content.md)), so ASCII and (raw-only) diacritic forms of
   a name compare equal.

## Rationale

- The miss is a *retrieval* gap, not a decision gap — the resolver already disambiguates well when it
  sees the right candidates. Widening retrieval + keeping the conf floor is the smallest change that
  fixes Horia/Horia Fenwick without loosening safety.
- Accretion makes resolution improve monotonically: every confirmed link teaches the hub a surface
  form, so the exact short-circuit covers more next time and fuzzy load drops.
- The low-entropy guard is the known precision lever; ambiguous cases still de-risk to review, never a
  silent wrong merge.

## Consequences

- `PgAliasStore` gains a token-overlap query; the resolver's candidate assembly unions it with the
  exact leg. `NodeWriter` gains alias accretion (reuses the frontmatter-rewrite machinery from
  merge/retype). New config: `entity_min_token_len` (+ optional stop-token list); `entity_candidate
  _max` reused.
- Precision risk is bounded but real; the M3 Accept's entity-resolution criterion is the check, and
  `ENTITY_MATCH_MIN_CONF` stays live-tunable. Fuzzy **auto-link without the LLM** is still forbidden
  (only the exact short-circuit auto-links).
- Existing prod split (Horia) is healed by the reprocess op replaying captures in order through this
  resolver; or, pre-reprocess, by a manual merge (endpoint exists).
- `02-data-model` (alias accretion) + `04-pipelines` (resolver candidate retrieval) + `08-logs/m3.md`
  updated in the same change set.
- ❌ Rejected: *accretion-only* (doesn't fix the miss — the hub is never retrieved to link to);
  *unguarded fuzzy auto-link* (precision blowup on short names); *embedding-similarity candidate
  retrieval now* (heavier; token-overlap suffices at M3 scale, revisit if recall lags).
