---
name: ig-advanced-datasets
description: "Advanced DataSet types in InformationGeometry.jl: DataSetExact, CompositeDataSet, GeneralizedDataSet, DataSetUncertain, UnknownVarianceDataSet."
version: 0.1.0
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

Advanced `DataSet` types in InformationGeometry.jl v1.32.2: `DataSetExact`, `CompositeDataSet`, `GeneralizedDataSet`, `DataSetUncertain`, and `UnknownVarianceDataSet`.

This skill is a **reference** — copy-paste constructors, examples, and accessor tables directly. Every code block is runnable Julia.

## 1. When to Use Each Type

The basic `DataSet` (documented in `ig-reference`) handles the simplest case: Gaussian y-errors of known magnitude, no x-errors, no missing data, no non-Gaussian distributions. When your data goes beyond that, pick the advanced type using the comparison table below.

| Type | Non-Gaussian y | Missing values | x-uncertainty | Mixed x-y unc. | y-unc. estimation | x-unc. estimation |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| `DataSet` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `DataSetExact` | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `CompositeDataSet` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `GeneralizedDataSet` | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `DataSetUncertain` | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| `UnknownVarianceDataSet` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

**Quick decision guide:**

- **x-values uncertain?** → `DataSetExact` (independent x-y) or `GeneralizedDataSet` (correlated x-y).
- **Non-Gaussian y-errors?** → `DataSetExact` or `GeneralizedDataSet` (distribution-based) or `CompositeDataSet` (per-component distributions).
- **Missing y-values?** → `CompositeDataSet` (different x-points per observable) or `DataSetUncertain` (same x, some y missing).
- **Unknown y-variance (estimate from data)?** → `DataSetUncertain` or `UnknownVarianceDataSet`.
- **Unknown x-variance too?** → `UnknownVarianceDataSet`.
- **Multiple observables at different time points?** → `CompositeDataSet`.
- **Correlations between x and y errors?** → `GeneralizedDataSet`.

## 2. Type Hierarchy

```
AbstractDataSet
├── AbstractFixedUncertaintyDataSet     (uncertainties known a-priori)
│   ├── DataSet                         (basic — see ig-reference)
│   ├── DataSetExact                    (distribution-based, optional x-unc)
│   ├── CompositeDataSet                (multi-observable, missing data)
│   └── GeneralizedDataSet              (joint x-y distribution, mixed cov)
└── AbstractUnknownUncertaintyDataSet   (uncertainties estimated via error model)
    ├── DataSetUncertain                (y-variance estimated, optional missing)
    └── UnknownVarianceDataSet          (both x- and y-variance estimated)
```

Key distinctions:
- `AbstractFixedUncertaintyDataSet`: uncertainties are given at construction time and never change. `HasEstimatedUncertainties()` returns `false`. `SplitErrorParams()` returns identity.
- `AbstractUnknownUncertaintyDataSet`: uncertainties depend on error parameters that are jointly fit with model parameters. `HasEstimatedUncertainties()` returns `true`. `SplitErrorParams()` splits θ into (modelparams, errorparams).

## 3. DataSetExact

### Purpose

`DataSetExact` stores data as **probability distributions** over x-space and y-space, allowing:
- Uncertainties in the independent variables (x-errors)
- Non-Gaussian y-uncertainties (Cauchy, Student-t, log-normal, etc.)
- Exact (non-approximate) likelihood evaluation via `logpdf`

When x-uncertainty is zero, `DataSetExact` with a `Dirac` x-distribution is numerically equivalent to `DataSet`.

### Constructors

```julia
# From arrays with y-errors only (x-errors = 0)
DataSetExact(x::AbstractArray, y::AbstractArray, allysigmas::Real=1.0; kwargs...)
DataSetExact(x::AbstractArray, y::AbstractArray, yerr::AbstractArray; kwargs...)

# From arrays with x-errors and y-errors
DataSetExact(x::AbstractArray, allxsigmas::Real, args...; kwargs...)
DataSetExact(x::AbstractArray, xerr::AbstractArray, y::AbstractArray, allysigmas::Real; kwargs...)
DataSetExact(x::AbstractArray, Σ_x::AbstractArray, y::AbstractArray, Σ_y::AbstractArray, Dims=nothing; kwargs...)

# From a DataSet + x-uncertainties
DataSetExact(DS::DataSet, σ_x::Real; kwargs...)
DataSetExact(DS::DataSet, Σ_x::AbstractArray; kwargs...)

# From any AbstractDataSet (converts via distributions)
DataSetExact(DS::AbstractDataSet; kwargs...)

# From a DataModel
DataSetExact(DM::AbstractDataModel, args...; kwargs...)

# From distributions directly
DataSetExact(xd::Distribution, yd::Distribution; kwargs...)                        # assumes (N,1,1)
DataSetExact(xd::Distribution, yd::Distribution, dims::Tuple{Int,Int,Int}; kwargs...)

# From Measurement type
DataSetExact(x::AbstractArray{<:Number}, y::AbstractArray{<:Measurement}; kwargs...)

# Full inner constructor (rarely called directly)
DataSetExact(xd::Distribution, yd::Distribution, dims::Tuple{Int,Int,Int},
             InvCov::AbstractMatrix{<:Number}, WoundX::Union{AbstractVector,Nothing},
             xnames, ynames, name)
```

### Worked Examples

```julia
using InformationGeometry, Distributions, LinearAlgebra

# Example 1: Basic — y-errors only, x-errors zero (equivalent to DataSet)
DS1 = DataSetExact([1,2,3,4], [4,5,6.5,7.8], [0.5,0.45,0.6,0.8])

# Example 2: With x-errors
DS2 = DataSetExact([1,2,3,4], [0.1,0.1,0.1,0.1], [4,5,6.5,7.8], [0.5,0.45,0.6,0.8])

# Example 3: Non-Gaussian y-uncertainty using Cauchy distribution
X = product_distribution([Normal(0, 1), Cauchy(2, 0.5)])
Y = MvTDist(2, [3, 8.], [1 0.5; 0.5 3])
DS3 = DataSetExact(X, Y, (2,1,1))

# Example 4: Dirac for zero x-uncertainty (numerically == DataSet)
DS4a = DataSetExact(InformationGeometry.Dirac([1,2]), MvNormal([5,6], Diagonal([0.1, 0.2].^2)))
DS4b = DataSetExact([1,2], [5,6], [0.1, 0.2])
DS4c = DataSet([1,2], [5,6], [0.1, 0.2])
# DS4a == DS4b == DS4c  →  true

# Example 5: Convert from existing DataSet, adding x-uncertainties
DS_basic = DataSet([1,2,3,4], [4,5,6.5,7.8], [0.5,0.45,0.6,0.8])
DS_with_x = DataSetExact(DS_basic, 0.2)  # uniform x-sigma of 0.2

# Example 6: Multi-dimensional x and y
X = [0.9, 1.0, 1.1, 1.9, 2.0, 2.1, 2.9, 3.0, 3.1, 3.9, 4.0, 4.1]
Y = [1.0, 5.0, 4.0, 8.0, 9.0, 13.0, 16.0, 20.0]
Σ_x = Diagonal(0.1 * ones(12))   # 4 points × 3 x-components
Σ_y = Diagonal([2.0, 4.0, 2.0, 4.0, 2.0, 4.0, 2.0, 4.0])  # 4 points × 2 y-components
DS6 = DataSetExact(X, Σ_x, Y, Σ_y, (4, 3, 2))
```

