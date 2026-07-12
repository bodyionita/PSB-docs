# ADR-007: Auth = single password + httpOnly sessions, behind Cloudflare proxy

**Status:** Accepted · 2026-07-12

## Context
Single-user system exposed on the public internet, holding an entire life's memory.
Candidates: in-app password+session, Supabase Auth (JWT), edge identity
(Cloudflare Access / Tailscale).

## Decision
In-app auth: one password (argon2/bcrypt hash in env), `POST /auth/login` issues a
long-lived httpOnly Secure session cookie (hashed token in `auth_sessions`, revocable).
Login rate-limited. The VPS sits behind Cloudflare proxied DNS (TLS at edge, origin IP
hidden). Cloudflare Access is a pre-approved future upgrade, not built now.

## Consequences
- ✅ Zero third parties in the hot path of capture (phone lock-screen → capture must
  never depend on an external IdP being reachable).
- ✅ Sessions are inspectable/revocable in the DB; PWA-friendly (cookie just works).
- ⚠️ We own rate-limiting and session hygiene — small, well-understood code, but ours.
- ❌ Rejected: Supabase Auth (multi-user machinery for one user); Tailscale-only (breaks
  the "any device, instantly" PWA promise — requires client install everywhere).
