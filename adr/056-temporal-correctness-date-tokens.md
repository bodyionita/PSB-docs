# ADR-056: Temporal correctness — anchored resolution, inline date tokens, and "LLMs classify, code computes"

**Status:** Accepted · 2026-07-17 (grilled decision-by-decision) · **Defines** half of the M8.2
milestone in [08-implementation-plan.md](../08-implementation-plan.md) · **Absorbs** the backlog's
`occurred-enrichment` item (its own planning session = this one) · **Builds on**
[ADR-031](031-m3-organizer-and-contract-extensions.md) (`occurred` partial-ISO → range),
[ADR-042](042-reprocess-all-from-raw-and-data-survival.md) (P10 / reprocess determinism),
[ADR-048](048-m6-chat-distiller-build-decisions.md) §1 (conversation-time anchoring),
[ADR-049](049-dedup-sweep-merge-core-build-decisions.md) §3 (the occurred-overlap gate this
strengthens) · **Adds a new hard rule** to the code repo's CLAUDE.md.

## Context

A memory saying "10 days ago" stays literally "10 days ago" forever — and the organizer prompt
carries **no date context at all**, so relative references are today unresolvable (the model can
only emit `occurred: null` or fabricate). Beyond ingestion, dates *inside* memory prose go stale,
LLM consumers (chat, MCP) receive temporally uninterpretable text, and the user has no way to
correct a wrong date where they see it. Grilled 2026-07-17.

## Decisions

**1. Anchor injection, never wall-clock.** The organizer prompt carries the capture's **stored**
timestamp ("This capture was recorded on Thursday, 2026-07-17 08:40 Europe/Bucharest") —
`captures.created_at`, or `anchor_at` for chat-distilled captures (ADR-048 §1). A live "now"
tool was **rejected**: under reprocess it would resolve "10 days ago" against the replay date,
silently shifting memories — the fix itself violating P10. The anchor is data, so replay is
deterministic forever.

**2. "LLMs classify, code computes" (new hard rule, product-wide).** The LLM never emits a
computed date or arithmetic result. For every time-reference it emits a **symbolic
classification** — `{"phrase": "10 days ago", "kind": "relative_days", "offset": -10}`,
`{"kind": "season", "season": "summer", "year_offset": -1}`, `{"kind": "weekday_last",
"weekday": "tue"}`, explicit dates as `{"kind": "explicit", ...}` with year-snapping left to code —
and **deterministic Python** (`datetime`/`dateutil`) resolves it against the anchor: offset
arithmetic, weekday walks, season expansion (northern-hemisphere default), month lengths, ranges,
granularity. **Fail semantics:** a symbolic form that doesn't validate produces **no token** (the
phrase stays prose) and may file a review item — never a guessed date. An agentic tool loop was
rejected (latency, fallback-provider parity, failure surface) — symbolic emission gives the same
guarantee one-shot. The rule is general and binding on all future work (connectors, reflection):
no LLM-emitted numerical/temporal derivation is ever stored.

**3. Inline date tokens in derived bodies.** The resolver's output lands in the derived body as a
machine-readable token: **`[[t:START[/END][|label]]]`** — partial-ISO with honest granularity
(`[[t:2025]]`, `[[t:2026-07]]`), optional time-of-day (`[[t:2026-07-07T22:00]]`), **ranges** in
ISO-interval form (`[[t:2025-06/2025-08|summer 2025]]`), and an optional **absolute** display label
(never "last summer" — the relative form is computed at render). Raw captures are untouched (rule
2); tokens live only in organizer-written prose. **Recurrence is fenced out**: habitual time ("on
weekends", "every Tuesday") is a pattern, not a point — no token, stays prose (M11 territory).

**4. Rendering contracts (tokens are never shown raw).**
- **Web:** tokens render as a live phrase (relative where natural, absolute where natural), each
  carrying the M8.1 `<TimeAgo>`-style tap/hover exact-time tooltip — always current because it is
  computed at render.
- **Indexing:** the indexer expands tokens to natural absolute text **before chunking/embedding** —
  vectors see language, not token noise.
- **LLM-bound rendering contract (binding, all paths):** every render that puts node/capture
  content in front of *any* LLM — MCP tools, the chat grounded prompt, the capsule distiller's
  source, profile generation, consolidation — MUST ship (i) **token expansion** to absolute (+ a
  freshly computed relative where it aids the model: "on 7 Jul 2026, 10 days before this
  conversation") and (ii) a **per-item temporal metadata header** (recorded-at · occurred), so even
  *unmarked* prose ("last Tuesday" in a pre-reprocess node) is interpretable against stated
  context. Build-time checklist over every existing render path.

**5. Two-tier edit semantics.** Every rendered date is a tap-to-edit target (the token is the edit
anchor — no text-span bookkeeping):
- **Token edit** (a date mentioned in the body): mechanical — rewrite the token, update `occurred`
  if that token is the node's event date, re-embed the node's chunks. **No LLM, instant.**
- **Anchor edit** (the capture's recorded-at was wrong): invalidates every relative resolution →
  save triggers a **background one-capture `reorganize_capture_now`** (the inbox-drainer's
  machinery) re-resolving against the corrected anchor.
- Graph-wide ripple (dedup occurred-gate, profiles) rides the **nightly pipeline**; a full
  reprocess is never auto-triggered by a date edit.

**6. Schema: `occurred_*` stays `date`.** Every DB consumer (dedup overlap gate, recency prior,
`since`/`until`/`as_of`) is day-granular by nature; **tokens own sub-day precision** (display /
narrative / editing). If a sub-day *query* need ever emerges, tokens hold the data — a later
migration is a backfill, not a re-derivation.

**7. `occurred-enrichment` review kind (absorbed from the backlog).** A nightly step flags
undated / coarsely-dated content nodes (graph-health already counts them); the review card asks
the user to tag the event time in natural language ("summer 2019", "last Tuesday ~6pm"); the
answer goes through the **same symbolic-classification + deterministic resolver** (anchored to
the answer's own date) and applies via the **mechanical tier** of §5. Directly strengthens the
ADR-049 dedup occurred-signal.

**8. Backfill by reprocess (P10).** One prod `reprocess-all-from-raw` at milestone end backfills
tokens (+ ADR-055's interiority) onto existing data — anchors are stored per capture, so
resolution is exact regardless of when the replay runs. Known ADR-042 caveat (standing manual
merges re-applied manually) accepted and reported loudly by the op.

## Consequences

- New pure-logic temporal engine (symbolic schema, resolver, token parse/render) — heavily
  unit-testable with zero mocks; organizer prompt version bumps; render-path sweep across MCP /
  chat / capsule / profiles / indexer.
- The dedup gate, recency prior, and `as_of` all get materially better as `occurred` densifies.
- Rejected: a live now-tool (P10 trap), an agentic date-tool loop (complexity for no added
  guarantee), LLM-computed dates (calculus in a language model), recurrence tokens (different
  machine), a `timestamptz` migration (no day-granular consumer needs it).