### Accessor Methods

| Method | Returns |
|--------|---------|
| `dims(DS)` | `(Npoints, xdim, ydim)` tuple |
| `xdata(DS)` | Mean of x-distribution (`GetMean(xdist(DS))`) |
| `ydata(DS)` | Mean of y-distribution (`GetMean(ydist(DS))`) |
| `xdist(DS)` | The x distribution |
| `ydist(DS)` | The y distribution |
| `xsigma(DS)` | Vectorized x standard deviations from `Sigma(xdist(DS))` |
| `ysigma(DS)` | Vectorized y standard deviations from `Sigma(ydist(DS))` |
| `xInvCov(DS)` | Inverse covariance of x-distribution (`InvCov(xdist(DS))`) |
| `yInvCov(DS)` | `DSE.InvCov` (cached inverse y-covariance) |
| `HasXerror(DS)` | `false` if xdist is Dirac, else checks if any xsigma > 0 |
| `WoundX(DS)` | Vector of SVectors (x-values grouped by point), or flat vector if xdim=1 |
| `Xnames(DS)` / `Ynames(DS)` | Vector of Symbol names |
| `xnames(DS)` / `ynames(DS)` | Vector of String names |
| `name(DS)` | Symbol name of dataset |

### Likelihood Implementation

```julia
# LogLike evaluates logpdf of both distributions
LogLike(DSE::DataSetExact, x::AbstractVector, y::AbstractVector) =
    logpdf(xdist(DSE), x) + logpdf(ydist(DSE), y)

# _loglikelihood fixes x to xdata and evaluates y at model prediction
_loglikelihood(DSE::DataSetExact, model, θ) =
    LogLike(DSE, xdata(DSE), EmbeddingMap(DSE, model, θ))
```

The likelihood is **exact** — it uses `logpdf` of the stored distributions rather than a Gaussian approximation. This means non-Gaussian distributions (Cauchy, Student-t, etc.) are handled natively.

`_Score` uses `gradlogpdf(ydist(DSE), ...)` for the score function.

`DataMetric(P::Distribution)` returns `InvCov(P)` for most distributions, with special cases:
- Cauchy (df=1): `0.5 * InvCov(P)` (the Fisher information of a Cauchy is half the inverse covariance)
- Product of Cauchy: per-component `0.5` scaling

### Conversions

```julia
# DataSetExact → DataSet (drops x-uncertainties with a warning)
DataSet(DSE::DataSetExact)

# DataSet → DataSetExact (adds x-uncertainties)
DataSetExact(DS::DataSet, σ_x)
DataSetExact(DS::DataSet, Σ_x)

# Any AbstractDataSet → DataSetExact
DataSetExact(DS::AbstractDataSet)
```

### Pitfalls

1. **x-uncertainty dropped on conversion to DataSet**: `DataSet(DSE)` silently drops x-errors (only warns). Always check `HasXerror(DSE)` before converting.
2. **Dirac is not exported**: Use `InformationGeometry.Dirac(...)` — it is not in the public export list.
3. **Non-Gaussian distributions need `gradlogpdf`**: The Score function requires `Distributions.gradlogpdf` to work for your chosen distribution. Cauchy has a custom `gradlogpdf` fix in IG.jl, but other exotic distributions may not.
4. **`DataMetric` for Student-t with df≠1**: Returns a warning and falls back to `InvCov(P)`, which may not be the correct Fisher information. Only df=1 (Cauchy) is handled exactly.
5. **Dimension inference from distributions**: When constructing from distributions without explicit `dims`, it assumes `(N,1,1)`. For multi-dimensional data, always pass the `dims` tuple.
6. **yInvCov is cached at construction**: Unlike xsigma/ysigma which compute from the distribution on each call, `yInvCov` is pre-computed and stored as `DSE.InvCov`.

## 4. CompositeDataSet

### Purpose

`CompositeDataSet` handles **multiple observables** (multi-component y-data) where:
- Different observables may be measured at **different x-values** (e.g., different time points)
- Some y-values may be **missing** (not recorded at every time point)
- Each observable can have its own uncertainty

It works by splitting the data into a vector of simpler `DataSet` (or `DataSetExact`) objects — one per observable — that share the same x-dimensionality. The constructor automatically handles missing values by dropping rows where a given observable is missing.

### Constructors

```julia
# From a vector of AbstractDataSets (one per observable)
CompositeDataSet(pDSs::AbstractVector{<:AbstractDataSet}; kwargs...)
CompositeDataSet(DS::AbstractDataSet, args...; kwargs...)  # single DS wrapped in vector

# From a DataFrame (most common entry point)
CompositeDataSet(df::AbstractDataFrame, xdims::Int=1, ydims::Int=(ncols-1)÷2;
    xerrs::Bool=false, stripedXs::Bool=true, stripedYs::Bool=true, kwargs...)

# From separate x, y, sigma DataFrames
CompositeDataSet(xdf::AbstractDataFrame, ydf::AbstractDataFrame, sig::Real=1.0; kwargs...)
CompositeDataSet(xdf::AbstractDataFrame, ydf::AbstractDataFrame, sigdf::AbstractDataFrame; kwargs...)

# From arrays
CompositeDataSet(xdf::AbstractArray, ydf::AbstractMatrix, sigdf::AbstractMatrix; kwargs...)

# From a DataModel
CompositeDataSet(DM::AbstractDataModel, args...; kwargs...)

# Full inner constructor (rarely called directly)
CompositeDataSet(DSs, InvCov, logdetInvCov, WoundX, SharedYdim::Val, name)

# ReadLongTable for PEtab-style long-format DataFrames
ReadLongTable(df; Time=:time, Measurement=:measurement, Noise=:noiseParameters,
    SimulationId=:simulationConditionId, ObservableId=:observableId)
```

