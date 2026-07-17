# ADR-055: Interiority — the user's inner voice as a first-class data dimension

**Status:** Accepted · 2026-07-17 (grilled decision-by-decision) · **Defines** half of the M8.2
milestone in [08-implementation-plan.md](../08-implementation-plan.md) · **Builds on**
[ADR-027](027-typed-vocabulary-governance.md) (governed vocabulary), [ADR-031](031-m3-organizer-and-contract-extensions.md)
(organizer contract), [ADR-037](037-profile-embedding-in-search-m4.md)/[ADR-032](032-prior-art-adoptions.md) §5/§7
(hybrid retrieval + priors), [ADR-046](046-m5-mcp-server-oauth-connectors.md) §4 (identity capsule),
[ADR-042](042-reprocess-all-from-raw-and-data-survival.md) (reprocess backfill).

## Context

The user wants their **feelings and internal thoughts** to hold a more pivotal place than records of
events/conversations with people — "pivotal **in the data**", not a display trick. Today a capture
like "walked with D., talked about work… it felt easy, I've missed this" organizes into event-record
prose; the inner voice is buried as adjectives. Grilled 2026-07-17: the foundation is an
ingestion-time marker (c), and chat retrieval (a) + the identity capsule (b) become cheap consumers.

## Decisions

**1. `interiority` marker on every content node.** The organizer stamps every content node
`interiority: internal | external | mixed` (frontmatter + a DB column on `nodes` for cheap
querying). *internal* = the user's inner voice (feelings, reflections, self-talk); *external* = a
record of the world (events, facts, what others said/did); *mixed* = genuinely both after
extraction (below). Orthogonal to `type` — a dimension of content, not a kind of content.

**2. Extraction, not just marking.** The organizer is instructed to **pull introspective content
out into its own node(s)** — existing types (`memory`/`insight`), stamped `internal`, edge-linked
to the event node they arose from. The feeling becomes a node: visible on the map orbiting its
event, independently retrievable, profile-buildable. The event node keeps its record character.
A **new governed node type was rejected for now** (type is exclusive where this dimension isn't;
feeling-vs-`insight` is fuzzy; proliferation costs) — the marker makes a later type migration
trivial since internal nodes are already identifiable.

**3. Consumers.**
- **(a) Chat retrieval:** a config-knobbed **multiplicative boost on the fused RRF×recency score**
  for `internal` nodes (same bounded pattern as the recency prior; default modest ~1.2×, tunable).
  `/search` stays neutral — only chat grounding gets the inner-voice thumb on the scale.
- **(b) Identity capsule:** the capsule source gains a **dedicated internal-nodes slice** (top-K
  recent `internal` alongside hubs/memories/insights) so the distiller sees the inner voice as its
  own labeled section, not diluted among event records.
- **(c) Visibility:** internal nodes get a subtle visual marker in Map/`NodePreview` (distinct
  halo/glyph) — the dimension is seen, not just stored.

**4. Backfill by reprocess (P10).** Applies forward from deploy; existing nodes gain the marker +
extraction via the standard `reprocess-all-from-raw` op — no hand-tagging, raw is truth. Known,
loudly-reported caveat carried from ADR-042: standing manual merges are not re-applied by id.

## Consequences

- Organizer prompt version bumps; one `nodes` column migration; retrieval/capsule/map touches are
  each a few lines against existing seams.
- Answers about the user's state ground in what they *felt/thought*, and the capsule — which fronts
  chat **and** MCP (`build_context` L0) — is shaped primarily by the inner life.
- Rejected: a new `feeling`/`reflection` node type (for now), a plane (wrong axis — planes are life
  areas), query-time introspectiveness guessing (flaky; the marker is stamped once at ingestion).
