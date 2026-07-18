# M9 T6 — live Accept runbook (operator + verification)

**Status:** prepared 2026-07-18 (implementation session). **Tooling ready; live steps pending.**
This is the executable procedure for **T6 — live M9 Accept** ([08 §M9](../08-implementation-plan.md),
[ADR-060](../adr/060-node-media-linkage-and-voice-unification.md), [ADR-057](../adr/057-multimodal-media-ingestion-substrate.md)).
T6 is a **live** milestone — it needs a prod deploy, real-device captures, and a running DB. The
gated steps (push, deploy, phone captures) are the **operator's** to perform; each drill lists the
PASS check to verify after.

> **Public-repo discipline** ([docs are public](../README.md#rules-of-this-repo)): when recording
> results, redact node titles / capture text / real names to placeholders (P1/P2). Paste **shapes
> and counts**, never raw personal content.

## 0. Prereqs / what's already prepared

- **Code:** M9 T1–T5 are committed locally (through `4adab51`) and **not yet pushed** (7 commits).
  This session added the **`rederive-capture <id>` CLI verb** (the drill's live re-derive trigger —
  no HTTP endpoint exists until M9.5's `POST /admin/connector/rederive`) + the **media-join SQL
  smoke** (`deploy/smoke/m9_media_join_smoke.sql`). These ride the same push.
- **Deploy mechanism:** push to `main` → CI (`.github/workflows/ci.yml`) runs server tests + web
  build, then the deploy job SSHes the VPS: `git pull` → `docker compose up -d --build` →
  **`alembic upgrade head`** (this is what applies **migrations 017 + 018** automatically). The
  **backfill op is NOT automatic** — run it by hand (step 2).
- **Operator command shape** (on the VPS, in `/srv/app`):
  ```
  docker compose --env-file deploy/defaults.env -f deploy/docker-compose.yml exec -T api \
    uv run python -m app.cli <verb>
  ```
- **DB** is external (Supabase — no `db` service in compose). Run the SQL smoke in the **Supabase
  SQL editor**, or `docker compose … exec -T api sh -c 'psql "$DATABASE_URL" -f -' < deploy/smoke/m9_media_join_smoke.sql`.

## 1. Deploy (migrations 017 + 018 apply)

1. **[operator — user's call]** Push the code: `git push origin main`.
2. Watch CI green (server + web + deploy jobs). The deploy job's last step runs `alembic upgrade
   head`.
3. **Verify migrations applied:** `… exec -T api uv run alembic current` shows **`018`** at head.
   Or SQL smoke **block 1** (both `media` + `node_media` tables + indexes present).

## 2. Backfill op (legacy voice → media)

1. **[operator]** `… exec -T api uv run python -m app.cli voice-media-backfill`
2. **PASS:** the command's `agent_runs` summary reads `voice-media-backfill: N legacy voice
   capture(s) — R relocated, D degraded (audio missing), L node-linked`. Idempotent — a second run
   reports 0 relocated (already done).
3. **Verify:** SQL smoke **block 9** → `unbackfilled_voice_captures = 0`; legacy voice nodes now
   carry a playable audio row.

## 3. Real-phone photo → node, inline (Accept ¶1)

1. **[operator, on the phone PWA]** Capture screen → photo affordance → pick a **real photo** (an
   iPhone HEIC exercises the client-side HEIC→JPEG path; a jpg/png skips it).
2. **PASS — capture surface:** the Recent-captures row shows the photo thumbnail + a status label
   walking `deriving` → done.
3. **PASS — node inline:** open the produced node's `NodePreview` → the **media strip** renders the
   photo thumbnail between header and body; **tap → lightbox** (zoom, swipe/Esc dismiss).
4. **PASS — traceability:** the strip's "see raw capture" opens the **capture detail sheet** (source
   badge, status, the photo, the fenced `<photo: …>` raw text, NodeChips to every node it produced).
5. **Verify (server):** `GET /nodes/{id}` carries `media:[{id,kind:"image",status:"derived",capture_id}]`;
   SQL smoke **blocks 6–7** list the node with `media_kinds = {image}`.
6. **Session gate:** `GET /media/{id}` **without** a session cookie → 401/redirect; with the session
   → the image bytes (never public).

## 4. Voice → playable on its node + capture, Range/206 scrub (Accept ¶2)

1. **[operator, on the phone]** Record a **voice capture**.
2. **PASS:** the node's media strip + the capture surface both show the **themed voice player**;
   it plays. **Scrub** the audio mid-clip — seeking works (Starlette `FileResponse` serves
   **Range/206**; confirm in devtools Network the `GET /media/{id}` audio request returns **206
   Partial Content** with a `Content-Range` header on a scrub).
3. **Verify:** `GET /nodes/{id}.media[]` has `kind:"voice"`; the transcript is the node body text.

## 5. Chat-screenshot attribution (Accept ¶3, ADR-057 §5)

1. **[operator]** Capture a **screenshot of a chat** (messages from other people).
2. **PASS:** the derived description attributes the contained messages to the **screenshot's
   internal speakers**, never to the user — the produced node(s) do **not** record the user as
   having said the other party's words. The two-layer rule: content inside a `<photo: …>` fence is
   shared material, never the capturer's utterance.
3. **Verify:** inspect the node(s) — no first-person mis-attribution of the depicted conversation.

## 6. Vision group edit forward-live + Claude-route warning (Accept ¶4)

1. **[operator]** Settings → Models → **Vision** group is present (Groq-seeded). Edit the primary
   model, save.
2. **PASS — forward-live:** the **next** photo capture derives via the edited model (check the media
   row's `model_used` / the derivation `agent_runs`), no redeploy.
3. **PASS — warning:** set the active/fallback vision model to a **`claude`-provider** model → the
   inline **Claude-route warning** appears (no vision path — images would silently drop, ADR-057 §4).
   Revert after.

## 7. Failure → placeholder → `rederive_capture` drill, BOTH kinds (Accept ¶5)

The forced failure needs **no code** — point the derivation at a broken model (reversible):

**Image:**
1. **[operator]** Settings → Vision group → set primary **and** fallback to a bogus/nonexistent
   model id (forces the VLM call to fail).
2. Capture a photo. **PASS:** derivation walks retry → `unavailable`; the node is filed with the
   **`<photo — description unavailable>`** placeholder (raw kept, pipeline not blocked, broken-media
   tile on the strip). Note the capture id.
3. **Restore** the Vision group to the working Groq/Nebius models.
4. **[operator]** `… exec -T api uv run python -m app.cli rederive-capture <capture_id>`
5. **PASS:** re-derive recovers the description, refreshes the capture's fenced `raw_text`,
   reorganizes — and the **node** now shows the real description (not just `GET /media/{id}`). This is
   the T4-review must-fix (node recovery, not only the media row) proven live.

**Voice:** same shape via the STT chain. If STT isn't Settings-routable to a bogus id, force the
failure by capturing an **empty/corrupt audio** clip; it walks to `<voice note — transcript
unavailable>`. Then `rederive-capture <capture_id>` after restoring a working STT → the node
recovers the transcript.

- **Verify both:** SQL smoke **block 2** (the item left `unavailable` before re-derive, `derived`
  after); the node body no longer shows the placeholder.

## 8. Merge inherits media (Accept ¶6, ADR-060 §4)

1. Have a media-backed node (from step 3/4). Merge another node **into** it (or it into another) via
   `POST /admin/entities/merge` (survivor S, loser L).
2. **PASS:** the survivor's `GET /nodes/{S}.media[]` now includes L's media; SQL smoke **block 4**
   returns **0 rows** (no link stranded on the tombstoned loser).

## 9. Media-join SQL smoke (Accept ¶7, the T3 follow-up)

Run `deploy/smoke/m9_media_join_smoke.sql` (Supabase editor or psql). **PASS:** blocks 1–9 as
commented — tables/indexes present, no dangling links (3), no tombstone strands (4), the get_node
media join + media_kinds array_agg + get_by_capture_id joins all return sane rows against the REAL
DB (not fakes).

## 10. Independent review + sign-off

- Run the **independent agent review** ([09 §Implementation-session loop step 2](../09-session-protocol.md)):
  fresh context, checks the T6 outcome (this runbook's results + the `rederive-capture` verb diff)
  against the M9 **Accept** criteria, ADR-060/057, and CLAUDE.md invariants. Resolve any must-fix.
- **Record** results into [08 §M9](../08-implementation-plan.md) (tick T6, note the review outcome)
  + update the README snapshot; push docs. **T6 done = every Accept ¶ above verified live.**
