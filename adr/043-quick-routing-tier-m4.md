# ADR-043: a third `quick` routing tier for trivial tasks (M4)

**Status:** Accepted · 2026-07-14 (M4 planning) · **Extends** [ADR-025](025-ui-editable-model-routing-and-per-task-effort.md)
(two configurable routing groups → **three**) · touches [ADR-004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) ·
updates [02-data-model §app_settings](../02-data-model.md), [03-api §Settings](../03-api.md),
[04-pipelines §5](../04-pipelines.md), [06-web-app](../06-web-app.md), [08 §M4](../08-implementation-plan.md).
Decided in the M4 kickoff grilling, recorded before coding per [09](../09-session-protocol.md).

## Context

[ADR-025](025-ui-editable-model-routing-and-per-task-effort.md) established **two** UI-editable
routing groups — `chat` (interactive chat) and `conspect` (organize/distill: capture organize, the
follow-up nudge, chat query-condensation, tag/edge consolidation, entity resolution, profile
generation). Each group = `{ active, fallback, effort_by_provider }`, sourced from `app_settings`.

M4 introduces the first genuinely **trivial** LLM call: **session titling** (a one-line summary of a
chat thread, [ADR-025]-adjacent, [08 §M4](../08-implementation-plan.md)). Running that on the
`conspect` group means titling inherits whatever heavyweight model + effort the user set for organize
quality — e.g. Opus at `high` — which is wasteful for a throwaway title. The user wants a **cheap,
fast lane** for short/low-stakes calls (titling now, "other very short simple tasks" later), tunable
from the UI like the other groups.

Two facts shaped the mechanism:
- `ClaudeMaxProvider.complete()` **already** accepts a per-call `model` override and the `claude` CLI
  already takes `--model` — so targeting a cheaper Claude (Sonnet vs Opus) is a supported axis, not a
  new provider capability.
- The registry's routing unit is a **provider id** (`claude-max`, `nebius`), one configured model per
  id. Exposing a second Claude model as a *new provider id* keeps the chain/`requested_model`/Settings
  machinery unchanged — no special-casing of an "underlying model" axis.

## Decision

**1. A third routing group, `quick`,** modeled **identically** to the ADR-025 groups —
`{ active, fallback, effort_by_provider }`, read from `app_settings` under the same `model_routing`
namespace, cache-busted on save. `ModelRoutingService` becomes three-group generic; nothing about the
group mechanics is special-cased for `quick`.

**2. Default routing (config seed, Rule 9):** `active = claude-max-sonnet` at **low** effort;
`fallback = nebius` (already a small/cheap model, effort-irrelevant). Overridable from Settings and by
config; the env seed is the default when `app_settings` is unset (ADR-025 §3 pattern).

**3. Sonnet is a distinct provider id — `claude-max-sonnet`.** A second `ClaudeMaxProvider` instance
over the **same** `claude` CLI with a config-driven `--model` (initially **Sonnet 4.6** — chosen as
the cheaper tier "for now"; a one-line config swap to Sonnet 5 or any alias later, no code change).
Opus stays `claude-max`. This reuses the existing "one provider id = one configured model" pattern, so
`available_*_models` / `_resolve_chain` / `GET /chat/models` need no new concept. The exact CLI model
alias is confirmed at build time.

**4. M4's only `quick` caller is session titling.** Existing `conspect` calls are **not**
retroactively reclassified — organize, resolution, and profile generation are not trivial. New tier,
opt-in per call-site; future short tasks migrate onto `quick` as they are identified.

**5. `quick` is UI-editable in Settings → Models** alongside `chat` and `conspect` (three live
controls). The Settings panel is group-generic, so the marginal cost is one more control, not new
plumbing.

**6. Titling runs best-effort and non-blocking.** It is generated **after** the first assistant
message is persisted, never on the request's critical path — a titling failure or slow call must not
delay or fail the user's first answer (degrades to an untitled/first-message-derived title, logged).

## Rationale

- Deliberate spend: the Claude Max window is finite; titling a thread should not consume Opus-at-high.
  A `quick` tier makes "cheap by default for trivia" a first-class, tunable dial.
- Cheapest possible mechanism: no new provider capability (per-call `model` already exists), no new
  routing axis (Sonnet is just another provider id), no special-casing in `ModelRoutingService`.
- Symmetry with ADR-025: same group shape, same `app_settings` namespace, same Settings surface — one
  more group, not a parallel system.

## Consequences

- ✅ `ModelRoutingService` resolves three groups; `build_registry` gains a `claude-max-sonnet` instance
  (config `claude_max_sonnet_model`); Settings → Models renders three controls.
- ✅ `quick` slots into the same `app_settings` `model_routing` jsonb — **no migration** for routing.
- ⚙️ Provider-boundary note (with ADR-025 §4): `effort` is a per-call arg on the chat interface; the
  `quick` group supplies `low` to `claude-max-sonnet` per call. `model_used`/`fallback_used` recording
  is unchanged and now can report `claude-max-sonnet`.
- ⚙️ `CHAT_MODEL_LABELS` (ADR-025 §6) gains a display label for `claude-max-sonnet`.
- ↩️ Only titling uses `quick` in M4; broader adoption is incremental and unforced.
- ❌ Rejected: **titling on `conspect`** (wastes Opus/high on trivia — the motivating problem);
  **a hardcoded cheap model** (Rule 9, non-tunable, non-extensible); **an `underlying-model` per-call
  axis threaded through the registry** (bigger boundary change than a second provider id, for no gain);
  **per-call-site effort overrides** (the granularity ADR-025 deferred — the group model suffices).
