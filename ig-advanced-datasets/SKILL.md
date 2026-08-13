---
name: ig-advanced-datasets
description: "Advanced DataSet types in InformationGeometry.jl: DataSetExact, CompositeDataSet, GeneralizedDataSet, DataSetUncertain, UnknownVarianceDataSet."
version: 0.2.0
author: Hermes Agent
created_by: agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [informationgeometry, julia, dataset-types, x-uncertainty, non-gaussian, missing-data, error-models]
    related_skills: [ig-reference, ig-julia-project, ig-calibration-diagnosis, ig-identifiability-uq, mechanistic-model-development]
---

# ig-advanced-datasets

Advanced `DataSet` types in InformationGeometry.jl. The basic `DataSet` (see `ig-reference`) handles Gaussian y-errors only. Use this skill when your data needs x-uncertainty, non-Gaussian errors, missing values, or estimated variances. For full constructor signatures and edge cases, read the source: `.julia/packages/InformationGeometry/<ver>/src/DataStructures/`.

## When to Use Each Type

| Type | Non-Gaussian y | Missing values | x-uncertainty | Mixed x-y unc. | y-unc. estimation | x-unc. estimation |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| `DataSet` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `DataSetExact` | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `CompositeDataSet` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `GeneralizedDataSet` | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `DataSetUncertain` | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| `UnknownVarianceDataSet` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

**Decision guide:**
- **x-uncertain** → `DataSetExact` (independent x-y) or `GeneralizedDataSet` (correlated x-y)
- **Non-Gaussian y** → `DataSetExact` / `GeneralizedDataSet` / `CompositeDataSet`
- **Missing y-values** → `CompositeDataSet` (different x per observable) or `DataSetUncertain` (same x)
- **Unknown y-variance** → `DataSetUncertain` (+ unknown x-variance → `UnknownVarianceDataSet`)

## Type Hierarchy

```
AbstractDataSet
├── AbstractFixedUncertaintyDataSet          (uncertainties known a-priori)
│   ├── DataSet                              (basic — see ig-reference)
│   ├── DataSetExact                         (distribution-based, optional x-unc)
│   ├── CompositeDataSet                     (multi-observable, missing data)
│   └── GeneralizedDataSet                   (joint x-y distribution, correlations)
└── AbstractUnknownUncertaintyDataSet        (uncertainties estimated via error model)
    ├── DataSetUncertain                     (y-variance estimated)
    └── UnknownVarianceDataSet               (both x- and y-variance estimated)
```

`AbstractFixedUncertaintyDataSet`: `HasEstimatedUncertainties()` → `false`. `SplitErrorParams()` → identity.
`AbstractUnknownUncertaintyDataSet`: `HasEstimatedUncertainties()` → `true`. `SplitErrorParams()` splits θ into (modelparams, errorparams).

## Shared Accessors (all types)

`dims(DS)` → `(Npoints, xdim, ydim)` · `xdata(DS)` · `ydata(DS)` · `xsigma(DS)` · `ysigma(DS)` · `xInvCov(DS)` · `yInvCov(DS)` · `HasXerror(DS)` · `HasMissingValues(DS)` · `WoundX(DS)` · `Npoints(DS)` · `xdim(DS)` · `ydim(DS)` · `DataspaceDim(DS)` · `xnames/ynames` · `name(DS)`

## DataSetExact

Stores data as **probability distributions** over x and y. Supports x-errors and non-Gaussian y-errors (Cauchy, Student-t, etc.) with exact `logpdf` likelihood. With `Dirac` x-distribution (zero x-unc), equivalent to `DataSet`.

```julia
using InformationGeometry, Distributions, LinearAlgebra

# y-errors only (equivalent to DataSet)
DS1 = DataSetExact([1,2,3,4], [4,5,6.5,7.8], [0.5,0.45,0.6,0.8])

# with x-errors
DS2 = DataSetExact([1,2,3,4], [0.1,0.1,0.1,0.1], [4,5,6.5,7.8], [0.5,0.45,0.6,0.8])

# non-Gaussian y (Cauchy) via distributions
X = product_distribution([Normal(0,1), Cauchy(2,0.5)])
Y = MvTDist(2, [3,8.], [1 0.5; 0.5 3])
DS3 = DataSetExact(X, Y, (2,1,1))  # dims=(Npoints, xdim, ydim)

# convert from DataSet, adding x-uncertainty
DS_with_x = DataSetExact(DataSet([1,2,3,4], [4,5,6.5,7.8], 0.5), 0.2)

# multi-dimensional: (4 points, 3 x-components, 2 y-components)
DS_md = DataSetExact(X_flat, Σ_x, Y_flat, Σ_y, (4, 3, 2))
```

**Conversions**: `DataSet(DSE)` drops x-errors (warns). `DataSetExact(DS::DataSet, σ_x)` adds them.

