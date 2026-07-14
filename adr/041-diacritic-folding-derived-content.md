# ADR-041: diacritic folding on all derived node content (M3)

**Status:** Accepted · 2026-07-14 · Fixes the diacritic-mangling bug found in the M3 task-10 live
Accept · scopes against the never-lose invariant ([CLAUDE.md rule 2](../templates/CLAUDE.md)) and the
slug/alias contract of [02-data-model](../02-data-model.md). Decided in the M3 organizer-quality
replanning session; recorded before coding.

## Context

Romanian diacritics are mangled in derived nodes: `_SLUG_INVALID = [^a-z0-9]+` (filenames) and
`_TAG_INVALID = [^a-z0-9_/-]+` (tags) replace non-ASCII with `-` instead of transliterating —
"Mădălina" → filename `m-d-lina`, "Ștefan" → `tefan` (**letters dropped**). Meanwhile
`normalize_alias` *keeps* diacritics, so slugs and alias-matching disagree, and an ASCII "Madalina"
won't match a diacritic "Mădălina" hub. Observed live during the Accept.

User decision (grilled): **there should be no diacritics anywhere in the graph store — stored or
otherwise.** The raw capture is the one exception: it is the never-lose source of truth and what
`reprocess-all-from-raw` replays, so it keeps its original bytes.

## Decision

**All derived node content is diacritic-folded to ASCII at the single write chokepoint; raw inputs
are never folded.**

1. **One fold utility.** A `fold_diacritics(text)` — Unicode NFKD + strip combining marks, plus the
   handful of Romanian letters NFKD doesn't decompose if any (`ș/ț` decompose; map defensively) —
   maps `ă→a â→a î→i ș→s ț→t` (and uppercase). Pure, unit-tested, the sole authority.
2. **Applied to every derived field.** Folding covers the **filename slug, title, aliases, disambig,
   tags, and the body prose** of every `NodeDocument`. Enforced centrally in `NodeWriter` (the single
   file-writer, ADR-028) so nothing written to the store can carry a diacritic — the organizer output
   is folded on the way to disk, not trusted to be clean.
3. **Matching folds too.** `normalize_alias` and the alias index fold as well, so retrieval +
   accretion ([ADR-040](040-token-overlap-retrieval-and-alias-accretion.md)) are diacritic-insensitive
   and consistent with the stored (already-folded) aliases.
4. **Raw is preserved.** The `captures` raw text/audio and the `source_ref` chain keep original
   diacritics (rule 2). Reprocessing re-derives folded nodes from the un-folded raw. The nudge
   (sourced from raw) is unaffected.

## Rationale

- Folding at the writer is the single chokepoint — one place guarantees the "zero diacritics in the
  store" invariant, and it can't be bypassed by a prompt/model change or a new call site.
- Full-field folding (incl. body) is what the user asked for and is the simplest contract to reason
  about — there is no "some fields have diacritics" ambiguity, and no field can ever feed a mangled
  slug/tag/alias.
- Keeping raw un-folded preserves the source of truth and the ability to re-derive, honoring
  never-lose and the data-survival principle ([ADR-042](042-reprocess-all-from-raw-and-data-survival.md)).

## Consequences

- `slugify`/`_slugify_tag` transliterate (fold) **before** the invalid-char regex, so a folded base
  letter survives instead of becoming `-`. `NodeWriter.write_nodes` folds all text fields;
  `normalize_alias` + alias index fold.
- Existing prod nodes carry diacritic-mangled slugs/tags; healed by `reprocess-all-from-raw` re-deriving
  every node through the folding writer (the NFKD fix must re-slug existing nodes, not only new ones).
- `02-data-model` (slug/tag/alias normalization = folded) + `08-logs/m3.md` updated in the same change
  set.
- ❌ Rejected: *fold identifiers only, keep body prose* (user wants zero diacritics anywhere);
  *fold-for-matching but keep original for display* (leaves diacritics stored — same rejection).
