# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

## Current status (snapshot — 2026-07-19)

> This section is a **snapshot, updated in place** each session. The full session-by-session
> history lives in **[08-logs/status-history.md](08-logs/status-history.md)** (append-only —
> a superseded snapshot entry moves there verbatim). The authoritative build state is
> **[08-implementation-plan.md](08-implementation-plan.md)** + **[08-logs/](08-logs/)**.

**Live at `https://braindan.cc` — M0 through M8.2 all closed.** The graph-native stack
([ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–029 pivot, M3): capture
(voice/text/MCP) → organizer (single writer; typed nodes + edges, entity resolution, alias
accretion, diacritic folding) → `PSB-graph` git store → index/hybrid search (vector ⊍ FTS RRF +
recency + profile leg). Grounded cited **chat** with UI-editable model routing
(`chat`/`conspect`/`quick`; provider ≠ model, [ADR-045](adr/045-provider-model-effort-separation.md))
+ provider observability (M4). **MCP server behind self-hosted OAuth 2.1** — connector-verified
on Claude (mobile/web) *and* ChatGPT (M5); identity capsule L0. **Pipeline scheduling primitive**
(M5.5, [ADR-047](adr/047-pipeline-scheduling-primitive.md)). **Chat distiller** with stance gate,
watermarks, salience, dedup sweep + merge core, inbox drainer, one-tap remove (M6,
[ADR-048](adr/048-m6-chat-distiller-build-decisions.md)/[049](adr/049-dedup-sweep-merge-core-build-decisions.md)).
**Map/Explore** constellation UI (M7/M8.1), **ops console + observability** (M8,
[ADR-053](adr/053-m8-ops-console-observability-build-decisions.md)), **graph-health**.
**Interiority + temporal correctness** (M8.2, [ADR-055](adr/055-interiority-inner-voice-first-class.md)/[056](adr/056-temporal-correctness-date-tokens.md)):
"LLMs classify, code computes" (hard rule 12), anchored symbolic time-refs → `[[t:…]]` tokens,
inner-voice extraction; prod reprocessed (41/41 captures, 160 nodes). Durability: everything
derived rebuilds from the store (`reprocess-all-from-raw`, vision P10,
[ADR-042](adr/042-reprocess-all-from-raw-and-data-survival.md)); reindex parity verified live.

**Where we are (2026-07-19):** **M9.7 + M9.6 T6 CLOSED — deployed, live Accept all verified.**
Batch A/B (`e91d1cb`…`f83268e`) shipped chat-screenshot self-attribution (organizer v9), live
per-part capture observability (M8 run-log tail + `derive.parts[]`), and general capture remove
(`DELETE /captures/{id}`, double-confirm UI). T7's live drill surfaced a live-ops issue —
**Groq decommissioned the `llama-4-scout-17b` vision primary (404 mid-drill)** — fixed by
**[ADR-063](adr/063-groq-vision-model-scout-decommissioned.md)**: vision primary → **`qwen/qwen3.6-27b`**
(Groq's current free VLM), with **`reasoning_effort=none`** (Groq-scoped `extra_body`) to keep the
Qwen3 thinking out of the derived text and stop long-screenshot truncation. **T7 Accept verified:**
own-chat attribution correct under v9 (graph-store + MCP); live processing view (the run-log even
caught the 404 in-flight); **remove** proven via Supabase (all removed captures `removed_at` set,
`leftover_media=0`, hubs preserved, `reprocess` replays `WHERE removed_at IS NULL`); migration
confirmed. Gate green throughout (1039 pytest).

**M9.8 GRILLED TO BUILD-READY ([ADR-064](adr/064-durable-merges-visual-dedup-gc.md)).** The T7 wrap
investigated duplicate `Diana`/`Diana Vance` person hubs and found: manual entity merges **don't
survive `reprocess-all`** (keyed on node id; the 07-17 rebuild silently dropped the merge), the merge
UI is **unusable** (paste two raw UUIDs), there's **no entity-hub dedup** (sweep is content-only), and
**graph-health is read-only**. Grilled to a plan (ADR-064): **durable merges** (keyed on surface-form
+ type, replayed by reprocess), a **shared visual name-typeahead picker** (no ids), **inline-actionable
graph-health** (merge/delete/keep), a **conservative entity-hub dedup detector** (suppresses the
genuinely-different `Diana Wren`), and **manual orphan GC**. Better entity-resolution-at-ingest
deferred. *(`Diana` stays duplicated until M9.8 ships — the user declined the id-paste stopgap.)*

**M9.8 T1 LANDED (server foundation, pushed).** **Durable, replayable merges** ([ADR-064](adr/064-durable-merges-visual-dedup-gc.md) §1):
new `entity_merges` table (migration 021) records each merge keyed on **surface form + type** (not
node id); merge apply upserts the decision, and `reprocess-all` re-applies them after the raw rebuild
(new `MergeReplayService`, wired into both the in-app `ReprocessService` and the CLI) — matched by
surface form, title-form ranked first so a survivor/loser sharing a short alias never cross;
unresolvable/ambiguous → skipped (never-lose). Fixes ADR-042 §4's silent drop (the Diana merge will
now survive a reprocess). Gate green (1049 pytest, +10). Docs updated (02 §3, 03 §Admin, 08 T1).

**Next:** continue **M9.8** server foundation — **T4** (conservative entity-hub dedup detector →
inline feed + Review for low-confidence; suppresses Diana Wren) and **T5** (node-delete path for
orphan hubs); then the shared picker + merge surfaces (T2/T3), then inline-actionable graph-health
(T6); T7 live Accept = merge Diana by name and confirm it survives a `reprocess-all`. *(Separate
background task in flight: the identity-capsule L0 generator-preamble leak.)*

> **Planning/replanning sessions start with `/grilling`; implementation sessions build
> against the approved plan (no grilling). Every session follows
> [09-session-protocol.md](09-session-protocol.md).**

## Reading order

| Doc | Contents |
|---|---|
| [00-vision.md](00-vision.md) | Why the system exists, principles, planes, non-goals, success criteria |
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA + MCP surfaces, VPS service, agents, graph store |
| [02-data-model.md](02-data-model.md) | Graph-store layout, typed-node frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the web↔server seam) + the MCP tool surface |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, chat-distillation, indexing, chat, agent pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract (conversation substrate, stance gate), Instagram + Slack specs |
| [06-web-app.md](06-web-app.md) | PWA screens, design language (premium, animated), auth UX |
| [07-infrastructure.md](07-infrastructure.md) | VPS, Docker Compose, Caddy, Cloudflare, CI/CD, secrets, backups |
| [08-implementation-plan.md](08-implementation-plan.md) | Phased delivery: per-milestone scope, acceptance, build decisions + a **task tracker** (done/open) |
| [08-logs/](08-logs/) | Per-milestone **implementation logs** + the archived **[status history](08-logs/status-history.md)** |
| [09-session-protocol.md](09-session-protocol.md) | How every session runs: grill → record → pause → respawn-friendly commits |
| [adr/](adr/) | Architecture Decision Records — the *why* behind every choice |

