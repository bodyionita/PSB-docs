# ADR-017: Origin TLS via Cloudflare Origin CA; cert+key delivered by CI

**Status:** Accepted · 2026-07-12
**Relates to:** [007 auth: password/session behind Cloudflare](007-auth-password-session-cloudflare.md) · [013 web stays on VPS, single origin](013-web-stays-on-vps-single-origin.md) · [016 secrets via GitHub Actions; CI renders `deploy/.env`](016-secrets-via-github-actions-ci-renders-env.md). **Updates** [07-infrastructure](../07-infrastructure.md) §Hosting and §Secrets.

## Context
Cloudflare fronts the origin with **proxied DNS** (orange-cloud) and **Full (strict)** TLS,
so the origin must present a certificate Cloudflare's edge trusts. Caddy terminates TLS on
the box. The M0b Caddyfile used a bare `{$BRAINDAN_DOMAIN}` site block, which triggers
Caddy's **automatic ACME** (HTTP-01 / TLS-ALPN). Behind this topology that cannot succeed:

- **HTTP-01** needs inbound **:80**, but the firewall opens **22 + 443 only** ([07-infra]
  (../07-infrastructure.md) / `provision.sh`), and Cloudflare would proxy :80 anyway.
- **TLS-ALPN-01** rides :443, which **Cloudflare terminates at the edge** — the challenge
  never reaches Caddy.

So an explicit cert strategy is required before Caddy can serve. Two options that both work
under Full (strict): **(a) Cloudflare Origin CA** — a Cloudflare-issued origin certificate
(validity up to 15 years), trusted only by Cloudflare's edge; **(b) Caddy DNS-01 ACME** — a
real Let's Encrypt cert obtained via a DNS-01 challenge using a Cloudflare API token.

## Decision
1. **Use a Cloudflare Origin CA certificate** for origin TLS. Caddy references it with an
   explicit `tls <cert> <key>` directive (no automatic ACME, no custom image, no renewal —
   the cert is static for its 15-year life).
2. **The cert + private key are delivered by CI**, reusing the ADR-016 render-and-scp path.
   Two new **GitHub `production` secrets** hold the PEM material — `ORIGIN_CERT_PEM` and
   `ORIGIN_KEY_PEM` — entered once in the UI by the human, never seen by the agent, never in
   git. The deploy job writes them to `deploy/origin.crt` / `deploy/origin.key` and `scp`s
   them to the VPS alongside `.env` (**mode 600**). Caddy mounts the two files read-only and
   points `tls` at them.
3. **Firewall/ports:** origin needs **:443 only**. Drop the `:80` publish from the caddy
   service; Cloudflare handles HTTP→HTTPS at the edge (“Always Use HTTPS”). UFW stays 22+443.

## Consequences
- ✅ Canonical pairing with Full (strict); no ACME machinery, no custom Caddy build, no cert
  renewal to operate or monitor.
- ✅ Smaller secret blast radius than DNS-01: an Origin CA key is trusted **only** by
  Cloudflare's edge, versus a DNS-edit API token that can rewrite the whole zone.
- ✅ DR stays one-button — "provision the box + trigger deploy" renders `.env` **and** the
  cert/key; nothing hand-placed (consistent with ADR-016 and the < 30-min DR target).
- ⚙️ +2 secrets beyond the ADR-016 "irreducible ~9" (the cert is public, but kept with the
  key as PEM files for symmetry). Accepted; static, low-churn.
- ⚠️ The origin cert is **not publicly trusted** — a browser hitting the origin IP directly
  gets a cert error. This is intended: the architecture never serves the origin off-Cloudflare
  (proxied DNS, origin IP hidden — ADR-013/007).
- ↩️ **Reopen trigger:** a need to serve the origin **without** Cloudflare in front (grey-cloud,
  direct access, or a second non-CF edge) → switch to **DNS-01 ACME** for a publicly-trusted,
  auto-renewing cert (custom Caddy image + a scoped `Zone:DNS:Edit` token).
- ❌ Rejected: Caddy automatic ACME (cannot complete under proxied Full-strict, as above);
  DNS-01 now (larger-blast-radius token + custom image, unnecessary while CF is always in front).
