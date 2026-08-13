---
name: ig-julia-project
description: "Use when building an InformationGeometry.jl project."
version: 0.2.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [julia, informationgeometry, project-setup, csv, ode, algebraic-models]
    related_skills: [mechanistic-model-development, ig-calibration-diagnosis, ig-reference, ig-advanced-datasets]
---

# InformationGeometry.jl Project Construction

Use this skill as phase A of `mechanistic-model-development`. Create a minimal, portable Julia project for either a static nonlinear/algebraic model or a deterministic ODE/state-space model.

## Julia environment setup — do this FIRST

Always use a **local project environment** (not the global one). From the project directory:

```julia
using Pkg
Pkg.activate(".")          # creates/uses Project.toml in the current directory
Pkg.add(["InformationGeometry"])  # IG.jl is self-contained; pulls its own deps
Pkg.instantiate()          # resolve and precompile
```

InformationGeometry.jl does not require CSV.jl — it has no CSV dependency. For reading CSV data, either:
- `Pkg.add("CSV")` and use `CSV.File(path)`, or
- Use Julia's built-in `DelimitedFiles.readdlm` or a simple `readlines` parser (no extra deps).

If `Project.toml` already exists, just run `Pkg.activate("."); Pkg.instantiate()` — do not re-add packages.

Run scripts with: `julia --project=. --startup-file=no run.jl`

⚠️ **Timeout**: `Pkg.add` and `Pkg.instantiate` trigger precompilation of IG.jl and all its dependencies. In a fresh environment this takes **2–5 minutes**. Always set `timeout=300` (or use `background=true`) for these commands. Do NOT use the default 120s timeout — it will kill the precompilation midway and corrupt the environment.

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

If measurement uncertainty is missing, use the additive absolute-error / OLS default.
For `DataSet`, need to pass known `σ` array. If the data requires non-Gaussian uncertainties, x-errors, missing values, or estimated variances, use an advanced dataset type — load the `ig-advanced-datasets` skill for constructors and examples.

If a material ambiguity remains, create `INPUT_CLARIFICATIONS.md`, commit it, write `STATUS.md` as `BLOCKED_INPUT`, and stop before calibration.

## Required project layout

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
  results/
  figures/
```

`run.jl` must run the current baseline from a clean working directory. Keep scratch files outside the handoff tree.

Initialize Git before any accepted model is implemented.

## Ready-to-run template: static (algebraic) model

Copy this template and adapt the `model` function, parameter count, and data loading. It covers the full pipeline: load data → fit → model comparison → identifiability → profile likelihood CIs.

```julia
using InformationGeometry, LinearAlgebra, Statistics, CSV, DataFrames, Printf


function main()
    ## Different column names must be adapted accordingly
    df = CSV.read(joinpath(@__DIR__, "data", "input.csv"), DataFrame)
    x, y, σ = df[!,"x"], df[!,"y"], df[!,"sigma"]
    ds = DataSet(x, y, σ; xnames=["x"], ynames=["y"])

    # --- Model definition ---
    # For ydim=1: return a SCALAR (Number), not a vector
    # θ is always a Vector{Float64}, even for 1 parameter
    model(xi, θ) = θ[1]*xi + θ[2]          # ← replace with your model
    θ0 = rand(2)                           # ← initial guess

    # --- Fit (DataModel auto-optimizes MLE on construction) ---
    dm = DataModel(ds, model, θ0)
    mle = MLE(dm)
    println("MLE: ", mle)
    println("logL: ", LogLikeMLE(dm))
    println("AIC: ", AIC(dm), "  BIC: ", BIC(dm), "  AICc: ", AICc(dm))
    println("MLEuncert: ", MLEuncert(dm))   # MLE ± SE directly

    # --- Asymptotic (Fisher) confidence intervals ---
    println("MLEuncert: $(MLEuncert(dm))")

    # --- Profile likelihood confidence intervals ---
    println("\nProfile likelihood 95% CIs:")
    profiles = ParameterProfiles(dm, 2.5, collect(1:np); plot=false, SaveTrajectories=true)
    ci95 = Tuple(ConfidenceIntervals(profiles, InvConfVol(0.95)))
    for i in 1:np
        @printf("  θ[%d] = %.6f  CI95 = [%.6f, %.6f]\n", i, mle[i], ci95[i][1], ci95[i][2])
    end

    # --- Identifiability ---
    svs, svecs = StructurallyIdentifiable(dm; threshold=1e-10)
    println("\nSingular values: ", svs)
    pract = PracticallyIdentifiable(profiles)
    println("Practically identifiable up to σ = ", pract)
end

main()
```

⚠️ **Julia scoping**: The entire script is wrapped in a `function main() ... end` block to avoid Julia's top-level scoping issue where variables assigned in `for`/`try` blocks at global scope require `global` prefix. This is the #1 source of errors for agents writing Julia scripts. Always use this pattern.

### Model comparison across candidates (e.g. polynomial degree selection)

```julia
function compare_degrees(ds, degrees)
    for deg in degrees
        np = deg + 1
        θ0 = zeros(np)
        pred(xi, θ) = sum(θ[i] * xi^(deg - i + 1) for i in 1:np)
        dm = DataModel(ds, pred, θ0)
        @printf("deg=%d  AIC=%.2f  BIC=%.2f  AICc=%.2f  logL=%.2f\n",
                deg, AIC(dm), BIC(dm), AICc(dm), LogLikeMLE(dm))
    end
end
compare_degrees(ds, 0:9)
# Select model with lowest BIC (most conservative) or AIC/AICc
```

### ODE model template

```julia
using InformationGeometry, ModelingToolkitBase
using ModelingToolkitBase: t_nounits as t, D_nounits as D

@parameters k
@variables x(t)
@named sys = System([D(x) ~ -k*x], t, [x], [k])
# Do NOT use @mtkcompile — it reorders states

# GetModel creates an IG-compatible model from the ODE system
ode_model = GetModel(sys, [x0], [1], [k0]; tol=1e-10)
dm = DataModel(ds, ode_model, [k0])
```

## Implementation checks before baseline commit

1. `Pkg.instantiate()` succeeds.
2. `julia --project=. --startup-file=no run.jl` loads data, builds a `DataModel`, prints MLE and diagnostics.
3. The model predicts the correct output dimension (scalar for ydim=1).
4. `README.md` contains the reproduction command and one-paragraph model description.
5. `MODELING_LOG.md` begins with an input/baseline decision table row.

Commit after these checks pass.

## Handoff to next phase

Pass the project root, `DataModel` construction location, declared error convention, parameter metadata, and the baseline Git commit to `ig-calibration-diagnosis`.
