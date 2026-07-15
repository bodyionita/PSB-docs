# ADR-045: first-class provider / model / effort separation (`claude-max` collapse)

**Status:** Accepted · 2026-07-15 (M4 follow-up 3 planning, grilled decision-by-decision) ·
**Supersedes** the routing-*unit* of [ADR-025](025-ui-editable-model-routing-and-per-task-effort.md)
(a group picks a **model id**, not a provider id) and the `claude-max-sonnet` *mechanism* of
[ADR-043](043-quick-routing-tier-m4.md) (Sonnet is a **model of the `claude` provider**, not a second
fake provider) · **refines** [ADR-004](004-provider-registry-claude-primary-nebius-fallback.md)
(a provider now maps to **N models**) · relates to
[ADR-044](044-provider-observability-surface.md) (the Providers card collapses to one `claude` row) ·
updates [02-data-model](../02-data-model.md), [03-api](../03-api.md), [04-pipelines](../04-pipelines.md),
[06-web-app](../06-web-app.md), [08](../08-implementation-plan.md). Recorded before coding per
[09](../09-session-protocol.md).

## Context

The system conflated **provider** with **model**. To route Opus for `chat`/`conspect` and a cheaper
Sonnet for `quick` (ADR-043), the registry constructs **two `ClaudeMaxProvider` instances over the same
`claude` CLI** — `id="claude-max"` (Opus) and `id="claude-max-sonnet"` (Sonnet) — differing only in the
`--model` flag ([registry.py:305-319](../../second-brain/server/app/providers/registry.py)). Everything
downstream treats a **provider id as the routable unit**: a routing group is `{active, fallback,
effort_by_provider}` where `active`/`fallback` are provider ids ([model_routing.py](../../second-brain/server/app/services/model_routing.py)),
the config seeds are provider-id chains (`chat_chain=["claude-max","nebius"]`), the chat composer's
`requested_model` is a provider id, and `chat_messages.model` stores one.

Two user-visible symptoms of the conflation (2026-07-15):
- The [ADR-044](044-provider-observability-surface.md) Providers card shows **two `claude` rows** for one
  underlying CLI + one OAuth/Max credential. Their `health()` is *always identical* (both probe the same
  `claude` binary), so the split is noise: a fallback is a **provider** event, but the model conflation
  forces it to look like two.
- The raw id `claude-max` leaks into the UI — a name that means nothing to a user thinking, correctly,
  "the provider is **Claude**; Opus and Sonnet are **models**; effort is a **parameter**."

Facts that shaped the design (grilled against the code):
- The `claude` CLI provider's `complete()` **already accepts a per-call `model=`** — one instance can
  serve any number of models. The two-instance split buys nothing the CLI doesn't already give.
- Only `claude` is fake-split. `nebius` (chat), `groq`/`openai` (STT), `ollama` (embedding) are genuinely
  distinct OpenAI-compatible endpoints (different base URL / key / model) — same *class*, real separate
  providers. "Fix the others too" means **apply the provider≠model vocabulary + a friendly provider
  label**, not collapse anything.
- The user-editable routing surface (Settings → Models groups `chat`/`conspect`/`quick` + the chat
  composer picker) is **chat-only**. STT (`stt_chain`) and embedding are **config-only**, never pickable.
- Saved routing lives in `app_settings.model_routing` **keyed by provider id**; historical
  `chat_messages.model` holds provider ids — both must survive the rename (**vision P10**).

## Decision

**1. The routable / pickable unit is a MODEL ID; the provider is an attribute of the model.**
The chat catalog becomes a flat list of models, each carrying `{id, provider, label, supports_effort,
effort_levels}`. A routing group's `active`/`fallback` stay **scalar** but now hold **model ids**;
`run_chain` resolves each model id → its provider instance + the vendor model string to pass. This is
the minimal realization of provider≠model: the stored shape is unchanged, only its *value set* moves
from provider ids to model ids. (Rejected: an explicit `{provider, model, effort}` object per pick — it
only disambiguates a "same model id, two providers" case that never occurs for chat here, at the cost of
a nested contract + UI.)

**2. Scope: fix the concept across every provider internally; the user-pickable surface stays chat-only.**
`claude`/`nebius`/`groq`/`openai`/`ollama` each become a provider with model(s); the fake `claude-max`/
`claude-max-sonnet` ids die. The editable model picker stays chat-only (Opus / Sonnet / Llama 3.3 with
the provider derived). STT/embedding keep their real provider/model internally but gain **no** new
editing UI — embedding especially is a schema/`embedding_dim` migration concern, not a casual dropdown.

**3. A model id is the RAW VENDOR MODEL STRING** — `claude-opus-4-8`, `claude-sonnet-4-6`,
`meta-llama/Llama-3.3-70B-Instruct`. No short-key indirection. The accepted cost: a vendor-string change
(e.g. Sonnet 4.6→5, or a Nebius path change) is a config **and** a saved-routing migration touch, not a
transparent remap. (Rejected: a stable short-key layer — cleaner against vendor churn, but adds an
indirection table the user did not want.)