## Development workspace layout (local machine)

```
PersonalSecondBrain/            # workspace folder, not a repo
├── second-brain-docs/          # THIS repo — documentation (public GitHub — no personal data!)
├── second-brain/               # code monorepo: server/ + web/ + deploy/ + tools/
├── PSB-graph/                  # local clone of the (private) production graph-store repo
├── instagram-…/                # the user's unzipped Instagram data export (local only,
│                               # NEVER committed anywhere — M9.5 prep-tool input)
└── _archive/                   # pre-pivot artifacts (ADR-026): ObisidanVault/ (dead dev
                                # scratch) + PSB-vault/ (pre-pivot vault repo — ARCHIVED
                                # 2026-07-14 at the M3 close; see _archive/README.md)
```

The production graph store lives on the VPS ([ADR-001](adr/001-vault-on-vps-with-git-backup.md) +
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)).

## Cold start — instructions for a fresh implementation session

If you are an AI (or human) picking this up with no prior context:

0. Follow the [session protocol](09-session-protocol.md): **planning/replanning** sessions
   `/grilling` first, record decisions to docs, and **pause before implementation** so the
   user can continue or respawn; **implementation** sessions build against the approved plan
   (no grilling), pausing between tasks. Commit + push docs at every pause.
1. Read the docs in the order above; skim every ADR — they are binding.
2. The code monorepo lives at `../second-brain/` (server/, web/, deploy/, tools/;
   its `CLAUDE.md` carries the binding implementation rules).
3. Implement strictly by phases in [08-implementation-plan.md](08-implementation-plan.md),
   starting at the first milestone whose acceptance criteria don't pass yet. Do not skip ahead.
4. Anything ambiguous or contradictory: fix the docs first (new ADR if architectural),
   then implement. Never silently diverge from these documents.
5. Things intentionally NOT decided yet (ask the user when reached): Slack app creation (M12);
   exact vision model ids (M9 T1 — verified against live Groq/Nebius catalogs at build).
   Already provisioned: domain (`braindan.cc`), Cloudflare, Supabase, code repo, `PSB-graph`
   (+ VPS deploy key, write access).
6. Update the **Current status** snapshot above **in place** at every pause; move the
   superseded entry verbatim to [08-logs/status-history.md](08-logs/status-history.md)
   (append-only). Never let state live only in chat.

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).
- **This repo is public: no real names, no relationship details, no personal data** — use
  placeholders (P1/P2) in logs; the triage manifest and export data stay local, gitignored. A
  **pre-commit guard** (`.githooks/pre-commit` → `pii_scan.py`, a SHA-256 denylist) blocks any real
  contact's full name; wire it on a fresh clone with `git config core.hooksPath .githooks`. Example
  people in docs/code use fabricated names (Diana Vance, Horia Fenwick, …).

## License

Source-available under the **PolyForm Noncommercial License 1.0.0** ([LICENSE.md](LICENSE.md)):
free for any noncommercial purpose, attribution required (keep the `Required Notice:` line).
**Commercial use requires a separate paid license** — see [COMMERCIAL.md](COMMERCIAL.md).
