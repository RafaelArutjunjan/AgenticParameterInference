---
name: ig-julia-project
description: "Use when building an InformationGeometry.jl project."
version: 0.1.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [julia, informationgeometry, project-setup, csv, ode, algebraic-models]
    related_skills: [mechanistic-model-development, ig-calibration-diagnosis]
---

# InformationGeometry.jl Project Construction

Use this skill as phase A of `mechanistic-model-development`. Create a minimal, portable Julia project for either a static nonlinear/algebraic model or a deterministic ODE/state-space model. Prefer ordinary InformationGeometry.jl containers and small source files; do not create a bespoke framework.

## Inputs and mandatory pre-flight

Require:

- CSV path, prose experiment/data description, and candidate mechanism outline.
- Destination project directory and hard time/iteration budget from the coordinator.
- An explicit or inferred map of columns to inputs/time, outputs, conditions, replicates, and units.

Inspect the actual CSV before implementation. Write `INPUT_INTERPRETATION.md` that names every used column and records:

- input/time and observed-output columns;
- condition/replicate encoding and units;
- observation map from model states/predictions to observations;
- parameter names, meanings, units, initial guesses, bounds/transforms if known;
- error model.

If measurement uncertainty is missing, use the agreed additive absolute-error / OLS default. A likelihood/profile calculation still needs an absolute residual scale: use a supplied scale when available; otherwise estimate the OLS residual scale after a provisional fit (normally `sqrt(SSE / max(n - p, 1))`) and record it as a fixed plug-in scale for that analysis. Do not present it as measured uncertainty, do not switch to proportional noise automatically, and state the limitation. For InformationGeometry.jl’s `DataSet`, use this declared fixed standard-deviation convention explicitly.

If a material ambiguity remains, create `INPUT_CLARIFICATIONS.md`, commit it, write `STATUS.md` as `BLOCKED_INPUT`, and stop before calibration.

## Required project layout

Create this small baseline layout, adapting filenames only when necessary:

```text
<project>/
  Project.toml
  Manifest.toml
  run.jl
  README.md
  STATUS.md
  INPUT_INTERPRETATION.md
  MODELING_LOG.md
  data/input.csv
  src/load_data.jl
  src/model.jl
  src/analysis.jl
  results/
  figures/
```

`run.jl` must include the source files and run the current baseline from a clean working directory. Keep intermediate scratch files outside the handoff tree or under an ignored `scratch/` directory.

Initialize Git before any accepted model is implemented. Commit the immutable input and input interpretation separately from the baseline model whenever useful.

## InformationGeometry.jl construction patterns

### Static model

Use `DataSet(x, y, sigma; xnames=..., ynames=...)` then `DataModel(dataset, predictor, initial_parameters; ...)`. `DataModel` normally attempts an MLE at construction; use `SkipOptim=true` only when deliberately controlling optimization later.

```julia
using InformationGeometry
x = collect(...)
y = collect(...)
σ = fill(σ_abs, length(y))
data = DataSet(x, y, σ; xnames=["input"], ynames=["response"])
predictor(x, θ) = θ[1] * x + θ[2]
dm = DataModel(data, predictor, θ0; SkipOptim=true)
```

For vector outputs, specify the mapping and preserve component names; do not flatten data without documenting the ordering.

### Deterministic ODE/state-space model

Use `ModelingToolkit` or an explicit `ODEFunction`, define the observation map deliberately, and construct a `DataModel` from the system, initial state, observable components or observation function, and parameter initial guess.

```julia
using InformationGeometry, ModelingToolkit
using ModelingToolkit: t_nounits as t, D_nounits as D
@parameters k
@variables x(t)
@named sys = System([D(x) ~ -k*x], t, [x], [k])
dm = DataModel(data, sys, [x0], [1], [k0]; tol=1e-10)
```

Avoid `@mtkcompile` if preserving declared state/equation ordering is important; InformationGeometry.jl documentation notes it may reorder them. Use a named system or an explicit `ODEFunction` and test the observation map.

## Implementation checks before baseline commit

1. `Pkg.instantiate()` succeeds in the new project environment.
2. `julia --project=. --startup-file=no run.jl` loads data, builds a `DataModel`, and writes a machine-readable setup/calibration placeholder.
3. The model predicts the correct output dimension and has a tested observation mapping.
4. `README.md` contains the reproduction command, declared status, one-paragraph baseline-model description, and results map.
5. `MODELING_LOG.md` begins with an input/baseline decision table row.

Commit only after these checks, e.g. `model: implement baseline <mechanism>`.

## Handoff to next phase

Pass the project root, `DataModel` construction location, declared error convention, parameter metadata, and the baseline Git commit to `ig-calibration-diagnosis`.

## Do not

- Guess whether a numeric column is a time, covariate, condition, or measured output when that distinction affects the model.
- Substitute a fitted error model for the declared additive-error default.
- Put all equations and analysis into a notebook or an untracked REPL session.
- Conceal data transformations, reshaping, unit conversion, state ordering, or initial-condition assumptions.