**DataFrame column layout** (with `stripedYs=true`, `xerrs=false`):
```
| x₁ | y₁ | σ₁ | y₂ | σ₂ | ... |
```
With `stripedYs=false`:
```
| x₁ | y₁ | y₂ | ... | σ₁ | σ₂ | ... |
```
With `xerrs=true` and `stripedXs=true`:
```
| x₁ | σx₁ | y₁ | σ₁ | y₂ | σ₂ | ... |
```

### Worked Examples

```julia
using InformationGeometry, DataFrames

# Example 1: From DataFrame with missing values (striped format)
t = [1,2,3,4]
y₁ = [2.5, 6, missing, 9]
y₂ = [missing, 5, 3.1, 1.4]
σ₁ = 0.3*ones(4)
σ₂ = [missing, 0.2, 0.1, 0.5]
df = DataFrame([t y₁ σ₁ y₂ σ₂], :auto)
cds = CompositeDataSet(df, 1, 2; xerrs=false, stripedYs=true)
# Creates two DataSet components:
#   - Component 1: y₁ at times [1,2,4] (row 3 dropped — y₁ missing)
#   - Component 2: y₂ at times [2,3,4] (row 1 dropped — y₂ missing)

# Example 2: NaN also works as "missing"
df2 = DataFrame([1:5. [1,2,3,NaN,5.] [2,NaN,4,6,8.] 0.5ones(5) 0.3ones(5)], :auto)
cds2 = CompositeDataSet(df2; stripedYs=false)

# Example 3: From individual DataSets
ds1 = DataSet([1,2,4], [2.5, 6, 9], 0.3)
ds2 = DataSet([2,3,4], [5, 3.1, 1.4], 0.2)
cds3 = CompositeDataSet([ds1, ds2])

# Example 4: With a model
cdm = DataModel(cds, (x,p)->[p[1]*x, p[2]*x])

# Example 5: ReadLongTable for PEtab-style data
# (long format: one row per measurement, columns for time, measurement, noise, condition, observable)
# ReadLongTable(df_long)

# Example 6: With x-errors via DataFrame
df_xerr = DataFrame([1:4 0.1ones(4) [2.5,6,missing,9] 0.3ones(4) [missing,5,3.1,1.4] 0.2ones(4)], :auto)
cds_xerr = CompositeDataSet(df_xerr, 1, 2; xerrs=true, stripedXs=true, stripedYs=true)
```

### Accessor Methods

| Method | Returns |
|--------|---------|
| `Data(CDS)` | Vector of component AbstractDataSets |
| `dims(CDS)` | Not directly defined — use `Npoints`, `xdim`, `ydim` |
| `xdata(CDS)` | Concatenated x-data from all components |
| `ydata(CDS)` | Concatenated y-data from all components |
| `xdim(CDS)` | x-dimension (from first component) |
| `ydim(CDS)` | Total y-dimension (sum of component ydims) |
| `Npoints(CDS)` | Total number of points across all components |
| `DataspaceDim(CDS)` | Sum of `Npoints(DS)*ydim(DS)` across components |
| `xsigma(CDS)` | Block-reduced x sigmas |
| `ysigma(CDS)` | Block-reduced y sigmas |
| `yInvCov(CDS)` | Block-diagonal inverse y-covariance |
| `WoundX(CDS)` | Unique concatenated WoundX values |
| `HasXerror(CDS)` | Inherited from components |
| `HasMissingValues(CDS)` | `true` if components have different Npoints |
| `Xnames(CDS)` | From first component |
| `Ynames(CDS)` | Concatenated Ynames from all components |
| `name(CDS)` | Symbol name |

### Likelihood Implementation

The likelihood is computed as the **sum of component likelihoods**, since each component is an independent `DataSet` or `DataSetExact`:

```julia
# EmbeddingMap dispatches to _CustomOrNot which evaluates the model
# at each component's x-values and fills the result vector.
# The likelihood is then the sum of individual component log-likelihoods.
```

The `InvCov` is a block-diagonal matrix assembled via `mapreduce(yInvCov, BlockMatrix, DSs)`.

### Conversions

```julia
# Type conversion (e.g., to Float16 or BigFloat)
T(CDS::CompositeDataSet) where T<:Number

# CompositeDataSet does not directly convert to DataSet
# (different components may have different x-values)
```

### Pitfalls

1. **All components must share the same xdim**: The constructor checks this and throws if violated.
2. **Components are split by y-component**: The constructor calls `SplitDS` which splits each input dataset by its y-components, so a single multi-y DataSet gets split into multiple single-y components.
3. **SharedYdim type parameter**: `CompositeDataSet{true,1}` means all components have ydim=1 (the common case). `SharedYdim=Val(false)` paths are not fully implemented — `_CustomOrNot` throws for `Val{false}`.
4. **In-place ModelMaps not supported**: `_CustomOrNot` throws for `ModelMap{true}` (in-place models).
5. **Missing values via `missing` or `NaN`**: Both are handled — `DitchMissingRows` treats both `missing` and non-finite (NaN) as missing.
6. **Whole rows missing**: If all y-components are missing for a given x-row, that row must be removed manually before constructing the `CompositeDataSet`. The `ReadIn` function handles this per-component but not for entire rows.
7. **`xerrs=true` changes column count expectation**: With x-errors, the DataFrame must have `2*xdims + 2*ydims` columns; without, it needs `xdims + 2*ydims`.
8. **Plotting limited to xdim=1 and ydim≤16**: The recipe throws for `xdim != 1` or too many observables.

## 5. GeneralizedDataSet

### Purpose

`GeneralizedDataSet` handles the most general case of **joint x-y uncertainty** with possible **correlations between x and y errors**. It stores a single multivariate distribution over the combined (x, y) space, where off-diagonal blocks of the covariance matrix encode x-y correlations.

Use this when:
- x and y errors are correlated (e.g., derived from a shared measurement process)
- You need non-Gaussian joint distributions over x and y
- `DataSetExact` is insufficient because it assumes x and y are independent

### Constructors

```julia
# From a joint distribution (most common)
GeneralizedDataSet(dist::ContinuousMultivariateDistribution; kwargs...)
GeneralizedDataSet(dist::ContinuousMultivariateDistribution, dims::Tuple{Int,Int,Int}; kwargs...)

# From a DataSetExact (converts to joint distribution)
GeneralizedDataSet(DS::DataSetExact; kwargs...)

# From any args that work for DataSetExact (delegates to DataSetExact → GeneralizedDataSet)
GeneralizedDataSet(args...; kwargs...)

# From a vector and covariance matrix (assumes MvNormal)
GeneralizedDataSet(X::AbstractVector{<:Number}, Σ::AbstractMatrix{<:Number}; kwargs...)

# From a DataModel
GeneralizedDataSet(DM::AbstractDataModel, args...; kwargs...)

# Full inner constructor (rarely called directly)
GeneralizedDataSet(dist, dims, WoundX, xnames, ynames, name)
```

