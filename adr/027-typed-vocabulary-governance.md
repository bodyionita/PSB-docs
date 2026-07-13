# ADR-027: Typed node/edge vocabulary with propose → approve → consolidate governance

**Status:** Accepted · 2026-07-13 · Builds on [ADR-026](026-graph-native-storage-obsidian-removed.md);
generalizes the consolidation pattern of [ADR-024](024-tag-vocabulary-reuse-and-consolidation.md).
**Extended by [ADR-030](030-entity-substrate-and-lifecycle.md)/[031](031-m3-organizer-and-contract-extensions.md) (M3 grilling):**
proposal storage = a `review_queue` kind (`vocab-proposal`); starter vocabulary seeded to 9 node
types / 6 edge rels.

## Context

The graph atom is a typed node with typed edges (ADR-026). Types make the graph queryable and
the map legible — but a frozen vocabulary can't follow a life, and a free-minting LLM recreates
the tag-sprawl problem at the structural level (the exact failure ADR-024 exists to clean up).
The user wants the vocabulary to "grow organically" — without ever growing silently.

## Decision

1. **Starter vocabularies (config-driven, like planes):**
   - **Node types:** `memory` (default — an experience/thought/event) · `person` · `idea` ·
     `conversation` · `insight`.
   - **Edge types:** `involves` (memory→person) · `about` (anything→idea/topic) · `part_of`
     (e.g. memory→conversation) · `led_to` (provenance/causality) · `follows` (temporal).
2. **The LLM proposes; the user approves.** The organizer/distiller must map everything to an
   existing type; when nothing fits it uses the default (`memory` for nodes; for edges it may
   omit) **and files a typed-vocabulary proposal** (name, definition, examples of the items
   that wanted it). Proposals surface in Settings + the activity feed; nothing changes until
   approved. The same mechanism covers node types and edge types.
3. **Approval triggers retro-consolidation.** Approving a type queues a **consolidation job**
   (ADR-024's propose→apply shape, run as an agent job): it re-walks the existing graph,
   proposes re-typings/new edges that should use the new type, and applies them on confirm —
   git-tracked, revertible, feed-visible. The vocabulary is therefore always retroactively
   consistent, not just forward-looking.
4. **Only the organizer writes vocabulary into the graph** ([ADR-028](028-one-service-layer-mcp-peer-surface.md)):
   every surface (UI capture, connectors, chat distillation, MCP `capture`) funnels through
   the same gatekeeper, so governance is enforceable at one choke point.

## Rationale

- "Propose, approve, retrofit" is the actual meaning of *organic*: the system tells the user
  what type it keeps reaching for and cannot find, growth comes from real usage, and the graph
  never bifurcates into pre-/post-vocabulary eras.
- One governance mechanism for node types, edge types (and, per ADR-024, tags) = one mental
  model and shared implementation.

## Consequences

- Settings gains a vocabulary panel (types, definitions, pending proposals); the consolidation
  job joins the agent-jobs roster (manually triggerable, live-observable — 08 M8).
- Schema: type/edge vocabularies live in config + `app_settings` (approved additions);
  proposals in an operational table (defined at M3 grilling).
- ❌ Rejected: user-only minting (growth then depends on unprompted introspection); free
  minting by the LLM (structural tag-sprawl); a fixed-forever taxonomy (fights the purpose).
