# Kairos: Architecture

**Status**: Design
**Depends on**: Dialectica (GODE API), `packages/core` (`thought_mode`), the
Ideological Materialism framework (`UD-FRAMEWORK-IM-001`)

---

## 1. Scope

Kairos owns three things and three things only:

1. **Ingestion** — scheduled pulls from open sources into normalized events.
2. **World-State Store** — a versioned, timestamped belief graph those events update.
3. **Calibration Loop** — forecast logging, resolution tracking, Brier scoring, and
   feedback into which analytical moves get trusted.

Kairos explicitly does **not** own dialectical synthesis logic, GRP parameter
extraction, or debate simulation — those already exist in Dialectica and Delphi and
are called as services. Duplicating them here would recreate the exact
fragmentation (three separate GRP implementations, three separate LLM clients) that
`MODULAR_ROADMAP.md` and the Phase 0 core-extraction were written to stop.

## 2. Non-goals

- Retraining or fine-tuning any model on ingested data. The frontier model(s) stay
  frozen; only the world-state and the move-reweighting table change.
- Replacing Dialectica's Studio UI or Dialogica's voting UI. Kairos is a backend
  service; if it needs a UI at all in v1, it's a read-only dashboard.
- Building a new GRP schema. Kairos consumes `ThoughtMode.GRP` as-is from
  `packages/core`.
- Guaranteeing forecast accuracy. The calibration loop's job is to *measure* and
  *report* accuracy honestly, including when it's bad — that's the whole value of
  logging every prediction before it resolves.

## 3. Domain model

Kairos's world-state store has to hold two different kinds of structure
simultaneously: the generic (entities, events, claims, confidence) and the
IM-specific (Ideological Objects, Horizons, Substrate configurations). Rather than
choosing one vocabulary, the schema keeps a thin generic core and lets IM- and
GRP-flavored data attach to it as typed extensions.

### 3.1 Generic core

```
Entity
  id              — UUID
  kind            — actor | institution | polity | movement | infrastructure
  name            — String
  aliases         — [String]
  first_seen_at   — DateTime
  last_updated_at — DateTime

Claim
  id              — UUID
  subject_id      — Entity.id
  predicate       — String            (e.g. "material_gravity", "controls_channel")
  object          — String | Entity.id | Number
  confidence      — Float 0.0–1.0
  source_ids      — [Source.id]       (provenance — every claim traces to ≥1 ingested event)
  observed_at      — DateTime          (when the underlying event happened)
  recorded_at      — DateTime          (when Kairos wrote this claim)
  superseded_by    — Claim.id | null   (claims are never mutated in place, only superseded)

Source
  id              — UUID
  origin          — String            (feed URL, dataset name, market identifier)
  origin_kind      — rss | gov_release | prediction_market | acled_style | manual
  fetched_at       — DateTime
  raw_payload_ref  — String            (pointer to stored raw payload, for audit)

Forecast
  id                — UUID
  claim              — String          (the actual prediction, natural language + structured)
  probability        — Float 0.0–1.0
  resolution_date     — Date
  resolution_criteria — String          (must be specific enough to score unambiguously)
  produced_by         — AnalyticalRun.id
  status              — pending | resolved_true | resolved_false | resolved_ambiguous | expired
  resolved_at         — DateTime | null
  brier_component      — Float | null   (computed at resolution)

AnalyticalRun
  id              — UUID
  engine          — :gode | :im_protocol | :grp_comparative
  engine_version   — String            (e.g. "GODE v5.2", "IM v1.0")
  world_state_snapshot_id — Snapshot.id (immutable pointer — what state produced this run)
  input_entities   — [Entity.id]
  raw_response      — Map              (full engine output, stored verbatim)
  started_at        — DateTime
  completed_at       — DateTime
```

### 3.2 IM extension (attaches to `Entity`/`Claim` when the subject is an
Ideological Object, Horizon, or Substrate)

Mirrors `UD-FRAMEWORK-IM-001` directly — no renaming, so analysts moving between the
framework document and the database aren't translating terms:

```
IdeologicalObject   (Entity subtype, kind = :ideological_object)
  essence               — String
  material_gravity      — Float        (Ψ, 0.0–1.0)
  transmission_profile   — Map          (channel → {fidelity, distortion_profile})

Horizon             (Entity subtype, kind = :horizon)
  population_ref         — Entity.id
  boundary_description    — String       (what's currently "outside" — updated, not replaced)

SubstrateStrategyObservation
  ideological_object_id   — Entity.id
  strategy                — sedimentary_dominance | substrate_insurgency
                             | substrate_capture | substrate_fragmentation
                             | ancestral_invocation
  evidence_claim_ids       — [Claim.id]
  observed_at              — DateTime

ConvergenceFieldAssessment
  population_ref           — Entity.id
  material_contradiction_present — Boolean
  contradiction_description        — String | null
  triggering_event_claim_id         — Claim.id | null
  assessed_at                       — DateTime
```

### 3.3 GRP extension

