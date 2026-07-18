# ADR-061: Composite multi-part capture (draft compose → one blended organize)

**Status:** Accepted · 2026-07-19 (grilled decision-by-decision) · **Adds milestone M9.6** in
[08-implementation-plan.md](../08-implementation-plan.md) and **supersedes the paused M9 T6** (its
remaining single-part live drills fold into the M9.6 Accept) · **Builds on**
[ADR-057](057-multimodal-media-ingestion-substrate.md) (the media substrate; its §5 `<photo: …>`
fence **format** is superseded here — the two-layer attribution **semantic** is preserved),
[ADR-060](060-node-media-linkage-and-voice-unification.md) (`node_media` linkage, voice
unification, kind-aware `rederive_capture` — all **generalized from 1 part to N**),
[ADR-042](042-reprocess-all-from-raw-and-data-survival.md) (P10 — `raw_text` stays the replay
source), [ADR-019](019-capture-pipeline-inbox-fallback-and-nudge.md) (the capture pipeline this
extends), [ADR-030](030-entity-resolution-never-guess.md) (never-guess — the attribution fallback).

## Context

Today a capture is **single-part**: one `kind` (`text` XOR `voice` XOR `image`) and at most one
`media` row, organized into node(s) on its own. The user wants to attach **several photos + a voice
note + text in ONE capture**, organized *together* — the voice note can explain a photo, a text
caption frames both — instead of three disconnected captures.

Grilling surfaced that the **storage substrate is already multi-capable**: `media.capture_id` is a
nullable, **non-unique** FK ([017_media.py](../../second-brain/server/migrations/versions/017_media.py)),
`_link_node_media` already rebuilds over `list_by_capture_id(...)` (a *list*), and `node_media` is
already many-to-many. The single-part-ness lives in four narrower places: `captures.kind` (one
modality), the `_process` orchestration (derives exactly one media, assembles one `raw_text`),
`CaptureView.media` (singular), and the web (one-shot-per-kind, no compose surface). So this is a
**kind / orchestration / assembly / UI** change, not a storage rewrite. Grilled 2026-07-19.

## Decisions

**1. One blended organize pass.** All parts are assembled into a **single** organizer input and
organized in **one** pass under one anchor, so the resulting node(s) can cross-reference across
parts. This is exactly today's media-capture path (`derive → render → set raw_text → organize`)
generalized from one part to N. Rejected: organizing each part independently and merely grouping
them by a shared id — that forbids the cross-part understanding that is the whole point.

**2. Generalize the capture into parts; `kind = composite`.** A capture becomes an **optional typed
text body + an ordered list of media parts (0..N photos + ≤1 voice)**. The single-modality captures
become the degenerate cases. `captures.kind` gains the value **`composite`** for the compose flow;
the node-frontmatter `source` for a web composite is **`web`** (not a modality — `_effective_source`
= `source or kind`, and a composite has no single modality). Rejected: a distinct `composite` kind
bolted **alongside** the untouched three single-part paths — it carries two orchestration paths that
drift; unifying is cleaner and the blended pass *is* the generalization.

