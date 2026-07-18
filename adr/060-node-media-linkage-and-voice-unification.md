# ADR-060: Node–media linkage, voice unification, and the media surfacing UX

**Status:** Accepted · 2026-07-18 (grilled decision-by-decision, M9 replan) · **Restructures** the
open tail of M9 in [08-implementation-plan.md](../08-implementation-plan.md) (T4/T5 superseded by
T4/T5/T6) · **Builds on** [ADR-057](057-multimodal-media-ingestion-substrate.md) (the media
substrate this makes node-visible; §2 video clause refined here), [ADR-049](049-dedup-sweep-merge-core-build-decisions.md)
(the shared merge-core this extends), [ADR-042](042-reprocess-all-from-raw-and-data-survival.md)
(P10 — the link is derived-tier, rebuilt by reprocess), [ADR-020](020-stt-fallback-chain-groq-primary.md)
(the STT chain, now driven through the derivation engine), [ADR-054](054-m8.1-ui-navigation-consolidation.md)
(NodePreview/NodeChip — the surface media renders on).

## Context

M9 T1–T3 built the media substrate: `media` rows hang off *captures* (`media.capture_id`), photos
are described and organized as fenced text, and `GET /media/{id}` serves files. But **nodes — the
thing the user actually reads — have no path to their media**: `GET /nodes/{id}` carries no media,
there is no node→capture reverse resolution, and the approved T4 accept line "photo visible on the
node" was unbuildable as specified. Voice audio predates the substrate entirely: saved to disk but
never a `media` row, never servable, transcripts not re-derivable. The user wants media a node
references viewable **in-app and inline** — images now, voice now, video keyframes later — plus a
"see raw capture" traceability affordance. Grilled 2026-07-18.

## Decisions

**1. A first-class `node → media` link, paid for now: the `node_media` join table.** Not a
read-time join through captures (rejected: leaves media second-class), not body-fence parsing
(rejected: fragile, no id), not media-as-graph-node (rejected: contradicts ADR-057 §3 — media
stays a DB+file citizen, git stays text). A plain many-to-many DB relationship:
`node_media (node_id fk → nodes ON DELETE CASCADE, media_id fk → media, unique (node_id, media_id))`,
written by the capture pipeline when a capture's content nodes are written, exposed as a
`media: [{ id, kind, status, capture_id }]` field on `GET /nodes/{id}`. It is a **media
attachment**, not a graph edge — it does not appear in `edges`, traverse, the Map, or MCP.

**2. Link policy: content nodes only.** A media item links to the **content node(s) in its
capture's `node_paths`** — the node built *from* the photo/voice note. Entity hubs the capture
merely mentions do **not** own the media (a screenshot mentioning five people depicts none of
them); a hub reaches media transitively through its edges to content nodes. Because the table is
many-to-many, hub-linking ("all photos of P1") can later be added as a pure write-policy change —
**no new migration** — gated on a real depicts-vs-mentions signal. Logged as a future enrichment.

