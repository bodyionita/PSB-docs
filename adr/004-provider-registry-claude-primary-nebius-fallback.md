# ADR-004: Provider registry; Claude Max (Agent SDK) primary with automatic Nebius fallback

**Status:** Accepted · 2026-07-12

## Context
The user pays for Claude Max 5× and wants it as the primary mind. A Max subscription is
not an API key: programmatic use goes through the Claude Agent SDK / headless Claude CLI
(OAuth), which shares the subscription's usage windows with interactive use and cannot do
embeddings or STT. Vendor independence (vision P6) must survive this preference.

## Decision
A **provider registry**: config declares named providers and per-task routing with
fallback chains.

- **Chat + distillation:** chain `claude-max` → `nebius`. `claude-max` is implemented
  over the Claude Agent SDK (CLI credentials on a Docker volume). Fallback triggers on
  rate-limit/timeout/error; every resolution is recorded (`model_used`, `fallback_used`)
  and surfaced in the activity feed / chat banner.
- **UI control:** chat model picked per conversation; agent (distillation) model + its
  fallback configured separately in Settings → Agents. Both lists come from the registry.
- **Embeddings:** fixed (`text-embedding-3-small` via OpenAI) — changing it means a
  migration + full reindex, so it is deliberately not UI-selectable.
- **STT:** fixed (Whisper via OpenAI).
- Nebius and any OpenAI-compatible endpoint share one client implementation; a new
  compatible provider is config-only.

## Consequences
- ✅ Max subscription utilized; costs drop to near-zero for the heaviest LLM stage.
- ✅ No pipeline ever blocks on Claude limits — Nebius answers, visibly flagged.
- ⚠️ Agent runs consume the same usage windows as interactive Claude Code → mitigated by
  the 03:00–05:00 window (ADR-010).
- ⚠️ CLI/OAuth provider is more fragile than HTTP APIs: health-checked, and re-login is a
  documented ops task.
- ❌ Rejected: Anthropic pay-per-token API (double paying next to Max); LangChain/LiteLLM
  (framework weight for a 3-method interface + custom fallback logic anyway).
