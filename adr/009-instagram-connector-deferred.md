# ADR-009: Instagram connector deferred to v2

**Status:** Accepted (deferral) · 2026-07-12

## Context
Instagram DMs are a desired memory source. Reality of Meta's Graph API: requires a
professional (business/creator) account, a Meta developer app, and app review for
messaging permissions; even then the Messaging API exposes conversations of the
*professional account inbox* (people messaging "the business"), with constraints on
history depth — it is not a personal-DM archive API. Feasibility for "my personal
conversation history" is unproven.

## Decision
Defer. v1 ships the connector contract (ADR-008) and the Slack connector only. Instagram
gets a time-boxed feasibility spike before any commitment: can a professional account +
Graph API (or, alternatively, periodic official data export ingestion) actually surface
the conversations that matter?

## Consequences
- ✅ v1 isn't held hostage by Meta app review; the contract already accommodates the
  connector when it proves feasible.
- ✅ The spike has a concrete fallback path to evaluate: Instagram's official "Download
  your information" export, ingested as files — no API approval needed, but not automatic.
- ⚠️ Until then, Instagram conversations are absent from the brain.
