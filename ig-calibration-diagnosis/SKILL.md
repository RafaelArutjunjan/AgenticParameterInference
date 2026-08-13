---
name: ig-calibration-diagnosis
description: "Use when calibrating InformationGeometry.jl models."
version: 0.2.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [informationgeometry, calibration, multistart, residuals, model-selection, diagnostics]
    related_skills: [mechanistic-model-development, ig-julia-project, ig-identifiability-uq, ig-reference, ig-advanced-datasets]
---

# InformationGeometry.jl Calibration and Fit Diagnosis

Use this skill as phase B of `mechanistic-model-development`, after `ig-julia-project` has created a runnable baseline. Its purpose is to establish whether a model explains the observed patterns sufficiently for reduction/UQ, and, if not, to guide the smallest defensible structural change.

## Calibration protocol

For every candidate model, record in `results/calibration_metadata.txt`:

- model Git commit and candidate identifier;
- Julia/InformationGeometry.jl version;
- declared error model and uncertainty scaling;
- parameter names, transforms/bounds, initial values;
- optimizer/method, tolerances, random seed, number of starts, time limit;
- final MLE, log-likelihood, AIC/BIC/AICc, convergence status.

### Standard fit (single start)

```julia
using InformationGeometry, LinearAlgebra, Printf

# DataModel auto-optimizes MLE on construction
dm = DataModel(ds, model, θ0)

# Extract results
mle = MLE(dm)                    # Vector{Float64}
logl = LogLikeMLE(dm)            # Float64
aic = AIC(dm)                    # Float64 — uses MLE automatically
bic = BIC(dm)                    # Float64 — uses MLE automatically
aicc = AICc(dm)                  # Float64 — uses MLE automatically
```

### Controlled fit (skip auto-optimization)

```julia
dm = DataModel(ds, model, θ0; SkipOptim=true)
mle = InformationGeometry.minimize(dm, θ0; meth=LBFGS(), tol=1e-10, maxtime=300.0)
```

### Multistart fit (for multimodality)

```julia
R = MultistartFit(dm; N=100, meth=LBFGS())
MLE(R)                           # best parameter vector
```

### Model comparison across candidates

```julia
# For each candidate model:
dm_i = DataModel(ds, model_i, θ0_i)
println("AIC=$(AIC(dm_i))  BIC=$(BIC(dm_i))  AICc=$(AICc(dm_i))  logL=$(LogLikeMLE(dm_i))")
# Select lowest BIC (most conservative) or AIC/AICc
# AIC/BIC/AICc all accept (dm) directly — they use MLE automatically
```

## Required diagnostics

For the selected model, compute and save:

```julia
mle = MLE(dm)
preds = [model(xi, mle) for xi in x]   # predicted values
residuals = y - preds                    # residuals
rmse = sqrt(mean(residuals .^ 2))
reduced_chi_squared = ChisquaredReduced(dm)
```

Save to CSV:
- `results/residuals.csv`: x, y, predicted, residual
- `results/calibration_metadata.txt`: all fit metadata
- `results/model_comparison.csv`: degree/candidate, nparam, logL, AIC, BIC, AICc, RMSE

Inspect residual patterns. In `MODELING_LOG.md`, explain any claimed discrepancy in one concise sentence tied to a visible pattern.

## Qualitative adequacy gate

A candidate is adequate when qualitative review finds no material, interpretable systematic residual pattern. Describe the conclusion in `STATUS.md` and `MODELING_LOG.md`.

If a systematic mismatch remains:
1. State the pattern (phase lag, saturation miss, condition-specific bias, etc.).
2. State the smallest mechanism hypothesis that could address it.
3. Implement in a reversible commit.
4. Re-run calibration and diagnostics.
5. Retain, reject, or mark ambiguous using direct evidence.

AIC/BIC may be reported as secondary evidence but must not replace the qualitative diagnostic gate or scientific plausibility.

## Candidate retention

Maintain `results/candidate_comparison.csv` with: candidate id, Git commit, mechanism summary, error model, calibration status, diagnostic conclusion, retain/reject rationale. Preserve only the preferred candidate plus diagnostically indistinguishable alternatives.

## Commit discipline and handoff

One accepted model change = one atomic commit + one log row:
```
model: add saturable clearance to address high-concentration residual curvature
```

When a candidate passes the qualitative gate, hand its commit, calibration metadata, and caveats to `ig-identifiability-uq`.

## Failure/budget behavior

At budget exhaustion, write `PARTIAL` `STATUS.md` naming the last calibrated candidate, last successful command, unresolved pattern, and next command. Never claim final model validation.
