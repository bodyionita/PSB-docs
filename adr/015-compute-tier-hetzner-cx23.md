# ADR-015: Compute tier pinned to Hetzner CX23 (4GB); free/sub-4GB tiers rejected

**Status:** Accepted · 2026-07-12
**Terminology note ([ADR-026](026-graph-native-storage-obsidian-removed.md), 2026-07-13):** read
"vault" as **graph store**; decision unchanged.
**Updates the tier in:** [ADR-003](003-single-service-on-vps.md) (CX22 → CX23). ADR-003's
core decision — single FastAPI service on an always-on VPS, no PaaS, no worker fleet —
stands unchanged.
**Relates to:** [ADR-004](004-provider-registry-claude-primary-nebius-fallback.md) (Claude CLI drives the RAM floor) · [ADR-013](013-web-stays-on-vps-single-origin.md) (web on the box) · [ADR-014](014-vault-history-durability.md) (durability makes the host disposable)

## Context
Budget is tight; the goal was the smallest, cheapest **always-on** host meeting the hard
constraints (persistent Claude CLI OAuth store, nightly 03:00–05:00 agent window, vault git
on disk). Two things reshaped the choice:

- **Topic B (ADR-014) made the host nearly stateless.** The vault is re-clonable (GitHub/R2
  WORM), the DB is off-box (Supabase + R2 `pg_dump`), `/srv/data` audio syncs to R2, and the
  Claude creds are one `claude login` away. So **host loss = a <30-min reprovision, not data
  loss** — we can shop on price, trading only availability/hassle, never memory.

### Findings
- **RAM floor is driven by the Node Claude CLI** (ADR-004), which spawns on *every*
  `claude-max` chat and across the nightly window, bursting ~300–600MB+ on top of a ~500MB
  baseline (FastAPI ~150 + Caddy ~40 + Ubuntu/Docker ~300). **Comfortable floor ≈ 2GB;
  512MB/1GB is not viable** (CLI OOM, worst during the nightly window); **swap is
  emergency-only** (Node GC thrash on swap wrecks exactly that window). 4GB gives headroom
  and builds the Python+Node image on-box (`compose up --build`) without OOM, so ADR-003's
  deploy flow is unchanged.
- **ARM is fully viable** (multi-arch `python:3.12-slim`, npm CLI on ARM64) but Hetzner **CAX
  (ARM) capacity was not selectable in-region** at provisioning time → x86.
- **Hetzner's entry tier is now 4GB** (CX11/CX21 retired); **CX23** = 2 vCPU / 4GB / 40GB
  NVMe / 20TB, Germany/Finland, ~€5.49/mo + €0.50 IPv4.
- **Web-on-VPS (ADR-013) is not a RAM driver** (static serving ≈ 0), so it doesn't affect the
  tier.
- **Oracle Cloud Always Free** (halved to 2 OCPU/12GB ARM in June 2026) was **rejected**: its
  idle-reclaim policy (95th-percentile CPU < 20% over 7 days) targets our bursty, mostly-idle
  workload almost by design, and free-tier support recourse is weak. ADR-014 keeps data safe
  through a reclaim, but not uptime on the daily-driver capture endpoint.

## Decision
**Hetzner CX23** (x86, 2 vCPU / 4GB / 40GB NVMe), EU region. All-in system cost **~€7/mo**:
VPS ~€5.49 + IPv4 €0.50 + domain ~€1 (amortized); web, vault/DB backups (R2 free tier),
Supabase, and Cloudflare are all €0.

## Consequences
- ✅ Reliable EU always-on host with headroom for the Node CLI and on-box builds.
- ✅ Cheapest *reliable* option actually provisionable; €0 alternatives rejected on
  availability risk, not data risk (ADR-014 covers the data).
- ✅ ARM re-choosable later at ~zero effort if CAX frees up — the image is already multi-arch.
- ↩️ **Reopen triggers:** Hetzner pricing shifts materially; workload RAM outgrows 4GB; or a
  reliable sub-€5 alternative appears. If build RAM ever becomes the constraint, decouple by
  building in CI → push to a registry → VPS pulls (no on-box build).
- ❌ Rejected: Oracle/other free tiers (reclaim risk on the daily-driver host); sub-4GB
  (Node CLI OOM in the nightly window); other budget VPS (marginal savings, worse
  reliability/latency to Europe).
