# Pipelines & Scheduling

**Version:** 2.3 · **Status:** Approved 2026-07-13 (2.3 = M2 indexing/search: nomic embeddings +
prefixes [ADR-022], nightly combined `reindex` job + relatedness graph [ADR-023], organizer tag
reuse [ADR-024]; 2.2 = M1 replan: STT fallback chain [ADR-020], capture interactions → `agent_runs`
[ADR-021], `claude_max_effort`; 2.1 = M1 capture follow-up nudge, ADR-019)

Five pipelines, decoupled: each fails, retries and evolves independently. Every step
transition is persisted (`captures.status`, `agent_runs`) — nothing silent
(vision P8). All LLM calls resolve through the provider registry with the
Claude-primary → Nebius-fallback chain ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md)).

**Claude Max effort ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md); M1 replan).**
Every `claude-max` call sets a reasoning-effort level on the CLI (`claude --print --model
claude-opus-4-8 --effort <level>`). A single global setting `CLAUDE_MAX_EFFORT` (default
**`medium`**, `deploy/defaults.env`) applies to every claude-max invocation in v1 — the knob
**ships in v1** (effort is never silently unset), the default is explicit. Per-task effort
(organize vs chat vs distill) is a **noted post-v1 extension**, not built now.

**STT resolves through a fallback chain ([ADR-020](adr/020-stt-fallback-chain-groq-primary.md)).**
Voice transcription walks `STT_CHAIN = ["groq", "openai"]` (Groq Whisper `large-v3` primary,
OpenAI `whisper-1` fallback) exactly like the chat chain — advancing past `ProviderUnavailable`
(an OpenAI 429 is one) and recording `model_used` / `fallback_used`.

## 1. Capture pipeline (user-initiated, immediate)

```
POST /capture/{voice,text}
   │ persist raw input + captures row, return 202 instantly
   │ open agent_runs row (agent="capture", running)          ← ADR-021
   ▼
[voice] TRANSCRIBE (STT chain: groq→openai)        status=transcribing
   ▼
ORGANIZE — LLM, JSON out                           status=organizing
   │ { notes: [ { title, plane, planes[], tags[], body } ] }
   │ may SPLIT into multiple atomic notes (ADR-005); "don't know" plane → Inbox
   │ tag reuse: existing vault tag vocabulary injected into the prompt —
   │ prefer an existing tag, coin new only if none fits (ADR-024, M2)
   ▼
WRITE NOTES to vault (frontmatter contract)        status=written
   ▼
INDEX each note + enqueue vault git backup         status=indexed
   │ close agent_runs row (succeeded/failed + model_used,     ← ADR-021
   │ fallback_used, details: stt/organize/nudge + timings)
```

Failure ⇒ `status=failed` + error; `retry` resumes from first incomplete step. Organizer
failure fallback: single note, title = first 8 words, plane = Inbox — capture is never
lost to a model error.

**Interaction logging ([ADR-021](adr/021-capture-interactions-agent-runs-logging.md)).** Each
run writes **one `agent_runs` row** (`agent="capture"`): `model_used` = organize model,
`fallback_used` = any step fell back (STT or organize), `details` = `{capture_id, kind,
stt:{provider,fallback_used,error?}, organize:{model,fallback_used}, nudge:{model}, timings_ms}`.
This is where the STT chain's rule-3 resolution lands. The `captures` row stays lean (drives the
strip); `agent_runs` is the queryable interaction log — explored via the Supabase dashboard /
`capture_interactions` view / optional Supabase MCP. Logging never changes capture behavior: a
model failure still ends the run `failed` *and* still yields the Inbox-fallback note.

**Follow-up nudge (M1, [ADR-019](adr/019-conversational-capture-minimal-in-m1.md)).** After a
*successful* organize (not the Inbox fallback), a single gentle "dig deeper" question is
generated as a **trailing, non-blocking** step (notes land first, protecting <30s) and stored
on the capture. `POST /captures/{id}/follow-up` re-organizes original+answer and **replaces**
the capture's notes (soft-delete via `git rm`, then write the enriched set).

## 2. Ingestion pipeline (agents, scheduled)

Connector contract and per-connector specs: [05-connectors.md](05-connectors.md).

```
scheduler fires connector (03:00–05:00 window)
   │ agent_runs row (status=running)
   ▼
FETCH new items since connector_cursors[connector]
   ▼
DISTILL — LLM conspects: filter noise, group by topic,
   split per plane → atomic notes (same frontmatter contract,
   source=<connector>, source_ref per item)
   ▼
WRITE notes → INDEX → advance cursor (only after successful write)
   ▼
finish agent_runs (summary + details per note, model_used, fallback_used)
   └─ enqueue vault git backup
```