Not reimplemented — this is literally `ThoughtMode.GRP`'s existing profile shape
(5-category Dialectica variant, superset of Delphi's 4-category variant per the core
package's "hybrid schema" merge). Kairos stores the profile as an attribute on an
`Entity` with a `produced_by` pointer back to the `AnalyticalRun` that generated it, so
a GRP profile is never orphaned from the world-state snapshot and analytical engine
version that produced it.

### 3.4 Why append-only claims, not mutable entity fields

Two reasons, both non-negotiable for a calibration loop to mean anything:

- **Auditability.** A forecast made against "the world-state as of March 3" has to be
  reproducible after the world-state has moved on. `AnalyticalRun.world_state_snapshot_id`
  only works if claims are versioned, not overwritten.
- **Contradiction is data, not an error state.** Two sources can produce
  contradictory claims about the same entity. IM's own methodology (Material
  Contradiction, Section 2.3 of the framework) treats the *presence* of a
  contradiction as an analytically significant signal. Overwriting one claim with
  another would destroy exactly the signal the framework cares about most.

## 4. Ingestion layer

### 4.1 Source classes (v1)

| Class | Examples | Notes |
|---|---|---|
| RSS / news | Major wire services, regional outlets via RSS | Highest volume, lowest reliability per-item; entity/claim extraction via LLM |
| Government releases | Federal Register, congressional record, agency press releases | Structured or semi-structured; higher trust weight |
| Prediction markets | Public market APIs (e.g. Polymarket, Kalshi where accessible) | Gives *external* calibration baseline — see §6 |
| ACLED-style conflict/event data | Open conflict-event datasets | Structured, good for material-conditions claims |

No specialized/paywalled access in v1, matching the original design constraint: the
edge comes from structure and the calibration discipline, not from data no one else
has.

### 4.2 Pipeline shape

```
Source poll (scheduled, per-source interval)
  → raw payload stored (Source.raw_payload_ref) — nothing is extracted-then-discarded
  → extraction pass (LLM via ThoughtMode.LLM, structured-output prompted)
      → candidate Entities (new or matched against existing via name/alias fuzzy match)
      → candidate Claims, each tagged with source + confidence
  → dedup / merge pass (same event reported by 3 outlets → 1 claim, 3 sources)
  → write to World-State Store
```

Extraction confidence is a first-class field, not a filter threshold applied at
ingestion time — low-confidence claims are still stored (with low confidence), because
whether they matter is a question for the analytical engine and the calibration loop
to answer over time, not a decision to make permanently at ingestion.

### 4.3 What ingestion deliberately does not do

It does not attempt sentiment analysis, bias scoring, or truth adjudication on
individual sources. That's exactly the kind of judgment IM's own Convergence Field /
Material Contradiction analysis is designed to produce from *patterns* across claims —
building it into ingestion would duplicate the analytical engine's job with cruder
tools and no traceability.

## 5. Analytical engine integration

Kairos treats analytical engines as pluggable backends behind one interface:

```
AnalyticalEngine.run(world_state_snapshot, entities, mode) 
  -> {:ok, AnalyticalRun} | {:error, reason}
```

### 5.1 GODE backend (Dialectica)

Calls Dialectica's existing REST API directly — no code sharing, no forking:

- `POST /api/v1/analyze/pipeline` — primary integration point; runs a configured
  multi-stage GODE pipeline against a set of profile keys.
- `GET /api/v1/profiles/:key/counterpoint` — used to find the antithesis pole for a
  given Ideological Object before requesting synthesis, so Kairos doesn't have to
  reimplement dialectical opposition-finding.
- `POST /api/v1/sessions` + `POST /api/v1/sessions/:id/results` — Kairos opens a
  Dialectica session per analytical run so results are visible/inspectable in
  Dialectica's own Studio UI, not just in Kairos's database.

Kairos's job here is purely translation: world-state `Entity`/`Claim` data in,
Dialectica GRP-profile-shaped requests out; Dialectica's synthesis response back,
parsed into an `AnalyticalRun` and any `Forecast` records the synthesis output implies.

### 5.2 IM protocol backend (new, thin — implemented in Kairos)

This is the one piece of "engine" logic that lives in Kairos, because the IM 8-step
protocol (`UD-FRAMEWORK-IM-001` §III) has never been implemented as software — it
exists only as a document. It is deliberately kept thin: each step is a structured LLM
call over `ThoughtMode.LLM` with the world-state snapshot as context, not a novel
reasoning engine.

