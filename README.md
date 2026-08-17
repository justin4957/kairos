# Kairos

**The missing continuous-learning layer for the ThoughtMode Works ecosystem.**

Kairos is not a new reasoning engine. Dialectica already has one (GODE v5.2, dialectical
synthesis). Delphi already has one (GRP thought cards, 16-parameter reasoning profiles).
Dialogica already knows how to run debates and cohorts against them. What none of the
three have is a *world* to reason about that updates itself.

Kairos closes that loop. It pulls from open sources on a schedule, maintains a
versioned, timestamped belief store of entities/actors/material conditions, drives
Dialectica's synthesis engine and the Uncertainty Department's Ideological Materialism
protocol against that store instead of one-shot prompts, and scores every resulting
forecast against what actually happened. That last step is what makes the system
"continuously trained" in the only sense that's actually achievable without retraining
model weights: the analytical *moves* get reweighted by their track record, even though
the underlying LLM stays frozen.

## Why this exists

> "continuously trained world models" in the literal sense — retraining model weights
> on incoming data — isn't feasible or even desirable at this scale... But the
> functional goal *is* achievable by moving the "continuous learning" out of the weights
> and into a persistent state layer. The frontier model stays frozen; the *world model*
> is a structured, versioned belief store that updates continuously.

Three of the four pieces this requires already exist in this workspace, spread across
three separate apps that have never been wired to live data:

| Layer | Status before Kairos | Where it lives |
|---|---|---|
| Analytical engine | ✅ Built | Dialectica (GODE synthesis), Delphi (GRP cards) |
| Domain framework | ✅ Written, not automated | `rand_briefings` — Ideological Materialism, ~40 essays |
| Shared infra (LLM/GRP/Debate) | ✅ Built, partially adopted | `packages/core` (`thought_mode`) |
| **Ingestion layer** | ❌ Does not exist | — |
| **World-state store** | ❌ Does not exist | — |
| **Calibration loop** | ❌ Does not exist | — |

Kairos is scoped to build exactly the missing three, and to treat Dialectica's existing
`/api/v1/analyze`, `/api/v1/analyze/pipeline`, and `/api/v1/profiles` endpoints as the
analytical backend it drives, rather than reimplementing synthesis logic.

## Architecture in one picture

```
 open sources (RSS, gov releases,        ┌─────────────────┐
 prediction markets, ACLED-style)  ─────▶│  Ingestion       │
                                          │  (Kairos)        │
                                          └────────┬─────────┘
                                                   │ normalized events
                                                   ▼
                                          ┌──────────────────┐
                                          │  World-State      │
                                          │  Store (Kairos)    │◀──── analysts / corrections
                                          │  entities, IM      │
                                          │  Objects, GRP      │
                                          │  profiles, claims  │
                                          └────────┬───────────┘
                                                   │ current snapshot
                                                   ▼
                        ┌──────────────────────────────────────────┐
                        │        Analytical Engine (existing)        │
                        │  Dialectica GODE synthesis  │  Delphi GRP  │
                        │  IM 8-step protocol runner (new, thin)     │
                        └──────────────────┬───────────────────────┘
                                           │ forecasts w/ probability + resolution date
                                           ▼
                                  ┌──────────────────┐
                                  │  Calibration Loop  │
                                  │  (Kairos)          │
                                  │  Brier scoring on   │
                                  │  resolution, move    │
                                  │  reweighting          │
                                  └──────────────────┘
```

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full design and
[`docs/WHITEPAPER.md`](docs/WHITEPAPER.md) for the conceptual case, prior art
survey, and open research questions.

## Relationship to the rest of ThoughtMode Works

- **Dialectica** — Kairos calls its REST API (`POST /api/v1/analyze/pipeline`) as the
  primary analytical backend. Kairos does not fork or reimplement GODE.
- **Delphi** — Kairos can request GRP thought cards for entities in the world-state as
  a secondary/comparative analytical lens, and stores resulting profiles back as
  world-state attributes.
- **Dialogica** — future integration point: world-state-derived scenarios could seed
  Dialogica voting sessions / simulated cohorts for stress-testing forecasts against
  simulated stakeholder reactions. Not in scope for v1.
- **`packages/core` (`thought_mode`)** — Kairos depends on it directly for LLM
  provider abstraction (`ThoughtMode.LLM`) and GRP schema (`ThoughtMode.GRP`), rather
  than adding a fourth implementation of either.
- **`rand_briefings`** — the Ideological Materialism framework document
  (`UD-FRAMEWORK-IM-001`) is Kairos's second analytical backend, codified as a
  pipeline. Kairos does not touch `rand_briefings`'s Phoenix app; it only consumes the
  framework's published methodology as a spec to implement against.

## Status

Design phase. This repository currently contains architecture and whitepaper
documents only — no runtime code yet. See the roadmap in `docs/ARCHITECTURE.md`
§8 for the phased build plan.

## License

TBD — matches whatever license posture the rest of ThoughtMode Works settles on.
