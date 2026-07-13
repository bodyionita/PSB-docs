# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

**Status:** design approved 2026-07-12 (grilled decision-by-decision). **M0 grilled and
recorded 2026-07-12** (see [ADR-011](adr/011-alembic-migrations-plain-sql-no-orm.md),
[ADR-012](adr/012-m0-implementation-stack.md)). **M0 / M0b ACCEPT COMPLETE 2026-07-12** —
the code monorepo `../second-brain/` is **deployed live at `https://braindan.cc`**: PWA over
HTTPS, login works, `/health` green (db/vault/git_remote), Cloudflare TLS **Full (strict)**,
`claude login` done on the box. The one accept clause with no M0 surface — the *live*
Claude-limit→Nebius chain-and-record — is formally **deferred to M3** (satisfied for M0 by
unit tests); see [08-implementation-plan.md](08-implementation-plan.md) Accept amendment.
**M1 grilled + recorded to build-ready detail 2026-07-12** (see
[ADR-019](adr/019-conversational-capture-minimal-in-m1.md) and the M1 build-decisions block in
[08-implementation-plan.md](08-implementation-plan.md)) — minimal conversational capture pulled
into M1, full ADR-014 durability set, online-only web capture. **M1 implementation IN PROGRESS:
Task 1 (migration 002 + capture domain core / `CapturePipeline`), Task 2 (capture routers +
lifespan wiring), Task 3 / durability **Slice A** (git-backed `VaultBackupService` — one-lock
ff-only push + heal-on-reject merge, debounced commits, empty-repo bootstrap, gc/reflog pins,
`POST /admin/backup`), and durability **Slice B1** (the four R2/WORM jobs — `git bundle`→R2 +
fingerprint, integrity drill, `pg_dump`→R2, `/srv/data`→R2 — `agent_runs` writer, boto3 object
store, CLI) done, reviewed, and verified 2026-07-12** (see the *M1 progress* block in
[08](08-implementation-plan.md)). **Durability Slice B2 (in-process APScheduler + `/health` `backups`
4th leg) done, reviewed (no must-fix), and verified 2026-07-12 — the durability task is complete
(Slices A+B1+B2); 116 tests + ruff green.** **Web capture screen (06) done, 2026-07-12** — the last
M1 surface: record orb + Web-Audio `AnalyserNode` visualizer, quick text capture, recent-captures
strip (`GET /captures?limit=20`, TanStack Query polling ~2s while in-flight), failed→retry, inline
follow-up nudge; online-only (offline text queue stays M5). `tsc`+`eslint`+`vite build` green; code
committed locally (not pushed — user's call). **M1 replan recorded 2026-07-12** — the first live
drive on `braindan.cc` + an independent review reopened M1 with recorded, not-yet-built scope
(see the *M1 replan* block in [08](08-implementation-plan.md) and
[ADR-020](adr/020-stt-fallback-chain-groq-primary.md)/[ADR-021](adr/021-capture-interactions-agent-runs-logging.md)):
**(1)** STT fallback chain (Groq `whisper-large-v3` primary → OpenAI fallback) after voice hit an
OpenAI 429; **(2)** capture interactions logged to `agent_runs` + a `capture_interactions` view,
explored via the Supabase dashboard/MCP; **(3)** global `CLAUDE_MAX_EFFORT=medium` on every
claude-max call. **M1 replan IMPLEMENTED 2026-07-12** — all replan code is built, independently reviewed, and
static-verified; committed locally (**not pushed — user's call**). Commits: server `d9b21e8` (STT
chain, capture `agent_runs` logging + migration 003 view, `CLAUDE_MAX_EFFORT`), web `4d988ea` +
`26a1c09` (nudge/Pass-2 render live, reduced-motion), deploy `a56038f` (vault-push SSH wiring:
openssh-client + entrypoint installs the mounted deploy key + pins GitHub's host key). The
independent review found **1 must-fix** (answered-nudge Pass-2 not rendering live) — **fixed in
`26a1c09`** — and reconciled Inbox-fallback semantics (a *success*, now flagged via
`inbox_fallback`). Server 124 pytest + ruff green; web `tsc`+`eslint`+`vite build` green; **not yet
run live**. `GROQ_API_KEY` added to GitHub Actions secrets. **What remains (all operational): push
code; confirm the `PSB-vault` GitHub repo + write-enabled deploy key on the box; then run the M1
*live Accept*** on `braindan.cc` (voice→plane-note <30s in vault **and GitHub history**, nudge
appears, organizer-down→Inbox note, a nightly WORM bundle + drill). Then M1 closes and M2
(indexing/search) begins. **M1 polish batch SHIPPED 2026-07-13** — English-only vault + valid tags +
vault git hygiene + `POST /admin/captures/{id}/reorganize` (`d469277`, deployed); the one Romanian
capture reorganized to English + pushed to `PSB-vault`. **Post-ship vault review (2026-07-13):** all
notes English/valid; one organizer-v2 name corruption `Braindone`→`Braindan` fixed by hand (vault
`70396dd`). Clarified two by-design points (shared `id` = capture id; `related` = same-source
siblings) and flagged **semantic relatedness linking** as a future-scope question for M2 — see the
*M1 polish batch* block in [08](08-implementation-plan.md). **M1 close POSTPONED to end of M2
(2026-07-13):** the Accept's nightly-WORM-bundle + weekly-integrity-drill tail needs real
overnight/weekly cycles, so the app is left running overnight and **M2 starts now**; the M1 Accept
confirmation (backup evidence + any overnight findings) folds into the M2 close. **M2 grilled to
build-ready detail 2026-07-13** (see [ADR-022](adr/022-embeddings-self-hosted-nomic.md) self-hosted
nomic embeddings, [ADR-023](adr/023-semantic-relatedness-graph.md) materialized relatedness graph,
[ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md) tag reuse, and the M2 build-decisions
block in [08](08-implementation-plan.md)): local nomic-via-Ollama embeddings (768-dim, single
provider, Nebius cold-swap), a full materialized `note_links` graph rendered as Obsidian-visible
wikilink blocks, note-grouped `/search` + read-only preview, async single-flight `/admin/reindex`,
a combined nightly pull+rescan+graph job, forward tag reuse + a manual consolidate tool, and a web
Search + Admin surface. **Overnight snapshot recorded** (box up; nudge/Inbox-fallback live-verified;
scheduled-backup evidence pending the next cycle — not chased). **M2 implementation IN PROGRESS:
Task 1 (migration 004 + Ollama/nomic provider wiring) done, independently reviewed (no must-fix),
committed locally (server `c66b562`, not pushed) 2026-07-13** — `vector(768)` resize +
`notes.embedding` + `note_links` (verified up/down/up on real pgvector), self-hosted
nomic-via-Ollama as the sole embedding provider (keyless localhost), OpenAI now STT-only, `ollama`
compose sidecar; 134 pytest + ruff green. **Task 2 (pure text chunker — `app/indexing/chunking.py`,
02 §4) done, independently reviewed, committed locally (server `fdd0f60`, not pushed) 2026-07-13** —
heading→paragraph→hard-split-with-overlap, frontmatter + `sb:related` stripping, 19 chunker tests
(153 total + ruff green). **Task 3 (indexer service — the real index step replacing the M1 stub)
done, independently reviewed (1 must-fix — batch resilience — fixed), committed locally (server
`684604e`, not pushed) 2026-07-13** — per-note `read → strip sb:related → sha256 → skip-if-unchanged
→ parse frontmatter → chunk_note → batch-embed (`search_document:` prefix) → per-note tx (upsert
note + replace chunks) + `notes.embedding` mean-pool`; `index_paths` (capture path) + `reindex_all`
(rescan + deletion reconcile); embed-fail → skip-and-continue → `partial`; graph left untouched
(nightly-only). No YAML dep (own frontmatter parser); a pgvector `vector` codec on the asyncpg pool
lets embeddings pass as plain float lists. Wired into the capture pipeline (best-effort — vault is
truth, so an index failure never fails a written capture; outcome logged to `agent_runs`). 173
pytest + ruff green; **verified end-to-end against a real pgvector DB** (codec round-trip, per-note
tx, mean-pool, hash-skip, cosine query, deletion reconcile). See the *M2 progress* block in
[08](08-implementation-plan.md). Next: `/search` + `GET /notes/{id}` (search-side `search_query:`
prefix + note-grouped results).

> **Planning/replanning sessions start with `/grilling`; implementation sessions build
> against the approved plan (no grilling). Every session follows
> [09-session-protocol.md](09-session-protocol.md).**

## Reading order

| Doc | Contents |
|---|---|
| [00-vision.md](00-vision.md) | Why the system exists, principles, planes, non-goals, success criteria |
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA client, VPS service, agents, storage |
| [02-data-model.md](02-data-model.md) | Vault layout (planes), note frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the only seam between web and server) |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, indexing, chat, analysis pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract, Slack connector spec, deferred connectors |
| [06-web-app.md](06-web-app.md) | PWA screens, design language (premium, animated), auth UX |
| [07-infrastructure.md](07-infrastructure.md) | VPS, Docker Compose, Caddy, Cloudflare, CI/CD, secrets, backups |
| [08-implementation-plan.md](08-implementation-plan.md) | Phased delivery with acceptance criteria |
| [09-session-protocol.md](09-session-protocol.md) | How every session runs: grill → record → pause → respawn-friendly commits |
| [adr/](adr/) | Architecture Decision Records — the *why* behind every choice |
| [templates/CLAUDE.md](templates/CLAUDE.md) | Ready-to-copy implementation rules for the code monorepo |

## Development workspace layout (local machine)

```
PersonalSecondBrain/          # workspace folder, not a repo
├── second-brain-docs/        # THIS repo — documentation
├── second-brain/             # code monorepo: server/ + web/ + deploy/
└── ObisidanVault/            # local dev vault (scratch, not canonical)
```

Production vault lives on the VPS (see [ADR-001](adr/001-vault-on-vps-with-git-backup.md)).

## Cold start — instructions for a fresh implementation session

If you are an AI (or human) picking this up with no prior context:

0. Follow the [session protocol](09-session-protocol.md): **planning/replanning** sessions
   `/grilling` first, record decisions to docs, and **pause before implementation** so the
   user can continue or respawn; **implementation** sessions build against the approved plan
   (no grilling), pausing between tasks. Commit + push docs at every pause.
1. Read the docs in the order above; skim every ADR — they are binding.
2. The code monorepo `../second-brain/` **may not exist yet**. If missing, create it per
   [01-architecture.md](01-architecture.md) layout (`server/`, `web/`, `deploy/`), git-init
   it, and copy [templates/CLAUDE.md](templates/CLAUDE.md) to its root as `CLAUDE.md`.
3. Implement strictly by phases in [08-implementation-plan.md](08-implementation-plan.md),
   starting at the first milestone whose acceptance criteria don't pass yet. Do not skip ahead.
4. Anything ambiguous or contradictory: fix the docs first (new ADR if architectural),
   then implement. Never silently diverge from these documents.
5. Things intentionally NOT decided yet (ask the user when reached): domain name,
   Cloudflare account setup, Supabase project credentials, Slack app creation,
   GitHub repo names for vault backup and code.

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).

## License

Source-available under the **PolyForm Noncommercial License 1.0.0** ([LICENSE.md](LICENSE.md)):
free for any noncommercial purpose, attribution required (keep the `Required Notice:` line).
**Commercial use requires a separate paid license** — see [COMMERCIAL.md](COMMERCIAL.md).
