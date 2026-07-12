# Personal Second Brain — Documentation

This repository is the **single source of truth for product and architecture decisions**.
It lives outside the code on purpose: the code repo (`../second-brain/`) contains only
implementation plus a `CLAUDE.md` that points here.

**Status:** design approved 2026-07-12 (grilled decision-by-decision). Implementation not started.

## Reading order

| Doc | Contents |
|---|---|
| [00-vision.md](00-vision.md) | Why the system exists, principles, planes, non-goals, success criteria |
| [01-architecture.md](01-architecture.md) | High-level architecture: PWA client, VPS service, agents, storage |
| [02-data-model.md](02-data-model.md) | Vault layout (planes), note frontmatter contract, database schema |
| [03-api.md](03-api.md) | HTTP API contract (the only seam between web and server) |
| [04-pipelines.md](04-pipelines.md) | Capture, ingestion, indexing, chat, analysis pipelines + scheduling |
| [05-connectors.md](05-connectors.md) | Connector contract, Slack connector spec, deferred connectors |
| [06-web-app.md](06-web-app.md) | PWA screens, design language (premium, animated), auth UX |
| [07-infrastructure.md](07-infrastructure.md) | VPS, Docker Compose, Caddy, Cloudflare, CI/CD, secrets, backups |
| [08-implementation-plan.md](08-implementation-plan.md) | Phased delivery with acceptance criteria |
| [adr/](adr/) | Architecture Decision Records — the *why* behind every choice |
| [templates/CLAUDE.md](templates/CLAUDE.md) | Ready-to-copy implementation rules for the code monorepo |

## Development workspace layout (local machine)

```
PersonalSecondBrain/          # workspace folder, not a repo
├── second-brain-docs/        # THIS repo — documentation
├── second-brain/             # code monorepo: server/ + web/ + deploy/
└── ObisidanVault/            # local dev vault (scratch, not canonical)
```

Production vault lives on the VPS (see [ADR-001](adr/001-vault-on-vps-with-git-backup.md)).

## Rules of this repo

- Behavior changes in code **must** be reflected here first or alongside — docs are the contract.
- New architectural choices get a new ADR; existing ADRs are never edited, only superseded.
- Docs are written to be directly ingestible by an AI implementer (Claude Code).