**4. Migrate saved routing; leave historical audit untouched.**
- `app_settings.model_routing` (saved overrides) is **migrated** by an idempotent Alembic revision
  (plain SQL, ADR-011): `claude-max`→`claude-opus-4-8`, `claude-max-sonnet`→`claude-sonnet-4-6`,
  `nebius`→`meta-llama/Llama-3.3-70B-Instruct`, in `active`/`fallback`/`effort_by_model`; a no-op when
  the row is absent/empty. This is the **P10-load-bearing** step — it preserves a deliberate routing
  choice across the rename instead of silently degrading it to the seed.
- `chat_messages.model` (historical "answered by" audit) is **left as-is**; label resolution stays
  **legacy-tolerant** (still maps the old ids to names). Rewriting past audit rows would falsify the
  record (a turn recorded as `nebius` during the broken-Llama-3.1 window must stay `nebius`) and it is
  display-only — the P10-faithful call for audit data is *tolerate, don't rewrite*.

**5. Config declares Claude's models as named scalars.** `CLAUDE_OPUS_MODEL` + `CLAUDE_SONNET_MODEL`
(+ `CLAUDE_EFFORT` as the default effort seed, replacing `claude_max_effort`/`quick_effort`'s
provider-scoped naming). `NEBIUS_CHAT_MODEL` stays a scalar (one model). The `*_chain` seeds keep their
names but now hold **model strings** (`chat_chain=["claude-opus-4-8","meta-llama/Llama-3.3-70B-Instruct"]`,
`quick_chain=["claude-sonnet-4-6", …]`). Rule-9 intact — models still declared in config, never
hardcoded. (Rejected: a single `CLAUDE_MODELS` CSV — more future-proof, but the explicit named scalars
were preferred for readability.)

**6. Identity splits into two labels; the Providers card is provider-only.**
- **Provider label** = a friendly provider name: `claude`→"Claude", `nebius`→"Nebius", `groq`→"Groq",
  `openai`→"OpenAI", `ollama`→"Ollama".
- **Model label** = the model-derived name ("Claude Opus 4.8", "Llama 3.3 70B") — used in Settings →
  Models and the chat "answered by …" caption.
- `GET /admin/providers` returns **one row per provider** (5 total), labeled by provider name, with **no
  raw id and no per-model breakdown** — a fallback/error is a *provider* event, and one CLI/credential is
  one health signal. This is the literal fix for the duplicate-`claude`-rows complaint.

## Consequences

- Provider id `claude-max`/`claude-max-sonnet` → one **`claude`**; class `ClaudeMaxProvider` →
  **`ClaudeProvider`**; file `claude_max.py` → **`claude.py`**. The `claude` provider serves both Opus and
  Sonnet via per-call `--model`.
- The registry gains a **chat-model catalog** (`{id, provider, label, supports_effort, effort_levels}`)
  and a **model→provider** index; `run_chain` walks model ids.
- `ModelRouting` keys by model id; the API/web field **`effort_by_provider` → `effort_by_model`**
  (keeping "provider" would relaunch the exact confusion this ADR removes). `/settings`, `/chat/models`,
  and `/admin/providers` response shapes update accordingly (03-api).
- One **idempotent Alembic migration** for saved routing (Decision 4); legacy-tolerant label mapping for
  `chat_messages.model`.
- The Providers status card (ADR-044) renders **5 provider-only rows**; `claude` appears once.
- **Superseded:** ADR-025's "routing unit = provider id" and ADR-043's "`claude-max-sonnet` second
  provider" — both restated here as model-id routing over a single `claude` provider. ADR-004's provider
  boundary (only `providers/` imports vendor SDKs) **stands**; this refines only the provider→model
  cardinality.
- **Not in scope:** STT/embedding editing UI (config-only, Decision 2); a stable short-key model layer
  (Decision 3); durable provider-reliability history (still the ADR-044 follow-up).

## Implementation (tasks tracked in [08 §M4 follow-up 3](../08-implementation-plan.md))

1. Server core — `ClaudeProvider` (N models); registry catalog + model→provider index; `model_routing`
   keyed by model id; config named scalars; chains → model strings; `effort_by_model`; tests.
2. Migration — Alembic rewrite of saved routing (idempotent) + legacy-tolerant `chat_messages.model` labels.
3. Server API — `/settings`, `/chat/models`, `/admin/providers` response shapes.
4. Web — types, ModelsPanel, chat picker, ProvidersPanel (provider-only).
5. Live Accept — deploy; migration applies; saved routing preserved; card shows 5 provider rows (one
   "Claude"); chat still grounded on Opus/Sonnet/Llama.
