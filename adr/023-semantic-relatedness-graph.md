# ADR-023: Semantic relatedness graph — `note_links` (canonical) + rendered vault wikilink block

**Status:** Accepted · 2026-07-13 (M2 planning)
**Partially superseded by [ADR-026](026-graph-native-storage-obsidian-removed.md) (2026-07-13):**
the derived similarity graph in the DB stands (as derived `similar` edges); the rendered
`sb:related` wikilink block, its churn-gating, and the content-hash exclusion are **deleted** —
similarity is never written into files anymore.
**Relates to:** [005 planes & atomic notes](005-planes-and-atomic-notes.md) · [002 Supabase/pgvector](002-supabase-pgvector-for-index.md) · [022 embeddings](022-embeddings-self-hosted-nomic.md) · [014 vault durability](014-vault-history-durability.md) · updates [02-data-model](../02-data-model.md), [03-api](../03-api.md), [04-pipelines](../04-pipelines.md), [08-implementation-plan §M2](../08-implementation-plan.md)

## Context

Live M1 use surfaced a want the docs flagged as **genuine new scope** (see
[08 §"odd observations"](../08-implementation-plan.md)): the user wants note links that reflect
**actual topical relatedness**, not merely **co-capture** (the existing `related:` frontmatter +
`## Related` section, which links siblings written from the *same* capture — [ADR-005](005-planes-and-atomic-notes.md)).
The two are different relations and conflating them re-creates exactly the confusion that was
raised (a park-picnic note and an app-idea note are co-capture siblings yet topically unrelated).

M2 introduces pgvector similarity, which is the enabling primitive. During M2 grilling the user
chose the **full materialized graph** (not just a search primitive, not just a read-time
"related notes" query) — and specifically wants it visible in **Obsidian's graph view**, which is
driven by `[[wikilinks]]` in note bodies.

## Decision

**1. Canonical store = a derived `note_links` table; the vault gets a rendered projection.**
- **`note_links`** (migration 004, rebuildable from embeddings — same durability tier as
  `notes`/`chunks`): `note_id → related_note_id`, `score` (cosine), directional rows.
- **Vault projection**: a machine-owned `## Related notes` block of `[[wikilinks]]` rendered into
  each note body so it shows in Obsidian's graph. The DB table is canonical (drives web
  search/preview + is the recompute source); the vault block is a rendered view of it.

**2. Semantic links stay strictly distinct from co-capture links.**
- Frontmatter `related:` and the human/organizer `## Related` section keep meaning **"co-capture
  siblings"** — **untouched** by this feature.
- Semantic links live in (a) `note_links` and (b) a **separate, delimited** body block:
  ```
  <!-- sb:related:start -->
  ## Related notes
  - [[Ideas/2026-07-12 Braindan — personal second brain app|Braindan — personal second brain app]]
  <!-- sb:related:end -->
  ```
  Path-target + title-alias wikilinks (so same-titled notes across planes resolve), placed at the
  end of the body. The delimiters make the region unambiguously machine-managed — a human edit
  outside the markers and a regenerated block never collide.

**3. Note-to-note similarity uses a note-level vector = mean-pool of the note's chunk embeddings.**
- New `notes.embedding vector(768)` ([ADR-022](022-embeddings-self-hosted-nomic.md)), written in the
  same transaction that writes the note's chunks — **free** (no extra embedding call). Well-matched
  to atomic notes ([ADR-005](005-planes-and-atomic-notes.md)), which are mostly 1–2 chunks.
- Neighbors = **top-K over `notes.embedding` cosine, above a floor**: `RELATED_TOP_K` (default 5),
  `RELATED_MIN_SCORE` (default 0.5) — **both settings, tuned live** during the M2 Accept (empty
  graph → lower the floor; junk links → raise it). Directional per-note rendering ("notes most
  related to this one"); **no mutual-only requirement**.
- Future swap to a dedicated per-note embedding (embed `{title}\n\n{body}`) is a drop-in on the same
  column — noted, not built.

**4. Recompute is nightly-only; the real-time capture path never touches the graph.**
- The graph is **global** (adding one note can change others' neighbors), so it is recomputed
  wholesale in the **one combined nightly `reindex` job** (M2): `git pull` → rescan → **recompute
  `note_links`** → **render changed `## Related notes` blocks** → commit+push (under the single
  vault git lock, [ADR-014](014-vault-history-durability.md)). `POST /admin/reindex` triggers the
  same full recompute on demand.
- Capture stays fast and churn-free; a freshly captured note gets its related block the next
  morning. Relatedness is a reflective/background feature — hours of latency is fine.

**5. Churn control + the feedback-loop fix (both mandatory).**
- **Churn:** a note file is rewritten only when its newly-rendered block **differs** from the
  on-disk block (compare block hash). A stable graph ⇒ zero nightly writes ⇒ zero git churn.
- **Feedback loop (designed out):** the indexer's `content_hash`, chunking, and embedding **strip
  the `sb:related` block** (exactly as they strip frontmatter). Otherwise writing the block would
  change the file's hash → the next rescan re-embeds the note (now including its own related links
  as text) → its neighbors shift → the block rewrites again: a self-feeding churn + embedding-
  pollution loop. Stripping the block makes a note's identity/embedding its **human content only**,
  and means writing the block never marks the note "changed."
  - Note: `content_hash` still covers **frontmatter + body minus the sb:related block**, so genuine
    metadata edits (tags/plane/planes) *do* trigger a reindex; only the machine block is excluded.

## Consequences
- ✅ Delivers the actual want: topical relatedness, visible in Obsidian's graph, distinct from
  co-capture linkage.
- ✅ Canonical `note_links` is rebuildable (reindex restores it), so the graph adds no new never-lose
  burden; the vault projection is a convenience view.
- ✅ Nightly-only + churn-gated + block-stripped ⇒ bounded, stable git history (no nightly diff storm).
- ⚙️ The nightly job now mutates human vault notes (the machine block). Mitigated by strict
  delimiters, block-diff gating, and the never-lose vault being the source of truth (a bad render is
  a git revert away).
- ⚙️ `RELATED_MIN_SCORE` is empirical — the M2 Accept must tune it against the real vault; a wrong
  floor yields an empty or noisy graph, not a correctness bug.
- ⚙️ Adds `note_links` + `notes.embedding` to migration 004; adds `GET /notes/{id}` (preview returns
  the note's related links) to [03](../03-api.md).
- ↩️ Read-time-only "related notes" (no materialization) and mutual-only linking were considered and
  rejected in favour of the materialized, Obsidian-visible, directional graph the user chose.
