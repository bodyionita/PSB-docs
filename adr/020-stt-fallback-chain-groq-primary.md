# ADR-020: STT fallback chain — Groq (Whisper large-v3) primary, OpenAI fallback

**Status:** Accepted · 2026-07-12
**Relates to:** [004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) (extends it to the STT task) · updates [04-pipelines](../04-pipelines.md), [02-data-model](../02-data-model.md), [07-infrastructure](../07-infrastructure.md), [08-implementation-plan §M1](../08-implementation-plan.md)

## Context
The M1 live drive on `braindan.cc` returned an **OpenAI 429** on voice capture. STT was a
**single** provider (OpenAI Whisper `whisper-1`, resolved via `stt_provider_id`), so a 429 —
rate-limit or quota — killed voice capture outright: the audio was persisted and the capture
marked `failed` (never-lose held), but no transcript was produced and no fallback existed.

Chat/distill already survive a primary outage because they walk a **chain**
(`_chat_over_chain`, [ADR-004](004-provider-registry-claude-primary-nebius-fallback.md)) that
advances past any `ProviderUnavailable` and records which model answered. STT had no equivalent.
A 429 already surfaces as `ProviderUnavailable` (via `raise_for_status` → `httpx.HTTPError`), so
the missing piece is only the *chain walk* for transcription.

## Decision

**1. STT becomes a chain, mirroring chat.** Replace the single `stt_provider_id` with an
ordered `stt_chain`; the registry gains a `_transcribe_over_chain` that walks the chain exactly
like `_chat_over_chain` — advance past `ProviderUnavailable`, raise `RegistryExhausted` only when
every provider is unavailable.

**2. `stt_chain = ["groq", "openai"]` — Groq primary, OpenAI fallback.**
- **Groq** serves Whisper on an **OpenAI-compatible** `/audio/transcriptions` endpoint, so it
  drops into the existing `OpenAICompatibleProvider` with just a base URL / key / model — **no
  new provider class**. Its generous free tier fits the "lean on subscriptions, minimise
  per-call cost" ethos (Claude Max is already the chat primary): making the free provider primary
  means voice stops costing OpenAI per-minute *and* dodges the quota that 429'd.
- Model: **`whisper-large-v3`** (config `groq_stt_model`, not turbo). The transcript is the seed
  the organizer LLM works from — transcription errors propagate into every downstream note and
  can't be fixed after the fact, so accuracy outranks the ~seconds turbo would save (the <30s
  Accept budget is dominated by the organize LLM step, not STT). `groq_stt_model` makes flipping
  to turbo a one-line config change.
- **OpenAI (`whisper-1`) stays as fallback** — free to keep (the OpenAI key already exists for
  embeddings, which must stay OpenAI for pgvector dimension stability), fires only when Groq is
  down/rate-limited.

**3. Rule-3 recording (forced).** With a chain now present, [CLAUDE.md rule 3] requires the STT
fallback resolution to be recorded, not swallowed. `transcribe()` therefore stops returning a
bare `str` and returns a **result carrying `model_used` + `fallback_used`** (as `ChatResult`
does). Where that resolution is persisted is [ADR-021](021-capture-interactions-agent-runs-logging.md)
(one `agent_runs` row per capture).

**4. No per-provider retry in v1.** On a 429 the chain falls straight through to OpenAI — no
same-provider backoff/retry. If **both** providers rate-limit, the capture is marked `failed`
with the chain errors and stays **user-retryable** (never-lose holds; the audio is on disk). A
retry/backoff policy can be added later if Groq's free-tier limits prove tight.

**5. Config & provisioning.** New non-secret config (`deploy/defaults.env`): `STT_CHAIN`,
`GROQ_BASE_URL`, `GROQ_STT_MODEL`. New **secret** (`.env` on the VPS + GitHub Actions secret):
`GROQ_API_KEY` — the human enters it directly; the agent never handles the value (rule 11).

## Consequences
- ✅ Voice capture degrades instead of dying on an OpenAI 429; the M1 live Accept can pass.
- ✅ Cheaper (free primary) **and** higher-quality (`whisper-large-v3` > `whisper-1`) than before.
- ✅ Reuses the ADR-004 chain machinery and `OpenAICompatibleProvider` — no new vendor class.
- ⚙️ `transcribe()` signature changes to a result object; call sites (`CapturePipeline._transcribe`)
  and the registry interface update accordingly.
- ⚙️ Adds one vendor dependency (Groq) at the provider boundary only (rule 3 / ADR-006 hold).
- ↩️ Per-provider retry/backoff and per-task STT selection are deferred; not needed for v1.
- ❌ Rejected: **Nebius for STT** (LLM/embeddings studio, no first-class Whisper endpoint);
  **self-hosted Whisper on the VPS** (the 4GB box is already tight, ADR-003/015);
  **Groq-only** (no safety net when the free tier rate-limits); **OpenAI-primary + Groq-fallback**
  (keeps paying OpenAI and keeps hitting the quota that caused this).
