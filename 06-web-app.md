# Web App (PWA)

**Version:** 2.0 · **Status:** Approved 2026-07-13 (2.0 = **mind-graph pivot**
[ADR-026](adr/026-graph-native-storage-obsidian-removed.md)–[029](adr/029-conversational-ingestion-stance-gate-review-queue.md):
nodes/graph vocabulary; new **Map** (M7) + **Review** (M6) screens; Activity becomes categorized
tabs + the **ops console** (M8); chat = M4. The PWA is one of the thin surfaces over the service
layer — its exclusives are the human affordances: voice (Romanian-capable STT), the map, review,
settings [ADR-028](adr/028-one-service-layer-mcp-peer-surface.md). 1.x history in git.)
**Stack:** React + Vite + TypeScript, installable PWA, served statically by Caddy on the
VPS at **single origin** (`/` = web, `/api` = API — [ADR-013](adr/013-web-stays-on-vps-single-origin.md)),
consumes only [03-api.md](03-api.md). Strictly decoupled from the server
([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)).

## Design mandate (explicit product requirement)

This is *not* an admin panel. It is "my memory as an app": **premium, beautiful,
wow-factor**. Fluid animation is a first-class feature.

**Product name:** **Braindan** (Bogdan + brain). Used in the PWA manifest, page title, and
login header; kept as a single config constant so it's changeable at zero cost.

- **Motion:** framer-motion throughout — page transitions, staggered feed entries, a
  living/breathing recording visualizer (waveform/orb reacting to voice), streaming-text
  reveal in chat, springy micro-interactions on every touch target. Animations must stay
  60fps on mid-range phones — transform/opacity only, no layout thrash.
- **Look:** dark-first, deep background with subtle gradient/glow accents, glass surfaces,
  one strong accent color per theme; large expressive type for the capture screen.
- **Themes (switchable, remembered):** a theme switcher offers **5 palettes**; the choice
  persists in `localStorage` (M0) and may later sync to `app_settings`. Default **Nebula**.

  | # | Theme | Base | Accent |
  |---|---|---|---|
  | 1 | **Nebula** (default) | near-black, faint violet haze | violet→indigo `#7C5CFF` |
  | 2 | **Aurora** | dark teal-slate | cyan/emerald `#2DD4BF` |
  | 3 | **Ember** | warm charcoal | amber/gold `#F5B041` |
  | 4 | **Rose Quartz** | dark plum | magenta/pink `#FF5C8A` |
  | 5 | **Daylight** | off-white | indigo `#5B5BD6` (the light option) |

  Themes are pure token swaps (CSS custom properties); no component knows a hard-coded
  color. `prefers-reduced-motion` is an independent concern from theme.
- **Feel:** capture is the hero action — reachable in one tap from anywhere, oversized
  record button, satisfying confirmation animation when a capture lands.
- Respect `prefers-reduced-motion`.

## Screens

### 1. Capture (home)
- Giant record button → hold-or-tap to record (MediaRecorder, m4a/webm) → upload to
  `POST /capture/voice` → optimistic "captured ✓" animation; transcript appears in the
  feed when the pipeline finishes.
- Text input for quick typed capture (`POST /capture/text`).
- Recent captures strip with live pipeline status (received → transcribing → … → indexed),
  animated state transitions; failed items expose retry.

### 2. Chat (M4, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))
- Conversation list (newest first, `GET /chat/sessions`) + thread view. **List / open / new**
  only in M4 — rename + delete are deferred (no endpoints yet).
- Answer rendering is a **client-side reveal animation** over the full non-streaming response
  (true token streaming is post-v1) — respects `prefers-reduced-motion`.
- **Model picker per conversation** (`GET /chat/models`, real ids + labels) — compact selector in
  the composer; overrides only the *active* model for this thread (fallback + effort inherited from
  the Settings **Chat** group). When the response's `fallback_used` is true, show a discreet banner
  "answered by <model>".