**Key behavior**: If the distribution is **separable** (x and y independent, e.g., `GeneralProduct` with 2 components), the constructor automatically returns a `DataSetExact` instead and prints an info message.

### Worked Examples

```julia
using InformationGeometry, Distributions, LinearAlgebra

# Example 1: Joint MvNormal with x-y correlations
# 2 data points, each with xdim=1, ydim=1
# Full covariance includes x-y cross terms
μ = [1.0, 2.0, 3.0, 4.0]  # [x₁, x₂, y₁, y₂]
Σ = [0.1 0.0 0.05 0.0;    # x₁-x₁, x₁-x₂, x₁-y₁, x₁-y₂
     0.0 0.1 0.0 0.05;     # x₂-x₁, x₂-x₂, x₂-y₁, x₂-y₂
     0.05 0.0 0.2 0.0;     # y₁-x₁, y₁-x₂, y₁-y₁, y₁-y₂
     0.0 0.05 0.0 0.2]     # y₂-x₁, y₂-x₂, y₂-y₁, y₂-y₂
joint_dist = MvNormal(μ, Σ)
gds = GeneralizedDataSet(joint_dist, (2, 1, 1))

# Example 2: From a vector and full covariance (assumes MvNormal)
X = [1.0, 2.0, 3.0, 4.0]
Σ_full = Matrix(Diagonal([0.1, 0.1, 0.2, 0.2]))
gds2 = GeneralizedDataSet(X, Σ_full)

# Example 3: Convert from DataSetExact (warns if xdist is Dirac)
dse = DataSetExact([1,2], 0.1, [3,4], 0.2)
gds3 = GeneralizedDataSet(dse)

# Example 4: With a model
gdm = DataModel(gds, (x,p)->p[1]*x + p[2], [1.0, 1.0])
```

### Accessor Methods

| Method | Returns |
|--------|---------|
| `dims(GDS)` | `(Npoints, xdim, ydim)` tuple |
| `dist(GDS)` | The joint distribution |
| `xdata(GDS)` | First `Npoints*xdim` entries of `GetMean(dist(GDS))` |
| `ydata(GDS)` | Last `Npoints*ydim` entries of `GetMean(dist(GDS))` |
| `xdist(GDS)` | Marginal x-distribution (if separable, the component; else reconstructed) |
| `ydist(GDS)` | Marginal y-distribution (if separable, the component; else reconstructed) |
| `xsigma(GDS)` | x block of `Sigma(dist(GDS))` |
| `ysigma(GDS)` | y block of `Sigma(dist(GDS))` |
| `xInvCov(GDS)` | x block of `InvCov(dist(GDS))` |
| `yInvCov(GDS)` | y block of `InvCov(dist(GDS))` |
| `fullsigma(GDS)` | Full covariance `Sigma(dist(GDS))` (unique to GeneralizedDataSet) |
| `isseparable(GDS)` | `true` if dist is a `GeneralProduct` with 2 components (x,y independent) |
| `WoundX(GDS)` | Vector of SVectors or flat vector |
| `Xnames`/`Ynames`/`name` | Standard name accessors |

### Likelihood Implementation

```julia
# LogLike evaluates logpdf of the JOINT distribution
LogLike(GDS::GeneralizedDataSet, x, y) = logpdf(dist(GDS), vcat(x, y))

# _loglikelihood fixes x to xdata and evaluates at model prediction
_loglikelihood(GDS::GeneralizedDataSet, model, θ) =
    LogLike(GDS, xdata(GDS), EmbeddingMap(GDS, model, θ))
```

The `_Score` function uses `gradlogpdf(dist(GDS), [woundX; ypred])` and extracts only the y-block of the gradient (since x is fixed).

### Conversions

```julia
# From DataSetExact
GeneralizedDataSet(DS::DataSetExact)

# Type conversion
T(GDS::GeneralizedDataSet) where T<:Number

# If the distribution is separable, construction returns DataSetExact automatically
```

### Pitfalls

1. **Separable distributions auto-convert**: If you pass a `GeneralProduct` of two independent distributions, the constructor returns a `DataSetExact`, not a `GeneralizedDataSet`. This is by design (use the more efficient type when possible), but can be surprising.
2. **Distribution length must be even**: When no `dims` is given, it assumes `length(dist)÷2` points with `xdim=ydim=1`. For anything else, pass `dims` explicitly.
3. **`xdim=0` is allowed**: The default `remake` constructor uses `dims=(1,0,1)`, but this is a degenerate case.
4. **Performance**: `GeneralizedDataSet` is less performant than `DataSetExact` for separable distributions because it works with the full joint covariance. Use `DataSetExact` when x and y are independent.
5. **Dirac xdist warning**: Converting from `DataSetExact` with a `Dirac` x-distribution prints a warning (the Dirac has zero variance, so x-y correlations are meaningless).
6. **`InvCov(GDS)` returns full inverse**: Use `xInvCov(GDS)` and `yInvCov(GDS)` to get the marginal blocks.

## 6. DataSetUncertain

### Purpose

`DataSetUncertain` handles data where the **y-variance is unknown a-priori** and must be estimated from the data via a parametric error model `σ(x, y_pred, c)`. The error parameters `c` are jointly fit with the model parameters `θ`.

Key features:
- Reciprocal error model `σ⁻¹(x, y_pred, c)` is specified (for performance)
- Supports missing y-values (via `datakeep` mask)
- Bessel correction for unbiased variance estimation
- Error parameters can be separated from model parameters via `errorparamsplitter`

### Constructors

```julia
# Simplest: x, y, reciprocal error model, initial error params
DataSetUncertain(x::AbstractVector, y::AbstractVector, σ⁻¹::Function, c::AbstractVector; BesselCorrection=false)

# With explicit errorparamsplitter
DataSetUncertain(x::AbstractVector, y::AbstractVector, σ⁻¹::Function, errorparamsplitter::Function, c::AbstractVector, dims::Tuple{Int,Int,Int}; BesselCorrection=false)

# From matrices (multi-dimensional x/y)
DataSetUncertain(X::AbstractArray, Y::AbstractArray, dims::Tuple{Int,Int,Int}=(size(X,1),...); verbose=true, kwargs...)
DataSetUncertain(X::AbstractArray{<:Number}, Y::AbstractArray{<:Number}, inverrormodel::Function, testpy::AbstractVector; kwargs...)

# From DataFrames
DataSetUncertain(X::AbstractDataFrame, Y::AbstractDataFrame, args...; xnames=names(X), ynames=names(Y), kwargs...)

# From a DataModel
DataSetUncertain(DM::AbstractDataModel, args...; kwargs...)

# From any AbstractDataSet
DataSetUncertain(DS::AbstractDataSet, args...; kwargs...)

# From a CompositeDataSet (reconstructs data matrices first)
DataSetUncertain(DS::CompositeDataSet, args...; kwargs...)

# Default error model (assumes σ(x,y,c) = exp10.(c))
DataSetUncertain(X::AbstractArray, Y::AbstractArray; verbose=true, kwargs...)

# Full inner constructor (rarely called directly)
DataSetUncertain(x, y, dims, inverrormodelraw, testout, inverrormodel,
    testpy, errorparamsplitter, datakeep, predkeep, woundXpred,
    nerrorparameters, ErrorModelDependencies, xnames, ynames, name; BesselCorrection=false)
```

