# Session Protocol (working agreement)

**Version:** 1.1 · **Status:** Approved 2026-07-12

How every session is run. This is binding process, not a suggestion — it exists so any
session (human or AI) can be started, paused, and **respawned fresh** without losing
context, because the docs always hold the full state.

## Two kinds of session

Grilling belongs to **planning**, not building. Which kind of session you're in decides
whether you grill:

- **Planning sessions** — architecture planning, implementation/scope planning, or
  **replanning** (when implementation reveals the plan was wrong). Here you decide *what*
  to build and *how*. **Grill first**, then record, then pause.
- **Implementation sessions** — build against an already-approved, already-recorded plan or
  design. **No grilling** — the decisions are made and written down; read the docs and
  implement, pausing between tasks.

If an implementation session hits an open question or a decision the docs don't answer,
**stop and switch to a planning/replanning pass** (grill the question, record it) rather
than deciding silently while coding.

## Planning-session loop

1. **Grill first.** Run `/grilling` before recording anything. Walk the decision tree one
   question at a time; look up *facts* in the code, put *decisions* to the user.
2. **Record after grilling.** Write every decision into the docs — a new **ADR** for
   architectural choices, updates to the relevant contract docs (00–08) for everything
   else. Nothing agreed lives only in chat.
3. **Pause before implementation.** After recording, **stop**. The user decides whether to
   implement in this session or respawn a fresh one. Do not start coding until told to.

## Implementation-session loop

1. **No grilling.** Read the approved docs/plan and implement against them.
2. **Pause between major features/tasks.** At each boundary, record progress to the docs
   and stop so the user can choose: continue here, or respawn. Long single sessions are not
   the goal; clean handoffs are.
3. **Hit an unrecorded decision? Stop and replan.** Don't resolve architecture or scope
   questions inline — grill them in a planning/replanning pass first, record, then resume.

## Commit & push discipline

- **Docs:** at **every pause** (after grilling, and between tasks), **commit *and* push**
  the docs repo. A freshly spawned session must be able to `git pull` and inherit the whole
  context. Docs pushes are automatic at pauses.
- **Code:** commit freely while implementing (small, coherent commits). **Pushing code is
  the user's call** — do not push the code repo without being asked.

## Why respawn-friendliness matters

A new session starts cold. Its only reliable memory is this docs repo. So: decisions →
docs, progress → docs, pauses → pushed. If it isn't written and pushed, a respawned session
won't know it happened.

## Checklist

**Planning / replanning session**
- [ ] `/grilling` run; decisions captured
- [ ] Decisions recorded to docs (ADR for architectural choices)
- [ ] Docs committed **and pushed**
- [ ] **Paused** — user chose continue vs respawn

**Implementation session**
- [ ] Built against the approved plan (no grilling)
- [ ] Code committed at each task; progress recorded to docs at each pause
- [ ] Docs committed **and pushed** at each pause
- [ ] Any unrecorded decision → stopped and replanned, not decided inline
