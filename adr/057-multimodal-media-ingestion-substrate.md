# ADR-057: Multi-modal media ingestion substrate — media as first-class raw, vision routing group, PWA photo capture

**Status:** Accepted · 2026-07-18 (grilled decision-by-decision) · **Defines** the M9 milestone in
[08-implementation-plan.md](../08-implementation-plan.md) · **Builds on**
[ADR-014](014-vault-history-durability.md) (the `/srv/data` → R2 raw-input sync this reuses),
[ADR-020](020-stt-fallback-chain-groq-primary.md) (the STT chain photos' sibling path reuses),
[ADR-042](042-reprocess-all-from-raw-and-data-survival.md) (P10 — derived media text must be
recomputable from kept raw), [ADR-045](045-provider-model-effort-separation.md) (routing groups
over model ids — the `vision` group follows its shape) · **Consumed by**
[ADR-058](058-instagram-dm-connector-and-conversation-substrate.md) (the first bulk producer of
media items).

## Context

The system ingests text and voice only. The Instagram DM export (ADR-058) carries photos, videos
and voice notes inside conversations; the user also wants ad-hoc image capture (photos,
screenshots of apps/whiteboards/pages). Today there is no vision capability anywhere: no
vision-capable routing, no media storage story, no derived-description machinery. Grilled
2026-07-18.

## Decisions

**1. Media are first-class raw inputs (architecture "A"), not connector-side preprocessing.**
Images, voice audio and (with the §2 exception) videos enter the same capture/connector
substrate as text: raw media is **stored and kept**, and a **media-understanding stage** derives
text from it — photo → vision description, voice → the existing Groq→OpenAI STT chain, video →
summary. Derived text is derived-tier data: recomputable from kept raw when models improve, so
P10/reprocess semantics hold for media exactly as for text. The rejected alternative — connectors
describing media during normalization and handing the distiller only text — would have made a
2026 vision model's output effectively raw, unrecoverable without re-running external scripts.

**2. Video exception: store the processed form, not the file.** Videos are transcribed/described
**at import time** (ADR-058: locally, in the prep tool — ffmpeg keyframes + audio-track STT +
vision on keyframes → compact summary) and the **summary text is stored as the raw, tagged
`video`** (+ optional small thumbnail). The original file is never uploaded and never kept
server-side. This is a **knowing, recorded P10 exception**, motivated by size (videos are ~80% of
the export's bytes; single threads carry 100–170MB) against a 4GB VPS; the user's export
zip/archive remains the cold-storage original if a re-pass is ever wanted. Photos and voice notes
are small, are kept raw, and stay fully re-derivable.

**3. Storage: `/srv/data/media/…` on the existing raw-inputs volume.** Media files live on the
filesystem (`/srv/data/media/<source>/<thread>/…` for connector media,
`/srv/data/media/captures/…` for ad-hoc captures) — **never in the git graph store** (git stays
text) and never as DB blobs. The volume already has a **nightly R2 sync** (ADR-014 operational
backup), so durability comes free. A `connector_media` table (02-data-model) tracks each item:
kind (`photo`/`voice`/`video`), file path (null for video — summary-only), **derivation status +
derived text + model used**. Derivation is a **resumable, status-tracked job**: bounded retries
with backoff; after N failures (config) the item is marked `unavailable` and downstream consumers
render an explicit placeholder (`<photo — description unavailable>` / `<voice note — transcript
unavailable>`) rather than blocking; **targeted re-derivation** (only `unavailable` items, or an
explicit list) can revisit any time — giving up is reversible because raw is kept.

**4. A fourth routing group: `vision`.** UI-editable in Settings → Models exactly like
`chat`/`conspect`/`quick` (ADR-045 semantics; per-call effort N/A). Seeded **Groq VLM primary,
Nebius VLM fallback** (user call — Groq first); exact model ids are config scalars verified
against live catalogs at build time. Mechanism: the existing OpenAI-compatible provider class
gains `image_url` content-part support — modest and clean. **Rejected:** vision through the
`claude` CLI provider (file-path-through-tools hackery, permission surface, fragile). The local
prep tool (ADR-058) calls the same vendor APIs directly with its own env keys — no dependency on
the server routing engine, same default model for uniform description style.

**5. One description contract, with a screenshot-attribution rule (binding, two layers).** Every
photo description is compact + factual and **transcribes any legible text verbatim** (many DM
"photos" are screenshots — the text is the value). When an image is itself a **chat screenshot**:
- **Vision layer:** the description must say it is a screenshot, transcribe the contained
  messages **with the screenshot's own internal attribution** (bubble alignment left/right, names
  visible inside the image), and never present them as speech of anyone in the outer conversation.
- **Distiller/organizer layer:** prompt rule — content inside `<photo: …>` placeholders is
  *shared material* ("X shared a screenshot showing …"), never X's own words, and **never the
  user's words** merely because the user sent the image. The same rule covers forwarded and
  inline-reply-looking content generally.
This operationalizes the user's hard requirement that other people's messages are never
mis-attributed as their own.

**6. Ad-hoc PWA photo capture (in M9); video/MCP-image capture out.** The capture strip gains an
image affordance (camera/file input) → upload → `captures` row (kind `image`, raw kept under
`/srv/data/media/captures/`) → vision description (server-side, resumable per §3) → organizer,
the description entering as fenced derived text, the photo surfaced on the capture/node. A
screenshot-of-a-conversation captured ad-hoc is handled by the §5 contract — it does **not**
spawn a message-level ingestion pipeline (no `connector_messages` minted from pixels); a
dedicated screenshot-conversation pipeline is **backlog**, only worth grilling if a source with
no export ever matters enough. PWA **video** capture stays out (no server video stage by design,
§2); MCP `capture` stays text-only.

**7. Serving.** One authenticated, session-gated endpoint (`GET /media/{id}`) streams a media
file — used by the session-transcript view (ADR-058) to show photos inline and play voice notes,
and by the capture/node surfaces for ad-hoc images.

## Consequences

- ✅ Media descriptions/transcripts are derived-tier: re-derivable under better models (P10),
  with the single recorded exception of videos (§2).
- ✅ Vision arrives as a governed, swappable routing group, not a hardcoded vendor call.
- ✅ The connector (ADR-058) and ad-hoc capture share one media pipeline, one storage layout,
  one serving path.
- ⚠️ Two media paths exist by design: video understanding is local-tool-only; photos/voice are
  server-side. A future PWA-video feature would need a server video stage — deliberately not
  built now.
- ⚠️ `/srv/data` grows by the selected threads' photos+audio (~hundreds of MB); R2 sync covers
  loss; disk watched via the existing ops surface.
