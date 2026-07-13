# ADR-025: UI-editable per-group model routing + per-group/per-provider effort

**Status:** Accepted · 2026-07-13 (M3 planning)
**Relates to / supersedes clauses of:** [004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) · [010 agent window](010-agent-window-3-5am.md) · updates [02-data-model §3 (app_settings)](../02-data-model.md), [03-api (Chat/Settings)](../03-api.md), [04-pipelines §4](../04-pipelines.md), [06-web-app (Chat/Settings)](../06-web-app.md), [08-implementation-plan §M3](../08-implementation-plan.md)

## Context

M3 (chat) is the first surface that lets a human drive `.complete()` interactively, and the point
where the user wants **direct control over which model runs which task, from the UI**, recorded on
the server.

Two things ADR-004 / the M1 replan fixed now need to change:

1. **Model chains were config-declared and static.** `build_registry(settings)` reads
   `chat_chain` / `distill_chain` from env **once at startup**; ADR-004 surfaced only the
   *per-conversation* chat pick and an (M4) agent-model Settings field. The user wants to choose the
   **active model + fallback per task from Settings**, persisted server-side, taking effect without
   a redeploy.
2. **Reasoning effort was a single global constant.** `CLAUDE_MAX_EFFORT` (default `medium`) is baked
   into the `claude-max` provider instance at construction; the M1 replan explicitly deferred
   per-task effort to "post-v1". The user wants effort tunable per task (e.g. sharp chat, cheaper
   organize) to spend the Claude Max window deliberately.

**Effort does not translate across providers.** `claude-max` takes `--effort low|medium|high|xhigh|max`;
Nebius (Llama 3.1 70B) has **no** reasoning-effort concept; OpenAI reasoning models have their own.
So effort is an **attribute of a model that supports it**, not a portable per-task scalar — a matrix,
surfaced only where the chosen model supports it.

## Decision

**1. Two configurable routing groups — `chat` and `conspect`.** The existing chains map to exactly
two task-groups, so Settings exposes two routing units (not per-call-site knobs):
- **`chat`** drives `chat_chain` — interactive chat.
- **`conspect`** drives `distill_chain` — organize (capture, [ADR-019](019-conversational-capture-minimal-in-m1.md)),
  the follow-up nudge, the M3 chat **query-condensation** call, M2 tag-consolidation propose, and
  the future M4/M5 distiller + summary jobs.
- Each group = `{ active, fallback, effort_by_provider }`. STT (`stt_chain`, groq→openai) and
  embeddings (single-provider ollama, [ADR-022](022-embeddings-self-hosted-nomic.md)) are **not**
  in scope — effort-irrelevant and provider-constrained; they stay config-only.

**2. Per-group, per-provider effort.** Each group stores an effort **only for models that support
one**. Today that means an independent Claude effort for `chat` and for `conspect` (Nebius shows no
effort control). The global `CLAUDE_MAX_EFFORT` is the **default** when a group hasn't set one. This
supersedes the "per-task effort is post-v1" note in [ADR-004](004-provider-registry-claude-primary-nebius-fallback.md) /
the M1 replan.

**3. `app_settings` is the source of truth; env config is the seed/default.** A **`ModelRoutingService`**
reads routing + effort from `app_settings` (key(s) under a `model_routing` namespace), falling back
to `chat_chain` / `distill_chain` / `CLAUDE_MAX_EFFORT` when unset. It caches in memory and **busts
the cache on save** — edits apply on the next request, no restart. The registry stays pure
provider-mechanics; the routing brain lives in one testable service.

**4. Effort becomes a per-call argument.** `registry.chat(...)` / `registry.distill(...)` accept an
`effort`, threaded down to `claude_max.complete(..., effort=…)` (the CLI already takes `--effort`;
today it's a constructor constant). This is a deliberate, recorded change to the provider boundary
([CLAUDE.md rule 3](../templates/CLAUDE.md)): providers still own vendor mechanics, but effort is now
passed per-call rather than fixed at build time. `fallback_used` / `model_used` recording is
unchanged.

**5. Settings default + per-conversation override.** The `chat` group sets the **default** active
model, fallback, and effort. The chat composer's **per-conversation picker** overrides only the
*active* model for that thread (the existing `model` param on `POST /chat`); the group's fallback +
effort still apply underneath — this is exactly the registry's existing `_resolve_chain(requested,
chain)` (requested first, configured chain as fallback). The per-conversation picker from ADR-004 /
[06](../06-web-app.md) is retained.

**6. Endpoints.** `GET /settings` returns the saved routing for both groups **plus** the available
models per group and which support effort + their effort levels (all sourced from the registry — no
hardcoded model lists or effort enums in the web). `PUT /settings/models` saves one group's routing
and busts the cache. `GET /chat/models` (composer picker) exposes the registry's **real ids**
(`claude-max`, `nebius`) with display labels from a `CHAT_MODEL_LABELS` config map. The M4-drafted
`PUT /settings/agents {conspect_model, fallback_model}` is **superseded** by `PUT /settings/models`.

## Consequences
- ✅ The user drives model + effort per task from the UI, server-recorded (`app_settings`), no redeploy.
- ✅ Deliberate spend of the Claude Max window: sharp chat (`high`), cheaper background conspect (`medium`).
- ✅ One routing brain (`ModelRoutingService`), unit-testable with fakes; the registry stays mechanics.
- ✅ No migration — `app_settings` already exists (revision 001); routing is jsonb under a namespaced key.
- ⚙️ Provider-boundary change: `effort` is now a per-call arg on the chat interface — recorded here,
  reflected in the base provider signature + registry.
- ⚙️ `app_settings` is now request-path state (cached). A bad saved value must degrade safely: unknown
  model id → fall back to the config default chain, never a hard failure ([CLAUDE.md rule 7](../templates/CLAUDE.md)).
- ↩️ Effort is per-group, not per-individual-call-site; finer granularity (e.g. organize vs nudge
  independently) remains a future option, unbuilt.
- ↩️ STT and embedding routing stay config-only; making them UI-editable is deferred (low value,
  effort-irrelevant, and an embedding change forces a full reindex — [ADR-022](022-embeddings-self-hosted-nomic.md)).
