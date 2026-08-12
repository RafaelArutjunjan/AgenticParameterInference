# v1 Design Decisions — 2026-08-02

This record captures the user-agreed design contract for the initial autonomous mechanistic-model development workflow. It does not replace per-run scientific assumptions, which belong in the generated project repository.

## Agreed scope

- Support deterministic ODE/state-space and static nonlinear/algebraic models through a shared artifact convention.
- Use Julia and InformationGeometry.jl for implementation, calibration, profile likelihoods, and related analysis.
- The system is exploratory: it may add, remove, or reparameterize mechanisms based on data diagnostics and documented assumptions.

## Model adequacy and uncertainty

- Treat qualitative fit and residual diagnostics as the acceptance gate; do not impose a universal numerical goodness-of-fit threshold.
- Make frequentist InformationGeometry.jl calibration and profile likelihoods the definitive UQ path.
- Use a user-specified measurement-error model whenever supplied.
- Otherwise use the simplest absolute-error model: independent additive residuals / OLS. Record the scale convention and limitations explicitly in every run.
- Treat InformationGeometry.jl rank/singular-direction results as local/numerical identifiability evidence. Do not call them formal global symbolic structural-identifiability proofs without a separate saved formal analysis.
- Aim for bounded finite practical confidence intervals when the data support them; report open, unbounded, or failed profiles faithfully.

## Reproducibility and human handoff

- Initialize a Git repository for every run.
- Make commits for accepted structural/scientific decisions, rather than optimizer retries.
- Retain the final preferred model and alternatives that are diagnostically indistinguishable or biologically/mechanistically plausible.
- Optimize for an expert InformationGeometry.jl user to understand and resume the work in under five minutes via a terse README, modeling log, source, results, figures, and commit history.

## Orchestration and bounds

- Use a coordinator with specialized project implementation, calibration/diagnosis, and identifiability/UQ workstreams.
- Enforce a hard wall-clock/iteration budget.
- At budget exhaustion, preserve reproducible commits and results, state partial completion clearly, and never claim work that was not completed.

## Input inference

- Infer CSV roles and observation semantics from CSV + prose where possible.
- Stop with a structured clarification artifact where ambiguity could materially change the model or inference.
