# ADR-044: provider-observability surface (in-memory status + `GET /admin/providers`)

**Status:** Accepted · 2026-07-15 (M4 follow-up planning, grilled decision-by-decision) ·
**Touches** [ADR-004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) ·
relates to [ADR-012](012-m0-implementation-stack.md) (the LLM-free `/health` rule — left untouched) ·
updates [03-api §Admin](../03-api.md), [06-web-app §Settings](../06-web-app.md),
[08 §M4](../08-implementation-plan.md). Recorded before coding per [09](../09-session-protocol.md).

## Context

The M4 live Accept surfaced a real defect: the committed `NEBIUS_CHAT_MODEL` was a model id Nebius
never served, so **every** live Nebius chat call raised `ProviderUnavailable` and **silently fell back
to `claude-max`**. The symptom was model-independent and took a diagnosis; nothing in the product made
the fallback *reason* visible ([08-logs/m4.md](../08-logs/m4.md) task 8). This violates the spirit of
**vision P8 ("everything visible")** and **CLAUDE.md rule 7** — a degradation with no visible reason.

Facts that shaped the design (grilled against the code):
- The registry already records `model_used`/`fallback_used` and `logger.warning`s each failed attempt,
  but the per-attempt `ProviderUnavailable` **error text is only logged, never surfaced** — a request
  path, not a job, so it never lands in `agent_runs`.
- **`Provider.health()` already exists** on every provider ([base.py](../../second-brain/server/app/providers/base.py))
  — a cheap, non-raising, LLM-free reachability probe — but it is **defined and never called**.
- Critically, `health()` checks **reachability**, not success: for Nebius it returns `True` whenever an
  API key is configured. It would have shown Nebius **green** the entire time every call was failing.
  So an availability snapshot alone gives **false comfort**; the signal that explains a silent fallback
  is the **actual runtime error captured at fallback time**.
- The app runs as a **single uvicorn worker** (no `--workers`), and the registry is a **process-lifetime
  singleton** on `app.state` — so in-memory per-provider state is consistent across all requests and
  resets only on redeploy/restart.

## Decision

**1. Capture the last runtime error per provider (the spine), fold reachability in as one field.**
The core signal is *why a provider last failed*. Availability (`reachable`) is a secondary field,
explicitly **config-reachability, not a success guarantee** — never rendered as "calls will succeed."

**2. State lives in-memory on the registry via a `ProviderStatusTracker` collaborator** — a separate,
unit-testable object (not dict-fields on the registry). **No migration, no persistence, no request-path
DB write.** A chat/STT fallback is a *degradation signal*, not a durable job failure (rule 7's
durability mandate is about jobs ending in `agent_runs`, which this is not). The failure mode that bit
us is **persistent** — every call failed — so even after a redeploy wipes memory, the next call
repopulates the last-error immediately; in-memory is still a 30-second read. Durable reliability
*history* is a clean follow-up (persist the same record shape) if it ever becomes a product need.

**3. Record at every provider call site — chat, STT, and embedding.** A one-line
`record_success(id)` / `record_failure(id, err)` at each registry catch/return point
([registry.py](../../second-brain/server/app/providers/registry.py)). Embedding has **no fallback**
(single provider), so a failure there is a total outage that today is recorded nowhere — the most
important blind spot to close.

**4. Sticky error + separate success stamp + failure counter.** Per provider:
`last_error {message, at}` (**sticky** — a later success does *not* wipe it), `last_success_at`, and
`consecutive_failures` (reset to 0 on success). This preserves the post-hoc forensic trail
("broke at 2pm, recovered at 2:05") that clear-on-success would throw away, while `consecutive_failures`
still gives the clean "is it broken *right now*" signal. The `message` is the `ProviderUnavailable`
text as-is (already `{id}`-prefixed, carries URL + status but no headers/keys — verified safe),
truncated to a sane length.

**5. Surface = new session-gated `GET /admin/providers`; `/health` is left untouched.** Per provider:
`{ id, label, capabilities: [chat|stt|embedding], reachable, last_error, last_success_at,
consecutive_failures }`. `reachable` is filled by a **live `health()` probe** per request (all providers
concurrently via `asyncio.gather`) — finally using the dormant seam; LLM-free and non-raising, so
ADR-012-safe. `/health` stays a clean **unauthenticated** liveness probe: per-provider error text
(endpoints, model ids, key-state) must not be public, and mixing diagnostics into `/health` would blur
its load-balancer contract.

**6. A read-only Settings "Providers" card** (thin TanStack-Query client over the endpoint, ADR-006):
one row per provider — a status dot (**green** `consecutive_failures == 0`, **amber** `> 0`) + the
sticky `last_error` line. **No actions, no editing.** The primary interface is the mobile PWA, so a
glanceable card — not an authenticated curl — is what makes the signal an actual 30-second read on the
device in use.

## Rationale

- Closes the exact P8/rule-7 gap the postmortem exposed, at the layer where the fallback happens.
- Cheapest mechanism that is *actually diagnostic*: reachability alone would have shown Nebius green;
  the captured runtime error is the thing that explains a silent fallback.
- In-memory is proportionate to a degradation diagnostic and matches the M4 "lean spine, defer the rest"
  ethos; the non-durability is a **deliberate** divergence from the `agent_runs` pattern, recorded here
  so a future session doesn't read it as an oversight (and doesn't "improve" `reachable` into a false
  green).

## Consequences

- ✅ New `ProviderStatusTracker` held by the registry; `record_*` calls at the three call sites;
  `GET /admin/providers` in the existing `/admin` router (session-gated); a read-only Settings card.
- ✅ **No migration** — state is in-memory; resets on redeploy (accepted, see Decision 2).
- ✅ The dormant `Provider.health()` seam is finally exercised (for `reachable`).
- ⚙️ `reachable` is labeled reachability, never success; `last_error`/`last_success_at` carry the runtime
  truth beside it.
- ↩️ Durable provider-reliability *history* remains a future follow-up (same record shape, persisted).
- ❌ Rejected: **extending the unauthenticated `/health`** (leaks per-provider detail, blurs the liveness
  contract); **persisting to `agent_runs`** (chat/STT fallback is not a job, and the request *succeeded*);
  **a static `configured` inference** (the live `health()` probe is more informative and uses the built
  seam); **clear-on-success `last_error`** (discards the forensic trail this follow-up exists to provide).
