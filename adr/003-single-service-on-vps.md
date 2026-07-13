# ADR-003: Single FastAPI service on a Hetzner VPS (Docker Compose), no PaaS, no worker fleet

**Status:** Accepted · 2026-07-12
**Terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md), 2026-07-13):** read
"vault" as **graph store**; the single-service-on-VPS decision stands unchanged (the M5 MCP server
is a surface inside the same service, [ADR-028](028-one-service-layer-mcp-peer-surface.md)).
**Tier updated by:** [ADR-015](015-compute-tier-hetzner-cx23.md) — the specific plan is now
**Hetzner CX23** (CX22 retired in Hetzner's 2026 refresh); this ADR's core decision
(single service on an always-on VPS, no PaaS/worker fleet) stands unchanged.

## Context
Everything (API, pipelines, scheduler, ingestion agents, vault) must run 24/7 without the
user's machine. Candidates: micro-VPS with one service, PaaS (Fly.io/Railway), separate
worker processes/containers, Supabase Edge Functions for agents.

## Decision
One Hetzner CX22 (~5€/mo, 2 vCPU/4GB), Docker Compose with two containers: `caddy`
(TLS + static PWA + /api proxy) and `api` (FastAPI + APScheduler + agents in one
process). Deploy via GitHub Actions (push to main → SSH → pull → compose up --build).

## Consequences
- ✅ Fixed ~5€/mo vs ~2× variable on PaaS; persistent disk for the vault; the Claude CLI
  OAuth session survives deploys (volume) — on PaaS ephemeral containers this is fragile.
- ✅ Async I/O-bound workload (API calls, file writes) fits one process comfortably at
  n=1 user; `ENABLE_SCHEDULER` flag keeps a future second instance safe.
- ⚠️ In-flight background tasks die on restart — recovered via persisted pipeline state,
  retry endpoint, nightly rescan.
- ⚠️ Server administration is ours; mitigated by scripted provisioning (<30min rebuild).
- ❌ Rejected: PaaS (cost, ephemeral FS vs vault+CLI creds); per-agent containers (RAM
  lux on 4GB, no isolation need yet); Supabase Edge Functions for agents (see ADR-008).
