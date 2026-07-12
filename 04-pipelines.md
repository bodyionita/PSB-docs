# Pipelines & Scheduling

**Version:** 2.0 · **Status:** Approved 2026-07-12

Five pipelines, decoupled: each fails, retries and evolves independently. Every step
transition is persisted (`captures.status`, `agent_runs`) — nothing silent
(vision P8). All LLM calls resolve through the provider registry with the
Claude-primary → Nebius-fallback chain ([ADR-004](adr/004-provider-registry-claude-primary-nebius-fallback.md)).

## 1. Capture pipeline (user-initiated, immediate)

```
POST /capture/{voice,text}
   │ persist raw input + captures row, return 202 instantly
   ▼
[voice] TRANSCRIBE (Whisper)                       status=transcribing
   ▼
ORGANIZE — LLM, JSON out                           status=organizing
   │ { notes: [ { title, plane, planes[], tags[], body } ] }
   │ may SPLIT into multiple atomic notes (ADR-005); "don't know" plane → Inbox
   ▼
WRITE NOTES to vault (frontmatter contract)        status=written
   ▼
INDEX each note + enqueue vault git backup         status=indexed
```

Failure ⇒ `status=failed` + error; `retry` resumes from first incomplete step. Organizer
failure fallback: single note, title = first 8 words, plane = Inbox — capture is never
lost to a model error.

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

## 3. Indexing pipeline

Triggers: capture/ingestion writes · nightly full rescan · `POST /admin/reindex`.

```
file → read → sha256 ── unchanged? skip
     → parse frontmatter (plane/planes/tags/source/created)
     → chunk (docs/02 §4) → embed (batched) → transactional upsert notes+chunks
```

Deletions reconciled on rescan (DB rows without files are removed). Fully idempotent.

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
| Full rescan | 03:40 | pipeline §3 over whole vault |
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