**3. Server-side incremental draft; raw persists on attach.** Chosen over a client-held draft
(all parts POSTed at Send). Each part is durable on the server the moment it is attached
(never-lose starts earlier; a flaky mobile connection can't lose a recorded voice note), and an
**open draft survives app-close and is resumable** — the payoff that justifies the extra surface.
Lifecycle:
- `captures.status` gains **`draft`** (pre-submit).
- `POST /capture/draft` opens a draft (**one active draft** at a time; opening the capture screen
  **resumes** the existing draft, plus a **Discard** action).
- `POST /capture/{id}/part` (multipart, one part per call) persists raw immediately + mints the
  `media` row; **≤1 voice enforced server-side** (a 2nd voice part is rejected).
- `DELETE /capture/{id}/part/{mediaId}` removes a draft part (the 'x'; hard-removes raw + row — a
  user-initiated pre-commit edit, not a pipeline drop, so rule 2 is not violated).
- the **text body** is one editable field on the draft (not N interleaved text parts).
- `POST /capture/{id}/submit` requires **≥1 non-empty part**, flips `draft`→`received`, spawns
  `_process`. **Idempotent** (rule 6): submit on a non-draft is a 409/no-op.

**4. Derivation deferred to Submit, then concurrent (bounded).** Attaching a part stores raw + shows
its thumbnail/player instantly (display needs no derivation); **all VLM/STT runs at Submit** inside
`_process` — nothing is wasted on a part the user removes. At Submit the parts derive
**concurrently under a small config-bounded semaphore** (multi-photo is the headline case;
Submit is 202 + background, status walks `deriving`→`organizing`, so it is never a synchronous
block). Assembly order is by part **ordinal**, independent of completion order.

**5. `raw_text` stays the cached assembled replay source; add `captures.text_body`.** New
`text_body` column holds the person's typed words (never-lose + the reassembly source). At Submit,
`raw_text` = `text_body` + each part rendered in ordinal order, **cached** — so `reprocess-all`
replays `raw_text` **verbatim**, byte-parity with today, zero new replay logic. `rederive_capture`
refreshes `raw_text` by reassembling. Rejected: reassembling on every replay (DRYer but changes
reprocess-all's contract from "replay a stored string" to "reassemble").

**6. A stable per-part ordinal.** A capture's parts carry an explicit **position** (a `media`
column for capture parts, or an equivalent ordering) so (a) assembly is deterministic across
reprocess and (b) the attribution indices in §7 map to the right `media` row even after a
draft-time delete + re-add reorders arrival.

**7. Per-node media attribution; supersedes the `<photo: …>` fence *format*.** A media item links
only to the node(s) it is *about*, not all-to-all. Each media part is introduced in the assembled
input by a **structural index marker** carrying its ordinal + kind, e.g.

```
<text_body — the person's words, plain, if present>

[[part 1 · photo]] <description — shared material, a record of an image, never the person's words>
[[part 2 · voice]] <transcript — the person's own words, plain>
[[part 3 · photo]] <description …>
```

The organizer emits, per node, a **bounds-checked `parts: […]`** list (validated exactly like
`arose_from`); `_link_node_media` links each node to only those media. This **supersedes the literal
`<photo: …>` fence format** (ADR-057 §5 / ADR-060 §5) while **preserving its two-layer semantic**:
photo-part content is shared material (never mis-attributed to the person — the binding
screenshot-attribution rule); voice-part content is the person's own words. The exact marker syntax
is a build pin (adjustable in implementation as long as the semantic and the bounds-checked
per-node ref survive).

- **Unattributed part → capture-only.** A part the organizer references from no node stays visible
  on the capture (media list + detail sheet) but pins to no node — nothing lost, nothing
  force-attached.
- **Total attribution failure → all-to-all fallback.** If `parts` is entirely absent/unparseable
  (older model, parse miss), fall back to linking every part to every content node (parity with
  today's behavior) so nothing is silently stranded.
- **Key risk:** the model reliably emitting valid part indices. Mitigated by the bounds-check +
  the two fallbacks above; the attribution is a *link-quality* concern, never a data-loss one.

**8. Fold the web capture surface fully into the draft flow.** The three one-shot HTTP endpoints
(`POST /capture/text|voice|image`) are **removed**; every web capture — including a one-word text
note — goes `draft → part/text → submit`. Single code path, no dead endpoints to drift. The
**internal** producers stay unchanged and do **not** open drafts: MCP `create_mcp_capture`, chat
`create_chat_capture`, and `reprocess-all` replay create a **committed** capture directly
(programmatic, not user-composed). Both styles funnel through the same `_process` / organizer
(single writer, rule 2b).

**9. `rederive_capture`, orphan-sweep, and GC over N parts.** `rederive-capture <id>` re-derives the
capture's **non-`derived`** parts only (`unavailable`/`pending` — never re-runs the VLM on an
already-good photo), reassembles `raw_text`, reorganizes (recovers the node, not just the media
row). The boot **orphan-sweep skips `draft`** captures (a draft is intentionally open, not a crashed
in-flight run). A boot/periodic sweep **deletes unsubmitted drafts older than 7 days** (raw files +
rows); it never touches submitted captures.

**10. Observability by reuse, not a new stepper.** The capture's `agent_runs` detail carries
**per-part** derivation sub-steps (today's single `derive`/`stt` entry becomes one per photo/voice),
and the capture surface **deep-links to that run in the Activity tab** so the user can expand and
follow the processing live. Reuses the M8 observability surface; no bespoke stepper.

**11. Web compose surface + list-valued media.** The capture screen becomes a **compose** surface:
text field + multi-photo attach + record-voice (≤1) + a per-part 'x' + **Send** (enabled at ≥1
part). `CaptureView.media` goes **singular → list** (part order) and exposes `text_body`; the web
renders the list. The `NodePreview` media strip already renders a list, so the node side needs no
new component (it now shows a node's **attributed** parts per §7).

**12. Milestone M9.6, superseding the paused T6.** Composite ships as its **own milestone (M9.6)**
with its own live Accept. The paused M9 **T6**'s remaining single-part live drills (voice
Range/206, screenshot attribution, group-edit forward-live, `rederive` both-kinds, merge-inherits,
media-join SQL smoke) **fold into the M9.6 Accept** — they are all exercised through the composite
flow — and M9 T6 is marked **superseded-by-M9.6**. Avoids running a single-part live Accept for a
capture UX we are reshaping. (M9 T1–T5 remain done and shipped; the media substrate they built is
what M9.6 composes.)

## Consequences

- ✅ One capture can carry several photos + a voice note + text, organized together — the voice
  can explain the photo, nodes cross-reference parts.
- ✅ The storage substrate barely moves: no `media`/`node_media` rewrite; the changes are
  `captures.text_body` + a part ordinal + `status='draft'` + `kind='composite'`.
- ✅ Reprocess-all keeps byte-parity (still replays the cached `raw_text`); `node_media`,
  merge-repoint, and `rederive` all generalize to N parts by construction.
- ✅ Never-lose strengthens: raw persists on **attach**, and an open draft is resumable across
  app-close.
- ⚠️ New surface area: a draft lifecycle (open/part/delete/submit), a `draft` status the
  orphan-sweep must skip, and a 7-day draft GC.
- ⚠️ The organizer contract changes (indexed part markers + per-node `parts`), superseding the
  `<photo: …>` fence *format*; the two-layer attribution semantic is preserved but the prompt +
  validation + tests that key on the old fence must move with it. Attribution reliability is the
  headline risk, bounded by the §7 fallbacks.
- ⚠️ The three one-shot web capture endpoints are removed (breaking for any non-web caller that
  used them; the web is the only real client, and MCP/chat/reprocess are internal).