**Pitfalls:**
- `Dirac` not exported → use `InformationGeometry.Dirac(...)`
- From distributions without `dims`, assumes `(N,1,1)` — always pass `dims` for multi-dim data
- Non-Gaussian distributions need `gradlogpdf` to work (Cauchy is fixed in IG.jl; others may not be)

## CompositeDataSet

Multiple observables at **different x-values** with possible **missing data**. Splits into a vector of `DataSet`/`DataSetExact` components (one per observable). Missing rows (`missing` or `NaN`) are dropped per-component.

```julia
using InformationGeometry, DataFrames

# From DataFrame (striped: x, y₁, σ₁, y₂, σ₂, ...)
# missing/NaN values are automatically dropped per-observable
df = DataFrame([1:4 [2.5,6,missing,9] 0.3ones(4) [missing,5,3.1,1.4] 0.2ones(4)], :auto)
cds = CompositeDataSet(df, 1, 2; stripedYs=true)
# Component 1: y₁ at [1,2,4]  (row 3 dropped)
# Component 2: y₂ at [2,3,4]  (row 1 dropped)

# From individual DataSets
cds = CompositeDataSet([DataSet([1,2,4], [2.5,6,9], 0.3), DataSet([2,3,4], [5,3.1,1.4], 0.2)])

# With model (model returns vector, one y per observable)
cdm = DataModel(cds, (x,p)->[p[1]*x, p[2]*x])
```

**DataFrame layouts**: `stripedYs=true` (default): `x | y₁ σ₁ y₂ σ₂ ...`. `stripedYs=false`: `x | y₁ y₂ ... σ₁ σ₂ ...`. With `xerrs=true`: prepend `σx` after each x column.

**Likelihood**: sum of component likelihoods. `InvCov` is block-diagonal.

**Pitfalls:**
- All components must share the same `xdim`
- `SharedYdim=Val(false)` (non-uniform component ydims) is not fully implemented
- In-place `ModelMap{true}` not supported
- `xerrs=true` doubles the expected x-column count
- Plotting limited to `xdim=1`, `ydim≤16`

## GeneralizedDataSet

Most general fixed-uncertainty type: **joint x-y distribution** with possible **x-y error correlations**. If the distribution is separable (x,y independent), auto-converts to `DataSetExact`.

```julia
using InformationGeometry, Distributions, LinearAlgebra

# Joint MvNormal with x-y cross-covariance (2 points, xdim=1, ydim=1)
μ = [1.0, 2.0, 3.0, 4.0]  # [x₁, x₂, y₁, y₂]
Σ = [0.1 0 0.05 0; 0 0.1 0 0.05; 0.05 0 0.2 0; 0 0.05 0 0.2]
gds = GeneralizedDataSet(MvNormal(μ, Σ), (2,1,1))

# From vector + full covariance (assumes MvNormal)
gds2 = GeneralizedDataSet([1,2,3,4.], Diagonal([0.1,0.1,0.2,0.2]))
```

**Likelihood**: `logpdf(joint_dist, vcat(x, y))` — uses the full joint distribution including cross-terms.

**Pitfalls:**
- Separable distributions auto-convert to `DataSetExact` (by design — use the efficient type)
- Less performant than `DataSetExact` for independent x-y; prefer `DataSetExact` when no correlations
- `InvCov(GDS)` returns full inverse — use `xInvCov`/`yInvCov` for marginal blocks

## DataSetUncertain

**Unknown y-variance** estimated via a parametric error model `σ(x, ȳ, c)`. Error parameters `c` are jointly fit with model parameters `θ`. You provide the **reciprocal** error model `σ⁻¹(x, y_pred, c)`.

```julia
using InformationGeometry

# Reciprocal error model: 1/σ, initial error params c
DS = DataSetUncertain([1,2,3,4], [4,5,6.5,7.8], (x,y,c)->1/exp10(c[1]), [0.5])

# With model — params are [model_params..., error_params...] by default
DM = DataModel(DS, (x,p)->p[1]*x + p[2], [1.0, 1.0, 0.5])

# Default error model (σ = exp10(c)) — no error model needed
DS_default = DataSetUncertain(1:5, rand(5))

# Missing y-values: pass Unwind(Y_matrix) with NaN for missing, plus dims
Yd = [[1.5,NaN,3.2] [NaN,8.0,NaN]]  # 3 points, 2 y-components
DSU = DataSetUncertain([1,2,3], InformationGeometry.Unwind(Yd), (3,1,2),
    (x,y,c)->[exp10(-c[1]), exp10(-c[1])],
    InformationGeometry.DefaultErrorModelSplitter(1), [0.1])

# Bessel correction for unbiased variance estimation
DS_b = DataSetUncertain([1,2,3,4], [4,5,6.5,7.8], (x,y,c)->1/exp10(c[1]), [0.5]; BesselCorrection=true)
```

