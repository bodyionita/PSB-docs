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
- **"Remember this" (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md)):**
  per-session action triggering immediate distillation (`POST /chat/sessions/{id}/remember`).

### 3. Activity (categorized tabs — full restructure at M8)
- "What did my brain do", as **multiple categorized tabs** (agents/jobs · conversations ·
  manual actions), recording automatic **and** manual events; staggered entrance animations;
  tap to expand run details (`GET /activity/runs/{id}`). Fallback events visibly badged.
- **Auto-ingested stance items** ([ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md))
  carry a flag + one-tap "that's wrong — remove".
- **Ops console (M8):** every job listed by category with schedule + next run (`GET /agents`),
  a manual **Run** button (no cron-only ghosts), and — while running — **live status + log
  tail** (`GET /activity/runs/{id}/logs`).

### 3b. Review (M6, [ADR-029](adr/029-conversational-ingestion-stance-gate-review-queue.md))
- Badge-counted queue of stance-unclear candidates from all conversational sources, grouped by
  conversation: proposed memory + short excerpt + **agree / disagree / maybe**. Agree ingests
  through the organizer; disagree discards (logged); maybe stays (no expiry).

### 3c. The Map (M7 — desktop-first)
- **Neighborhood explorer** over `GET /nodes/{id}/neighbors` (the same service as MCP
  `traverse`): enter from a search hit or node list; the node renders centered with edges one
  hop out; **click to expand**. Node shape/icon = type, color = plane; solid edges = canonical
  (labeled by `rel`), faint edges = derived similarity. Breadcrumb trail back.
- Phone: degrades to the same node-with-edges as a tappable list (no canvas).
- Growth path (post-M7): curated overviews — a custom-designed "world/continents" view (to be
  designed together) — then an aerial whole-graph mode only if a performant, genuinely pleasant
  rendering is found.

### 4. Settings
- **Vocabulary (M3, [ADR-027](adr/027-typed-vocabulary-governance.md)):** node + edge type
  vocabularies with pending LLM proposals — approve/reject; approval queues the
  retro-consolidation job.
- **Connectors (M9):** per-connector config incl. the lookback override (default 6 months).
- **Models section (M4, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md) +
  [ADR-043](adr/043-quick-routing-tier-m4.md)):** **three** routing groups — **Chat**, **Conspect**,
  and **Quick** (trivial tasks, e.g. session titling; default Sonnet-low / Nebius) — each an
  active-model dropdown + fallback dropdown + an **effort selector shown only for models that support
  it** (Claude yes, Nebius no). Choices + effort levels come from `GET /settings` (registry-sourced,
  never hardcoded); saved via `PUT /settings/models`. This is where the M0/M4 model-and-effort control
  lives; the chat composer picker is a per-conversation override of the Chat group's active model.
- **Providers status (M4 follow-up, [ADR-044](adr/044-provider-observability-surface.md)):** a
  **read-only** card over `GET /admin/providers` — one row per provider with a status dot (**green** =
  `consecutive_failures == 0`, **amber** = `> 0`) + the sticky `last_error` line. Surfaces *why* a
  provider last fell back (the M4 silent-fallback gap, vision P8). No actions, no editing — a thin
  TanStack-Query read (ADR-006).
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

### 6. Admin (M2; absorbed into the M8 ops console)
A lightweight operations panel with a few buttons, each showing live run status:
- **Reindex** (`POST /admin/reindex`) — async store rescan + edge recompute; shows the run's
  live counts (`indexed/skipped/deleted/failed`); single-flight (guarded against overlap).
- **Backup now** (`POST /admin/backup`) — force an immediate store commit + push.
- **Consolidate tags** (`POST /admin/tags/consolidate`) — two-step tag cleanup: **Propose** →
  review the merge plan → **Apply** ([ADR-024](adr/024-tag-vocabulary-reuse-and-consolidation.md)).

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