### Worked Examples

```julia
using InformationGeometry, Distributions

# Example 1: Simplest case — estimate a single y-sigma
DS = DataSetUncertain([1,2,3,4], [4,5,6.5,7.8], (x,y,c)->1/exp10(c[1]), [0.5])
# Error model: σ = exp10(c[1]), reciprocal: 1/exp10(c[1])
# Initial guess: c = [0.5] → σ ≈ 3.16

# Example 2: With a model and DataModel
DMU = DataModel(DS, (x,p)->p[1]*x + p[2], [1.0, 1.0, 0.5])
# Parameters: [p₁, p₂, c₁] — model params first, error params last (by default)

# Example 3: Default error model (assumes σ = exp10.(c))
DS_default = DataSetUncertain(1:5, rand(5))
# Equivalent to: DataSetUncertain(1:5, rand(5), (x,y,c)->exp10(-c[1]), [0.1])

# Example 4: Missing y-values
T = Float64[1,4,3,3,2,5]
Y = [1.5, 8.0, 3.2, 6.1, 4.8, 10.2]
Yd = [[1.5, NaN, 3.2, 6.1, 4.8, NaN] [NaN, 8.0, NaN, 6.1, NaN, 10.2]]  # 6 points, 2 y-components, some missing
DSU_missing = DataSetUncertain(T, InformationGeometry.Unwind(Yd), (6,1,2),
    (x,y,c)->[exp10(-c[1]), exp10(-c[1])],
    InformationGeometry.DefaultErrorModelSplitter(1), [0.1])

# Example 5: Matrix error model for multi-dimensional y
DSU_matrix = DataSetUncertain(T, Y, (3,2,2),
    (x,y,c)->diagm([exp10(-c[1]), exp10(-c[1])]),
    InformationGeometry.DefaultErrorModelSplitter(1), [0.1])

# Example 6: Bessel correction for unbiased variance estimation
DS_bessel = DataSetUncertain([1,2,3,4], [4,5,6.5,7.8],
    (x,y,c)->1/exp10(c[1]), [0.5]; BesselCorrection=true)

# Example 7: Proportional error model (relative weights fixed, overall scale estimated)
using InformationGeometry
σ_known = [0.3, 0.5, 0.2, 0.4]  # relative uncertainties
errmod = ProportionalInvErrorModel([1,2,3,4], 1 ./ σ_known)
DS_prop = DataSetUncertain([1,2,3,4], [4,5,6.5,7.8], errmod, [0.0])

# Example 8: From SIR ODE model (real-world example from tests)
times = 1:14
infected = [3, 8, 28, 75, 221, 291, 255, 235, 190, 126, 70, 28, 12, 5]
SIRDSU = DataSetUncertain(times, infected, (x,y,p)->1/abs(p[1]), [15.0]; BesselCorrection=false)
```

### Accessor Methods

| Method | Returns |
|--------|---------|
| `xdata(DS)` | `DS.x` |
| `ydata(DS)` | `DS.y` (missing values removed) |
| `dims(DS)` | `DS.dims` = `(Npoints, xdim, ydim)` |
| `xsigma(DS, mle)` | Zeros (no x-errors) |
| `ysigma(DS, c)` | y-sigmas from error model evaluated at error params `c` |
| `yInvCov(DS, c)` | y inverse covariance from error model |
| `xInvCov(DS)` | Not defined (no x-errors) |
| `HasXerror(DS)` | `false` |
| `HasEstimatedUncertainties(DS)` | `true` |
| `HasMissingValues(DS)` | `true` if `predkeep` is not Nothing |
| `HasBessel(DS)` | The BesselCorrection type parameter (Bool) |
| `SplitErrorParams(DS)` | The `errorparamsplitter` function |
| `yinverrormodel(DS)` | Wrapped inverse error model (matrix output) |
| `yinverrormodelraw(DS)` | Raw inverse error model (Number/Vector/Matrix output) |
| `yerrorparams(DS, mle)` | Extracts error params from full MLE via splitter |
| `xerrorparams(DS, mle)` | `nothing` |
| `errormoddim(DS)` | Number of error parameters (`length(DS.testpy)`) |
| `NumberOfErrorParameters(DS, mle)` | `DS.nerrorparameters` |
| `DataspaceDim(DS)` | `length(ydata(DS))` (accounts for missing) |
| `WoundY(DS)` | WoundYmasked (missings filled) |
| `WoundYmasked(DS)` | Y data wound up, with NaN for missing entries |

### Likelihood Implementation

```julia
# For no missing data:
_loglikelihood(DS::DataSetUncertain, model, θ) = begin
    normalparams, errorparams = SplitErrorParams(DS)(θ)
    woundYpred = Windup(EmbeddingMap(DS, model, normalparams), ydim(DS))
    woundInvσ = map((x,y) -> Bessel .* yinverrormodelraw(x,y,errorparams), woundX, woundYpred)
    _EvalWoundsLogLikelihood(woundYpred, woundInvσ, woundY)
end
```

The likelihood is a Gaussian evaluated at the **model prediction** (not the data), because the variance depends on the predicted value. This is the "prediction-weighted" Gaussian likelihood:

```
log L = -½ N log(2π) + Σᵢ log(σ⁻¹(xᵢ, ȳᵢ, c)) - ½ Σᵢ [σ⁻¹(xᵢ, ȳᵢ, c) · (yᵢ - ȳᵢ)]²
```

where `ȳᵢ = model(xᵢ, θ)` is the model prediction.

For missing data, the prediction is evaluated at a sparsified set of unique x-values (`woundXpred`) and then indexed via `predkeep` to reconstruct only the observed entries.

`_FisherMetric` includes both the model Jacobian contribution and the error model Jacobian contribution (via `AddCovarianceContribution!`).

### Conversions

```julia
# From UnknownVarianceDataSet (drops x-error model)
DataSetUncertain(DS::UnknownVarianceDataSet; kwargs...)

# To UnknownVarianceDataSet (adds default x-error model)
UnknownVarianceDataSet(DS::DataSetUncertain{<:Any,<:Nothing}, invxerrormodel, testpx; kwargs...)

# Type conversion
T(DS::DataSetUncertain) where T<:Number
```