- Source citations `[n]` rendered as **expandable cards** — only the **cited** nodes
  (`sources[]`), title + type icon + plane badge + snippet; tap to expand.
- **Plane-filter chips** in the composer (optional retrieval scoping → `POST /chat` `planes`).
- **Grounding indicator:** an answer with **no source cards** (empty `sources[]` — both the
  general-knowledge and "not in your memories" cases) carries a subtle **"not from your memories"
  chip** so ungrounded replies are always visibly distinct from cited ones (M4 kickoff grill). This
  is orthogonal to the `fallback_used` model banner above.
- **Session titles** are **LLM-generated** on the `quick`/Sonnet tier ([ADR-043](adr/043-quick-routing-tier-m4.md)),
  best-effort + non-blocking after the first exchange; the list shows the title (or the first
  message until the title lands).
- **"Remember now" (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)/[048](adr/048-m6-chat-distiller-build-decisions.md) §6):**
  per-session action → `POST /chat/sessions/{id}/remember`, showing the returned
  `{ endorsed, to_review }` (or "nothing new to record") summary; the organize runs in the background.
- **"Recently auto-recorded" list (M6, [ADR-048](adr/048-m6-chat-distiller-build-decisions.md) §12):**
  a **chat-scoped** audit list (`GET /chat/auto-recorded`) of memories the nightly distiller
  auto-endorsed — node preview + salience + **one-tap "that's wrong — remove"**
  (`POST /chat/auto-recorded/{capture_id}/remove`, soft-delete). This is M6's home for ADR-029 §6's
  reversal loop; M8's general Activity feed absorbs it later (Activity stays the M8 placeholder in M6).

### 3. Activity (full restructure at M8 — [ADR-053](adr/053-m8-ops-console-observability-build-decisions.md))
One top-level **Activity** tab with a **Feed / Ops segmented control** (no new top-level tab — there
are already 8; poll cadence `refetchInterval` is active **only while a run is live**):

**Feed** — "what did my brain do", as **3 categorized tabs** over the merged `GET /activity`
(agents/jobs · conversations · manual actions), automatic **and** manual events, newest-first
infinite scroll (`before=` keyset); staggered entrance animations; tap to expand run details
(`GET /activity/runs/{id}`). Fallback events visibly badged.
- **Pipeline runs (M5.5, [ADR-047](adr/047-pipeline-scheduling-primitive.md)):** a pipeline run is a
  parent entry expandable to its per-step child runs.
- **Conversations tab** absorbs the M6 **"recently auto-recorded"** list — auto-ingested stance
  items carry a flag + **one-tap remove** inline (was chat-scoped in M6).

**Ops** — the console (invariant 4; the M2 Admin panel is absorbed here):
- **Pipelines** (`GET /pipelines`): each with schedule + next-run + its ordered steps + a
  whole-pipeline **Run** (`POST /pipelines/{name}/run`).
- **Agent roster** (`GET /agents`) in **nightly-pipeline order**: per-job last-run status + a bare
  **Run** button (`POST /agents/{name}/run`) for every zero-arg job (all steps + `graph-health` +
  reindex/backups); no cron-only ghosts.
- **Live log tail** — while a run is active, its **status + streaming-by-poll log tail**
  (`GET /activity/runs/{id}/logs?after_seq=`, ~1s poll).
- **graph-health card** — the latest `graph-health` run's findings (orphans, `inbox/` depth, review
  aging, missing `occurred`, alias-less entities, tombstone integrity, freshness) read from its run
  `details`; **read-only** in M8 (acting on flags → M10).