Idempotency: cursor advances only past successfully-materialized items; re-running a
failed window reprocesses it; `source_ref` lets the distiller skip already-noted items.

## 3. Indexing pipeline (M2 — [ADR-022](adr/022-embeddings-self-hosted-nomic.md), [ADR-023](adr/023-semantic-relatedness-graph.md))

Triggers: capture/ingestion writes · the nightly combined `reindex` job · `POST /admin/reindex`.

```
file → read → strip frontmatter + sb:related block → sha256 ── unchanged? skip
     → parse frontmatter (plane/planes/tags/source/created)
     → chunk (docs/02 §4) → embed batched via nomic ("search_document: …")
     → per-note TRANSACTION: delete old chunks + upsert note + insert chunks
       + notes.embedding = mean-pool(chunk vectors)
```

- **Embedding provider = self-hosted nomic (Ollama), single-provider, no hot fallback**
  ([ADR-022](adr/022-embeddings-self-hosted-nomic.md)). One index = one vector space.
- **Robustness ([ADR-022](adr/022-embeddings-self-hosted-nomic.md) / M2):** per-note transaction (a
  note is never half-indexed); on an embed failure, **skip-and-continue** — leave the note's
  existing rows intact, log it, keep going, mark the run **`partial`**; batch-embed a note's chunks
  in one call with 429 backoff. Retry re-indexes only still-stale notes (hash-skip). Never crashes
  the job (rule 7).
- **Deletions** reconciled on rescan (DB rows without files removed, `chunks`/`note_links` cascade);
  a rename = delete old `vault_path` + insert new. Fully idempotent.
- **Relatedness graph** ([ADR-023](adr/023-semantic-relatedness-graph.md)) is recomputed **nightly
  only** (and on `/admin/reindex`), never on the real-time capture write: top-K over
  `notes.embedding` cosine above `RELATED_MIN_SCORE` → `note_links` → render changed `sb:related`
  blocks (churn-gated by block-diff). The capture path leaves the graph untouched.
- **`/admin/reindex` is async + single-flight** ([03-api](03-api.md)): `202`, an `agent="reindex"`
  run, `409` if a reindex/rescan is already running.

## 4. Chat / search pipeline (interactive)

```
query → embed → pgvector cosine top_k (optional planes filter)
      → [search] scored results
      → [chat] context blocks [n] + last N session turns
              → selected chat model (per-conversation picker; registry applies
                fallback if the pick is unavailable → flagged in response)
              → answer with [n] citations → persist messages
```

Prompt rules: answer only from context for questions about the user's notes; cite `[n]`;
say explicitly when the context lacks the answer; reply in the user's language.

## 5. Analysis pipeline (background intelligence)

| Job | Schedule (Europe/Bucharest) | Behavior |
|---|---|---|
| Slack ingest | 03:00 | pipeline §2 |
| Full rescan (`reindex`) | 03:40 | **one combined job** (M2, [ADR-023](adr/023-semantic-relatedness-graph.md)): `git pull` vault (external/other-device edits) → pipeline §3 over whole vault → recompute `note_links` → render `sb:related` blocks → commit+push (one vault git lock). Writes `agent="reindex"` (`details.trigger="nightly"`). Single-flight with `/admin/reindex`. |
| Daily summary | 04:10 | notes with `note_created_at` = yesterday → LLM: themes, decisions, open questions, **sectioned per plane** → `Summaries/Daily/YYYY-MM-DD.md` → upsert `summaries` → index. Skips silently when no notes. |
| Weekly review | Sunday 04:40 | inputs = week's daily summaries (fallback: week's notes) → patterns, recurring themes, cross-plane insights, suggested questions → `Summaries/Weekly/YYYY-Www.md` |
| Vault backup | after every write batch + 04:55 sweep | git add/commit/push ([ADR-001](adr/001-vault-on-vps-with-git-backup.md)) |

## Scheduling policy ([ADR-010](adr/010-agent-window-3-5am.md))

- All heavy agent work runs **03:00–05:00**, staggered as above: it's when the user
  doesn't compete for Claude Max usage windows, RAM, or attention.
- Jobs are idempotent and manually triggerable via `POST /agents/{name}/run` — the
  scheduler decides *when*, never *what*.
- Every run lands in `agent_runs` → activity feed, including fallback events
  ("conspected on Nebius: Claude limit reached").
