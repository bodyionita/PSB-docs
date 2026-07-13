# ADR-022: Embeddings on self-hosted nomic-embed-text (Ollama), single-provider, 768-dim

**Status:** Accepted · 2026-07-13 (M2 planning) · **Supersedes the embedding-provider choice in**
[ADR-004](004-provider-registry-claude-primary-nebius-fallback.md) (which named OpenAI
`text-embedding-3-small`, 1536-dim)
**Relates to:** [002 Supabase/pgvector](002-supabase-pgvector-for-index.md) · [004 provider registry](004-provider-registry-claude-primary-nebius-fallback.md) · [ADR-023 relatedness graph](023-semantic-relatedness-graph.md) · updates [02-data-model](../02-data-model.md), [04-pipelines](../04-pipelines.md), [07-infrastructure](../07-infrastructure.md), [08-implementation-plan §M2](../08-implementation-plan.md)

## Context

M2 builds the real index, so the embedding model becomes a live dependency for the first time
(the M1 index step was a no-op stub — `notes`/`chunks` are still empty). Two facts make **now**
the right — and cheapest — moment to revisit the embedding provider:

1. **The index is empty.** Changing the embedding model means a dimension change + full reindex.
   With zero chunks embedded, that cost is ~nothing today; it grows with the corpus.
2. **The user is deliberately reducing reliance on OpenAI.** OpenAI is currently only: embeddings
   + the STT *fallback* (Groq is STT primary, [ADR-020](020-stt-fallback-chain-groq-primary.md)).
   Moving embeddings off OpenAI leaves it as STT fallback only.

Facts established during grilling (2026-07-13, verified against provider docs):
- **Anthropic/Claude has no first-party embeddings API** — not an option.
- **Groq embeddings are not dependable** — availability was unconfirmed/thin at the time; treated
  as not available.
- **Self-hosting is feasible on the box.** The VPS (Hetzner CX23, 2 vCPU / 4 GB; DB is external
  Supabase) has ~3 GB free. `nomic-embed-text-v1.5` (137M params, 768-dim) served by **Ollama**
  exposes an **OpenAI-compatible `/v1/embeddings`** — a drop-in for the existing
  `OpenAICompatibleProvider` (base-URL change only, no new plumbing). Quantized resident size
  ~0.1–0.3 GB, CPU inference ~100 ms/text — safe even under stacked nightly jobs.
- **A multilingual local model (bge-m3, ~0.6–1.1 GB) was ruled out on this box** — too tight
  under the nightly `pg_dump` + `git bundle` + reindex peak (OOM risk); it would argue for a RAM
  bump first.

## Decision

**1. The sole embedding provider is self-hosted `nomic-embed-text-v1.5` via Ollama.**
- **768 dimensions** (was 1536). `chunks.embedding` and the new `notes.embedding`
  ([ADR-023](023-semantic-relatedness-graph.md)) are `vector(768)`; HNSW cosine.
  `embedding_dim` / `embedding_model` become settings; migration 004 resizes the (empty) columns.
- Wired as an `OpenAICompatibleProvider` pointed at the on-box Ollama endpoint; the registry's
  `embedding_provider_id` becomes the local provider. New settings (non-secret): the Ollama base
  URL + model name. No API key (localhost).
- **Asymmetric task prefixes are mandatory** (nomic requirement, or retrieval quality drops):
  the indexer embeds documents as `search_document: {title}\n\n{chunk}`; `/search` embeds the
  query as `search_query: {q}`. This is a build-contract detail, enforced in code, not left to
  the prompt.

**2. Embeddings are single-provider — no hot fallback.** Unlike chat/STT (where a fallback answers
one request and nothing persists cross-provider), embeddings **persist into a shared index**, and
different models produce **incompatible vector spaces** (different dims, different geometry). A
query embedded by model A cannot be compared to chunks embedded by model B — a cross-model
"fallback" would silently corrupt retrieval. So the index has exactly one embedder.
- **Resilience instead comes from locality + retry:** the model runs on-box (no network, no rate
  limits); its only real failure is "container down," which takes the whole app down anyway. The
  M2 indexer's **skip-and-continue** ([ADR-023](023-semantic-relatedness-graph.md) / M2 spec)
  re-embeds any note that missed embedding on the next rescan.

**3. Nebius is the documented cold-swap escape hatch (not a hot fallback).** If English-centric
search disappoints (e.g. cross-lingual RO→EN queries retrieve poorly), the embedder is a single
config value: switch `embedding_provider_id` to Nebius (`Qwen3-Embedding`, multilingual,
OpenAI-compatible, already a configured provider), adjust `embedding_dim`, and run one reindex.
This is a deliberate migration, not a per-request switch.

**4. A same-model hot fallback is a noted future option (not built).** The *only* index-safe hot
fallback is another host running the **identical** model — local Ollama nomic → Nomic's hosted
`nomic-embed-text-v1.5` (same 768-dim space). Adds a Nomic account/key; deferred until locality
proves insufficient.

## Consequences
- ✅ Embeddings leave OpenAI entirely (OpenAI stays only as STT fallback); no per-request embedding
  cost; private/on-box.
- ✅ Reuses the `OpenAICompatibleProvider` + registry seam — a base-URL/model config change, not new
  code. Query/index prefixing is the one behavioral addition.
- ✅ Correct-by-construction index: one model, one space; no cross-provider corruption path exists.
- ⚙️ **English-centric.** Cross-lingual (RO query → EN note) retrieval is weaker than a multilingual
  model. Accepted for M2; Nebius cold-swap is the exit if it bites (tune/decide during the M2 live
  Accept).
- ⚙️ **New operational component:** an Ollama container in compose ([07](../07-infrastructure.md)),
  a model pull at provision time, ~0.1–0.3 GB resident on the 4 GB box. Startup ordering: the app
  tolerates Ollama-not-ready (a capture's index step fails retryably; the next rescan converges).
- ⚙️ Migration 004 resizes the empty `vector(1536)` columns to `vector(768)` and recreates the HNSW
  indexes; `embedding_dim` is a setting so a future swap is a migration + reindex, never a code edit.
- ↩️ Multilingual-on-box (bge-m3) deferred behind a RAM upgrade; hosted-nomic hot fallback deferred.
