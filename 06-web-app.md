# Web App (PWA)

**Version:** 1.2 · **Status:** Approved 2026-07-13 (1.2 = **M3 chat screen finalized** (reveal render,
per-conversation picker, cited-only source cards, plane chips, "not in notes") + **Settings → Models**
routing panel — [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md); 1.1 = M2 adds the
Search + Admin screens —
[ADR-022](adr/022-embeddings-self-hosted-nomic.md)/[023](adr/023-semantic-relatedness-graph.md)/[024](adr/024-tag-vocabulary-reuse-and-consolidation.md))
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

### 2. Chat (M3, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md))
- Conversation list (newest first, `GET /chat/sessions`) + thread view. **List / open / new**
  only in M3 — rename + delete are deferred (no endpoints yet).
- Answer rendering is a **client-side reveal animation** over the full non-streaming response
  (true token streaming is post-v1) — respects `prefers-reduced-motion`.
- **Model picker per conversation** (`GET /chat/models`, real ids + labels) — compact selector in
  the composer; overrides only the *active* model for this thread (fallback + effort inherited from
  the Settings **Chat** group). When the response's `fallback_used` is true, show a discreet banner
  "answered by <model>".
- Source citations `[n]` rendered as **expandable cards** — only the **cited** notes
  (`sources[]`), title + plane badge + snippet; tap to expand.
- **Plane-filter chips** in the composer (optional retrieval scoping → `POST /chat` `planes`).
- **"Not in your notes"** answers render plainly, no source cards.

### 3. Activity
- The "what did my brain do" feed (`GET /activity`): agent runs, captures, errors —
  high-level entries with staggered entrance animations; tap to expand per-note details
  (`GET /activity/runs/{id}`). Fallback events visibly badged.

### 4. Settings
- **Models section (M3, [ADR-025](adr/025-ui-editable-model-routing-and-per-task-effort.md)):** two
  routing groups — **Chat** and **Conspect** — each an active-model dropdown + fallback dropdown +
  an **effort selector shown only for models that support it** (Claude yes, Nebius no). Choices +
  effort levels come from `GET /settings` (registry-sourced, never hardcoded); saved via
  `PUT /settings/models`. This is where the M0/M3 model-and-effort control lives; the chat composer
  picker is a per-conversation override of the Chat group's active model.
- **Agents (M4):** connector list with last-run status and "run now". (The conspect model that was
  drafted as `PUT /settings/agents` is now the **Conspect** group above — ADR-025.)
- Session management (logout), theme, reduced motion override.

### 5. Search (M2)
Standalone semantic-search screen over the whole vault (`POST /search`, no LLM call):
- Query box + **plane-filter chips** (scopes on `notes.planes` membership).
- Results as **note cards** — title, plane badge, snippet (the best-matching chunk), tags,
  score — ranked by relevance.
- **Expand a card → read-only note preview** (`GET /notes/{id}`): the note body read live from
  the vault, plus its **semantic neighbours** from the `note_links` relatedness graph
  ([ADR-023](adr/023-semantic-relatedness-graph.md)). No in-app editing (Obsidian/git covers that).

### 6. Admin (M2)
A lightweight operations panel (movable later) with a few buttons, each showing live run status:
- **Reindex** (`POST /admin/reindex`) — async vault rescan + relatedness recompute; shows the
  run's live counts (`indexed/skipped/deleted/failed`); single-flight (guarded against overlap).
- **Backup now** (`POST /admin/backup`) — force an immediate vault commit + push.
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
