# Session Protocol (working agreement)

**Version:** 1.7 · **Status:** Approved 2026-07-16

*(v1.7 — 2026-07-16: added **Parallel task batches**, a provisional, opt-in
implementation-session mechanism for fanning independent tasks out to ≤3 side-agents.)*

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
   else. Nothing agreed lives only in chat. **When a milestone's task list is written,
   annotate each task's parallel-eligibility** (`parallel-with:` / `depends-on:` / a batch
   id) per *Parallel task batches* — the fan-out structure is decided here, at planning time,
   and lives in the docs; the coordinator never re-derives it live.
3. **Pause before implementation.** After recording, **stop**. The user decides whether to
   implement in this session or respawn a fresh one. Do not start coding until told to.

## Implementation-session loop

Tasks run **sequentially by default** (the loop below). Where planning declared an eligible
batch, independent tasks may instead fan out to side-agents — see *Parallel task batches*. The
review, must-fix, and record-then-pause discipline is the same either way.

1. **No grilling.** Read the approved docs/plan and implement against them.
2. **Independent agent review before each pause.** When a task/feature is done, and *before*
   recording it complete, spawn a **fresh, independent** agent to review it — in case
   something was missed. Independence matters: the reviewer starts from minimal prior context
   and reads the **approved plan/docs + the diff**, not the implementer's reasoning, so it
   isn't anchored by the same assumptions. It checks the change against: the milestone's
   **acceptance criteria** ([08-implementation-plan.md](08-implementation-plan.md)), the
   **ADRs / contract docs** (00–08), and the **CLAUDE.md hard rules + architecture
   invariants** — hunting specifically for gaps, missed criteria, contract/invariant
   violations, and untested paths. (Use the `/code-review` skill or a `code-reviewer`
   subagent as the mechanism; the requirement is tool-agnostic.)
   - **Triage the findings before proceeding:** contract, correctness, invariant, or
     acceptance-criteria violations are **must-fix** — resolve them (or, if they reveal the
     plan was wrong, stop and replan) before the task counts as done. Minor/style items are
     logged as follow-ups. The review outcome (and any must-fix resolutions) is recorded to
     the docs alongside the progress note.
3. **Pause between major features/tasks.** At each boundary — *after* the review passes —
   record progress to the docs and stop so the user can choose: continue here, or respawn.
   Long single sessions are not the goal; clean handoffs are.
4. **Hit an unrecorded decision? Stop and replan.** Don't resolve architecture or scope
   questions inline — grill them in a planning/replanning pass first, record, then resume.

## Parallel task batches (v1.7 — provisional)

Within an **implementation session only**, independent tasks may be run **concurrently** by
side-agents instead of one-at-a-time. This is **opt-in and provisional** — the default is the
strictly-sequential loop above, and reverting to it is a documented option (see *Reversal*
below), not a failure. **Planning and grilling never fan out** (grilling is interactive; it
stays in the foreground with the user).

**Live coordinator.** The main session runs as a thin **coordinator**: it does **not** write
code itself, it spawns **≤3 sibling side-agents at a time** (one level deep), and it keeps
every interactive/destructive step — git, the quality gate, review, and **all doc writes** —
in the foreground where permission prompts pass through. The coordinator stays
**minimal-context**: it shuttles task numbers and file paths, never the diffs or specs (the
side-agents hold that context and discard it on return). A longer-lived coordinator is
accepted *only* under this minimal-context discipline.

**Eligibility (decided at planning time, written into [08](08-implementation-plan.md)).** A
set of tasks may form a fan-out batch only if **all three** hold for every task against its
batch siblings:
1. **Disjoint files** — no two tasks in the batch write the same file.
2. **≤1 migration per batch** — at most one task creates an Alembic migration (migrations are
   strictly serial; two in one batch would collide on numbering/order).
3. **No intra-batch dependency** — no logical, data, or deploy ordering between batch members.

Planning annotates each task accordingly; the coordinator never re-derives eligibility live.

**Execution shape.**
`≤3 parallel implementers (effort by difficulty) → ≤3 parallel fresh per-task reviewers (high
effort) → coordinator foreground integration gate → resolve must-fix → record → pause.`

- **Implementers** edit in the **shared working tree** (the disjoint-files guarantee makes
  concurrent writes safe — no worktrees). Effort is chosen per task: **medium** for
  straightforward tasks, **high** for genuinely hard ones.
- **Review stays independent and per-task**, just *decoupled from the fan-out* — it runs
  **after** tasks land, not interleaved. One **fresh, high-effort** reviewer per landed task
  reads the approved plan + that task's diff; reviewers run in parallel. Because tasks are
  disjoint, cross-task integration is covered by the coordinator's gate, not a reviewer.
- **Integration gate (mandatory).** Once the batch has landed, the coordinator runs the full
  quality gate on the **merged** tree in the foreground — lint, typecheck, the whole test
  suite, migration apply — **once**, before recording.
- **Must-fix stays a hard blocker**, batched not downgraded: the batch is not recorded done
  and **nothing deploys** until every task's must-fix findings are resolved (scoped editor
  side-agent or coordinator inline) and re-reviewed on a fresh diff.

**Durability (a batch must survive a dead coordinator).**
- **Commit per task**, the moment it lands and passes review — completed tasks are durable in
  git immediately, so a mid-batch crash can only orphan tasks still in flight (which the docs
  never ticked; a respawn discards their partial edits and re-dispatches).
- **Coordinator is the sole doc writer** — side-agents never touch `README`/`08` (single
  files; concurrent writes would clobber). It records **batch dispatched** *before* fan-out,
  **ticks each task done** (sha + review outcome) as it lands, and records **batch complete**
  once the gate passes.
