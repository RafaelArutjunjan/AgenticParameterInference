---
name: ig-identifiability-uq
description: "Use for InformationGeometry.jl profile-likelihood UQ."
version: 0.1.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [informationgeometry, identifiability, profile-likelihood, confidence-intervals, model-reduction]
    related_skills: [mechanistic-model-development, ig-julia-project, ig-calibration-diagnosis]
---

# InformationGeometry.jl Identifiability and UQ

Use this skill as phase C of `mechanistic-model-development`, only after `ig-calibration-diagnosis` has produced a diagnostically adequate candidate under a declared error model. It implements reduction and frequentist profile-likelihood UQ without overstating what the numerical evidence proves.

## Terminology and evidence standard

InformationGeometry.jl provides `StructurallyIdentifiable(dm)` and `IdentifiabilityReport(dm)` based on the local numerical rank/singular directions of the model embedding/Fisher geometry near the MLE. Report this as **local/numerical identifiability evidence**. It does not, by itself, prove global symbolic structural identifiability.

Use the following separate labels in all results:

- **Local/numerical identifiability:** rank/singular-value result at or near the fitted point.
- **Practical identifiability:** finite profile-likelihood support at the specified confidence level under the declared error model.
- **Formal structural identifiability:** only claim this if a dedicated formal symbolic analysis was independently executed and saved.

## Analysis sequence

1. Freeze a candidate commit with calibration metadata and diagnostics.
2. Run and save local/numerical rank evidence and associated singular directions.
3. Run `IdentifiabilityReport` to associate affected parameters/mechanisms, saving output together with the chosen threshold.
4. Calculate preliminary profile likelihoods for all final candidate parameters.
5. Interpret profile paths and parameter compensation to formulate the smallest reduction/reparameterization consistent with the model science.
6. Apply one accepted reduction at a time, commit it, then return to `ig-calibration-diagnosis` for full recalibration and residual checks.
7. Repeat until no removable rank deficiency remains or the hard budget is exhausted.
8. For the final adequate candidate, compute definitive profiles, confidence intervals, and key plots.

Never remove/fix a parameter solely because a profile is wide. Document the implicated relationship, the scientific assumption made by the reduction, and the post-reduction diagnostic result.

## InformationGeometry.jl operations

Typical calls, with the exact package API verified in the run environment:

```julia
using InformationGeometry
singular_values, singular_vectors = StructurallyIdentifiable(dm; threshold=1e-10,
                                                              showall=true)
# NOTE: judge rank using the chosen `threshold`; the package convenience
# predicate `IsStructurallyIdentifiable` tests only whether singular values are > 0.
report, affected_parameter_names = IdentifiabilityReport(dm; threshold=1e-10)
profiles = ParameterProfiles(dm, 2; N=100, plot=true, IsCost=false,
                             adaptive=true, SaveTrajectories=true)
intervals_95 = Tuple(ProfileBox(profiles, InvConfVol(0.95)))
practical_limit = PracticallyIdentifiable(profiles)
```

`ParameterProfiles(..., IsCost=false)` presents profile levels on a confidence-level scale; document the requested level and degrees-of-freedom convention. `ProfileBox` returns `-Inf`/`+Inf` for an interval unbounded on a side. Preserve those values—never replace them with a finite plotting range.

For higher-dimensional candidates, profiles are normally preferred over expensive global confidence boundaries. If computing `ConfidenceRegions`, note that it requires/assumes locally identifiable behavior and can be expensive; record solver tolerance and direction/parallel settings.

## Required result artifacts

Save machine-readable files such as:

```text
results/identifiability_local.json     # threshold, singular values/directions, interpretation
results/profile_intervals.csv          # parameter, level, lower, upper, boundedness
results/profile_summary.csv            # practical-identifiability level / status
results/profile_paths_summary.csv      # material compensating parameters/mechanisms
figures/profile_likelihoods.*
figures/profile_paths.*                # only when informative
```

Every result must include the model Git commit, error-model statement, profile method/options, and timestamp/environment metadata.

## Reduction log rule

For each accepted reduction, add a row to `MODELING_LOG.md` with:

| Commit | Change | Numerical/profile evidence | Scientific assumption | Refit/diagnostic outcome | Status |
|---|---|---|---|---|---|

Use a commit message such as:

```text
model: reduce redundant rate pair after local rank deficiency in k1 and k2
```

Return to phase B after every structural change. A reduction that harms fit or leaves the rationale unsupported must be reverted or documented as rejected.

## Completion and partial completion

A phase-C result is complete only when the final repository contains saved local/numerical diagnostics, profiles, intervals, and an explicit practical-identifiability conclusion for each fitted parameter, including unbounded/failed cases.

When formal symbolic structural identifiability was not run, say so directly in `README.md` and `STATUS.md`.

At budget exhaustion, save all partial profiles/results and write `STATUS.md` with the exact last completed operation, model commit, unresolved parameters, and next command. Do not label practical intervals “finite” unless they are actually bounded in saved `ProfileBox` output.