**3. Derived-tier, rebuilt on every content-node write, keyed on `nodes.id`.** The link composes
raw truth (`media.capture_id` — the user attached this file to this capture) with a derived fact
(organize decides the capture's content nodes), so `node_media` is **derived**: recomputed
whenever a capture's content nodes are (re)written — initial organize, retry, reorganize,
`rederive_capture`, and `reprocess-all-from-raw` (P10: it falls out of replay like the search
index does; no independent durability). Keyed on the stable `nodes.id` (FK, cascade), never
`store_path` — matching `GET /nodes/{id}` addressing and surviving path churn.

**4. Merges repoint media, in the shared merge-core.** When loser L merges into survivor S,
`MergeCore` (ADR-049 §1 — the one choke point both entity-merge and dedup content-merge funnel
through) repoints `node_media` L→S alongside the inbound-edge retarget, `ON CONFLICT DO NOTHING`
against the unique pair. A tombstoned loser is kept, not deleted, so the FK cascade never fires —
the explicit repoint is what prevents stranding photos on tombstones. Standing merges re-apply
after a reprocess, so the rebuilt links land on the survivor again.

**5. Voice joins the substrate by full unification, not a minimal handle.** The ad-hoc voice
pipeline re-routes onto the **existing** T2 derivation engine (which already has a voice→STT leg,
built for M9.5): `POST /capture/voice` mints a `media` row (kind `voice`, source `capture`, audio
stored under the uniform `/srv/data/media/capture/…` layout), `derive_until_settled` drives the
ADR-020 STT chain, and the transcript lives as `media.derived_text`, **mirrored to
`captures.raw_text` plain/unfenced** (the person's own words — unlike the shared-material
`<photo: …>` fence) as the organize/reprocess replay source. Reprocess still replays `raw_text`
without re-running STT (parity kept). Targeted **re-transcription** comes free:
`redescribe_image_capture` generalizes to a kind-aware **`rederive_capture`** (photo → re-fence,
voice → plain refresh, then reorganize). **Backfill:** an idempotent deploy-time op relocates
legacy voice audio into the media layout, mints `media` rows with `derived_text` = the existing
transcript, and links `node_media`; a missing audio file degrades that item, never fails the op.

**6. One failure contract: voice degrades to placeholder, symmetric with image.** STT was classed
as infrastructure (failure → capture `failed`, blocked on a human). Under unification it is a
**derivation**: bounded retries → `unavailable` → the capture **organizes anyway** with the
`<voice note — transcript unavailable>` placeholder — never blocks, audio kept + playable,
re-transcribe→reorganize recovers the node later. `failed` remains for true infrastructure only
(store write, disk). Recorded cost: a placeholder voice node is a near-empty stub until
re-derived — but visible, linked, and recoverable, versus a silent `failed` row.

**7. Surfacing UX: NodePreview is where media lives.** One coherent package:
- A **media strip** in the shared `NodePreview` drawer (between title and body): photo thumbnails
  (lazy-loaded `GET /media/{id}`, browser-scaled — **no server thumbnailing** for a single-user
  app, logged as backlog), voice as a compact themed audio player (native `<audio>` under a
  styled shell; Starlette `FileResponse` already serves Range/206 for scrubbing). `pending` =
  shimmer tile, `unavailable` = explicit broken-media tile — never a silent gap.
- **Tap photo → full-screen lightbox** (framer-motion zoom from the thumbnail, swipe/tap dismiss,
  left/right nav across a node's photos).
- **"See raw capture":** media items ride with `capture_id`; the strip's overflow row opens a
  **capture detail sheet** over `GET /captures/{id}` — raw text (fenced description /
  transcript), status, source badge, its media, and NodeChips to every node it produced. The
  same component serves the Activity › Captures expanded row (shared, not new surface area).
  This is the node → capture → sibling-nodes traceability hop.
- **Everywhere else stays lean:** search result cards and chat source cards get at most a tiny
  media glyph (📷/🎙) riding a new `media_kinds` field; **no** thumbnails in lists, **nothing**
  on the Map canvas.

**8. HEIC is converted client-side at capture.** The PWA converts HEIC→JPEG before upload (a
lazy-loaded converter activated only when a HEIC file is picked; synthetic `photo.jpg` filename
per the upload contract). The server stays image-library-free; every stored photo is
browser-renderable (Chrome/Android cannot display HEIC) and VLM-safe by construction. **Recorded
P10 nuance:** the camera-original HEIC is never kept — the converted JPEG *becomes* the raw, and
is itself re-derivable forever. Server keeps accepting HEIC on the API (non-PWA callers); such
files may degrade to placeholder and may not render — the PWA path never produces them.

**9. Video (M9.5 refinement of ADR-057 §2): summary + 1–2 representative thumbnails.** The prep
tool selects **1–2 representative keyframes** (most informative frames, not arbitrary) and
uploads them as servable thumbnail media alongside the summary text, so a video-derived node can
show its representative frame(s) inline even though the file itself is never uploaded. Refines
§2's "optional small thumbnail" (singular, arbitrary). No work in M9.

**10. MCP exposure of media: backlog.** `get_node`/`build_context` render stays text-only; a
`[photo attached]`-style marker (or MCP resource serving) is deliberately deferred.

## Consequences

- ✅ "Photo visible on the node" becomes real and buildable: nodes carry their media; the M9 T4
  accept gap is closed by restructuring, not bolting on.
- ✅ Voice reaches full parity: playable, node-linked, re-transcribable, one failure contract
  and one storage layout across all derived media.
- ✅ The link survives merges, retries, reorganizes, and full reprocess by construction; media
  can later attach to hubs with zero schema change.
- ⚠️ One new migration (`node_media`) + a deploy-time voice backfill op (idempotent, degrading).
- ⚠️ STT failures no longer block as `failed` — they produce placeholder stubs a re-derive must
  recover; the ops surface should keep `unavailable` items visible.
- ⚠️ Full-size images serve into thumbnails (no server thumbnailing yet); acceptable single-user,
  revisit if lists ever grow media.
