# ADR-033: External inspirations — adoptions from `obsidian-second-brain` (agent behaviors, not storage)

**Status:** Accepted · 2026-07-13 (user: "adopt all"; #7 promoted to the roadmap as a must-have
ingestion surface) · Companion to [ADR-032](032-prior-art-adoptions.md); updates
[08](../08-implementation-plan.md) M4/M5/M6/M8/M9/M10 + backlog.

## Context

Review of [eugeniughelbur/obsidian-second-brain](https://github.com/eugeniughelbur/obsidian-second-brain)
(3.2k★, v0.12, 2026-07): a Claude Code plugin (44 commands + cron-run CLI agents + small Python
helpers) over a local Obsidian vault. **Architecturally not a peer** — no server/DB/index, no
durability tier, agents need the laptop awake, its "vault rewrites itself" philosophy is the
LLM-read-modify-write accretion model ADR-030 rejected, and its MCP lets any agent write notes
directly (vs our organizer-single-writer, ADR-028). Its strength is the layer we had
under-designed: **what agents say and do with the memory**. Cost note (user's framing,
confirmed): its research features bill per call (Grok ~$0.05, Perplexity ~$0.02–0.50,
research-deep ~$0.40–0.80); our Claude-Max plan + always-on VPS + self-hosted embeddings make
the equivalents fixed-cost, and the MCP pattern (#6) makes external research zero-marginal.

## Adopted

1. **Identity capsule** (their `/obsidian-world` L0, ~170 tokens). A **derived** ~200-token
   "who I am / current state" preamble (core facts + current priorities distilled from recent
   `insight` nodes), refreshed in the sleep cycle, injected into every chat system prompt
   (**M4**) and served as level-0 of MCP `build_context` (**M5**). Derived-tier, rebuildable,
   never hand-maintained files (no SOUL.md — ours is regenerated).
2. **Contradiction sweep** (their `/obsidian-reconcile`, mapped to our substrate). Nightly
   reconciler job (**M6**): new memories vs entity profiles/older claims — clear supersession →
   close the old edge with `until` + profile refresh + feed flag (one-tap revert); ambiguous →
   review queue, new kind **`contradiction`**. Their detect / auto-resolve / flag triage on our
   invalidate-never-delete rails.
3. **Freshness discipline lite** (their OKM rule: *"every stored fact must be timeless, dated,
   or a pointer"*). For us: derived-profile observation lines carry **`(as of …)` stamps**
   (from edge `since` / latest supporting `occurred`) — lands with the M3 profile job;
   graph-health (#5) flags stale volatile claims; the reflection agent (**M10**) runs
   occasional **staleness interviews** ("Is Alex still at Google?") — staleness becomes
   conversation, not rot. We do **not** adopt their body-text lint (our facts live in
   edges/profiles, not prose claims).
4. **Reflection-agent enrichments** (**M10** scope; from their thinking tools). The `/emerge`
   insight taxonomy: recurring themes (3+ occurrences, never named) · **energize-vs-drain**
   emotional patterns · unstated implications · emerging directions — under their rule,
   adopted verbatim as prompt law: *"surface what they haven't named yet, do not restate what
   they already know."* Plus **challenge mode** (argue against an idea using the user's own
   cited history — also a chat mode), **belief timeline** ("how my view of X evolved" —
   `occurred`-ordered memories + `until`-closed edges), and **catch-up on demand**.
5. **Graph-health job** (their `/obsidian-health`) → **M8** ops console: orphan nodes, `inbox/`
   depth, pending-review aging, memories missing `occurred`, alias-less entities, tombstone
   integrity, freshness flags — nightly job + panel card.
6. **Research-via-MCP is the canonical research pattern** (their vault-first `/research-deep`,
   inverted to our cost structure): at **M5**, an MCP-connected LLM (a) queries the graph for
   what is already known, (b) identifies gaps, (c) does external research on *its own* plan,
   (d) submits the distillate via `capture` with source refs. Recorded as the documented
   pattern; a server-side research connector exists on the backlog **only if** this proves
   insufficient.
7. **Telegram capture connector — promoted to the roadmap (M9)** as a must-have ingestion
   surface: a private bot polled **from the VPS** (always-on — beats their laptop poller);
   voice/text messages enter the **same capture pipeline** (STT chain, organizer, never-lose;
   `source: telegram`). Small and independent — depends only on the M3 capture pipeline, may be
   **pulled forward** ahead of Slack at the user's call. **Open question for its grilling:**
   image capture (their version vision-routes photos) — image ingestion is new scope, decide
   there.

## Explicitly rejected (safeguard — do not relitigate without new evidence)

- ❌ **"Vault rewrites itself"** (LLM destructively rewriting canonical pages, `## History`
  sections as audit): conflicts with immutable memories + thin hubs + derived profiles
  (ADR-030); git + `until` edges are our audit trail.
- ❌ **Two-Output Rule** ("every answer also updates pages"): conflicts with the stance gate
  (ADR-029) — ingestion from conversation stays salience- and stance-gated, nightly.
- ❌ **Server-side paid research APIs** (Grok/Perplexity/Gemini per-call): the MCP pattern (#6)
  delivers it at zero marginal cost on the Max plan.
- ❌ **AI-first prose conventions** ("For future Claude" preambles, prose confidence markers):
  our frontmatter/edges/`conf` already encode this structurally.
- ❌ **Raw MCP write/update tools** (`save_note`/`update_note` from any agent): ADR-028's
  organizer-single-writer stands.

## References (for future milestone grillings)

Repo: <https://github.com/eugeniughelbur/obsidian-second-brain> · freshness policy (OKM):
`references/freshness-policy.md` · reconcile: `commands/obsidian-reconcile.md` · emerge:
`commands/obsidian-emerge.md` · world/context engine: `commands/obsidian-world.md` · challenge:
`commands/obsidian-challenge.md` · health: `commands/obsidian-health.md` · MCP server:
`integrations/obsidian-mcp-server/` · Telegram: `integrations/telegram-journal/` (poller +
BotFather setup + Whisper transcription flow). Consult at the M5/M6/M8/M9/M10 kickoff
re-checks; the rejected list above is binding context for those reads.
