# ADR-034: External inspirations round 2 — evidence-tiered entity profiles; ecosystem references

**Status:** Accepted · 2026-07-13 · Companion to [ADR-033](033-external-inspirations-obsidian-second-brain.md)
(same review discipline: adopt only what fits, safeguard the skips) · refines the ADR-030/032/033
derived-profile design · updates [08 §M3](../08-implementation-plan.md) (profile job) + §M11 (reference).

## Context

Reviewed three further repos (user request, 2026-07-13):
[NicholasSpisak/second-brain](https://github.com/NicholasSpisak/second-brain) (612★ — a 4-command
Karpathy-LLM-Wiki starter; a thinner variant of ADR-033's subject, same vault-rewrites-itself
family), [huytieu/COG-second-brain](https://github.com/huytieu/COG-second-brain) (597★, v3.7.1 —
a work/PM-oriented multi-agent CLI: Opus lead + Sonnet workers, role packs, Jira/Linear/Confluence
sync, and a **knowledge-based People CRM**), and
[aristoapp/awesome-second-brain](https://github.com/aristoapp/awesome-second-brain) (465★ —
ecosystem index; its entries are already covered by the ADR-032 survey).

## Adopted

**Evidence-tiered derived profiles** (from COG's People CRM: Tier 3 @ 1 mention → Tier 2 @ 3+ →
Tier 1 @ 8+/direct interaction, observations cited + confidence-stamped). Mapped onto our
substrate — profile **depth scales with graph degree**, applied to all entity-like types by the
nightly profile-refresh job:

- **Tier 3 (1–2 connected memories):** mechanical stub — name, disambig, the observation lines
  derivable without any LLM call (edge rels + `(as of …)` stamps).
- **Tier 2 (3+):** LLM-synthesized snapshot — categorized observations (ADR-032 format) +
  a one-line "current state".
- **Tier 1 (high degree, threshold config e.g. 8+):** full profile — observations + themes +
  open threads, refreshed whenever the neighborhood changes.

Consequences: richer profiles are *earned* by connectivity (quality follows evidence), and the
nightly LLM spend is **structurally capped** — the long tail of once-mentioned entities never
costs a model call. Thresholds are config (`PROFILE_TIER_*`), tuned live. Every profile line
keeps source linkage (the supporting node ids), consistent with COG's citation discipline and
our rebuildability rule. Lands with the M3 profile-refresh job (task 6, not yet started).

## References saved (for future grillings)

- **awesome-second-brain** — ecosystem index; consult at future kickoff re-checks alongside
  ADR-032's source map.
- **COG two-way external sync** (`worker-executor`: pre-approved mutations to Linear/GitHub) —
  reference for the **M11 life-manager** grilling's "advisor vs state-manager" question.
- COG's verification-first freshness (7-day windows, confidence stamps) — validates ADR-033 #3;
  no further action.

## Skipped (nothing saved — per the "if not, don't" rule)

- ❌ **NicholasSpisak/second-brain in its entirety** — no idea not already present in richer
  form via ADR-033 (its sources/entities/concepts/synthesis taxonomy ⊂ our 9 types).
- ❌ COG's lead/worker CLI orchestration + `/tmp` file-passing — CLI-session mechanics; our
  server pipeline + ADR-025 routing + ADR-032 day/night split already cover the economics.
- ❌ COG's role packs, PM skill set (PRDs, release notes, Confluence), content-factory —
  work-tool scope, not a life-brain need; revisit only if M11 grilling pulls them in.
- ❌ Monthly "knowledge-consolidation" summarize pass — ADR-032's warning stands:
  consolidation deduplicates, never summarizes away atomic memories.
