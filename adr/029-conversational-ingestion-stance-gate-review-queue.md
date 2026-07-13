# ADR-029: Conversational ingestion — stance-gated distillation + one review queue

**Status:** Accepted · 2026-07-13 · Builds on [ADR-026](026-graph-native-storage-obsidian-removed.md),
[ADR-027](027-typed-vocabulary-governance.md), [ADR-028](028-one-service-layer-mcp-peer-surface.md);
extends the connector pattern of [ADR-008](008-connectors-run-on-vps.md) / [05-connectors.md](../05-connectors.md).
**Extended by [ADR-030](030-entity-substrate-and-lifecycle.md) (M3 grilling):** the `review_queue`
table lands **in M3** (kind-generic; first tenants = entity ambiguity + vocab proposals); this
ADR's M6 scope becomes the stance kinds + the polished Review UX.

## Context

The vision reframe makes conversations first-class memory sources: in-app chats with the LLM
should themselves become memories when they carry substance. But two failure modes threaten the
graph: (a) **memory lint** — most chat is retrieval ("what did I say about X?"), and recording
it creates summaries of the user asking about his summaries; (b) **authorship confusion** — an
LLM saying something clever is not the user's memory; *the user agreeing with it* is. The brain
must record **the user's stance**, not the model's output. The same stance problem exists in
people-conversations (Slack etc.): what did *the user* commit to, agree with, decide?

## Decision

1. **In-app chat is an ingestion source** — a **chat-distiller agent job** on the connector
   pattern: nightly (agent window), cursor over ended/idle chat sessions, every run in
   `agent_runs`, feed-visible ("3 sessions read, 1 insight recorded, 2 skipped"). Plus a
   per-session **"remember this now"** manual trigger for immediacy.
2. **Salience gate:** a session is distilled only if new information, a decision, a reflection,
   or a stated intention surfaced. Pure-retrieval sessions are skipped (logged as skipped).
3. **Stance-first distillation.** Candidates anchor on the **user's messages** — what he
   asserted, decided, agreed to, pushed back on. LLM statements enter a candidate only with
   evidence of user uptake. Per-candidate outcomes:
   - **endorsed** (clear uptake) → auto-ingested through the organizer;
   - **rejected** (disagreed/ignored) → dropped, kept in the run log;
   - **unclear** (no inferable stance) → the **review queue**.
4. **One review queue for all conversational sources** — LLM chats *and* people-conversations
   (Slack now, others later). Each item = proposed memory + short conversation excerpt +
   **agree / disagree / maybe**. Agree → organizer ingests; disagree → discarded (logged);
   maybe → stays queued, **no expiry**.
5. **The queue is DB-tier (operational), not graph-store content.** A candidate becomes
   canonical memory only on approval. Consistent with never-lose: the raw source (chat
   session / connector payload) is already persisted, so the queue is re-derivable.
6. **Auto-ingest is one tap from reversal:** every endorsed (auto) item is flagged in the
   activity feed with a "that's wrong — remove" action (soft-delete via git, history kept).
7. Distilled output goes **through the organizer** (ADR-028), typically as `conversation` /
   `insight` nodes edged (`about`, `involves`, `led_to`) to the nodes discussed;
   `source: chat`, `source_ref: <session-id>`.

## Rationale

- Reuses three committed mechanisms — the connector contract, the agent window + feed, the
  organizer-as-single-writer — instead of inventing a chat-specific pipeline.
- The stance gate is the difference between "a log of things an LLM said at me" and "my mind".
- Auto-ingest with feed-level reversal (option b of the trust question) keeps the flow
  frictionless while every stance call stays auditable and reversible.

## Consequences

- Ships as milestone M6 (after chat exists, M4): review-queue table + endpoints, Review
  surface in the web app, feed flags, the distiller job. Slack's distiller adopts the same
  stance gate when it lands (M9).
- ❌ Rejected: manual-only saving (loses most reflection); distill-everything (memory lint);
  guessing stance when unclear (silent corruption of the graph); queue items expiring
  (silently drops maybes).
