# ADR-019: Conversational capture (minimal) in M1 — one follow-up nudge, replace-on-answer

**Status:** Accepted · 2026-07-12
**Relates to:** [005 planes/atomic notes](005-planes-and-atomic-notes.md) · [004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) · [014 vault durability](014-vault-history-durability.md) · updates [04-pipelines](../04-pipelines.md), [03-api](../03-api.md), [02-data-model](../02-data-model.md), [08-implementation-plan §M1](../08-implementation-plan.md)
**Supersedes scope of:** the "conversational capture" v2-backlog item (08) — a **minimal** form is pulled into M1; the full multi-turn interviewer stays v2.

## Context
The vision wants capture to draw the thought *out*, not just record it — a gentle "dig
deeper" nudge after a capture. The full version (an adaptive multi-turn interviewer/therapist
during ingestion) is the hardest UX in the product and was parked in the v2 backlog. During
M1 grilling (2026-07-12) the user chose to pull a **minimal** version into M1: exactly **one**
short follow-up question per capture, answerable or ignorable, with the answer **enriching the
same memory** rather than fragmenting it.

Two hard constraints shape the design:
- **The <30s accept criterion** ([08 M1](../08-implementation-plan.md)): a capture must reach
  the vault fast. A human may never answer the nudge (or answer minutes later), so the nudge
  must **not** block the first result.
- **One capture = one coherent memory** (the user's explicit intent): answering the nudge
  should *enrich* the note-set, not leave two half-told fragments.

## Decision

**1. Pipeline placement — nudge is a trailing, non-blocking step.** Pass 1 always runs to
completion first: `transcribe → organize → write → index (stub)`. The capture reaches
`indexed` and satisfies "<30s to vault" regardless of the nudge. **Only then** is the nudge
generated, from the held organize result, via `registry.distill()` (cheap chain, ADR-004),
and stored on the capture. If Pass-1 organize took the **Inbox fallback** (model failure),
**no nudge** is generated — there is no understanding to dig into.

**2. Answering triggers Pass 2 — replace, not augment.** `POST /captures/{id}/follow-up
{answer}` appends the answer and re-organizes **original + answer** into a fresh note-set. The
Pass-1 notes recorded in `captures.note_paths` are **soft-deleted** (`git rm`, content stays
in history per [ADR-014](014-vault-history-durability.md) §3), the enriched set is written,
and `note_paths` is overwritten. Result: one capture → one enriched, coherent set of notes.
Re-organizing may produce a different split (1 note → 2, or vice-versa); replacing the whole
`note_paths` set handles that cleanly.

**3. Schema (migration 002, explicit SQL per [ADR-011](011-alembic-migrations-plain-sql-no-orm.md)).**
Add nullable `follow_up_question text` and `follow_up_answer text` to `captures`. No new
*blocking* status: after Pass 1 the capture is already `indexed`; `follow_up_question` present
+ `follow_up_answer` absent is the "nudge pending" signal. Answering cycles the status
`organizing → written → indexed` again.

**4. Prompt.** A versioned constant beside the organizer prompt: "one short, warm, open
question inviting them to expand on the most emotionally or substantively significant thread;
≤20 words; gentle, never an interrogation." Preserves the input's language. **No server-side
expiry** in M1 — the nudge simply ages out of the recent-captures strip in the UI.

**5. Web.** A recent-capture item with a pending `follow_up_question` shows the question with
an inline answer input; submitting calls the follow-up endpoint and shows re-processing via
the same live-status polling as any capture.

## Consequences
- ✅ Signature "dig deeper" feel is present from M1 without blocking the fast path or the
  <30s criterion.
- ✅ Enriched, non-fragmented memories; the replace path reuses the organize+write machinery.
- ⚙️ Pulls a contained **supersede** path (soft-delete + rewrite of a capture's notes) into
  M1 — the same primitive a future "undo a manual ingestion" (v2) will build on.
- ⚙️ One extra `distill` call per successful capture (nudge generation) + one more per
  answered nudge (Pass 2). Accepted — cheap chain, personal volume.
- ↩️ The **full** multi-turn interviewer remains v2; this ADR deliberately ships the smallest
  version that delivers the feel.
- ❌ Rejected: **augment** (add-only notes — fragments the thought); **append-and-defer**
  (rewrite only in the nightly pass — enriched note wouldn't appear until next morning,
  defeating the point of pulling it into M1); **blocking Pass 1 on the answer** (breaks <30s
  and never-lose timing).
