# ADR-063 — Groq vision primary: Scout decommissioned → qwen3.6-27b

**Status:** Accepted 2026-07-19 · **Supersedes** the `groq_vision_model` id in
[ADR-057](057-multimodal-media-ingestion-substrate.md) §4 and [ADR-062](062-chat-screenshot-self-attribution.md)
§4 (the two-tier `vision` chain — Groq primary, Nebius fallback — otherwise stands) ·
**Found by** the M9.7 T7 live drill.

## Context

M9.7 deployed. On the first live screenshot capture of T7, the vision derive's Groq primary
returned a hard error and fell back to the Nebius 72B (which produced a clean transcript). The
new live run-log (M9.7 T2/T3) surfaced the exact cause:

```
chat model meta-llama/llama-4-scout-17b-16e-instruct (groq) unavailable:
groq chat failed: Client error '404 Not Found' for url '.../openai/v1/chat/completions'
```

A **404 on the model** — Groq **decommissioned `meta-llama/llama-4-scout-17b-16e-instruct`**
(verified against Groq's live docs, which now list a single documented vision model). The
recorded vision primary (ADR-057 §4, seeded 2026-07-18) is therefore not "weak on dense
screenshots" as ADR-062 §4 hedged — it simply no longer exists, so **every** media derive was
paying a failed Groq call + latency before falling back to the paid Nebius 72B.

ADR-062 §4 already made the vision primary evidence-driven and revisable ("promote the 72B only
if measurement says the 17B can't hold §2"). The evidence is now "the 17B is gone" — but the
replacement is a model §4 didn't name, so it gets its own ADR rather than a silent config edit.

## Decision

**Groq vision primary `meta-llama/llama-4-scout-17b-16e-instruct` → `qwen/qwen3.6-27b`.** Groq's
current free multimodal VLM (dedicated model page; 20MB / ≤5 images per request). It is chosen
over simply promoting the Nebius 72B because:

* **Free-tier cost posture preserved** (the ADR-057 §4 "Groq-first" intent — user call). The 72B
  is paid Nebius; keeping a free Groq primary avoids paying on every derive.
* **Same model family as the fallback that just worked.** qwen3.6-27b is a Qwen VLM, like the
  Nebius `Qwen2.5-VL-72B-Instruct` that cleanly held ADR-062 §2's per-message format in the drill
  — so it is the free option most likely to hold that contract.

The `vision` chain shape is unchanged: **`qwen/qwen3.6-27b` (Groq, primary) → `Qwen/Qwen2.5-VL-72B-Instruct`
(Nebius, fallback)**. Config touched in three places (the runtime value is `deploy/defaults.env`,
which overrides the code default): `deploy/defaults.env`, `deploy/.env.example`,
`server/app/config.py` (`groq_vision_model` scalar + `vision_chain` seed).

**Still evidence-gated (ADR-062 §4 A/B, corrected).** The §4 A/B now runs `qwen/qwen3.6-27b`
(Groq, free) vs `Qwen/Qwen2.5-VL-72B-Instruct` (Nebius, paid) on the same real screenshot via the
Settings vision-group flip. The 72B is promoted to primary only if qwen3.6-27b can't hold §2.
Two watch-items during the A/B: (1) a **saved runtime routing override wins over the config seed**
— the T7 tester flipped Settings to the 72B during the failed pass, so the new Groq primary only
takes effect once the vision group is re-set in Settings → Models; (2) qwen3.6-27b has a
**thinking mode** — watch for reasoning-preamble leakage in the derived text and disable it if it
appears (the derived text is the byte-parity reprocess replay source, so it must stay clean prose).

## Consequences

* Media derivation has a working free primary again; no more guaranteed fail-then-fallback on
  every capture.
* Prompt/routing contracts are untouched: ADR-062 §2 (vision text format) and §3 (organizer v9)
  are model-agnostic, so this is purely a model-id swap. `reprocess-all` byte-parity holds per raw
  input version (a newly-derived screenshot's text changes with the model, as with any prompt/model
  bump — ADR-042).
* The M9.7 T7 migration (ADR-062 §5, rederive existing screenshots) now re-derives under whichever
  model the A/B selects — do the A/B first, then the migration.
* If Groq later decommissions qwen3.6-27b too, this is a one-line config swap; the fallback 72B
  keeps vision working in the interim (rule 3 — fallback resolution is recorded, never swallowed).
