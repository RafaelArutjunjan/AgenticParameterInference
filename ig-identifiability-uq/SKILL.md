---
name: ig-identifiability-uq
description: "Use for InformationGeometry.jl profile-likelihood UQ."
version: 0.2.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [informationgeometry, identifiability, profile-likelihood, confidence-intervals, model-reduction]
    related_skills: [mechanistic-model-development, ig-julia-project, ig-calibration-diagnosis, ig-reference, ig-advanced-datasets]
---

# InformationGeometry.jl Identifiability and UQ

Use this skill as phase C of `mechanistic-model-development`, only after `ig-calibration-diagnosis` has produced a diagnostically adequate candidate. It implements reduction and frequentist profile-likelihood UQ without overstating what the numerical evidence proves.

## Terminology

- **Local/numerical identifiability**: rank/singular-value result at the fitted point. What `StructurallyIdentifiable` returns. NOT a proof of global symbolic structural identifiability.
- **Practical identifiability**: finite profile-likelihood support at a given confidence level under the declared error model.
- **Formal structural identifiability**: only claim this if a dedicated symbolic analysis was independently executed and saved.

## Analysis sequence — copy-paste code

### 1. Local/numerical identifiability

```julia
using InformationGeometry, LinearAlgebra, Printf

# Returns (singular_values::Vector{Float64}, singular_vectors::Matrix{Float64})
svs, svecs = StructurallyIdentifiable(dm; threshold=1e-10)
println("Singular values: ", svs)
# All > threshold → locally identifiable at MLE

# Human-readable report (returns (nothing, nothing) if identifiable; (matrix, param_names) if not)
report, affected = IdentifiabilityReport(dm; threshold=1e-10)
# When identifiable: report === nothing, affected === nothing
# When non-identifiable: report = coupling submatrix, affected = Vector{String} of param names
# The function also prints a human-readable summary to stdout regardless
```

### 2. Asymptotic (Fisher information) confidence intervals

```julia
println("MLEuncert: $(MLEuncert(dm))")
```

### 3. Profile likelihood confidence intervals

```julia
# ParameterProfiles(dm, Confnum, Indices; kwargs...)
# Confnum: confidence level in σ units (e.g. 2 = compute up to 2σ)
# Indices: which parameters to profile (Vector{Int})
profiles = ParameterProfiles(dm, 2.5, collect(1:pdim(dm));
                             plot=false, SaveTrajectories=true)

# Tuple() gives [(lo, hi), ...] pairs
ci95 = Tuple(ConfidenceIntervals(profiles, InvConfVol(0.95)))
# ci95[i] = (lower_bound, upper_bound) for θ[i]
# May contain -Inf or +Inf for unbounded intervals — PRESERVE these values

# Practical identifiability directly from DataModel
pract = PracticallyIdentifiable(profiles)    # Float64 — max σ with bounded profile

println("Profile likelihood 95% CIs:")
for i in 1:pdim(dm)
    @printf("  θ[%d] = %.6f  CI95 = [%.6f, %.6f]\n", i, MLE(dm)[i], ci95[i][1], ci95[i][2])
end

# Practical identifiability from profiles
pract = PracticallyIdentifiable(profiles)   # Float64 — max σ with bounded interval both sides
```

### 4. Save results

```julia
# results/identifiability_local.txt
# results/profile_intervals.csv: parameter, mle, ci95_lower, ci95_upper, method
# results/fisher_info_cis.csv: parameter, mle, se, ci95_lower, ci95_upper, method
```

## Reduction loop

If `StructurallyIdentifiable` reveals rank deficiency:

1. Examine singular vectors to identify which parameters are non-identifiable.
2. Formulate the smallest reduction/reparameterization (fix a parameter, combine parameters, etc.).
3. Implement in a reversible commit.
4. Return to `ig-calibration-diagnosis` for full recalibration and residual checks.
5. Repeat until no removable rank deficiency remains or budget is exhausted.

Never remove/fix a parameter solely because a profile is wide. Document the implicated relationship, the scientific assumption, and the post-reduction diagnostic result.

For each accepted reduction, add a row to `MODELING_LOG.md`:

| Commit | Change | Numerical/profile evidence | Scientific assumption | Refit outcome | Status |
|---|---|---|---|---|---|

## Completion

A phase-C result is complete when the repository contains:
- `results/identifiability_local.txt` — singular values and interpretation
- `results/profile_intervals.csv` — profile-likelihood CIs for all parameters
- `results/fisher_info_cis.csv` — asymptotic CIs for comparison
- Practical-identifiability conclusion for each parameter (including unbounded/failed cases)

When formal symbolic structural identifiability was not run, say so in `README.md` and `STATUS.md`.

At budget exhaustion, save all partial results and write `STATUS.md` with: last completed operation, model commit, unresolved parameters, next command. Do not label practical intervals "finite" unless they are actually bounded in saved `Tuple(ConfidenceIntervals(dm))` output.
