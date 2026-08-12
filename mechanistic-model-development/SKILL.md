---
name: mechanistic-model-development
description: "Use when autonomously developing mechanistic data models."
version: 0.2.0
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

Use this coordinating skill to turn a CSV dataset, a prose experiment description, and a candidate model outline into an auditable Julia/InformationGeometry.jl project. Load the focused skills in phase order:

1. `ig-julia-project` — interpret inputs and build the reproducible project.
2. `ig-calibration-diagnosis` — calibrate candidates and use visual/residual evidence to guide minimal structural changes.
3. `ig-identifiability-uq` — reduce nonidentifiable structure and compute profile-likelihood UQ.

This is a scientific-development workflow, not generic curve fitting. The product is a small, runnable Git repository an InformationGeometry.jl user can understand and resume quickly.

## Contract agreed for v1

- Support deterministic ODE/state-space and static nonlinear/algebraic models under one artifact convention.
- The agent may exploratorily add, remove, or reparameterize mechanisms, but each accepted change must state its scientific rationale and diagnostic target.
- Adequacy is assessed by qualitative fit and residual diagnostics unless the user provides explicit numerical criteria. Never invent a universal fit cutoff.
- Use InformationGeometry.jl for calibration, local/numerical identifiability diagnostics, profile likelihoods, and confidence intervals where applicable.
- When no measurement uncertainties are supplied, use the simplest **absolute-error model**: independent additive residuals / ordinary least squares. Record this assumption in `INPUT_INTERPRETATION.md`; do not infer a scale-dependent error model automatically.
- Preserve the preferred final model plus only comparably adequate, mechanistically plausible alternatives.
- Commit accepted scientific states, not every optimizer attempt.
- Require and obey a hard wall-clock and/or iteration budget. At exhaustion, leave a runnable partial repository and truthful status; never claim a phase was completed when it was not.

## Scientific terminology guardrail

`InformationGeometry.StructurallyIdentifiable` evaluates the local numerical rank/singular directions of the model embedding at an MLE. It is useful evidence for local identifiability, but is **not by itself a proof of global symbolic structural identifiability**. Call outputs “local/numerical identifiability diagnostics” unless an additional formal structural-identifiability method has been run and saved.

Likewise, finite profile intervals support practical identifiability under the declared error model; they do not establish a model mechanism as physically true.

## Pre-flight gate — never bypass

Before model code is written, establish all of the following:

- Input CSV path is immutable/copied and readable.
- Prose description and candidate mechanism outline are available.
- Destination directory and hard budget are explicit.
- Column roles, units, conditions, replicates, observation map, and initial-condition semantics have been inferred or supplied.
- Error model is supplied, or the additive/OLS default is declared together with its residual-scale convention (supplied scale or an explicitly documented post-fit plug-in estimate).

If an ambiguity could materially change the observation model, equations, units, conditions, or parameter interpretation, write `INPUT_CLARIFICATIONS.md`, commit it, set `STATUS.md` to `BLOCKED_INPUT`, and stop. Do not fit a guessed interpretation.

## Phase gate sequence

### Phase A — project baseline

Delegate/load `ig-julia-project`. Require a clean Julia project, input interpretation, baseline model implementation, one entrypoint, and a baseline commit before calibration.

### Phase B — calibration and diagnosis loop

Delegate/load `ig-calibration-diagnosis`. For each accepted candidate:

1. Calibrate with documented starts, bounds/transforms, method, and results.
2. Save observed-vs-predicted and residual diagnostics.
3. Decide qualitatively whether an interpretable systematic mismatch remains.
4. If so, record the smallest mechanism hypothesis that targets it; implement the change in a new candidate branch/directory or reversible Git commit.
5. Retain/reject the candidate with evidence in `MODELING_LOG.md`.

Do not enter UQ merely because an optimizer reports convergence.

### Phase C — reduction and UQ loop

Delegate/load `ig-identifiability-uq` only for a diagnostically adequate candidate. Iterate: local/numerical rank diagnosis → minimal reparameterization/reduction → recalibration and diagnostic recheck. Then compute profile likelihoods and intervals, faithfully recording finite, one-sided, unbounded, failed, or unevaluated outcomes.

## Required handoff

The final/partial repository contains at least:

```text
README.md                         # first-read entrypoint, model and status summary
STATUS.md                         # COMPLETE | PARTIAL | BLOCKED_INPUT, last completed phase
MODELING_LOG.md                   # terse accepted-decision table
INPUT_INTERPRETATION.md           # CSV roles, units, observation and error model
Project.toml
Manifest.toml                     # when resolved for the run
run.jl                            # one reproducible entrypoint
data/
src/
results/                          # machine-readable tables and serialized results
figures/                          # only essential figures
ALTERNATIVES.md                   # only if retained alternatives exist
```

A five-minute reader should find the final model equations/source, the declared observation/error model, key fit/profile figures, status, and the exact command to reproduce the last completed state.

## Commit and stop rules

Use commits like:

- `model: implement baseline <mechanism>`
- `model: add <mechanism> to address <diagnostic pattern>`
- `model: reduce <mechanism> after local rank deficiency in <parameters>`
- `uq: add profile likelihood analysis for <model>`
- `status: record partial completion at <phase>`

At budget exhaustion, write `STATUS.md` with: budget consumed, last successful command, completed/reproducible commit, unresolved issue, and next command. Do not silently extend the budget.

## Final verification gate

Before calling a run complete, verify from the project root:

```bash
julia --project=. --startup-file=no run.jl
```

If this cannot be run successfully, report the exact failure and preserve the last successful command/output in `STATUS.md`. Check that parameter names and units agree across source, CSV interpretation, tables, figures, profiles, and prose.
