# ADR-028: One service layer; MCP is a peer surface (query + store through the organizer)

**Status:** Accepted · 2026-07-13 · Elevates the v2-backlog "MCP server layer" note to core
architecture; builds on [ADR-003](003-single-service-on-vps.md), [ADR-026](026-graph-native-storage-obsidian-removed.md),
[ADR-027](027-typed-vocabulary-governance.md).

## Context

The brain must be usable from other LLMs (Claude in any chat, other clients) with capabilities
similar to the app: query the graph **and** feed it. The user's own observation: the PWA and
MCP differ only in a small subset of human-only features (voice capture with Romanian STT,
visual exploration, custom UX, settings). Architecture should therefore not privilege the web
app — both are skins over the same backend.

## Decision

1. **One service layer, thin surfaces.** All capabilities live in services; the REST API
   (PWA), the **MCP server**, and later the map consume the same primitives — `search`,
   `get_node`, `traverse`, `capture`, vocabulary/plane listings. No logic in any surface;
   no duplication between them.
2. **MCP scope = query + store.** Tools: `search`, `get_node`, `traverse` (expand a node's
   edges), `list_planes` / `list_types`, and `capture(text)`.
3. **Store = a capture.** MCP `capture(text)` enters the **exact same organizer pipeline** as
   a UI capture: typing, entity resolution, edge creation, inbox fallback, vocabulary
   governance. **The organizer is the single writer of graph structure** for every surface —
   UI, connectors, chat distillation ([ADR-029](029-conversational-ingestion-stance-gate-review-queue.md)),
   MCP. External LLMs never author nodes/edges directly; if they want to influence typing,
   they say so in the submitted text and the organizer picks it up.
4. **No chat tool on MCP.** The calling LLM *is* the chat — it composes answers itself from
   the query primitives (which also lets the user feel what graph-aware retrieval adds before
   it is built into the in-app chat pipeline).
5. **Auth:** MCP uses a **bearer token**, separate from the web session cookie; same
   single-user trust model, revocable independently.
6. **PWA exclusives** are the human affordances only: voice recording + STT (Claude clients
   can't take Romanian voice), the visual map, settings/admin, review queue UI.

## Rationale

- The organizer-as-single-gatekeeper keeps vocabulary governance (ADR-027), stance discipline
  (ADR-029), and never-lose semantics enforceable at one choke point, no matter who knocks.
- MCP early (M5 in the roadmap) multiplies ingestion volume while later milestones are built —
  the smallest milestone with the biggest compounding effect.

## Consequences

- The MCP server ships as milestone M5 ([08-implementation-plan.md](../08-implementation-plan.md)),
  token-authenticated, running on the VPS beside the API (same service layer, per ADR-003).
- `traverse`/`get_node` are built once and reused by the map (M7).
- ❌ Rejected: query-only MCP (contradicts the stated purpose); raw graph writes over MCP
  (external models authoring structure without the organizer's discipline); a separate MCP
  business-logic layer (logic duplication, drift).
