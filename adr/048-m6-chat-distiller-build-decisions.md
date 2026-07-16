# ADR-048: M6 chat-distiller + review queue — build decisions

**Status:** Accepted · 2026-07-16 (grilled decision-by-decision) · **Refines**
[ADR-029](029-conversational-ingestion-stance-gate-review-queue.md) (the stance-gate contract)
and [ADR-042](042-reprocess-all-from-raw-and-data-survival.md) (reprocess / data survival);
builds on [ADR-028](028-one-service-layer-mcp-peer-surface.md) (organizer = single writer),
[ADR-030](030-entity-substrate-and-lifecycle.md) (kind-generic `review_queue` + entity merge),
[ADR-032](032-prior-art-adoptions.md) (dedup verbs, salience), and
[ADR-047](047-pipeline-scheduling-primitive.md) (the nightly pipeline the jobs run in). Scope
= M6 core + addendum (a)–(d); **contradiction sweep (e) deferred**.

## Context

M6 turns in-app chat into stance-gated memory (ADR-029) and completes "the sleep cycle." The
kickoff grill resolved the seams the ADR left open — chiefly how a chat-distilled memory
**survives `reprocess-all`** (vision P10), given reprocess replays only `captures` rows and
`TRUNCATE`s the graph + `review_queue`. These are the build-ready decisions.

## Decisions

