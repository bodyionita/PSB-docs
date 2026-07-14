# ADR-039: entity types are mention-only — content-node coercion guard (M3)

**Status:** Accepted · 2026-07-14 · Fixes organizer over-extraction found in the M3 task-10 live
Accept · tightens the organizer contract of [ADR-031](031-m3-organizer-and-contract-extensions.md)
and the entity substrate of [ADR-030](030-entity-substrate-and-lifecycle.md). Decided in the M3
organizer-quality replanning session; recorded before coding.

## Context

The task-10 Accept produced **`person`-typed content nodes with narrative bodies** — e.g. a node
titled "How I know Horia" whose body is a memory ("I know Horia because he is the husband of
Mădălina…"), typed `person`. That is a category error: entity hubs are meant to be **thin** (`body:
""`, aliases, disambig — exactly what the resolver-minted hubs correctly are), and people/places/orgs
should appear **only as mentions** on content nodes, which the resolver turns into hubs + edges.

Two code paths currently produce entity-type nodes: (a) the resolver minting thin hubs (correct), and
(b) the **organizer emitting an entity-typed content node** (wrong). Path (b) inflates the entity
types with narrative dupes and defeats the thin-hub + derived-profile model (ADR-030).

## Decision

**The organizer may reference entity types only as *mentions*; it must never emit them as content
nodes. Both a prompt rule and a deterministic structural guard enforce this.**

1. **Prompt rule.** The organizer prompt states that people, places, organizations, projects, events
   and topics are expressed **only** in a node's `entities[]` (mentions), never as a node of that
   type. Content nodes are `memory`/`conversation`/`insight`/`idea`.
2. **Structural guard (authoritative).** Independent of the prompt, any node the organizer emits
   whose `type` is an **entity type** is **coerced to `memory`**, keeping its `body`, `title`,
   `tags`, planes and `entities[]` intact — so the narrative survives as a memory and its mentions
   still mint/link the proper thin hubs + edges. The guard runs before resolution/write, so a prompt
   or model slip **cannot** corrupt the graph with a fat entity node.
3. **Thin hubs stay the resolver's job.** Entity hubs continue to be created solely by the resolver
   (mint) from mentions; the guard never creates a hub, it only reclassifies a mis-typed content
   node.

## Rationale

- Determinism: the guard makes the invariant ("entity types are thin hubs, minted only by the
  resolver") hold regardless of LLM behavior — the same defense-in-depth posture as the injection
  hygiene rules (ADR-031).
- Coercing to `memory` (rather than dropping) preserves the person's content (rule 2, never lose) and
  yields the correct shape: a `memory` "How I know Horia" —involves→ `person/Horia`.
- Prompt + guard together reduce both the frequency (prompt) and the blast radius (guard) of the
  error.

## Consequences

- The organizer post-processing gains the entity-type→`memory` coercion, driven by the config
  `entity_types` set (rule 9). Organizer prompt updated. A coerced node is flagged in the capture's
  `agent_runs` interaction (visible, rule 7).
- Existing over-extracted person nodes on prod are healed by the `reprocess-all-from-raw` op
  ([ADR-042](042-reprocess-all-from-raw-and-data-survival.md)).
- `03-api`/`04-pipelines` (organizer contract) + `08-logs/m3.md` updated in the same change set.
- Edge case noted: a coerced node's `title` may read like an entity name (e.g. "Mădălina Fairfax —
  childhood friend…"); acceptable — it is now a memory that links to the hub, and titles are not
  identity. ❌ Rejected: *prompt-only* (a regression silently reintroduces fat entity nodes) and
  *guard-only* (LLM keeps emitting the wrong shape; coercion titles drift).
