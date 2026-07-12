# Web App (PWA)

**Version:** 1.0 · **Status:** Approved 2026-07-12
**Stack:** React + Vite + TypeScript, installable PWA, served statically by Caddy,
consumes only [03-api.md](03-api.md). Strictly decoupled from the server
([ADR-006](adr/006-monorepo-with-strict-server-web-decoupling.md)).

## Design mandate (explicit product requirement)

This is *not* an admin panel. It is "my memory as an app": **premium, beautiful,
wow-factor**. Fluid animation is a first-class feature.

- **Motion:** framer-motion throughout — page transitions, staggered feed entries, a
  living/breathing recording visualizer (waveform/orb reacting to voice), streaming-text
  reveal in chat, springy micro-interactions on every touch target. Animations must stay
  60fps on mid-range phones — transform/opacity only, no layout thrash.
- **Look:** dark-first theme, deep background with subtle gradient/glow accents, glass
  surfaces, one strong accent color; large expressive type for the capture screen; light
  theme secondary.
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

### 2. Chat
- Conversation list + thread view; streaming answer rendering.
- **Model picker per conversation** (`GET /chat/models`) — compact selector in the
  composer; when the response's `fallback_used` is true, show a discreet banner
  "answered by <model>".
- Source citations `[n]` rendered as expandable cards (note title, plane badge, snippet).
- Plane filter chips (optional query scoping).

### 3. Activity
- The "what did my brain do" feed (`GET /activity`): agent runs, captures, errors —
  high-level entries with staggered entrance animations; tap to expand per-note details
  (`GET /activity/runs/{id}`). Fallback events visibly badged.

### 4. Settings
- **Agents section** (separate from chat, by decision): conspect model + fallback model
  (`PUT /settings/agents`), connector list with last-run status and "run now".
- Session management (logout), theme, reduced motion override.

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