### Pitfalls

1. **Reciprocal error model**: You must provide `σ⁻¹`, not `σ`. The function signature is `(x, y_pred, c) -> σ⁻¹`. For `ydim=1`, return a Number; for `ydim>1`, return a Cholesky-like matrix `S` where `Σ = S' * S`.
2. **Error parameters are last by default**: `DefaultErrorModelSplitter(n)` assumes the last `n` parameters of `θ` are error parameters. If you need a different split, provide a custom `errorparamsplitter`.
3. **Exponentiate error parameters**: The docstring advises using `exp10(c)` (or `exp(c)`) because error parameters are penalized proportional to `log(c)` in the Gaussian normalization term. Using raw `c` can lead to negative variance.
4. **`ysigma` warns when called with `testpy`**: Calling `ysigma(DS)` without arguments uses the initial guess `DS.testpy` and warns about "cheating" — the uncertainty is not constructed around the MLE prediction.
5. **Missing data requires careful `dims`**: When y has missing values, `ydata(DS)` has the missings removed. The `dims` tuple still refers to the full `(Npoints, xdim, ydim)` including missing entries. The `datakeep` mask tracks which entries are observed.
6. **Bessel correction formula**: `sqrt((length(ydata) - DOF) / length(ydata))` where DOF is the total degrees of freedom (model + error parameters + estimated x-values if applicable).
7. **Error model dependencies**: The `ErrorModelDependencies` tuple `(Xdep, Ydep, Pdep)` tracks whether the error model depends on x, y, and parameters. By default it's `(true, true, true)`. Incorrect values can lead to wrong Fisher metrics.
8. **`WoundY` returns masked version**: `WoundY(DS::DataSetUncertain)` is overridden to return `WoundYmasked(DS)`, which fills missing entries with NaN. This is intentional for prediction purposes.

## 7. UnknownVarianceDataSet

### Purpose

`UnknownVarianceDataSet` extends `DataSetUncertain` to handle **unknown x-variance in addition to unknown y-variance**. Both x- and y-uncertainties are estimated via separate parametric error models:
- `σ_x⁻¹(x, y_pred, c_x)` — reciprocal x-error model
- `σ_y⁻¹(x, y_pred, c_y)` — reciprocal y-error model

This is the most general dataset type for the "errors-in-variables" problem where both independent and dependent variables have unknown uncertainty.

### Constructors

```julia
# Simplest: x, y, reciprocal x/y error models, initial x/y error params
UnknownVarianceDataSet(x::AbstractVector, y::AbstractVector,
    σ_x⁻¹::Function, σ_y⁻¹::Function, cx::AbstractVector, cy::AbstractVector;
    BesselCorrection=false)

# With explicit dims and errorparamsplitter
UnknownVarianceDataSet(x::AbstractVector, y::AbstractVector, dims::Tuple{Int,Int,Int},
    σ_x⁻¹::Function, σ_y⁻¹::Function, cx::AbstractVector, cy::AbstractVector,
    errorparamsplitter::Function; BesselCorrection=false)

# From matrices (multi-dimensional x/y)
UnknownVarianceDataSet(X::AbstractArray, Y::AbstractArray,
    dims::Tuple{Int,Int,Int}=(size(X,1),...); testpx, testpy, verbose=true, kwargs...)

# From a DataModel
UnknownVarianceDataSet(DM::AbstractDataModel, args...; kwargs...)

# From any AbstractDataSet
UnknownVarianceDataSet(DS::AbstractDataSet, args...; kwargs...)

# Default error models (assumes σ = exp10.(c) for both x and y)
UnknownVarianceDataSet(X::AbstractArray, Y::AbstractArray, dims; testpx, testpy, verbose, kwargs...)

# From DataSetUncertain (adds default x-error model)
UnknownVarianceDataSet(DS::DataSetUncertain{<:Any,<:Nothing}, invxerrormodel, testpx; kwargs...)

# Full inner constructor (rarely called directly)
UnknownVarianceDataSet(x, y, dims, invxerrormodelraw, invyerrormodelraw,
    testoutx, testouty, invxerrormodel, invyerrormodel,
    testpx, testpy, errorparamsplitter, SkipXs,
    xnames, ynames, name; BesselCorrection=false, datakeep, predkeep, woundXpred)
```

### Worked Examples

```julia
using InformationGeometry, Distributions

# Example 1: Simplest — estimate both x and y sigma
DS = UnknownVarianceDataSet([1,2,3,4], [4,5,6.5,7.8],
    (x,y,cx)->1/exp10(cx[1]),    # σ_x⁻¹
    (x,y,cy)->1/exp10(cy[1]),    # σ_y⁻¹
    [0.25],                       # initial cx
    [0.8])                        # initial cy

# Example 2: With a model
DMU = DataModel(DS, ModelMap((x,p)->p[1]*x + p[2]))
# Parameters: [x̂₁, x̂₂, x̂₃, x̂₄, p₁, p₂, cx₁, cy₁]
# (estimated x-values + model params + x-error params + y-error params)

# Example 3: Default error models
DS_default = UnknownVarianceDataSet(1:10, rand(10))
# Assumes σ_x = σ_y = exp10(c) with initial c = 0.1

# Example 4: Matching DataSetExact (from tests)
X = 1:10; Y = rand(10)
DSE = DataSetExact(X, 0.3ones(length(X)), Y, 0.2ones(length(Y)))
DME = DataModel(DSE, LinearModel)

DSU = UnknownVarianceDataSet(X, Y,
    (x,y,c)->exp10(-c[1]), (x,y,c)->exp10(-c[1]),
    [log10(0.3)], [log10(0.2)])
DMU = DataModel(DSU, ModelMap(LinearModel))

# The likelihoods agree:
# loglikelihood(DME, MLE(DME)) ≈ loglikelihood(DMU, [xdata(DME); MLE(DME); log10(0.3); log10(0.2)])

# Example 5: With a prior on the ratio σ_x/σ_y (from tests)
dmu = DataModel(
    UnknownVarianceDataSet(1:7, [2.7, 4.4, 5.3, 6.6, 6.5, 6.3, 7.7]),
    ModelMap((x,p)->p[2]*x/(x+p[1])),
    rand(11),
    p->logpdf(Normal(0,0.3), p[10]-p[11])  # prior on cx - cy
)

# Example 6: From DataSetUncertain (upgrade to also estimate x-variance)
dsu = DataSetUncertain(1:5, rand(5), (x,y,c)->exp10(-c[1]), [0.0])
uvd = UnknownVarianceDataSet(dsu)
```