- **Parameterized admin ops** — `reindex` (live counts), `backup now`, `reprocess` (confirm-gated),
  `entities/merge` (pick two nodes), `tags`/`vocab` consolidate (two-step propose→apply): rehomed
  here with their **input controls** (they can't collapse to a bare Run), each showing live run
  status.

### 3b. Review (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)/[048](adr/048-m6-chat-distiller-build-decisions.md) §8)
- Badge-counted queue, **salience-ordered** (high first), of items from all conversational sources:
  - **stance-candidate** — proposed memory + short excerpt + **agree / disagree / maybe**. Agree
    ingests through the organizer (the auto-endorse path); disagree discards (logged); **maybe stays
    and is re-openable** (no expiry) with an **aging** indicator.
  - **dedup-proposal** — two node previews + **merge / keep / link** (survivor pick on merge).
- **Batch actions:** multi-select → agree/disagree/maybe (or keep/dismiss) in one call
  (`POST /review/batch`). A weekly **maybe digest** keeps the parked pile from stalling silently.

### 3c. The Map (M7 — [ADR-051](adr/051-m7-map-build-decisions.md); desktop-first, canvas on phone too)
- **Neighborhood explorer** over `GET /nodes/{id}/neighbors` (the same `GraphService.neighbors`
  as MCP `traverse`), a new **full-width top-level tab** (opts out of the shell's 640px column).
  Built on **`react-force-graph-2d`** (2D canvas only — ADR-032 #12).
- **Re-center navigation (not accumulate).** Exactly **one focal node at a time**: it renders
  pinned at center with its 1-hop neighbors fanned around it in **rel-based zones** (per-zone
  directional forces over a live d3-force sim — "people one side, topics another"). **Single-click
  a neighbor = re-center** on it (plex-style animated re-center: reheat, old fades out, new fans
  in). A **breadcrumb strip** records the path; a crumb re-centers back + truncates forward. The
  client only ever holds one node's neighborhood — **never a whole-graph client layout**.
- **Node encoding:** **emoji glyph = type** (reuses `ui/nodeTypes`), **ring + larger size = entity
  hub** vs plain disc = content, **halo color = primary plane** (a new theme-independent plane
  palette; the theme accent is reserved for the focal node). **Edges:** canonical = solid +
  subtle **arrowhead** + `rel` label on hover/zoom; derived `similar` = faint (no arrowhead);
  **superseded (`until`-closed) = dashed + dimmed**, `until` on hover.
- **Read vs move:** **hover** peeks (label/tooltip, no movement); a compact caption chip on the
  focal node (title · type · plane · tags); **clicking the focal node** opens the shared
  `ui/NodePreview` drawer (full body + edges, rule 10). Per-zone **"show more"** pages a hub's
  overflow (`?rel=` cursor) without refetching the neighborhood.
- **Entry:** from a **Search card** or a **`NodePreview` edge row** (sets a `mapSeed` + switches
  tab). **Empty state** (no seed): an embedded search to start + **restore the last-centered node**
  (`localStorage`).
- **Phone:** the **canvas is primary on phone too** (tap = re-center, tap center = drawer) — 2D
  canvas is phone-safe and node count is capped by the re-center model. This **supersedes** the
  earlier "phone = list, no canvas" plan. A **tappable-list renderer** (same `(origin, rel)` zones)
  is retained as the **`prefers-reduced-motion` fallback** + a manual view toggle.
- Growth path (post-M7): **auto-center on your top hubs** as the entry (needs a new top-degree
  endpoint — [ADR-051](adr/051-m7-map-build-decisions.md) backlog) · curated "world/continents"
  overviews · aerial whole-graph mode only if a performant, genuinely pleasant rendering is found.

### 4. Settings
- **Vocabulary (M3, [ADR-027](adr/027-typed-vocabulary-governance.md)):** node + edge type
  vocabularies with pending LLM proposals — approve/reject; approval queues the
  retro-consolidation job.
- **Connectors (M9):** per-connector config incl. the lookback override (default 6 months).
- **Models section (M4, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md) +
  [ADR-043](adr/043-quick-routing-tier-m4.md), [ADR-045](adr/045-provider-model-effort-separation.md)):**
  **three** routing groups — **Chat**, **Conspect**, and **Quick** (trivial tasks, e.g. session titling;
  default Sonnet-low / Nebius) — each an active-**model** dropdown + fallback dropdown + an **effort
  selector shown only for models that support it** (Claude yes, Nebius no). The dropdowns pick a **model**
  (ADR-045 — "Claude Opus 4.8" / "Claude Sonnet 4.6" / "Llama 3.3 70B"; the *provider* is derived, so both
  Claude models sit under the one `claude` provider and no `claude-max` id is shown); effort is keyed by
  model (`effort_by_model`). Choices + effort levels come from `GET /settings` (registry-sourced, never
  hardcoded); saved via `PUT /settings/models`. This is where the M0/M4 model-and-effort control lives;
  the chat composer picker is a per-conversation override of the Chat group's active model.
- **Providers status (M4 follow-up, [ADR-044](adr/044-provider-observability-surface.md);
  [ADR-045](adr/045-provider-model-effort-separation.md)):** a **read-only** card over
  `GET /admin/providers` — **one row per provider** (5: Claude, Nebius, Groq, OpenAI, Ollama) labeled by
  **friendly provider name** (ADR-045 — `claude` appears **once**, serving Opus+Sonnet; no raw id, no
  per-model breakdown) with a status dot (**green** = `consecutive_failures == 0`, **amber** = `> 0`) +
  the sticky `last_error` line. Surfaces *why* a provider last fell back (the M4 silent-fallback gap,
  vision P8). No actions, no editing — a thin TanStack-Query read (ADR-006).
- **Jobs & connectors:** the roster (last-run status, "run now", schedules) lives in the **M8 ops
  console** (screen 3); per-connector *config* lands here at M9. (The conspect model that was
  drafted as `PUT /settings/agents` is now the **Conspect** group above — ADR-025.)
- Session management (logout), theme, reduced motion override.

### 5. Search (M2; retargeted to nodes at M3)
Standalone semantic-search screen over the whole graph (`POST /search`, no LLM call):
- Query box + **plane-filter chips** (+ type filter, M3).
- Results as **node cards** — title, type icon, plane badge, snippet (best-matching chunk),
  tags, score — ranked by relevance.
- **Expand a card → read-only node preview** (`GET /nodes/{id}`): body read live from the graph
  store, plus its **edges** (canonical, labeled by `rel`, and derived similarity) — each a
  jump-off point (and the entry into the Map from M7). No in-app editing (git covers that).

### 6. Admin (M2 panel — **absorbed into the M8 ops console**, screen 3 › Ops)
The M2 lightweight operations panel (Reindex `POST /admin/reindex` with live
`indexed/skipped/deleted/failed` counts; Backup now `POST /admin/backup`; Consolidate tags
`POST /admin/tags/consolidate` two-step propose→apply, [ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md))
is **no longer a standalone screen at M8** — these parameterized ops are rehomed, with their input
controls, into **Activity › Ops** (screen 3), alongside `reprocess`, `entities/merge`, and
`vocab/consolidate` ([ADR-053](adr/053-m8-ops-console-observability-build-decisions.md) §8).

## Auth UX
Password screen (single field, biometric-friendly via password manager autofill) →
long-lived session cookie; re-auth only on expiry/revocation. Rate-limit feedback on
failed attempts.

## PWA specifics
- Installable (manifest + icons + splash), standalone display.
- Offline: app shell cached; capture screen queues text captures offline and syncs when
  back online (voice offline-queueing is v2).
- No service-worker caching of API responses beyond the models/settings bootstrap.

## Engineering rules
- TypeScript strict; API types generated from OpenAPI or hand-kept in one `api/` module —
  the only place that knows URLs.
- State: TanStack Query for server state; no global store unless proven needed.
- Component structure by feature (`features/capture`, `features/chat`, …), shared
  design-system primitives in `ui/`.
