# Session Protocol (working agreement)

**Version:** 1.0 · **Status:** Approved 2026-07-12

How every implementation session is run. This is binding process, not a suggestion — it
exists so any session (human or AI) can be started, paused, and **respawned fresh** without
losing context, because the docs always hold the full state.

## The loop

1. **Grill first.** Run `/grilling` at the start of any new session, before touching code.
   Walk the decision tree one question at a time; look up *facts* in the code, put
   *decisions* to the user.
2. **Record after grilling.** Write every decision into the docs — a new **ADR** for
   architectural choices, updates to the relevant contract docs (00–08) for everything
   else. Nothing agreed lives only in chat.
3. **Pause before implementation.** After recording, **stop**. The user decides whether to
   implement in this session or respawn a fresh one. Do not start coding until told to.
4. **Pause between major features/tasks.** At each boundary, record progress to the docs
   and stop so the user can choose: continue here, or respawn. Long single sessions are not
   the goal; clean handoffs are.

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

## Checklist per session

- [ ] `/grilling` run; decisions captured
- [ ] Decisions recorded to docs (ADR for architectural)
- [ ] Docs committed **and pushed**
- [ ] **Paused** — user chose continue vs respawn
- [ ] (if implementing) code committed at each task; progress recorded to docs at each pause
- [ ] Docs committed **and pushed** at each pause