### 1. Durability — endorsement materializes a `captures` row (Option A)
When a candidate is **endorsed** (auto, or via review-agree), the system writes a **`captures`
row**: `raw = the candidate's memory statement` (clean prose in the user's voice), `source =
chat`, `source_ref = <session-id>`, `created_at = the anchoring message's time`. It then flows
through the **existing organizer** (rule 2b) and is **naturally replayed by `reprocess-all`** —
P10 satisfied with zero chat-specific reprocess machinery; a chat memory is indistinguishable
downstream from any other capture. The chat session (`chat_messages`) remains the deeper raw.
**One `captures` row per endorsed candidate** (not one per session), so each is an independent
unit for edit / remove / reprocess. Trade-off accepted: reprocess replays the *distilled text*,
not the chat, so a future distillation-logic *improvement* reaches only new sessions, not old
ones (a retro-re-distill is a separate, non-P10-critical op). ❌ Rejected: chat session as the
only raw with reprocess re-running the distiller — it would have to replay every human
agree/disagree verdict to reproduce the graph.

### 2. Plain-text handoff — the organizer re-resolves everything
The `captures` raw is **plain prose**. The organizer does all typing (`conversation`/`insight`/
…), entity resolution, edges (`about`/`involves`/`led_to`), and the vocabulary gate via its
normal governed path. The distiller's own entity list + shape guess are used **only** for the
review excerpt and salience — never passed to the organizer as authority (keeps the stance-gate
and the structure-writer as clean, separately-testable seams). ADR-029 §7's "typically
conversation/insight" stays descriptive.

### 3. The distill pass — single-pass, multi-candidate (segmentation is emergent)
One LLM call (**`conspect` tier, night effort**; input **fenced** as data-not-instructions)
over a session's **new turns** returns a **list of candidates**, each anchored to specific user
message(s) with `{candidate_text, stance, salience, evidence_excerpt, referenced_entity_names}`.
Pure-retrieval spans yield no candidate → the **salience gate** is "zero candidates → session
skipped-and-logged." This is how "segment before the gate" (addendum a) is honored without a
separate segmenter: asking for a *list of user-stance candidates* rather than a session summary
is exactly what surfaces the buried decision and drops the 90% retrieval noise. Chain down ⇒
degrade: session left un-distilled, watermark not advanced, run logged `failed`/`partial`,
retried next window (rule 7). **No `inbox/` fallback** — nothing is lost (the session is the
raw).

### 4. Stance — three outcomes, hedge/affect biased to review
Per candidate, stance ∈ **{endorsed, unclear, rejected}**:
- **endorsed** (clear uptake) → `captures` row → organizer (auto-ingest).
- **unclear** (no inferable stance) → `review_queue` (agree / disagree / maybe).
- **rejected** (user disagreed with / ignored an LLM suggestion) → **run-log detail only** —
  never a review item, never a node.
The prompt **routes sarcasm, hedging, and affect-laden statements to `unclear`, not
`endorsed`** (addendum a) — the ADR-029 anti-goal "guessing stance = silent corruption" made
operational. A light post-check may downgrade endorsed→unclear on hedge markers in the excerpt.

### 5. Eligibility, cursor, idempotency
`chat_sessions` has no close/idle signal, so idleness is derived from
`max(chat_messages.created_at)`. A session is distillable on a run when
`max(created_at) < now() − chat_distill_idle_hours` (config). A new **`chat_distill_state`**
table (`session_id pk, last_message_at, distilled_at, run_id`) holds a **message-timestamp
watermark**; a run processes **only messages after the watermark**, so re-runs (crash recovery,
manual-then-nightly, or a reopened thread) never re-emit old turns and are a no-op with no new
activity (idempotent re-distillation, addendum a). The watermark advances once a session's
candidates are **materialized** (endorsed → captures written, unclear → filed, rejected →
logged); pending review items count as materialized (rule 6). A within-session candidate-content
dedup guard backstops duplicate review items.

### 6. `POST /chat/sessions/{id}/remember`
Runs the **same** single distill pass **synchronously** on the delta-after-watermark and returns
`{endorsed, review}` (or `{skipped: reason}`); the resulting `captures` **organize in the
background** (202-style). Advances the same watermark (so the nightly run then skips / re-distills
only a later delta) — fully idempotent with the nightly path, **same salience + stance gate** (a
manual trigger changes timing, not judgment; lowering the manual bar is a later tuning knob).

### 7. `review_queue` — payloads, reopening, reprocess
- **`stance-candidate` payload** (jsonb): `{candidate_text, referenced_entity_names[], salience,
  why_unclear}`; `excerpt` = the anchoring conversation snippet; `source=chat`, `source_ref=
  session-id`. **No node ids** — names + text only — so a reprocess that rebuilds the graph can't
  strand it. **agree** → materialize a `captures` row = **the exact auto-endorse path** (one
  ingest path, not two); **disagree** → discarded (logged); **maybe** → parked and
  **re-openable**. This requires fixing `PgReviewQueue.resolve` (today it guards `WHERE
  status='pending'`, making `maybe` terminal): `pending` and `maybe` are both still-decidable,
  `resolved`/`discarded` are terminal.
- **`dedup-proposal` payload** *may* reference node ids freely (see §9).
- **Kind-aware reprocess reset** (refines ADR-042 §2): `reprocess-all` **preserves
  `stance-candidate`** rows (their backing — chat sessions — is never touched by capture replay,
  and the watermark won't re-file them) and **truncates the capture-derived + graph-derived
  kinds** (`entity-ambiguity`, `vocab-proposal`, `dedup-proposal` — all re-derivable by replay +
  the nightly sweeps). Endorsed chat nodes rebuild from their captures; parked human decisions
  survive.

### 8. Review ergonomics (addendum b)
- **Salience** for triage = a **coarse LLM tag (`high`/`med`/`low`)** emitted in the single pass
  (the ADR-032 "graph-degree + pins + edge-conf" formula is for *node* re-split, doesn't fit a
  not-yet-a-node candidate, and **no pin feature exists**). Orders the Review list + ranks
  auto-endorsed feed items. (Fallback if LLM scoring is ever unwanted: referenced-entity graph
  degree.)
- **Batch actions** — `POST /review/batch {ids[], action}`, best-effort per item, reusing the
  single-item resolution logic.
- **Maybe digest** — a **weekly** job (step in the `weekly` pipeline) emits one feed-visible
  `agent_run` summarizing parked maybes; the Review UI shows an **aging indicator + count badge**.
  No push (that's M10).

### 9. Dedup sweep (addendum c)
A **nightly, all-source** sweep (chat + capture + MCP — graph hygiene) runs **after reindex** so
embeddings exist: for each recently-ingested node, **high cosine (`nodes.embedding`) + shared
entity edges + overlapping `occurred` window** (thresholds config) files a **`dedup-proposal`**
review item (two node ids + signals). Actions: **keep** (dismiss), **link** (write an explicit
typed edge), **merge** (fold the duplicate). **Merge generalizes the M3 primitive:** extract the
shared "retarget inbound edges → tombstone loser `merged_into: survivor` → reindex → force-commit"
core out of `MergeService`; entity-merge keeps its alias-union on top, **content-merge is the core
alone** (rule 10 — no parallel implementation). Survivor picked in the review item (default:
higher-degree / older). **`augment`** ("same event, new fact" — an LLM prose-reconciliation) is
**deferred** to a follow-up.

### 10. Inbox drainer (addendum d)
A **nightly** job (its own step, before reindex) finds captures currently materialized as an
`inbox/` fallback (`captures.node_paths` pointing into `inbox/`) and re-runs the existing
**`reorganize_capture`** on each — with the now-richer entity registry, a previously
unorganizable capture may resolve into real typed nodes (replaced only on success; still-failing
stays in `inbox/`). Idempotent, **bounded per run** (config, like `backfill_max_links`); residual
entity ambiguity files the normal `entity-ambiguity` items (no new review kind).

### 11. One-tap remove (ADR-029 §6)
**Remove = git-rm the node file(s) + DB-delete (`nodes`/`chunks`/`edges`) + tombstone the backing
`captures` row** (a `removed_at`, so replay/reprocess skips it — else `reprocess-all` would
resurrect a removed memory). Soft-delete: git history keeps the file, the capture raw is retained
but replay-excluded. Not a P10 violation — the **user** deliberately removed it and nothing is
truly destroyed. **Scope:** M6 builds the first user-facing node removal, **keyed to
chat-distilled nodes**, flagged **only on auto-endorsed items** (agree-from-review is user-vetted;
general node removal stays backlog).

### 12. Web surfaces
- **Review screen** (extend): `stance-candidate` + `dedup-proposal` rendering, salience ordering,
  multi-select batch actions, maybe aging + count badge.
- **Chat screen** (extend): a per-session **"Remember now"** action → `POST …/remember`.
- **Auto-ingest audit/remove**: a **chat-scoped "recently auto-recorded" list** (node preview +
  one-tap remove), keeping M8's general Activity feed untouched (it's still the M8 "Coming Soon"
  placeholder). ❌ Rejected for M6: building a minimal real Activity feed (pre-empts M8);
  endpoint-only with no visible reversal (weakens ADR-029's trust loop during M6).

## Consequences

- Migrations: `chat_distill_state` (new table); `captures.removed_at` (tombstone). No schema
  change for maybe-reopen (status column already supports it) or the kind-aware reset (reprocess
  code). `agent_runs.parent_run_id` comes from M5.5/ADR-047.
- New endpoints: `POST /chat/sessions/{id}/remember`, `POST /review/batch`, chat-node remove,
  a "recently auto-recorded" list. Recorded in [03-api.md](../03-api.md).
- Pipelines (ADR-047): distiller + inbox-drain + dedup join the `nightly` pipeline; maybe-digest
  joins the `weekly` pipeline.
- ❌ Deferred out of M6: contradiction sweep (e) + `contradiction` review kind; dedup `augment`;
  retro-re-distillation of old sessions; the general (non-chat) node-remove.
