---
name: mechanistic-model-development
description: "Use when autonomously developing mechanistic data models."
version: 0.3.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [mechanistic-models, julia, informationgeometry, orchestration, calibration, identifiability, uncertainty-quantification]
    related_skills: [ig-julia-project, ig-calibration-diagnosis, ig-identifiability-uq]
---

# Mechanistic Model Development — Coordinator

Turn a CSV dataset, a prose experiment description, and a candidate model outline into an auditable Julia/InformationGeometry.jl project. Load the focused skills in phase order:

1. `ig-julia-project` — set up Julia environment, interpret inputs, build baseline model.
2. `ig-calibration-diagnosis` — calibrate candidates, use residual diagnostics to guide structural changes.
3. `ig-identifiability-uq` — reduce nonidentifiable structure, compute profile-likelihood UQ.

If necessary, the `ig-reference` skill can also be loaded for a more complete API reference and copy-paste cheat sheet.

## Core principles

- Each accepted model change must state its scientific rationale and diagnostic target.
- Adequacy is assessed by qualitative fit and residual diagnostics unless the user provides explicit numerical criteria.
- Use InformationGeometry.jl for calibration, identifiability diagnostics, profile likelihoods, and confidence intervals.
- When no measurement uncertainties are supplied, use additive absolute-error / OLS. Record this in `INPUT_INTERPRETATION.md`.
- Preserve the preferred final model plus only comparably adequate, mechanistically plausible alternatives.
- Commit accepted scientific states, not every optimizer attempt.
- Require and obey a hard wall-clock and/or iteration budget. At exhaustion, leave a runnable partial repository and truthful status.

## Pre-flight gate

Before model code is written:
- Input CSV path is readable and copied into `data/`.
- Prose description and candidate mechanism outline are available.
- Destination directory and hard budget are explicit.
- Column roles, units, conditions, replicates, observation map, and initial-condition semantics have been inferred or supplied.
- Error model is supplied, or the additive/OLS default is declared with its residual-scale convention.
- Julia environment is set up: `Pkg.activate("."); Pkg.add("InformationGeometry"); Pkg.instantiate()`. **Use `timeout=300` for these commands** — IG.jl precompilation takes 2–5 min in a fresh environment. The default 120s terminal timeout will kill it midway.

If an ambiguity could materially change the observation model, equations, units, conditions, or parameter interpretation, write `INPUT_CLARIFICATIONS.md`, set `STATUS.md` to `BLOCKED_INPUT`, and stop.

## Phase gate sequence

### Phase A — project baseline
Delegate to `ig-julia-project`. Require: clean Julia project, input interpretation, baseline model, one entrypoint `run.jl`, and a baseline commit.

### Phase B — calibration and diagnosis loop
Delegate to `ig-calibration-diagnosis`. For each candidate: calibrate → save diagnostics → decide if systematic mismatch remains → if so, record smallest mechanism hypothesis and implement → retain/reject with evidence.

### Phase C — reduction and UQ loop
Delegate to `ig-identifiability-uq` only for a diagnostically adequate candidate. Iterate: local rank diagnosis → minimal reparameterization → recalibration → definitive profiles and intervals.

## Required handoff artifacts

```text
README.md                         # model and status summary, reproduction command
STATUS.md                         # COMPLETE | PARTIAL | BLOCKED_INPUT
MODELING_LOG.md                   # accepted-decision table
INPUT_INTERPRETATION.md           # CSV roles, units, observation and error model
Project.toml / Manifest.toml
run.jl                            # one reproducible entrypoint
data/                             # immutable input
results/                          # machine-readable tables and serialized results
figures/                          # only essential figures
```

## Commit messages

```
model: implement baseline <mechanism>
model: add <mechanism> to address <diagnostic pattern>
model: reduce <mechanism> after local rank deficiency in <parameters>
uq: add profile likelihood analysis for <model>
status: record partial completion at <phase>
```

## Final verification gate

```bash
julia --project=. --startup-file=no run.jl
```

If this fails, report the exact failure and preserve the last successful output in `STATUS.md`.
