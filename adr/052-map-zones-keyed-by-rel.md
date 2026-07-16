# ADR-052: M7 map zones are keyed by `rel` (dual-origin `similar` collapses to one zone)

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision, mid-implementation replan) ·
**Refines / supersedes** parts of [ADR-051](051-m7-map-build-decisions.md) §2 (zone key) and §3
(derived `similar` as its own zone) · **Builds on** [ADR-049](049-dedup-sweep-merge-core-build-decisions.md)
(the human `link` → canonical `similar` edge) and the M5 `GraphService.neighbors` primitive
([ADR-046](046-m5-mcp-server-oauth-connectors.md)).

## Context

M7 task 1 (the grouped `GET /nodes/{id}/neighbors` endpoint) surfaced a contract defect during
implementation. ADR-051 §2 keyed a map zone by **`(origin, rel)`** yet specified that the
single-zone **"show more"** mode "reuses the M5 rel-filtered cursor **untouched**" — i.e. it pages
by **`rel` only**. Those two statements are consistent only if `rel` uniquely identifies a zone.

It does — for every rel **except `similar`**. Confirmed from the code:

- **Derived edges are only ever `rel='similar'`** — the nightly recompute hardcodes
  `edges(origin='derived', rel='similar')` (`graph/store.py` `compute_similar` /
  `replace_derived_edges`). No other rel is ever derived.
- **Canonical `similar` also exists** — the ADR-049 `link` action writes a *canonical* `similar`
  edge (`review_service._link_similar`, `DEDUP_LINK_REL = "similar"`), which **deliberately
  survives** the nightly derived recompute (the recompute only touches `origin='derived'` rows).

So `similar` is the **only** dual-origin rel, and canonical + derived `similar` coexist on a node
by design. Under `(origin, rel)` zones they form two zones sharing one rel; "show more" (rel-only
keyset, ordered `(origin, rel, dir, node_id)`) on the *canonical*-`similar` zone would therefore
page past its cursor and **bleed derived-`similar` rows into the page** — rendered as solid
canonical edges and duplicating the derived zone. Rare in practice (needs > `map_zone_fanout`
canonical `similar` edges on one node, i.e. many human `link`s, plus coexisting derived similars),
but wrong.

## Decision

**A map zone is keyed by `rel` alone.** Canonical and derived `similar` collapse into a **single
`similar` zone**; every other rel is single-origin, so `(origin, rel)` ≡ `rel` and nothing changes
for them. This eliminates the bug class by construction rather than patching the paging:

1. **Zone identity = `rel`.** The grouped first page emits one zone per distinct `rel`
   (`ROW_NUMBER`/`COUNT` partitioned by `rel`; the zone's neighbors ordered `(origin, dir,
   node_id)` — the `(origin, rel, dir, node_id)` keyset with `rel` fixed, so canonical `similar`
   surfaces before derived on the first page and derived pages in after).
2. **"Show more" reuses the M5 rel-filtered cursor, now correct.** Because a zone *is* a rel, the
   rel-only keyset resumes exactly one zone — no origin param, and the shared M5
   `GraphService.neighbors` primitive stays **untouched** (still used verbatim by MCP `traverse`).
   ADR-051 §2's "reuses the M5 rel-filtered cursor untouched" is preserved and made true.
3. **Origin is per-edge, not per-zone.** The zone-level `origin` field is **dropped** from the
   response — a mixed `similar` zone has no single origin. Each neighbor still carries its own
   `origin`, which is the **sole** signal the canvas needs for ADR-051 §6's rendering (canonical
   solid + arrowhead, derived faint, superseded `until` dashed+dimmed). §6 is unchanged.
4. **Force layout.** Derived `similar` no longer gets its own force-sector separate from canonical
   `similar` (superseding ADR-051 §3's "derived `similar` is always its own zone"); they share the
   one `similar` sector. They remain visually distinct via the per-edge faint styling (§6). This is
   a client-side (task 2) consequence with no server effect.

## Consequences

- **Response shape (03-api):** a zone is `{ rel, neighbors[…≤map_zone_fanout], total, next_cursor }`
  — `origin` removed. `total` counts all neighbors of that rel (both origins for `similar`), so
  "show N of M" spans the merged zone.
- **Server diff is small:** `PARTITION BY rel` (was `origin, rel`) in
  `PgNeighborStore.neighbor_zones`; the service groups by `rel`; `MapZone`/`NeighborZone` drop
  `origin`. The primitive, the "show more" mode, the cursor codec, and every non-`similar` zone are
  unchanged.
- **Regression test:** the fix ships with the dual-origin `similar` case (a node carrying both a
  canonical and a derived `similar` edge) asserted at the unit + real-PG smoke level — the exact
  path that was untested and let the original defect through.
- **No migration**, no organizer touch, no new config.

## Alternatives considered

- **Keep `(origin, rel)` zones; add an `origin` param to "show more"** (and an origin filter on the
  M5 primitive) — rejected: grows the shared `traverse` primitive, adds a per-zone `origin` echo and
  a sometimes-meaningful field, and contradicts ADR-051 §2's "untouched" for a rel that is usually
  all-derived anyway. More moving parts to fix a one-rel edge case.
- **Keep `zone.origin`, null it for mixed zones** — rejected: a field meaningful for some zones and
  not others is a trap; per-edge origin already suffices for rendering.
