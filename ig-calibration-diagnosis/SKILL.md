---
name: ig-calibration-diagnosis
description: "Use when calibrating InformationGeometry.jl models."
version: 0.1.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [informationgeometry, calibration, multistart, residuals, model-selection, diagnostics]
    related_skills: [mechanistic-model-development, ig-julia-project, ig-identifiability-uq]
---

# InformationGeometry.jl Calibration and Fit Diagnosis

Use this skill as phase B of `mechanistic-model-development`, after `ig-julia-project` has created a runnable baseline. Its purpose is to establish whether a model explains the observed patterns sufficiently for reduction/UQ, and, if not, to guide the smallest defensible structural change.

## Calibration protocol

For every candidate model, record in `results/calibration_metadata.toml` or JSON:

- model Git commit/hash and candidate identifier;
- Julia/InformationGeometry.jl environment;
- declared error model and uncertainty scaling;
- parameter names, units, transforms/bounds, and initial values;
- optimizer/method, tolerances, random seed, number of starts, time limit;
- final MLE/objective/log likelihood, convergence/result status, and all materially distinct optima.

Do not treat a convergence flag as scientific adequacy.

### InformationGeometry.jl methods

A `DataModel` ordinarily finds an MLE during construction. For controlled fitting, construct with `SkipOptim=true`, then call `InformationGeometry.minimize`. For uncertain initial values or multimodality, run `MultistartFit`; save a `WaterfallPlot` and inspect `ParameterPlot` for distinct or dispersed good optima.

```julia
using InformationGeometry, Optim
mle = InformationGeometry.minimize(dm; meth=LBFGS(), tol=1e-10,
                                    maxtime=60.0, Domain=domain)
multistart = MultistartFit(dm; N=100, meth=LBFGS())
```

Use methods/domains compatible with the model and package versions actually installed. Do not copy an example optimizer without checking that it runs in the project environment.

## Required diagnostics

Write code that saves at least:

- observed vs predicted values, faceted by output and condition where relevant;
- residual vs fitted values;
- residual vs time/input and condition;
- a compact parameter table and calibration metadata;
- for time series, a clearly time-ordered overlay.

Inspect plots; do not merely generate them. In `MODELING_LOG.md`, explain any claimed discrepancy in one concise sentence, tied to a pattern visible in a saved figure.

## Qualitative adequacy gate

There is no universal numerical pass threshold in v1. A candidate is adequate only when an informed qualitative review finds no material, interpretable systematic residual pattern relative to the stated scientific goal. Describe the conclusion and remaining limitations in `STATUS.md` and `MODELING_LOG.md`.

If a systematic mismatch remains:

1. State the pattern: e.g. phase lag, saturation miss, condition-specific bias, decay-rate error, heteroscedastic funnel (the latter does **not** automatically authorize a new noise model).
2. State the smallest mechanism hypothesis and why it could address that pattern.
3. Implement it in a reversible commit or explicitly named candidate branch/directory.
4. Re-run the same calibration and diagnostics.
5. Retain, reject, or mark ambiguous using direct evidence.

Do not add general flexibility, nuisance parameters, or a more complex error model merely to lower an objective.

## Candidate retention

Maintain `results/candidate_comparison.csv` with candidate id, Git commit, mechanism summary, declared error model, calibration status, diagnostic conclusion, and retain/reject rationale. Preserve only the preferred candidate plus alternatives that are diagnostically indistinguishable or biologically/mechanistically plausible. Document retained alternatives in `ALTERNATIVES.md` with their commit and reproduction command.

AIC/BIC may be reported as secondary descriptive evidence, but must not replace the qualitative diagnostic gate or override scientific plausibility.

## Commit discipline and handoff

One accepted model change gets one atomic commit and one terse log row, for example:

```text
model: add saturable clearance to address high-concentration residual curvature
```

When a candidate passes the qualitative gate, hand its commit, calibration metadata, plots, retained alternatives, and remaining caveats to `ig-identifiability-uq`.

## Failure/budget behavior

If the time/iteration budget ends, write a `PARTIAL` `STATUS.md` naming the last calibrated candidate, last successful command, unresolved diagnostic pattern, and next command. Commit these artifacts. Never claim final model validation or UQ completion.