### Accessor Methods

| Method | Returns |
|--------|---------|
| `xdata(DS)` | `DS.x` |
| `ydata(DS)` | `DS.y` (missing values removed if datakeep set) |
| `dims(DS)` | `DS.dims` = `(Npoints, xdim, ydim)` |
| `xsigma(DS, c)` | x-sigmas from x-error model at error params `c` |
| `ysigma(DS, c)` | y-sigmas from y-error model at error params `c` |
| `xInvCov(DS, c)` | x inverse covariance from error model |
| `yInvCov(DS, c)` | y inverse covariance from error model |
| `HasXerror(DS)` | `true` (always) |
| `HasEstimatedUncertainties(DS)` | `true` |
| `HasMissingValues(DS)` | `!isnothing(DS.datakeep)` |
| `HasBessel(DS)` | The BesselCorrection type parameter |
| `SplitErrorParams(DS)` | `θ -> (modelparams, xerrorparams, yerrorparams)` |
| `xinverrormodel(DS)` | Wrapped x inverse error model (matrix output) |
| `yinverrormodel(DS)` | Wrapped y inverse error model (matrix output) |
| `xinverrormodelraw(DS)` | Raw x inverse error model |
| `yinverrormodelraw(DS)` | Raw y inverse error model |
| `xerrorparams(DS, mle)` | Extracts x-error params from MLE |
| `yerrorparams(DS, mle)` | Extracts y-error params from MLE |
| `xerrormoddim(DS)` | `length(DS.testpx)` |
| `yerrormoddim(DS)` | `length(DS.testpy)` |
| `errormoddim(DS)` | Total error params (allows overlap) |
| `SkipXs(DS)` | Function to skip estimated x-values from parameter vector |
| `xpars(DS)` | `length(xdata(DS))` — number of estimated x-values |
| `DataspaceDim(DS)` | `sum(datakeep)` or `length(y)` if no missing |
| `WoundY(DS)` | Wound y-data (fills zeros for missing entries) |

### Likelihood Implementation

```julia
_loglikelihood(DS::UnknownVarianceDataSet, model, θ) = begin
    normalparams, xerrorparams, yerrorparams = SplitErrorParams(DS)(θ)
    # normalparams includes estimated x-values AND model parameters
    LiftedEmb = LiftedEmbedding(DS, model, n_model_params)
    XY = LiftedEmb(normalparams)  # [x̂₁..x̂ₙ, ŷ₁..ŷₙ]
    woundXpred = Windup(XY[1:Nx], xdim)
    woundYpred = Windup(XY[Nx+1:end], ydim)
    # Likelihood = y-likelihood + x-likelihood
    _EvalWoundsLogLikelihood(woundYpred, woundInvYσ, WoundY(DS))
    + _EvalWoundsLogLikelihood(woundXpred, woundInvXσ, WoundX(DS))
end
```

The total likelihood is the **sum of the x-likelihood and y-likelihood**:

```
log L = log L_y + log L_x
```

where each term is a Gaussian evaluated at the predicted values, with variance from the respective error model.

The `LiftedEmbedding` function combines the estimated x-values with the model parameters, runs the model, and produces the concatenated `[x̂, ŷ]` vector. This is the "lifted" or "errors-in-variables" formulation where x-values are treated as latent variables.

`_FisherMetric` includes contributions from:
1. Model Jacobian (via `LiftedEmbedding` Jacobian)
2. Y-error model Jacobian (via `AddCovarianceContribution!`)
3. X-error model Jacobian (via `AddCovarianceContribution!`)

### Conversions

```julia
# From DataSetUncertain (adds default x-error model)
UnknownVarianceDataSet(DS::DataSetUncertain{<:Any,<:Nothing}, invxerrormodel, testpx; kwargs...)

# To DataSetUncertain (drops x-error model)
DataSetUncertain(DS::UnknownVarianceDataSet; kwargs...)

# Type conversion
T(DS::UnknownVarianceDataSet) where T<:Number
```

### Pitfalls

1. **Estimated x-values are parameters**: The x-values themselves become parameters to be optimized. The total parameter vector is `[x̂₁, ..., x̂ₙ, model_params, cx, cy]`. This dramatically increases the parameter count (by `Npoints * xdim`).
2. **`SkipXs` function**: Used to skip the estimated x-values when extracting "pure" model parameters. The default is `(p -> view(p, n+1:length(p)))` where `n = length(xdata)`.
3. **Missing x-values not supported**: The constructor asserts `all(!ismissing(z) && isfinite(z) for z in x)`. Only missing y-values are handled.
4. **`errorparamsplitter` returns 3-tuple**: Unlike `DataSetUncertain` (2-tuple), the splitter must return `(modelparams, xerrorparams, yerrorparams)`.
5. **`DefaultErrorModelSplitter(n, m)`**: Splits `θ` into `θ[1:end-n-m]` (model+xpars), `θ[end-n-m+1:end-m]` (x-error), `θ[end-m+1:end]` (y-error).
6. **High parameter dimensionality**: With `N` data points, `p` model parameters, `nx` x-error params, and `ny` y-error params, the total is `N*xdim + p + nx + ny`. Profile likelihood computations can be expensive.
7. **`errormoddim` allows overlapping parameters**: The `errormoddim` function uses set union (`∪`) to count unique error parameters, allowing x and y error models to share parameters.
8. **Bessel correction uses both x and y dimensions**: `sqrt((Nx + Ny - DOF) / (Nx + Ny))` where `Nx = length(xdata)`, `Ny = DataspaceDim`.

## 8. Helper Types and Functions

### Dirac

```julia
# Not exported — use qualified name
InformationGeometry.Dirac(μ::AbstractVector)
```

A "distribution" that is a point mass at `μ`. Used to represent zero x-uncertainty in `DataSetExact`.

- `mean(Dirac(μ))` → `μ`
- `cov(Dirac(μ))` → `Diagonal(Zeros(length(μ)))`
- `invcov(Dirac(μ))` → `Diagonal(Inf * ones(length(μ)))`
- `logpdf(Dirac(μ), x)` → `log(x == μ ? 1.0 : 0.0)`

### GeneralProduct

```julia
# Exported
GeneralProduct(v::Vector{<:Distribution})
product_distribution(v::Vector{<:ContinuousMultivariateDistribution})
```

Extends `Distributions.product_distribution` to support products of arbitrary distributions (not just univariate). If all components are `MvNormal`, it collapses into a single `MvNormal`; otherwise it stores the vector of component distributions.

- `mean(P)` → `reduce(vcat, map(GetMean, P.v))`
- `cov(P)` → block-diagonal of component covariances
- `logpdf(P, X)` → sum of component `logpdf`s
- `isseparable(P::GeneralProduct)` → `true` if `length(P.v) == 2` (used by `GeneralizedDataSet`)