- **Pause at the batch boundary** (not per task); emit the handoff prompt there. A respawn
  reads `08`, sees which batch members are ticked vs pending, reconciles against `git log`,
  discards orphaned partial edits, and re-dispatches the rest.

**Effort selection (both loops).** The coordinator picks effort per agent by task difficulty,
not by role blanket: **medium** for straightforward implementation, **high** for genuinely
hard implementation, **high** for every reviewer, and **high** for the coordinator itself and
any grilling/planning pass.

**Reversal (provisional-change clause).** This mode ships as a trial. Reverting to the
strictly-sequential loop is a **human judgment call**, not an automated trigger — drop it with
no ceremony if any warning sign shows: a collision/merge incident despite the disjoint-files
guarantee, a defect a batched review let slip that an interleaved one would have caught, or a
respawn that couldn't cleanly reconstruct an in-flight batch from the docs.

## Security & secrets discipline

Secrets are the one class of data that must never be mishandled — a leaked or committed
credential is the single worst outcome of any session, worse than any bug.

- **The agent (LM) never handles secret values.** Private keys, passwords, API tokens,
  session secrets, and connection strings are never pasted into chat, never echoed back,
  never written to a tracked file by the agent. The agent works with **references** — env
  var names, `.env.example` placeholders, dashboard locations — not values. When a real
  secret is needed, the **human enters it directly** into the target (the VPS `deploy/.env`,
  the process environment, a provider dashboard, or a GitHub Actions secret); the agent may
  guide but must not receive it.
- **Secrets never enter git.** Real values live **only** in `deploy/.env` on the VPS
  (gitignored), the process environment, or GitHub Actions secrets. The repo tracks only
  `.env.example` (placeholders). Key material (`*.pem`, `id_*`, `*.key`, …) is gitignored.
- **Enforced, not just asked.** The code repo carries `.githooks/pre-commit` (blocks key
  files + secret-shaped values; enable with `git config core.hooksPath .githooks`) plus a
  **gitleaks** CI job as an un-bypassable backstop. **`git commit --no-verify` is forbidden
  for secret material.**
- **Exposure = compromise.** If a secret is ever pasted, echoed, or committed, treat it as
  compromised: **rotate it immediately** at the provider, then purge. Do not rationalise
  ("it was only for a second"); rotate.

## Commit & push discipline

- **Docs:** at **every pause** (after grilling, and between tasks), **commit *and* push**
  the docs repo. A freshly spawned session must be able to `git pull` and inherit the whole
  context. Docs pushes are automatic at pauses.
- **Code:** commit freely while implementing (small, coherent commits). **Pushing code is
  the user's call** — do not push the code repo without being asked.
- **Lint gate (local, mandatory).** The code repo's `.githooks/pre-commit` runs the exact
  linters CI runs, scoped to the staged files, so a lint/format error can never reach `main`
  and gate the deploy: Python (`server/**.py`) `ruff check` + `ruff format --check`, web
  (`web/**` source) `eslint --max-warnings 0`. It fails closed if `uv`/`pnpm` is absent for a
  staged subtree. Don't `--no-verify` past it — fix the code (see `.githooks/README.md`).

## Recording state for respawn (the docs are the single source)

Before any pause, **the docs must record where we are and what's next** — progress into the
relevant contract doc (e.g. a milestone's progress note in [08](08-implementation-plan.md)),
decisions as ADRs — and be pushed. That, plus the README cold-start procedure, is what lets
a fresh session continue. If it isn't written and pushed, a respawned session won't know it.

**Handoff prompt — ALWAYS emit one at every pause (mandatory, user preference 2026-07-12).**
End every pause with a short respawn prompt — never skip it, never bury it. Keep it to ~2–3
lines: a pointer to the docs plus the immediate next action. It is a pointer, not a restated
summary (full state lives in the docs). Format:
```
Read second-brain-docs/ (git pull first): README → 09 → <doc that records where we are>.
Next: <the immediate next action>.
```

## Why respawn-friendliness matters

A new session starts cold. Its only reliable memory is this docs repo. So: decisions →
docs, progress → docs, pauses → pushed. If it isn't written and pushed, a respawned session
won't know it happened.

## Checklist

**Every session**
- [ ] **No secret handled by the agent or committed** — values only in VPS `.env` / env /
      Actions secrets; hook + gitleaks green; any exposure rotated immediately

**Planning / replanning session**
- [ ] `/grilling` run; decisions captured
- [ ] Decisions recorded to docs (ADR for architectural choices)
- [ ] **Where-we-are + next action recorded in the docs** and pushed
- [ ] **Short handoff/respawn prompt emitted** (mandatory at every pause — pointer + next action)
- [ ] **Paused** — user chose continue vs respawn

**Implementation session**
- [ ] Built against the approved plan (no grilling)
- [ ] **Independent agent review** run at each task boundary (fresh context, checks diff vs
      acceptance criteria + ADRs + invariants); **must-fix findings resolved**; outcome recorded
- [ ] Code committed at each task; **progress + next action recorded to docs** at each pause
- [ ] Docs committed **and pushed** at each pause
- [ ] **Short handoff/respawn prompt emitted** (mandatory at every pause — pointer + next action)
- [ ] Any unrecorded decision → stopped and replanned, not decided inline
- [ ] **If a parallel batch** (opt-in): ≤3 side-agents, disjoint files, ≤1 migration; live
      minimal-context coordinator owns git/gate/review/doc-writes; per-task **fresh high-effort**
      review; **integration gate on the merged tree**; **per-task commits**; pause at the batch
      boundary; a respawn can reconstruct in-flight state from the docs + `git log`
