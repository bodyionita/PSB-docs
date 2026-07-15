# ADR-046: M5 MCP server — remote OAuth connectors, transport, surface format, capsule scope

**Status:** Accepted · 2026-07-15 (grilled decision-by-decision) · **Refines**
[ADR-028](028-one-service-layer-mcp-peer-surface.md) §5 (bearer token → OAuth 2.1) and
[ADR-033](033-external-inspirations-obsidian-second-brain.md) #1 (identity-capsule source +
budget); builds on [ADR-003](003-single-service-on-vps.md), [ADR-013](013-web-stays-on-vps-single-origin.md),
[ADR-032](032-prior-art-adoptions.md) (traverse pagination + `build_context`).

## Context

M5 exposes the service layer over MCP so external LLMs can query the graph and feed it
(ADR-028). The M5 kickoff grill pinned the surface to build-ready detail. The decisive fact:
the connection surfaces the user wants — **the mobile Claude app and claude.ai web ("custom
connectors")** — authenticate to a remote MCP server with an **OAuth 2.1 flow**, not the
static bearer ADR-028 §5 sketched. That flips M5 from "smallest milestone" to one whose real
weight is a self-hosted authorization server. The same OAuth build also unlocks Claude Desktop
and ChatGPT connectors off one server. Claude Code CLI (static-header capable) is deferred.

## Decision

1. **Transport & deployment.** Official `mcp` Python SDK, **Streamable HTTP** (SSE is
   deprecated), mounted under **`/mcp`** on the existing FastAPI app — one `api` container
   (ADR-003), **single origin** `braindan.cc` (ADR-013). Caddy grows root-level routes
   (`/mcp`, `/.well-known/oauth-*`, `/authorize`, `/token`, `/register` → `api:8000`);
   Cloudflare passes them **un-cached**. No subdomain, no new container.

2. **Auth = self-hosted OAuth 2.1 authorization server** (supersedes ADR-028 §5's static
   bearer), built on `authlib` for the protocol crypto (PKCE, code exchange, metadata) so we
   do not hand-roll security-critical code. The `api` app is both authorization server and
   resource server:
   - **Discovery**: `/.well-known/oauth-protected-resource` (RFC 9728) +
     `/.well-known/oauth-authorization-server` (RFC 8414). The MCP resource identifier is
     `https://braindan.cc/mcp` (RFC 8707).
   - **Open Dynamic Client Registration** (`/register`, RFC 7591) — spec-required so the
     connectors self-register; **inert on its own** (registration grants nothing).
   - **`/authorize` is the choke point**: server-rendered, authenticates with the **existing
     Argon2id password** (reuses `AuthService`; a valid PWA session cookie short-circuits to
     just the consent step), **explicit Approve/Deny consent** (defense against a malicious
     site silently driving the flow in a logged-in browser), **CSRF-protected + rate-limited**
     (reuse `login_rate_limiter`), **PKCE** required.
   - **Tokens: opaque, HMAC-hashed, DB-stored** — same pattern as session tokens
     (`security.py:hash_session_token`); plaintext to the connector once, only the hash in the
     DB. **Access token ~1h**; **long-lived sliding refresh** (renewed on use → no daily
     re-approve for an active connector). **Revocation = a single DB update; a "revoke all MCP
     access" switch** is the M5 control (instant, total; per-connector management deferred to
     the M8 ops console). Satisfies ADR-028 §5 "independently revocable" + the Accept criterion.
   - **Single full-access scope** for M5 (read/write scope split deferred). The `capture`
     write path stays protected by the organizer-single-writer discipline (ADR-028 §3) behind
     the consent gate + revoke-all.
   - This **resolves the deferred "MCP token distribution" open question** (README cold-start
     #5): there is no token to hand out — the user adds a custom connector and completes the
     in-app OAuth flow.

3. **Tool surface** — thin over the service layer (ADR-028, no logic duplication), the six
   tools of the 03-api table: `search`, `get_node`, `traverse`, `build_context`,
   `list_planes`/`list_types`, `capture`.
   - **`traverse`** is backed by a new **`GraphService.neighbors`** one-hop primitive (rel
     filter, both directions, **cursor-paginated**) — the *same* primitive M7's
     `GET /nodes/{id}/neighbors` map endpoint reuses (built once here). **`build_context`**
     bundles `get_node` + neighbors and reaches **depth ≤ 2** via iterated one-hop with hard
     per-hop fanout caps (config); deep interactive expansion stays M7's job.
   - **Surface format = LLM-optimized Markdown**, rendered at the MCP boundary (a thin
     presentation serializer — no business logic; DTOs unchanged). Chosen over JSON for token
     efficiency (no per-record re-keying on hub edge lists / result lists), native LLM parsing,
     and **cross-model** fit (Claude **and** GPT, ruling out Claude-only XML framing). **IDs
     rendered verbatim and labeled** so the model chains into `get_node`/`traverse`/
     `build_context` with no precision loss. Node bodies are already markdown → near-free,
     lossless. **Hub guard**: inline edges capped at a config N with an overflow pointer
     ("N more; use `traverse`"). Optional JSON `structuredContent` dual-output **deferred to
     backlog** (avoids paying tokens twice when a client forwards both to the model).
   - **Usage guidance to the connected LLM**: the MCP `initialize` **`instructions`** capsule
     (authored, static, ~250–400 tokens: what the brain is · the six tools + when to use each ·
     the efficient `search`→`build_context`→`capture` loop · temporal filters · the
     research-via-MCP pattern), **rich per-tool descriptions**, **annotations** (`readOnlyHint`
     on reads, a write marker on `capture`), and an invokable **research-via-MCP Prompt**
     ([ADR-033](033-external-inspirations-obsidian-second-brain.md) #6 — documented "at M5"):
     query the graph for what's known → find gaps → research externally → `capture` the
     distillate with source refs. This authored *usage* capsule is **distinct from** the
     derived *identity* capsule (#5).

4. **`capture` over MCP.** Threads a **`source`** through capture creation (`web` default,
   `mcp` here; later `telegram`/`slack`) → node frontmatter `source: mcp` + `agent_runs` (so
   MCP captures are **visible in activity**). **Burst queue**: an asyncio semaphore bounds
   **MCP-source** captures to N in-flight (config `mcp_capture_burst_limit`; beyond N they
   wait — ADR-031 #1); UI captures stay immediate. **Fast ack return** (`capture_id` + status
   — the burst queue makes blocking on the organize call untenable; the tool description tells
   the LLM to `search` for confirmation). Reads are **not** logged as runs (high-volume, would
   flood activity; read telemetry is at most an M8 concern).

5. **Identity capsule** (refines [ADR-033](033-external-inspirations-obsidian-second-brain.md)
   #1 — `insight` nodes barely exist at M5, so an insight-only capsule would ship hollow):
   - **Broadened source** — a blend of high-salience entity-profile **hubs** (top by graph
     degree, from `node_profiles`), **recent memories**, and `insight` nodes **when present**;
     naturally enriches as M6/M10 produce insights. **300-token** budget, distilled on the
     `conspect` routing tier, authored prompt.
   - **Storage**: a derived-tier blob in **`app_settings`** (`identity_capsule`: text +
     `generated_at` + source node refs) — rebuildable, no new table, rule-1 clean.
   - **Refresh**: a nightly **`identity-capsule-refresh`** job on the APScheduler roster (the
     sleep cycle) + an on-demand admin trigger; `build_context` serves the last-generated
     capsule (cheap read; omitted if absent, never generated inline on the hot path).
   - **Serving**: `build_context` **level-0** (per ADR-033) **and** a read-only MCP resource
     **`identity://me`** (up-front grounding without picking a node). **Also wired into the
     existing M4 chat system prompt in M5** — the capsule is only *built* here, so M5 is where
     in-app chat finally consumes it (the compounding payoff lands the same day for internal
     chat and external LLMs).

6. **Accept gate = a real Claude connector** (mobile app / claude.ai web) live against
   `braindan.cc/mcp`: OAuth approve → `capture` → organized node; `search`/`get_node`/
   `traverse`/`build_context` (capsule L0) answer a question; **revoke-all locks it out**; MCP
   capture visible in activity. **ChatGPT is a fast-follow verification before M6** (may need
   thin `search`/`fetch` tool aliases for its deep-research connector; its quirks must not
   block M5's close).

## Rationale

- Self-hosted OAuth keeps the whole surface single-user-simple and vendor-free (matches the
  ADR-003 single-service, self-hosted ethos + secrets discipline), reusing auth primitives
  already trusted; `authlib` covers only the fiddly protocol crypto.
- Markdown-at-the-boundary optimizes the *only* thing that reaches the model (text), without
  touching the single service layer — presentation, not logic.
- Building `GraphService.neighbors` and the capsule here pays forward: M7 reuses the traverse
  primitive; M4 chat gains identity grounding immediately.

## Consequences

- M5 ships as **six tasks** (08 §M5): traverse primitive + `build_context` core · identity
  capsule + M4 wiring · OAuth 2.1 AS + migration · MCP server + tools + rendering + instructions/
  prompt + capture source/burst · deploy + infra · live Accept + OAuth-focused security review.
- New migration: MCP OAuth **client** + **token** tables; a capture **`source`** column.
- New config: `mcp_capture_burst_limit`, `build_context` fanout/depth caps, inline-edge cap,
  identity-capsule budget/source knobs, `MCP_TOKEN_HMAC_SECRET` (provisioning; **replaces** the
  static "MCP bearer-token secret" of 07-infra — the agent never handles the value).
- The independent review at the Accept boundary is **scoped to hammer the OAuth/security
  surface** (DCR abuse, PKCE, token leakage, the consent gate, no secret in git/logs) on top of
  the usual acceptance/ADR/invariant check.
- ❌ Rejected: static bearer only (fails the mobile-app/web surfaces the user requires);
  external IdP for OAuth (third-party in a private brain's auth path, needless for single-user);
  a dedicated `mcp.` subdomain (new vhost/DNS/cert for zero gain — against ADR-013); JSON as the
  primary surface format (token-heavy, re-keyed); dual JSON `structuredContent` now (double
  token cost); per-connector revocation in M5 (deferred to M8); read/write scope split (deferred).