### DataDist

```julia
# Exported
DataDist(Y::AbstractVector, Sig::AbstractVector, Dist=Normal)  # product of univariate
DataDist(Y::AbstractVector, Sig::AbstractMatrix, Dist=MvNormal)  # multivariate
```

Helper to construct a distribution from data and uncertainties. For a vector of sigmas, creates a product of univariate `Dist(yᵢ, σᵢ)` (default: `Normal`). For a covariance matrix, creates a single `MvNormal(y, Σ)`.

### xDataDist / yDataDist

```julia
# Not exported, but used internally
xDataDist(DS::AbstractDataSet)  # Dirac if no x-error, else DataDist(xdata, xsigma)
yDataDist(DS::AbstractDataSet)  # DataDist(ydata, ysigma)
```

### Error Model Helpers

```julia
# ProportionalInvErrorModelFixed: exact x-value matching
ProportionalInvErrorModelFixed(Ts, YsigmaInvHalf)
ProportionalErrorModelFixed(Ts, Ysigma; kwargs...)

# ProportionalInvErrorModel: nearest-x matching (tolerant)
ProportionalInvErrorModel(Ts, YsigmaInvHalf; MaxDiff=minimum(diff(sort(Ts))))
ProportionalErrorModel(Ts, Ysigma; kwargs...)

# DefaultErrorModelSplitter: splits θ into (modelparams, errorparams)
DefaultErrorModelSplitter(n::Int)  # for DataSetUncertain
DefaultErrorModelSplitter(n::Int, m::Int)  # for UnknownVarianceDataSet: (model, xerr, yerr)

# Identity2Splitter: θ -> (θ, θ) — used when all params are both model and error
Identity2Splitter
Identity3Splitter  # θ -> (θ, θ, θ) for UVD
```

### Key Generic Functions (from GeneralDataStructures.jl)

```julia
# Winding / unwinding
WoundX(DS)      # xdata grouped into per-point vectors (or SVectors)
WoundY(DS)      # ydata grouped into per-point vectors
Windup(vec, dim) # reshape flat vector into vector of dim-length chunks
Unwind(vec)      # flatten vector of vectors into flat vector

# Dimensions
Npoints(DS)     # number of data points = dims[1]
xdim(DS)        # x-dimension per point = dims[2]
ydim(DS)        # y-dimension per point = dims[3]
DataspaceDim(DS) # total number of observed y-values

# Uncertainty queries
HasXerror(DS)              # any x-uncertainty?
HasMissingValues(DS)       # any missing y-values?
HasEstimatedUncertainties(DS)  # is uncertainty estimated (AbstractUnknownUncertaintyDataSet)?
HasBessel(DS)              # Bessel correction applied?

# Name accessors
xnames(DS) / ynames(DS)    # String names
Xnames(DS) / Ynames(DS)    # Symbol names
name(DS)                   # dataset name (Symbol)

# Error parameter management
SplitErrorParams(DS)       # θ -> (modelparams, errorparams...)
xerrorparams(DS, mle) / yerrorparams(DS, mle)
errormoddim(DS)            # number of error parameters

# Distributions
xdist(DS) / ydist(DS)      # marginal distributions
dist(DS)                   # joint distribution (GeneralProduct)
```

## 9. Cross-References

- **`ig-reference`**: Complete API reference including the basic `DataSet` type, `DataModel`, `ModelMap`, confidence regions, profile likelihoods, and all shared methods. Start here for any IG.jl question not covered by this skill.
- **`ig-julia-project`**: Setting up a Julia project with IG.jl dependencies, environment management, and package installation.
- **`ig-calibration-diagnosis`**: Phase B — model calibration and diagnostic workflows using these dataset types with `DataModel`.
- **`ig-identifiability-uq`**: Phase C — profile likelihood UQ. Especially relevant for `DataSetUncertain` and `UnknownVarianceDataSet` where high parameter dimensionality affects profile computations.
- **`mechanistic-model-development`**: Coordinator skill that orchestrates the full workflow. References this skill when advanced dataset types are needed.

## 10. General Pitfalls

### Across All Advanced Types

1. **`dims` tuple ordering**: Always `(Npoints, xdim, ydim)`. Mixing up the order is the most common construction error. The constructor checks `Npoints == length(x)/xdim == length(y)/ydim`.

2. **Unwinding**: Multi-dimensional data is stored as flat vectors (concatenated components). `Unwind(X)` flattens a matrix or vector-of-vectors. `Windup(flat, dim)` reverses this. The constructors handle this automatically, but be aware when accessing raw fields.

3. **`HealthyCovariance`**: Covariance matrices are automatically checked for positive-definiteness. Non-PD matrices are symmetrized; if still not PD, an error is thrown. Near-diagonal matrices are converted to `Diagonal` for performance.

4. **Type conversions**: `T(DS)` converts the numeric type (e.g., `Float64(DS)` or `BigFloat(DS)`). Not all types support all conversions — `DataSetUncertain` type conversion can fail in edge cases with complex error models.

5. **`remake` support**: All types support `SciMLBase.remake` via keyword-only constructors. This is used internally for `InformNames` and other operations that modify metadata.

6. **Plotting**: Most types have `RecipesBase` recipes, but:
   - `xdim > 1` is generally not supported for plotting
   - `CompositeDataSet` requires `ydim ≤ 16`
   - `DataSetUncertain` and `UnknownVarianceDataSet` have specialized plotting that shows estimated uncertainties

7. **`==` comparison**: Equality is defined for `DataSet` and `DataSetExact` (including distribution comparison). `CompositeDataSet` equality compares component-wise. `DataSetUncertain` and `UnknownVarianceDataSet` equality compares struct fields.

8. **Error model output types**: For `ydim=1`, the reciprocal error model must return a **Number**. For `ydim>1`, it must return either:
   - A **Vector** (interpreted as diagonal Cholesky), or
   - A **Matrix** (full Cholesky `S` where `Σ = S' * S`)
   
   The `ErrorModelTester` function automatically wraps vector output into `Diagonal` and unwraps `Diagonal` input.

9. **Parameter ordering in `DataModel`**: For `AbstractUnknownUncertaintyDataSet` types, the parameter vector visible to the outside is processed by `errorparamsplitter` **first**, before forwarding into the model. The model only sees `modelparams`. The `ModelMap` domain must account for this — it should be set up for the full parameter vector, not just the model parameters.

10. **`DOF` (degrees of freedom)**: Used in Bessel correction. `DOF(DS, θ)` is the total number of estimated parameters (model + error + estimated x-values). Make sure your `ModelMap` or `DataModel` correctly reports `pdim` including error parameters.