| Protocol step | Kairos implementation |
|---|---|
| 1. Domain Scoping | Query world-state for entities matching population/time-period filter; explicit Object-identity criteria stored on the `AnalyticalRun` |
| 2. Substrate Mapping | LLM pass over matched entities' claim history → `SubstrateStrategyObservation` candidates |
| 3. Transmission Ecology Assessment | Query `IdeologicalObject.transmission_profile` deltas over the snapshot window — this step is mostly world-state querying, not LLM generation, per the framework's own finding that Ecology changes are the highest-value leading indicator |
| 4. Convergence Field Identification | LLM pass producing `ConvergenceFieldAssessment` |
| 5. Strategy Identification | LLM pass producing `SubstrateStrategyObservation` per competing Object |
| 6. Temporal Profiling | Classifies claims into τ_c / τ_i / τ_p per the framework; stored as a `Claim` metadata field, not a new table |
| 7. Contingency Identification | LLM pass, explicitly prompted to identify where structural analysis is *not* predictive — output stored verbatim, never dropped, since this is a required output per the framework, not an optional caveat |
| 8. (Kairos addition, not in the original framework) Forecast Emission | Structured pass that turns steps 4–7's output into one or more scored `Forecast` records with resolution dates — the framework document stops at "structure detector, not deterministic predictor"; Kairos adds this step because calibration requires a falsifiable claim, and it's the piece the framework explicitly left as future work |

Step 8 is flagged in the table because it's the one place Kairos extends the IM
framework rather than just implementing it — worth being honest about in any writeup,
including the whitepaper.

### 5.3 GRP comparative backend (Delphi)

Lowest priority integration. Used when an analytical run wants a second, independent
reasoning-pattern lens on the same entity (Delphi's 4-category card vs. Dialectica's
5-category profile) — mainly useful for the calibration loop's move-reweighting (§6.3),
since disagreement between backends is itself a signal worth tracking.

## 6. Calibration loop

### 6.1 Forecast logging

Every `Forecast` is written at creation time with `status: pending`, before its outcome
is knowable. This ordering is enforced at the database/application level, not just by
convention — a forecast record cannot be created with a `resolved_*` status. This is
the single most important discipline in the whole system: it's the only thing that
makes "continuously trained" more than a metaphor.

### 6.2 Resolution

Resolution is intentionally not automated in v1. `resolution_criteria` is written to
be checkable by a human analyst against public record on `resolution_date`, and the
resolution pass is a manual review queue. Auto-resolution against ingested claims is a
listed v2 goal (§8), gated behind v1 proving the manual loop actually produces honest
scores instead of analysts unconsciously softening `resolution_criteria` after the fact.

### 6.3 Scoring and reweighting

Standard Brier score per forecast: `(probability - outcome)²`, outcome ∈ {0, 1}, lower
is better. Aggregated per:

- **Engine** (`gode` vs `im_protocol` vs `grp_comparative`) — are they differentially
  well-calibrated?
- **Protocol step / move** — for IM-protocol forecasts specifically, which step's
  output correlated with eventual accuracy? (E.g.: do forecasts that cited a
  Convergence Field assessment resolve better than ones that didn't?)
- **Domain** — political/economic/institutional domains likely calibrate
  differently; the loop should surface that rather than average it away.

Reweighting is a visible, inspectable table (`engine × domain → calibration
multiplier`), not a hidden adjustment — every forecast's raw probability and its
calibration-adjusted probability are both stored, so the adjustment itself remains
auditable and reversible.

### 6.4 External baseline

Where a Kairos forecast overlaps a live prediction-market question (ingested per
§4.1), the market price is logged alongside Kairos's forecast at the same timestamp.
This gives a free, continuously-updating external calibration baseline without Kairos
having to bootstrap its own long resolution history before it's useful.

## 7. What "continuously trained" cashes out to, concretely

Restating this plainly because it's the whole thesis of the project and it's easy to
let the phrase drift back into meaning "the model learns":

- The LLM weights (Claude, or whatever `ThoughtMode.LLM` provider is configured) never
  change.
- What changes continuously: the world-state (new claims, superseded old ones), the
  calibration multiplier table (§6.3), and — because Kairos always includes the
  current calibration multiplier in the context passed to the analytical engine on
  each run — the *effective* behavior of the system, even though no weights moved.

This is a meaningfully weaker claim than "the model learns," and the docs should never
imply otherwise. It is also the only version of the claim that's actually true.

## 8. Phased roadmap

**Phase 1 — World-state store, no ingestion, no engine calls.**
Schema from §3 implemented and migrated. Manual/seed-script entity and claim creation
only. Goal: prove the append-only-claims model is workable to query against before
building anything on top of it.

**Phase 2 — GODE backend integration.**
`AnalyticalEngine.run/3` with the Dialectica backend only (§5.1). Manually curated
world-state snapshots fed into Dialectica pipelines; results stored as
`AnalyticalRun` + `Forecast`. No calibration scoring yet — just prove the round-trip.

**Phase 3 — Calibration loop.**
Manual resolution queue, Brier scoring, per-engine aggregation. This is the phase
that makes everything before it worth having; do not skip ahead of it to add more
ingestion sources.

**Phase 4 — Ingestion layer.**
RSS + one government-release source to start. Extraction pipeline from §4.2. Now the
world-state updates itself instead of being manually seeded.

**Phase 5 — IM protocol backend.**
The 8-step pipeline from §5.2, run against the now-live world-state.

**Phase 6 — Prediction-market ingestion + external baseline, GRP comparative
backend, auto-resolution experiments.**
Everything explicitly deferred above.

Each phase should ship independently usable. A system that stops after Phase 3 is
already more disciplined than the status quo (essays written and forgotten, with no
mechanism to ever discover whether they were right).
