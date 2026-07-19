# ADR-062 — Chat-screenshot self-attribution (own-chat default)

**Status:** Accepted 2026-07-19 · **Refines** [ADR-057](057-multimodal-media-ingestion-substrate.md)
§5 (the two-layer screenshot-attribution contract — the split stands; its "never the sharer" rule is
corrected for the own-chat case) · **Feeds** 08 §M9.7 (build) · **Found by** the M9.6 T6 live drill
([08-logs/m9.6-accept-drill.md](../08-logs/m9.6-accept-drill.md) §E.2).

## Context

The M9.6 Accept drill exposed a real-world failure: a screenshot of the **user's own** chat was
organized as "a conversation between P1 and *the sender*" — the user's own messages became a
phantom, nameless third party, and left/right bubble attribution was unreliable to begin with.

Root cause is a deliberate rule doing its job in the wrong case. ADR-057 §5 (vision layer) and the
organizer prompt (v7) both say: attribute a chat screenshot's messages to the people **inside** the
image, "NEVER to the person who saved it and NEVER to 'you'". That is right for a *forwarded*
screenshot of someone else's conversation — and exactly wrong for the overwhelmingly common case,
a screenshot of **your own chat**, where the unlabeled right-side bubbles *are* the user. The
system also has **no identity signal at ingest**: no self-entity, no configured user name/handles;
the identity capsule feeds chat, not the organizer. So the organizer literally cannot name the
right side, and the "never you" rule forbids the one correct resolution.

Two further facts from the drill + code:

* **Reply-quote insets break naive side-attribution:** a reply bubble inlines the *other* party's
  quoted message (faded/recolored/marked) inside the replier's bubble — pure right=me would
  misattribute the quote to the bubble owner, in both directions.
* **The vision layer's output is loose prose**, so even a correct read of the layout reaches the
  organizer in a shape it can't reliably map. The `vision` group's primary is a small VLM
  (`llama-4-scout-17b`); the fallback (`qwen2.5-vl-72b`) is much stronger at dense screenshot
  layout — reliability may be model as much as prompt.

## Decision

**§1 — Scope.** Screenshots of the user's **own chats (1:1 and group) are first-class** and resolve
by default. Forwarded/third-party screenshots: **default-mine now** — v1 may misattribute a
forwarded 1:1's right side to the user (accepted, rare); an **explicit "not mine" signal**
(per-photo toggle or equivalent) is **deferred** to a follow-up. Until it ships, a plain statement
in the capture's own text ("this is P1 and P2's chat, not mine") must be honored by the organizer
(attribute purely by internal names; never "you").

**§2 — Vision layer: disciplined per-message text (identity-agnostic, position-faithful).** The
two-layer split stands: the vision layer still never says "you". For a chat screenshot it emits,
in plain text (the derived text stays the byte-parity reprocess replay source — no JSON):

```
Chat screenshot (<app if identifiable>). Header: <name / group title as shown>.
[left · P1] <message text verbatim>
[right] <message text verbatim>
[right · quoting P1] <the quoted inset's text>
[right] <the reply text under that quote>
```

Rules: one line per message bubble, tagged with its **side** and any **visible sender label**;
a reply's **quoted inset** (recognizable by fade/color/inset styling) becomes its **own line**
attributed to the **quoted** party (`quoting <name>`), never to the bubble owner; the reply's own
text is a separate line for the bubble owner. Non-chat images keep the existing one-paragraph
description contract unchanged.

**§3 — Organizer: identity mapping (prompt v8).** The organizer owns the identity interpretation
of §2's tags, replacing the v7 "never you" rule with:

* `[right]` (and right-side labeled with what the header/context shows to be the capturer) → the
  **user's own words** — organized first-person, exactly like any voice/text capture. **No
  self-entity is minted and no "sender"/"unknown" person entity is ever created for the unlabeled
  side** — the phantom-sender failure mode is banned outright.
* `[left · Name]` → that **named person** (normal entity resolution; per-sender in group chats;
  in a 1:1 the header names the left party even when bubbles are unlabeled).
* `[… · quoting Name]` → the **quoted party's** words.
* The §1 "not mine" text override wins over all of the above: attribute by internal names only,
  never "you".

"You said X to P1" is representable today (content nodes are implicitly the user's; P1 stays an
`involves` entity) — **no schema change, no self-entity** (a first-class "me" hub was considered
and rejected as a much larger architectural change with no present need).

**§4 — Reliability: prompt-first, then measured A/B.** Ship §2's prompt, then A/B the **same real
screenshot** on `llama-4-scout-17b` (Groq, primary) vs `qwen2.5-vl-72b` (Nebius, fallback) via the
Settings vision-group flip (the M9 T5 forward-live group edit). Promote the 72B to primary **only
if the measurement says the 17B can't hold §2** — an evidence-driven routing change, not a
preemptive one.

**§5 — Migration: rederive + reorganize existing screenshots.** The organizer change alone cannot
fix already-ingested screenshots — their cached `raw_text` holds the old loose description. After
deploy, existing photo captures are **re-derived (fresh VLM pass under §2) + reorganized (§3)** via
the existing `rederive-capture` seam, correcting the drill's misattributed capture. Prod media is
tiny at this writing, so the pass is cheap; it is targeted (photo captures), not a full
`reprocess-all`.

**§6 — Deferred.** The explicit per-photo "not mine" control (UI + API field); configured user
identity (name/handles) as an additional labeled-bubble signal for group chats; any self-entity.

## Consequences

* The common case (own chat) organizes correctly: the user's side is their words, the other party
  is a real named entity, quotes land on the quoted party — and the graph stops accreting phantom
  "sender" people.
* A forwarded 1:1 screenshot can be misattributed to the user until §6's explicit signal ships —
  accepted consciously in the grill (rare; correctable by the §1 text override or a re-capture
  with the note).
* The vision prompt becomes chat-screenshot-aware but stays identity-agnostic and plain-text, so
  ADR-057's layering, `reprocess-all` byte-parity (ADR-042), and the marker/fence assembly
  (ADR-061 §7) are all preserved. Prompt version bumps (vision description + organizer v8) mean
  **newly derived** text differs — parity holds per raw input version, as with every prior prompt
  bump.
* Routing defaults change only on A/B evidence (§4), keeping the Groq-first cost posture unless
  proven inadequate.