**Likelihood**: Gaussian evaluated at the **model prediction** (not data), since variance depends on `ȳ = model(x, θ)`:
`log L = -½N log(2π) + Σ log(σ⁻¹) - ½ Σ [σ⁻¹·(y - ȳ)]²`

**Key accessors**: `yinverrormodel(DS)`, `SplitErrorParams(DS)`, `yerrorparams(DS, mle)`, `errormoddim(DS)`.

**Pitfalls:**
- Provide `σ⁻¹`, not `σ`. For `ydim=1` return a Number; for `ydim>1` return a Cholesky-like matrix `S` (`Σ = S'S`)
- Error params are **last** in θ by default (`DefaultErrorModelSplitter(n)`); provide custom splitter otherwise
- Use `exp10(c)` to keep variance positive — raw `c` can go negative
- `dims` refers to full data including missing entries; `ydata(DS)` has missings removed

## UnknownVarianceDataSet

Extends `DataSetUncertain` to also estimate **x-variance**. Both x- and y-uncertainties have separate error models. The x-values themselves become **latent parameters** — total parameter vector is `[x̂₁...x̂ₙ, model_params, cx, cy]`.

```julia
using InformationGeometry

# σ_x⁻¹, σ_y⁻¹, initial cx, cy
DS = UnknownVarianceDataSet([1,2,3,4], [4,5,6.5,7.8],
    (x,y,cx)->1/exp10(cx[1]), (x,y,cy)->1/exp10(cy[1]), [0.25], [0.8])

# With model — x-values are estimated as part of θ
DM = DataModel(DS, ModelMap((x,p)->p[1]*x + p[2]))
# θ = [x̂₁,x̂₂,x̂₃,x̂₄, p₁, p₂, cx₁, cy₁]

# Default error models (σ = exp10(c) for both)
DS_default = UnknownVarianceDataSet(1:10, rand(10))

# Upgrade from DataSetUncertain
uvd = UnknownVarianceDataSet(DataSetUncertain(1:5, rand(5), (x,y,c)->exp10(-c[1]), [0.0]))
```

**Likelihood**: `log L_y + log L_x` — sum of y-likelihood and x-likelihood, each Gaussian at predicted values.

**Key accessors**: `xinverrormodel(DS)`, `xerrorparams(DS, mle)`, `SkipXs(DS)`, `xpars(DS)`.

**Pitfalls:**
- Parameter count increases by `Npoints * xdim` (x-values are parameters) — profile likelihoods can be expensive
- `SplitErrorParams` returns **3-tuple**: `(modelparams, xerrorparams, yerrorparams)`
- `DefaultErrorModelSplitter(n, m)`: `θ[1:end-n-m]` = model+xpars, `θ[end-n-m+1:end-m]` = x-error, `θ[end-m+1:end]` = y-error
- Missing x-values not supported (only missing y)

## Helper Types

- **`InformationGeometry.Dirac(μ)`** — point mass distribution for zero x-uncertainty. Not exported.
- **`DataDist(Y, Sig)`** / `DataDist(Y, Σ)` — build a distribution from data + uncertainties (univariate Normal product or MvNormal).
- **`ProportionalInvErrorModel(Ts, σ_inv_half)`** — fixed relative weights, estimate overall scale.
- **`DefaultErrorModelSplitter(n)`** — for `DataSetUncertain`: last `n` params are error params. `(n, m)` for `UnknownVarianceDataSet`: last `n` = x-error, last `m` = y-error.

## General Pitfalls (all types)

1. **`dims` ordering**: always `(Npoints, xdim, ydim)`. Most common construction error.
2. **Error model return types**: `ydim=1` → Number; `ydim>1` → Vector (diagonal) or Matrix (Cholesky `S`, `Σ=S'S`).
3. **Parameter ordering in DataModel**: for estimated-uncertainty types, `errorparamsplitter` processes θ **before** forwarding to the model. The model only sees `modelparams`.
4. **`DOF`**: total estimated parameters (model + error + estimated x-values). Affects Bessel correction and must match `pdim` of your `ModelMap`.
5. **Plotting**: `xdim>1` generally unsupported; `CompositeDataSet` needs `ydim≤16`.

## Cross-References

- **`ig-reference`**: basic `DataSet`, `DataModel`, `ModelMap`, MLE, confidence regions, profile likelihoods, all shared API.
- **`ig-julia-project`**: project setup, data interpretation, package installation.
- **`ig-calibration-diagnosis`**: calibration + residual diagnostics with these types.
- **`ig-identifiability-uq`**: profile-likelihood UQ — high parameter dims from `UnknownVarianceDataSet` make this expensive.
- **`mechanistic-model-development`**: coordinator — references this skill when advanced dataset types are needed.
