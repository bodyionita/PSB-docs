# ADR-051: M7 map (neighborhood explorer) build decisions

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision) · **Implements** the M7 scope in
[08-implementation-plan.md](../08-implementation-plan.md) · **Builds on**
[ADR-032](032-prior-art-adoptions.md) #12 (react-force-graph 2D, plex re-center, rel-based zones,
fanout cap) and the M5 `GraphService.neighbors` primitive
([ADR-046](046-m5-mcp-server-oauth-connectors.md) / [ADR-028](028-one-service-layer-mcp-peer-surface.md)).
· **Supersedes** the "phone = tappable list, **no canvas**" wording in
[06-web-app.md](../06-web-app.md) §3c / the M7 scope in 08.

## Context

M7 is "The map": desktop-first point-and-click graph exploration over the same one-hop primitive
MCP `traverse` already uses (`GraphService.neighbors`, M5 — cursor-paginated, returns rich
`NeighborEdge` rows carrying `origin/rel/dir/node_id/type/title/plane/score/since/until`, so a
render needs no second fetch). The service exists; the REST route, the render, and the interaction
model did not. ADR-032 #12 pinned the **library** (`react-force-graph` 2D canvas only, bundle-lean,
phone-safe) and the **flavour** (plex-style animated re-center à la TheBrain, rel-based zones over
pure physics, per-hop fanout cap with "show more") but left the product shape open. This ADR records
the shape, grilled decision-by-decision (2026-07-16).

The docs carried a genuine tension: "click **to expand**" (Obsidian, accumulate a growing subgraph)
vs "plex-style **re-center**" + "breadcrumbs" + "**never a whole-graph client layout**" (TheBrain,
one focal node at a time). They describe different products; only the second is compatible with
"never a whole-graph client layout." That fork had to be resolved first because it determines the
endpoint, the physics, and the perf ceiling.

## Decisions

**1. Navigation = re-center, not accumulate.** Exactly **one focal node at a time**. The client
only ever holds the current center + its 1-hop neighborhood (bounded, fanout-capped). Clicking a
neighbor makes *it* the new center: the sim reheats with the new center pinned at origin, the old
neighborhood fades/flies out, the new neighbors fan in and settle (plex/TheBrain). The node you came
from is a genuine 1-hop neighbor of the new center, so it stays on screen with no special-casing. A
**breadcrumb strip** records the click-path; a crumb click re-centers back to it and truncates the
trail there (exploring onward from mid-trail forks — forward history drops). "Expand three hops" =
three smooth re-centers. The client never holds more than one node's neighborhood → honors "never a
whole-graph client layout" and keeps the perf ceiling flat.

**2. Endpoint = grouped-by-zone with per-zone caps.** `GET /nodes/{id}/neighbors`, one route, two
modes:
- **No `rel`** → the **grouped** first page: `{ center, zones: [{ origin, rel, neighbors[…≤cap],
  total, next_cursor }, …] }`, one zone per distinct `(origin, rel)`, each independently capped at
  **`map_zone_fanout` (seed 8)** with a per-zone `total` count and per-zone `next_cursor`.
- **With `rel`** (+ optional `cursor`) → a **single zone's** next flat page, thin over the existing
  M5 `neighbors` primitive (rel-filtered keyset) — this is what "show more" calls.

Rationale: a **global** cap starves zones on a hub node (25 slots all eaten by `involves` → the
`about`/`at`/`similar` zones render empty though they have content), which breaks the accept
criterion "see their **constellation**." Per-zone caps + per-zone `total` ("show 8 more of 47")
fix it, and zone "show more" **reuses the M5 rel-filtered cursor untouched**. Zone key is
`(origin, rel)`; `direction=both`; each neighbor carries its own `dir` for arrowheads. Tombstoned
endpoints stay excluded (M5 behavior); **superseded (`until`-closed) edges are returned** (rendered
distinctly — decision 6).

**3. Layout & motion = force engine + per-zone directional forces.** Keep `react-force-graph`'s
d3-force sim (install **`react-force-graph-2d`** specifically — 2D-only, no three.js, per ADR-032's
"bundle-lean"). Pin the focal node at origin (`fx/fy=0`); add a per-zone **angular/radial force** so
each `(origin, rel)` group settles into its own sector ("people one side, topics another"); keep
stock collision + link forces for organic, breathing spacing. For **content** nodes this naturally
yields the ADR's people/topics/places separation (each rel targets a characteristic type); for
**entity hubs** the neighborhood is mostly one rel + a `similar` zone — zoning can't separate what's
homogeneous, so there the **fanout cap + "show more"** carries the view, not the zones. Derived
`similar` is always its own zone with faint edges.

**4. Node encoding = emoji glyph (type) + size/ring (hub vs content) + plane color.** Reuse the
existing `ui/nodeTypes` emoji map as the node mark (🧠/👤/💡/… — legible at a glance, no 9-shape
legend, on-brand) rather than inventing geometric shapes. Encode the one topologically-real shape
distinction: **entity hubs** (person/place/topic/event/project) render **larger with a ring**,
**content** nodes as smaller plain discs. Plane → a **new theme-independent categorical color map**
(6 seed planes + a fallback for governed/unknown planes + a neutral for `inbox`/none), validated for
contrast on both dark and light bases — *not* derived from the theme accent (a plane means the same
thing in every theme; the current single-`--accent` `PlaneBadge` is wrong for a map). Color by
**primary `plane`** only; full `planes[]` shows in the drawer (a multi-plane ring is deferred
polish). The **theme accent** is reserved for the focal node + selection.

