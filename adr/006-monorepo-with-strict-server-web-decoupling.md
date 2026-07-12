# ADR-006: Code monorepo with strict server/web decoupling (split-ready)

**Status:** Accepted · 2026-07-12

## Context
One developer, an API and a PWA that evolve together. Separate repos mean duplicated PRs
and version sync for no benefit now — but the user requires that a future split into two
repos be trivial ("this is a must").

## Decision
One repo `second-brain/` with `server/` (FastAPI), `web/` (React PWA) and `deploy/`.
Decoupling rules, enforced as law:

- `server/` and `web/` share **nothing** but the HTTP contract (docs/03-api.md, OpenAPI).
  No shared code, no cross-imports, no shared build tooling.
- Each has its own dependency manifest, lint config, tests, and README; each builds and
  runs standalone from its own directory.
- CI runs per-directory pipelines (path filters), not one blended pipeline.
- Web knows exactly one server fact: the base URL (env). Server knows zero web facts
  (Caddy does the static serving).

**Split procedure (documented test of the rule):** `git filter-repo` per directory (or
plain copy), add web CORS config, point CI at two repos. Target effort: ~1 hour.

## Consequences
- ✅ Single-commit features across API+UI now; painless separation later.
- ✅ The discipline doubles as clean architecture: the OpenAPI contract stays honest
  because nothing can bypass it.
- ⚠️ Requires vigilance in review (no "just import that type from server") — codified in
  CLAUDE.md; types on the web side are generated from OpenAPI, not shared.
