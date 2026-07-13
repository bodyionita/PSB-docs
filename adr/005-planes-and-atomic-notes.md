# ADR-005: Life planes as folders; multi-plane membership in frontmatter; split into atomic notes

**Status:** Accepted · 2026-07-12
**Partially superseded by [ADR-026](026-graph-native-storage-obsidian-removed.md) (2026-07-13):**
planes survive as node attributes (`plane`/`planes[]` — filtering/scoping truth) and atomic
splitting survives; **folder = plane is dead** (folders are node types now), and cross-linking is
typed `edges` frontmatter, not wikilinks/`related:`.

## Context
The second brain mirrors a life: professional, personal, family, friends, health, ideas.
Content often spans planes (a Slack chat that is both work and personal). Options: planes
as folders (one home), planes as tags only (flat vault), emergent LLM taxonomy, notes
duplicated/symlinked into multiple folders.

## Decision
1. **Planes = top-level vault folders**, taxonomy defined in config (not code, not
   LLM-invented).
2. **Default: split.** One source yielding separable content produces multiple atomic
   notes, one per plane, sharing `source_ref` and cross-linked (`related` + wikilinks).
3. **Exception: inseparable content** gets one note: primary plane = folder,
   full membership in frontmatter `planes: [..]`. Queries and analysis filter on
   frontmatter, never on folder — the folder is navigation convenience.
4. The classifier may answer "don't know" → global `Inbox/`, never a guess.

## Consequences
- ✅ Navigable vault for manual exploration; Zettelkasten-style atomic linked notes make
  the Obsidian graph meaningful; per-plane analysis is a frontmatter filter.
- ✅ Adding a plane = config change.
- ⚠️ Splitting costs extra LLM tokens per ingested source — accepted (quality over cost,
  and distillation runs on the cheap chain).
- ❌ Rejected: tags-only (unnavigable vault); emergent taxonomy (category sprawl);
  duplication/symlinks (divergence; symlinks fight git and Obsidian).