**5. Interaction = single-click re-center; hover peek; center-click reads.** Single click/tap a
neighbor = **re-center** (the map's whole reason to exist is frictionless wandering; breadcrumbs
make it non-destructive). **Hover** (desktop) highlights + shows the label/tooltip **without
moving**. Clicking the **focal** node opens the full node in the shared **`ui/NodePreview`** drawer
(body + edges, rule 10) — read deep without leaving the map. Phone: tap neighbor = re-center, tap
center = drawer, long-press = peek. A compact caption chip on the focal node (title · type · plane ·
tags) renders **immediately from the `NeighborEdge` data** that rode along on the click, so there's
no flash while the new center's neighbors fetch resolves.

**6. Edges = canonical solid + arrowhead + gated label; derived faint; superseded dashed-dimmed.**
Canonical edges: solid, **subtle directional arrowheads** (center→out vs in→center), `rel` label on
**hover/focus or zoom-gated** (always-on when a zone has few edges) — never all-on (clutter).
Derived `similar`: faint, no arrowhead (symmetric), reads "similar" on hover. **Superseded edges**
(`until` set — e.g. "worked at X *until* 2025"): **shown, dashed + dimmed, `until` on hover** — the
belief-timeline / "how things changed" value (ADR-033) is exactly why `until` exists; hiding it
throws away the most interesting part of the graph. A "hide past connections" toggle is optional
polish.

**7. Screen = new full-width tab; canvas on phone too; list as fallback.** Map is the **8th
top-level tab**; it **opts out of the shell's 640px column** (a per-tab `wide` flag) to a
full-viewport canvas — without this "desktop-first" is dead on arrival. **Canvas is primary on both
desktop and phone** (ADR-032 chose 2D canvas *because* it is phone-safe; the re-center model caps
node count ≤~50). This **supersedes** the 06 §3c / 08 "phone = list, no canvas" wording. The
**tappable-list renderer is kept** — same `(origin, rel)` zones — as (a) the `prefers-reduced-motion`
fallback and (b) a manual view toggle. (Note: this app reads reduced-motion **only** from the OS
setting via framer-motion `useReducedMotion()`; the in-app override 06 §4 lists was never built, so
the list is carried in practice by the manual toggle + a11y correctness, not day-to-day use.)

**8. Entry + empty state.** Enter the map by setting a `mapSeed` node-id into `AppShell` (or a tiny
context) and flipping the tab — from **Search result cards** and from the **`NodePreview` drawer's
edge rows** (06 §5 already promised the edges are the entry into the map). No router / deep-linking
(single-user). **Empty state** (tab opened with no seed): an **embedded search** ("search your graph
to start exploring") → pick a result → center, plus **restore the last-centered node** from
`localStorage`. No new "list nodes / top hubs / me" endpoint — the system is search-first and has no
single "me" node. M7 adds exactly **one** new endpoint (neighbors).

## Consequences

- One new REST route (`GET /nodes/{id}/neighbors`) + one config knob (`map_zone_fanout`); zone
  "show more" reuses the M5 primitive. No migration, no new store surface, no organizer touch.
- The client stays trivial (one `neighbors` page in memory), the sim runs only on the desktop/phone
  canvas (not under reduced-motion), and the perf ceiling is bounded by the re-center model.
- A new theme-independent plane→color palette enters the design system (a `ui/` token map),
  reusable by any future plane-colored surface.

## Backlog (recorded, not M7)

- **Auto-center entry** — open the map on your highest-degree hubs (or recent captures) instead of
  the search box; needs a **new top-degree / entry-nodes endpoint** (deferred per the grill; the
  user explicitly wants it *later*).
- **In-app reduced-motion override** — 06 §4 lists it but it was never built; the app follows the OS
  setting only. Out of M7.
- **Multi-plane ring/pie** on the node mark (M7 colors by primary plane only).
- The ADR-032 **"continents" architecture** (nightly server-side UMAP/community layout → static
  tiles, LLM-named clusters → aerial whole-graph) stays the post-M7 growth path, its own design
  session.

## Alternatives considered

- **Accumulate (Obsidian-style growing subgraph)** — rejected: incompatible with "never a
  whole-graph client layout"; unbounded perf; makes the fanout cap meaningless.
- **Flat global-capped endpoint** (wrap `neighbors` as-is, client buckets) — rejected: starves zones
  on hubs, breaking the "constellation" accept criterion.
- **Deterministic radial sectors (no physics)** — rejected: static, fights the physics engine, and
  loses the living/breathing motion the design mandate makes first-class.
- **9 geometric shapes for the 9 types** — rejected: humans can't reliably distinguish 9 shapes; the
  emoji map already solves at-a-glance type with no legend.
- **Two-step select-then-center** — rejected: adds a click to every move, dilutes fluid wandering;
  breadcrumbs already cover the "lost my place" risk that usually justifies it.
- **Persistent split-pane detail** — rejected: cramps the hero canvas; on-demand `NodePreview`
  drawer reuses existing UI (rule 10).
