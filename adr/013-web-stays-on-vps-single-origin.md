# ADR-013: Web PWA stays on Caddy-on-VPS, single origin (Vercel/Pages/Netlify rejected)

**Status:** Accepted · 2026-07-12
**Terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md), 2026-07-13):** read
"vault" as **graph store**; decision unchanged.
**Relates to:** [003 single-service-on-VPS](003-single-service-on-vps.md) · [006 monorepo-decoupled](006-monorepo-with-strict-server-web-decoupling.md) · [007 auth-cookie](007-auth-password-session-cloudflare.md)

## Context
The default instinct was to host the web PWA on Vercel (familiarity), moving off only if
another option were **clearly cheaper**. Options priced for this exact workload — a
single-user static Vite `dist/` (HTML/CSS/JS + service worker, no SSR, negligible
bandwidth):

| Option | Monthly cost |
|---|---|
| Vercel (Hobby) | €0 |
| Cloudflare Pages | €0 |
| Netlify | €0 |
| Keep Caddy-on-VPS | €0 marginal (VPS already running; static files ≈ 0 RAM) |

Cost is therefore a **tie** — nothing is "clearly cheaper" than the €0 already paid for
Caddy. The decision axis is not price but **cross-origin auth complexity**:

- Web on the VPS (Caddy) → web `/` and API `/api` are **same-origin** → ADR-007's
  `SameSite=Lax` httpOnly cookie works untouched, no CORS.
- Web on Vercel/Pages/Netlify (a different origin) → the session cookie becomes
  **cross-site** → forces `SameSite=None; Secure`, CORS-with-credentials, and cookie-domain
  management — real ADR-007 rework for €0 saved.

Two facts closed the "but domains/subdomains are easier on Vercel" argument:
1. **The API forces a custom domain regardless.** The VPS is a bare IP; to be PWA-usable it
   needs HTTPS with a valid cert and, per ADR-007, a hostname inside a Cloudflare zone
   (Cloudflare cannot proxy a bare IP; no public cert for `https://<ip>`). So a domain is
   driven by the API. Vercel's free `*.vercel.app` subdomain does not remove that, and
   *using* it while the API sits on `api.<domain>` would itself be cross-site.
2. Once a domain is owned for the API, the web belongs on the **same registrable domain** to
   stay same-site — which is exactly what staying on the VPS gives for free.

"Deploy the compute to Supabase instead" was re-confirmed as rejected (same wall as ADR-003):
Supabase remains the **database** (ADR-002), but Supabase Edge Functions are ephemeral,
stateless, short-timeout, and Deno — they cannot hold the persistent `claude login` OAuth
credential store, cannot run the Node Claude CLI, cannot host the vault git working tree, and
cannot run the up-to-two-hour nightly agent window. Architectural mismatch, not cost.

## Decision
Keep the web PWA served by **Caddy on the VPS**, **single origin**:
`https://<domain>/` serves `web/dist`, `https://<domain>/api` proxies to FastAPI. One
hostname, one Cloudflare proxied A-record → VPS, one TLS mechanism (Caddy ACME), one
first-party `SameSite=Lax` cookie. No move to Vercel/Pages/Netlify.

## Consequences
- ✅ Zero change to ADR-007: cookie stays httpOnly `SameSite=Lax`, first-party. No CORS.
- ✅ ADR-003's web-serving half is **affirmed, not superseded**.
- ✅ CI/CD stays unified (push to main → SSH → `docker compose up --build`); no host-owned
  web-deploy path to reconcile against ADR-006. Split-readiness (ADR-006) is unaffected —
  web still knows exactly one server fact (the base URL).
- ✅ Total added cost for web hosting: €0.
- ⚠️ A custom domain is still required (driven by the API); its name is chosen at
  provisioning time (README cold-start open item).
- ↩️ **Reopen trigger:** if the web is ever moved off the VPS, revisit with the
  `app.<domain>` / `api.<domain>` **subdomain split** + `SameSite=None; Secure` + CORS —
  captured here so it need not be re-derived.
- ❌ Rejected: Vercel / Netlify / Cloudflare Pages (no cost benefit + forces cross-origin
  cookie rework); subdomain split now (buys a future-move option we resolved not to take);
  Supabase Edge Functions for compute (stateful/always-on constraints — see ADR-003).
