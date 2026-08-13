---
name: informationgeometry-jl
description: Complete reference for InformationGeometry.jl. Covers DataSet/DataModel/ModelMap construction, MLE, confidence regions, profile likelihoods, KL divergence, geodesics, curvature, model transformations, ODE models, ConditionGrid, optimization, plotting, and parallelization.
version: 1.0
tags: [julia, information-geometry, parameter-inference, confidence-regions, profile-likelihood, fisher-metric, geodesics, mle, optimization]
related_skills: [pgfplotsx-jl, latex-pgfplots]
---

# InformationGeometry.jl Reference

## Companion Skills for Plotting

- **`pgfplotsx-jl`**: Julia→LaTeX translation for PGFPlotsX.jl syntax. Use it when constructing `Axis`, `Plot`, `Coordinates`, etc. for thesis figures (see the "Thesis pythontex Workflow" section below and the `pgfplotsx-jl` skill for full syntax).
- **`latex-pgfplots`**: Comprehensive pgfplots LaTeX option/command reference. Consult it for exact key names, defaults, and LaTeX-level behavior (marker names, error bar options, tick formatting, colormaps, libraries like fillbetween/groupplots/statistics, etc.).

## Architecture Overview

**Package**: `RafaelArutjunjan/InformationGeometry.jl` (>= v1.32.0)  
**Purpose**: Differential-geometric analyses of parameter inference problems
**Key capabilities**: MLE, exact confidence regions, profile likelihoods, Fisher metric, geodesics, curvature tensors, KL divergences, model comparison


## Core Types

### DataSet

Container for observed data. Simplest form: `(x, y, σ)` vectors.

```julia
# Basic: x, y, sigma (1D x, 1D y)
DS = DataSet([1,2,3,4], [4,5,6.5,9], [0.5,0.45,0.6,1])

# With full covariance matrix
DS = DataSet(x, y, Σ_y)  # Σ_y is (N*ydim) × (N*ydim) covariance

# Multi-dimensional x and y via dims tuple (Npoints, xdim, ydim)
DS = DataSet([1,2,3,2,1,4,2,6.8,3.5], [0.5,0.5,0.45,0.45,0.55,0.6], (3,1,2))

# No uncertainties given → assumes σ=1 for all
DS = DataSet([1,2,3], [2.1,3.9,6.2])

# With named axes (provide names when known)
DS = DataSet(days, infected, 15ones(14); xnames=["Days"], ynames=["Infected"])

# From DataFrame (Ydf and sigma need to have same shape)
DS = DataSet(Xdf, Ydf, sigma)
```

**Accessors**: `xdata(DS)`, `ydata(DS)`, `ysigma(DS)`, `yInvCov(DS)`, `xdim(DS)`, `ydim(DS)`, `Npoints(DS)`, `dims(DS)`, `Xnames(DS)`, `Ynames(DS)`



### Advanced DataSet types

| Type | Non-Gaussian y | Missing values | x-uncertainty | Mixed x-y unc. | y-unc. estimation | x-unc. estimation |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| `DataSet` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `DataSetExact` | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `CompositeDataSet` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `GeneralizedDataSet` | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `DataSetUncertain` | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| `UnknownVarianceDataSet` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

If the given data cannot be expressed with the `DataSet` type, e.g. due to missing values, non-zero x-uncertainty or unknown y-uncertainty which needs to be estimated via an error model, look up the corresponding doc string and / or constructor methods to find the required syntax for defining them.


### DataModel

Combines a `DataSet` + model function `model(x, θ)` + auto-derived Jacobian `dmodel(x, θ)` + MLE.

```julia
# Auto-derive Jacobian via AD, auto-optimize
DM = DataModel(DS, model)

# With initial guess
DM = DataModel(DS, model, [1.0, 2.5])

# With explicit Jacobian (StaticArrays recommended for performance)
using StaticArrays
function dmodel(x::Number, θ::AbstractVector{<:Number})
    @SMatrix [x 1.0]  # rows=ydim, cols=pdim
end
DM = DataModel(DS, model, dmodel)

# Skip auto-optimization
DM = DataModel(DS, model, [1.3, 0.2]; SkipOptim=true)

# With custom log-prior
DM = DataModel(DS, model, mle, logprior_fn)

# With ModelMap (structured model with domain, names, constraints)
DM = DataModel(DS, ModelMap(model, InDomain, Domain; pnames=[:a, :b], startp=[1.0, 0.1]))

# From DataFrame
DM = DataModel(DataFrame, model, args...; kwargs...)
```

**Key accessors**: `Data(DM)`, `Predictor(DM)`, `dPredictor(DM)`, `MLE(DM)`, `LogLikeMLE(DM)`, `LogPrior(DM)`, `pdim(DM)`, `xdim(DM)`, `ydim(DM)`, `pnames(DM)`, `xnames(DM)`, `ynames(DM)`, `InformationGeometry.Domain(DM)`

**Model function conventions**:
- `ydim=1`: model returns a scalar `Number`, NOT a 1-element vector
- `ydim>1`: model should return `SVector{ydim}` for performance
- `θ` is ALWAYS a vector, even for 1 parameter
- In-place models: `model!(output, x, θ)` with `ModelMap(...; inplace=true)`

### ModelMap

Structured model container with domain bounds, parameter names, and constraints.

```julia
M = ModelMap(model, InDomain, Domain; startp, pnames, inplace, custom)
M = ModelMap(model, Domain::HyperCube, xyp; kwargs...)
M = ModelMap(model, startp, InDomain, Domain; kwargs...)
```

- `Map`: function `(x, θ) -> model(x, θ)` (or `(out, x, θ)` if inplace)
- `InDomain`: `θ -> scalar ≥ 0` on valid domain, OR `θ -> bool`
- `Domain`: `HyperCube` specifying parameter bounds
- `pnames`: `Vector{Symbol}` for parameter names
- `startp`: initial parameter vector

### HyperCube

N-dimensional rectangular domain (bounding box for parameters or integration).

```julia
HC = HyperCube([-1,-1], [3,4])           # lower and upper bounds
HC = HyperCube([-100, 100])              # symmetric: [-100,100]
HC = HyperCube([[-20,50],[-20,50]])      # 2D box
HC = 5*FullDomain(3, 1.0)                # [-5,5]^3
```

**Methods**: `Center(HC)`, `Corners(HC)`, `CubeWidths(HC)`, `CubeVol(HC)`, `IsInDomain(θ, HC)`, `TranslateCube(HC, v)`, `ResizeCube(HC, f)`, `IntersectCube(HC1, HC2)`, `Cuboid(lo, hi)`

### ConditionGrid

Links multiple `DataModel`s with shared parameters across experimental conditions.

```julia
DM1 = DataModel(DS1, (x,p)->p[1].*x .+ p[2]; name="Condition 1")
DM2 = DataModel(DS2, (x,p)->p[1].*x .+ p[3]; name="Condition 2")
ParameterTrafo = [ViewElements([1,2]), ViewElements([1,3])]
CG = ConditionGrid([DM1, DM2], ParameterTrafo, rand(3); pnames=["Slope", "Offset_1", "Offset_2"])
```

- Individual conditions: `Conditions(CG)[i]`
- MLE per condition from outer params: `CG.Trafos[i](MLE(CG))`
- Plots, optimization, profiles work on `CG` like on `DataModel`
- Condition names must be unique (set via `name` kwarg)

## Maximum Likelihood Estimation

### Automatic (on DataModel construction)

By default, `DataModel(DS, model)` auto-optimizes unless `SkipOptim=true`.

### Manual optimization

```julia
# Using Optim.jl (default)
using Optim
mle = InformationGeometry.minimize(DM; meth=Optim.NewtonTrustRegion(), tol=1e-12,
    maxtime=60.0, Domain=HyperCube(zeros(2), 10ones(2)), verbose=true)

# Using Optimization.jl ecosystem
using Optimization, OptimizationOptimisers
mle = InformationGeometry.minimize(DM, rand(2); meth=OptimizationOptimisers.AdamW())

# Full solution object
mle = InformationGeometry.minimize(DM; Full=true)
```

### Multistart optimization

```julia
R = MultistartFit(DM; N=200, maxval=500, meth=LBFGS())
R = MultistartFit(DM, MvNormal([0,0], Diagonal(ones(2)));
        MultistartDomain=HyperCube([-1,-1],[3,4]), N=200, meth=Newton())

MLE(R)                     # best parameter configuration
WaterfallPlot(R; BiLog=true)
ParameterPlot(R; st=:dotplot)  # :dotplot, :boxplot, :violin
GetStepParameters(R, 2)    # params from n-th waterfall step
```

### Robust prefitting

```julia
# Minimize with multistart
Minimize(DM; MultistartDomain=HyperCube([-1,-1],[3,4]), Multistart=200, meth=Newton())

# Chain optimizers sequentially
using OptimizationOptimisers, OptimizationOptimJL
Prefit(DM; meth=[OAdam(), LBFGS(), Newton()], maxiters=[3000, 500, 50], tol=[0, 1e-8, 1e-12])

# Nested: multistart with Prefit per run
Minimize(DM; Multistart=100, MinimizeFunc=Prefit,
    meth=[OAdam(), LBFGS(), Newton()], maxiters=[3000, 500, 50], tol=[0, 1e-8, 1e-12])
```

### Other optimization functions

```julia
IncrementalTimeSeriesFit(DM; kwargs...)  # incremental fitting for time series
RobustFit(DM; kwargs...)                 # robust fitting wrapper
LocalMultistartFit(DM; kwargs...)        # local multistart around current MLE
AlternatingMinimization(DM; kwargs...)   # alternating minimization
```

## Likelihood & Model Comparison

```julia
loglikelihood(DM, θ)       # log-likelihood at θ
MLE(DM)                    # maximum likelihood estimate
LogLikeMLE(DM)             # log-likelihood at MLE
MLEuncert(DM)              # ≈ MLE ± √(inv(Fisher)) as Measurements.Measurement

AIC(DM, θ)                 # Akaike Information Criterion
AICc(DM, θ)                # corrected AIC
BIC(DM, θ)                 # Bayesian Information Criterion
CAIC(DM, θ)                # consistent AIC
CAICF(DM, θ)               # corrected AICF
ModelComparison(DM; kwargs...)  # compare multiple model fits
FCriterion(DM, θ)          # F-criterion
FTest(DM, θ)               # F-test
```

## Confidence Regions

### 2D confidence boundaries (via ODE integration)

```julia
sols = ConfidenceRegions(DM, 1:2; tol=1e-9)
# Returns vector of ODESolution objects for 1σ and 2σ boundaries
VisualizeSols(DM, sols)

# For 3+ parameters: intersect with family of 2D planes
Planes, sols3 = ConfidenceRegion(DM3, 1; tol=1e-6, Dirs=(1,2,3), N=50)
VisualizeSols(DM3, Planes, sols3)
```

### Confidence bands (pointwise prediction uncertainty)

```julia
M = ConfidenceBands(DM, sols[2]; plot=false)  # 2σ bands from ODE boundary
ConfidenceBands(DM, sols[2])                   # plots directly

ApproxConfidenceBands(DM; Confnum=2)           # linearized (Gaussian) approximation
```

### Contour diagrams (2D slices of log-likelihood)

```julia
ContourDiagram(DM; kwargs...)
ContourDiagramLowerTriangular(DM; kwargs...)   # all pairs in lower-triangular grid
```

### Helper functions

```julia
ConfVol(nσ)            # nσ → confidence volume (e.g. 1→0.6827)
InvConfVol(p)          # p → nσ (e.g. 0.95→1.9596)
ConfAlpha(nσ)          # significance level α from nσ
DOF(DM)               # degrees of freedom = pdim(DM)
ChisqCDF(DOF, x)      # χ² CDF
InvChisqCDF(DOF, p)   # inverse χ² CDF
ConfidenceBoundary(DM, Conf)    # single boundary
ConfidenceRegionVolume(DM, sol) # volume of confidence region
```

## Profile Likelihoods

### Standard (re-optimization at each step)

```julia
P = ParameterProfiles(DM, 2; N=100, plot=true, IsCost=true, adaptive=true, SaveTrajectories=true)
# 2nd arg: confidence level in σ units
# N: approximate number of profile points
# IsCost=false: rescale y-axis to show confidence level (χ²-based)
# IsCost=true (default): show raw 2(ℓ_MLE - PL_i(θ_i))
# adaptive=true: adaptive step sizing
# SaveTrajectories=true: save nuisance parameter paths

plot(P)             # all profiles
plot(P[i])          # single profile
plot(P; Interpolate=true)  # smooth interpolation
```

### ODE-based (Chen-Jennrich integration)

```julia
IP = IntegrationParameterProfiles(DM2, 3; reltol=1e-3, N=201, IsCost=true, γ=nothing, plot=true)
# reltol: integration tolerance (use <1e-5 for reliable results)
# γ: stabilization term (default nothing = disabled; should stay disabled with AD Hessian)
# N: interpolation points for log-likelihood evaluation (not integrator steps)
# N=nothing: only evaluate at integrator steps
```

### Confidence intervals from profiles

```julia
ConfidenceIntervals(P, 1)                    # 1σ intervals
ConfidenceIntervals(P, InvConfVol(0.95))     # 95% intervals
Tuple(ConfidenceIntervals(P, InvConfVol(0.95)))  # as tuple of (lo, hi) pairs

PracticallyIdentifiable(P[2])       # max σ level where profile is bounded both sides
ConfidenceIntervals(P[2], 3)[1]              # interval for single parameter (may be (-Inf, Inf))
```

### Profile path analysis

```julia
Trajectories(P)                     # access saved paths
PlotProfilePaths(P3; RelChange=false, OnlyHighlightTop=1, TrafoPath=identity, idxs=1:length(P3))
PlotProfilePathDiffs(P)             # finite differences of paths
PlotProfilePathNormDiffs(P)         # normalized differences
```

## Geodesics & Differential Geometry

### Fisher metric and geometric quantities

```julia
FisherMetric(DM, θ)                 # Fisher information matrix g_ab(θ)
Score(DM, θ, nothing)               # score vector ∂ln(p)/∂θ
GeometricDensity(DM, θ)             # geometric density on manifold
```

### Christoffel symbols & curvature

```julia
ChristoffelSymbol(DM, θ)            # Γ^a_{bc} at θ
Riemann(DM, θ)                      # Riemann curvature tensor R^a_{bcd}
Ricci(DM, θ)                        # Ricci tensor R_{ab}
RicciScalar(DM, θ)                  # Ricci scalar R

# Efron's statistical curvature measures
EfronMeanCurvature(DM, θ)
EfronRiemannCurvature(DM, θ)
EfronRicciCurvature(DM, θ)
EfronScalarCurvature(DM, θ)
EfronCurvatureIsotropy(DM, θ)
```

### Geodesics

```julia
# Compute a geodesic from MLE in a given direction
sol = ComputeGeodesic(DM, MLE(DM), v; Endtime=50., tol=1e-11, approx=false)

# Radial geodesics (emanate in all coordinate directions from MLE)
geos = RadialGeodesics(DM, nσ; kwargs...)

# Geodesic between two points
GeodesicBetween(DM, θ1, θ2; kwargs...)

# Geodesic length and crossing
GeodesicLength(DM, sol)             # arc length L[γ]
GeodesicCrossing(DM, sol, ConfVol(1))  # parameter value where geodesic crosses confidence level
DistanceAlongGeodesic(DM, sol, L)  # parameter value at arc-length L
GeodesicDistance(DM, θ1, θ2)       # geodesic distance between two points
GeodesicEnergy(DM, sol)            # geodesic energy

# Visualization
VisualizeGeos(DM, geos)
PlotAlongGeodesic(DM, sol)
EvaluateAlongGeodesic(DM, sol, ts)
```

**Approximate geodesics**: Set `approx=true` in `ComputeGeodesic` to assume constant Christoffel symbols (faster but inaccurate for high curvature).

### Exponential and logarithmic maps

```julia
ExponentialMap(DM, θ, v)    # exp_θ(v): tangent vector → point on manifold
LogarithmicMap(DM, θ1, θ2)  # log_θ1(θ2): point → tangent vector
```

## Kullback-Leibler Divergence

```julia
# Between Distributions.jl distributions
KullbackLeibler(Cauchy(1.,2.), Normal(-4.,0.5), HyperCube([-100,100]); tol=1e-12)

# Multivariate (adaptive h-cubature)
KullbackLeibler(MvNormal([0,2.5],diagm([1,4.])), MvTDist(1,[3,2],diagm([2.,3.])),
    HyperCube([[-20,50],[-20,50]]); tol=1e-8)

# Monte Carlo integration
KullbackLeibler(p, q, HC; Carlo=true, N=Int(5e6))

# Generic functions (user ensures positivity & normalization)
KullbackLeibler(f_p, f_q, HC)
```

Analytical KL for: Normal, MvNormal, Cauchy, Exponential, Weibull, Gamma. ODE-based for 1D unknown. Monte Carlo for multivariate unknown.

## Model Transformations

### Componentwise transforms

```julia
LogTransform(DM)           # θ → log(θ)
ExpTransform(DM)           # θ → exp(θ)
Log10Transform(DM)         # θ → log10(θ)
Exp10Transform(DM)         # θ → 10^θ
ReflectionTransform(DM)    # θ → -θ
ScaleTransform(DM)         # scale parameters

# Apply to specific components only
LogTransform(DM, [true, false])    # only transform θ[1]
Exp10Transform(DM, [false, true])  # only transform θ[2]

# On model function only (not full DataModel)
LogTransform(model)
```

### General componentwise transform

```julia
ComponentwiseModelTransform(DM, F; F_inv=nothing, F_deriv=nothing)
# F: strictly monotonic scalar function
```

### Multivariate transforms

```julia
TranslationTransform(DM, v)       # θ → θ + v
LinearTransform(DM, M)            # θ → M·θ
AffineTransform(DM, M, v)         # θ → M·θ + v
LinearDecorrelation(DM)           # centers at MLE, applies Cholesky of inv(Fisher)
```

### General embedding

```julia
ModelEmbedding(DM, F, dim_new; F_inv=nothing, F_deriv=nothing)
# F: differentiable map from R^dim_new → R^pdim(DM)
```

### Other transforms

```julia
FixParameters(DM, fixed_indices, fixed_values)  # fix some parameters
EmbedModelVia(DM, F)                             # embed model via function
```

## Predefined Models

```julia
LinearModel(x, θ)       # θ[1]*x + θ[2] + ... (slope-intercept)
ExponentialModel        # exp ∘ LinearModel
PolynomialModel(n)      # degree-n polynomial: θ[1]*x^n + ... + θ[n+1]
SumExponentialsModel(x, θ)  # θ[end] + Σ exp(θ[i]*x[i])
QuadraticModel          # PolynomialModel(2)
NonLinearModel(x, θ)   # (θ[1]+θ[2])*x + exp(θ[1]-θ[2])

GetLinearModel(DS)               # auto-construct linear ModelMap from DataSet
GetGeneralLinearModel(DS)        # multi-ydim linear model
ProportionalErrorModel(DS)       # error model: σ ∝ y_model
```

## ODE-based Models

```julia
using InformationGeometry, ModelingToolkitBase, OrdinaryDiffEq
using ModelingToolkitBase: t_nounits as t, D_nounits as D

@parameters β γ
@variables S(t) I(t) R(t)
SIReqs = [D(S) ~ -β*I*S, D(I) ~ +β*I*S - γ*I, D(R) ~ +γ*I]
SIRsys = System(SIReqs, t, [S,I,R], [β,γ]; name=:SIR)

SIRinitial = [762, 1, 0.]
SIRobservables = [2]   # observe infected only
SIRmodel = GetModel(SIRsys, SIRinitial, SIRobservables;
    startp=[0.002, 0.5], Domain=HyperCube(1e-4ones(2), 2ones(2)), meth=Tsit5(), tol=1e-9)

SIRDM = DataModel(SIRDS, SIRmodel, [0.002, 0.5])
```

**GetModel** options:
- `SIRobservables`: array of component indices OR observation function `f(u)`, `f(u,t)`, `f(u,t,θ)`
- `startp`: initial parameter vector
- `Domain`: `HyperCube` parameter bounds
- `meth`: ODE solver (default `Tsit5()`)
- `tol`: ODE solver tolerance
- Splitter function `θ -> (u0, p)` for estimating initial conditions as parameters

⚠️ **Do NOT use `@mtkcompile`** — it reorders states/equations. Use `@named` or supply `ODEFunction` directly.

Instead of supplying a `System`, a `ODEFunction` can also be supplied to `GetModel`.


## Plotting (Plots.jl recipes)

```julia
# Basic fit plot with confidence bands
plot(DM; Confnum=[1,2])           # 1σ and 2σ linearized bands
plot(DM; Confnum=0)              # no bands
plot(DM, MLE(DM))                # with specific params
plot(DM; Validation=true)        # prediction bands (data + param uncertainty)

# Multi-component: split into separate plots
plot(DM, MLE(DM), :Individual; Confnum=1)

# Residuals
ResidualPlot(DM)
ResidualVsFittedPlot(DM)

# Parameter visualization
ParameterPlot(R; st=:dotplot)     # multistart results
TracePlot(DM)                     # optimization trace

# Log-likelihood surface
PlotLogLikelihood(DM; kwargs...)
PlotScalar(DM; kwargs...)
PlotScalarSlices(DM; kwargs.)

# Geodesic visualization
PlotAlongGeodesic(DM, sol)
VisualizeGeos(DM, geos)
Plot2DVF(DM; kwargs...)
```

## Identifiability

```julia
IsLinear(DM)                        # linear parameter dependence?
IsLinearParameter(DM, i)            # is θ[i] linear?
StructurallyIdentifiable(DM)        # structural identifiability
PracticallyIdentifiable(P)          # from already computed profile `P`: max σ with bounded interval
PracticallyIdentifiable(DM)         # from DataModel (computes ParameterProfiles first)
IdentifiabilityReport(DM)           # detailed report
IdentifiabilityReportGraph(DM)      # visual report
FixNonIdentifiable(DM; kwargs...)   # fix non-identifiable params
```

## Exporting Results

```julia
SaveConfidence(DM, sols; adaptive=true, filename="confidence.csv")
SaveDataSet(DS; filename="dataset.csv")
SaveGeodesics(DM, geos; filename="geodesics.csv")
SaveAdaptive(sol; kwargs...)        # sample ODE solution proportional to curvature
```

## Symbolic Computations

```julia
SymbolicModel(DM)         # String of symbolic model expression (with named vars)
SymbolicModel(DM; sub=false)  # with generic θ[1],θ[2],... names
SymbolicdModel(DM)        # symbolic Jacobian expression
OptimizedDM(DM)           # symbolically optimized model + Jacobian (via Symbolics.jl)
InplaceDM(DM)             # in-place version for performance
SpeedifyTransform(F)      # optimize any function via Symbolics
```

## Parallelization

```julia
using Distributed; addprocs(4)
@everywhere using InformationGeometry, ParallelDataTransfer

# Send DataModel to workers
sendto(workers(); DM=DM)

# Parallel confidence regions / profiles / geodesics
ConfidenceRegion(DM, 1; parallel=true)
ConfidenceRegions(DM, 1:2; parallel=true)
ParameterProfiles(DM, 2; parallel=true)
RadialGeodesics(DM, 2; parallel=true)
PlotScalar(DM; parallel=true)
```

## Extensions

### Lux.jl (neural networks)

```julia
using Lux
NN = NeuralNet(input_dim, output_dim, hidden_dims...; activation=tanh)
NM = NormalizedNeuralModel(input_dim, output_dim, hidden_dims...;
    pre_transform=x -> x, post_transform=y -> y, activation=tanh)
# Returns a ModelMap wrapping the neural network
```

### PEtab.jl (systems biology standard)

```julia
using PEtab
Boehm = PEtabODEProblem(PEtabModel(BoehmYamlPath); gradient_method=:ForwardEquations, hessian_method=:ForwardDiff)
DM = DataModel(Boehm)                      # single condition → DataModel
CG = ConditionGrid(Boehm; FixedError=true)  # multiple conditions → ConditionGrid
DM = Refit(CG; meth=IPNewton())            # re-fit with InformationGeometry optimizer
```


## Key Parameters & Keywords Reference

### DataModel constructor

| Keyword | Default | Description |
|---------|---------|-------------|
| `SkipOptim` | `false` | Skip MLE search |
| `SkipTests` | `false` | Skip model consistency tests |
| `tol` | `1e-12` | General tolerance |
| `OptimTol` | `tol` | Optimization tolerance |
| `meth` | `LBFGS()` | Optimizer (Optim.jl or Optimization.jl) |
| `startp` | auto | Initial parameter guess |
| `ADmode` | `Val(:ForwardDiff)` | AD mode |
| `verbose` | `true` | Print info |
| `name` | `Symbol()` | Condition name (for ConditionGrid) |

### ConfidenceRegions

| Keyword | Description |
|---------|-------------|
| `tol` | ODE solver tolerance |
| `Dirs` | Tuple of parameter indices for 3+ dim |
| `N` | Number of planes for 3+ dim |
| `parallel` | Distribute across workers |

### ParameterProfiles

| Keyword | Description |
|---------|-------------|
| `N` | Number of profile points |
| `plot` | Show plot during computation |
| `IsCost` | `true`=raw cost, `false`=confidence level rescaling |
| `adaptive` | Adaptive step sizing |
| `SaveTrajectories` | Save nuisance parameter paths |
| `parallel` | Distribute across workers |
| `meth` | Optimizer for re-optimization |
| `Domain` | Parameter domain |
| `ProfileDomain` | Domain for profile specifically |

### ComputeGeodesic

| Keyword | Description |
|---------|-------------|
| `approx` | Use constant Christoffel symbols |
| `tol` | ODE solver tolerance |
| `meth` | ODE solver algorithm |
| `Boundaries` | Termination function `f(u,t,int) -> bool` |
| `Endtime` | Max integration time |

## Quick API Cheat Sheet (copy-paste ready)

All verified against IG.jl v1.31.0. These are the exact call patterns and return types — no guessing needed.

### Construction → MLE → log-likelihood → AIC/BIC

```julia
using InformationGeometry, LinearAlgebra

ds = DataSet(x, y, σ; xnames=["x"], ynames=["y"])   # σ optional (defaults to 1)
model(xi, θ) = θ[1]*xi + θ[2]                         # scalar return for ydim=1
dm = DataModel(ds, model, θ0)                         # auto-optimizes MLE at construction

mle = MLE(dm)                          # Vector{Float64} — the MLE
logl = LogLikeMLE(dm)                  # Float64 — log-likelihood at MLE
logl_at_θ = loglikelihood(dm, θ)       # Float64 — log-likelihood at arbitrary θ  (NOT internal LogLike(dm, θ)!)
aic = AIC(dm)                          # Float64 — uses MLE automatically
bic = BIC(dm)                          # Float64
aicc = AICc(dm)                        # Float64
uncert = MLEuncert(dm)                 # Vector{Measurement{Float64}} — MLE ± SE directly
```


### Fisher metric and asymptotic CIs

```julia
# FisherMetric(dm) returns a CLOSURE; call it with θ to get the matrix
fm = FisherMetric(dm)(mle)             # Matrix{Float64} — Fisher information at MLE
# equivalently: FisherMetric(dm, mle)

mleu = MLEuncert(dm, mle)              # Computes symmetric Wald intervals based on Fisher metric and returns Vector{Measurement} and returns (mle ± uncert)
```

⚠️ `FisherMetric(dm)` returns a **function**, not a matrix. Call `FisherMetric(dm)(θ)` or use the two-arg form `FisherMetric(dm, θ)`. Always `using LinearAlgebra` for `inv`, `diag`.

### Identifiability

```julia
svs, svecs = StructurallyIdentifiable(dm; threshold=1e-10)  # Tuple{Vector{Float64}, Matrix{Float64}}
# svs = singular values (all > threshold → locally identifiable)
# svecs = singular vectors (columns = directions in parameter space)

# IdentifiabilityReport returns (Union{Nothing,Matrix}, Union{Nothing,Vector{String}})
# When identifiable: returns (nothing, nothing) — just prints a summary to stdout
# When non-identifiable: returns (coupling_matrix, affected_param_names)
report, affected = IdentifiabilityReport(dm; threshold=1e-10)

pract = PracticallyIdentifiable(dm)    # Float64 — max σ level with bounded profile (higher = more identifiable)
```

### Profile likelihoods and confidence intervals

```julia
profiles = ParameterProfiles(dm, 2, collect(1:pdim(dm));
                             plot=false, SaveTrajectories=true)
# 2nd arg: confidence level in σ units (2 = up to 2σ)
# 3rd arg: parameter indices to profile (Vector{Int})

ci_95 = Tuple(ConfidenceIntervals(profiles, InvConfVol(0.95)))
# Returns Vector{Tuple{Float64,Float64}} — [(lo_θ1, hi_θ1), (lo_θ2, hi_θ2), ...]
# May contain (-Inf, +Inf) for unbounded intervals — preserve these!

pract_from_profiles = PracticallyIdentifiable(profiles)  # Float64 — max σ with bounded interval
```

### PolynomialModel (predefined)

```julia
pm = PolynomialModel(n)                 # returns a callable: pm(x, θ) where θ has n+1 elements
# θ[1] = highest-degree coefficient, θ[end] = constant term
# e.g. PolynomialModel(2)(x, [a, b, c]) = a*x^2 + b*x + c
```

### Complete minimal example (static model)

```julia
using InformationGeometry, LinearAlgebra, Statistics

# Load data
x = [1.0, 2.0, 3.0, 4.0, 5.0]
y = [2.1, 3.9, 6.2, 7.8, 10.1]
σ = 0.5 .* ones(5)

# Build model
ds = DataSet(x, y, σ; xnames=["x"], ynames=["y"])
model(xi, θ) = θ[1]*xi + θ[2]
dm = DataModel(ds, model, [1.0, 0.0])

# Extract results
mle = MLE(dm)
logl = LogLikeMLE(dm)
aic, bic, aicc = AIC(dm), BIC(dm), AICc(dm)
uncert = MLEuncert(dm)                 # [2.01 ± 0.11, -0.05 ± 0.35]

# Fisher CIs
fm = FisherMetric(dm)(mle)
se = sqrt.(diag(inv(fm)))
ci = (mle .- 1.96.*se, mle .+ 1.96.*se)

# Profile likelihood CIs
profiles = ParameterProfiles(dm, 2, [1, 2]; plot=false, IsCost=true)
ci95 = Tuple(ConfidenceIntervals(profiles, InvConfVol(0.95)))
```

## Pitfalls

1. **Model output for ydim=1**: Must return a scalar `Number`, NOT a 1-element vector. For `ydim>1`, return `SVector{ydim}`.
2. **θ is always a vector**: Even single-parameter models use `θ::AbstractVector`.
3. **Do not use the internal method `LogLike(dm, θ)` directly**: Use `loglikelihood(dm, θ)` or `LogLikeMLE(dm)`.
4. **`FisherMetric(dm)` returns a closure**: Call `FisherMetric(dm)(θ)` or use `FisherMetric(dm, θ)`. Import `LinearAlgebra` for `inv`/`diag`.
5. **Jacobian convention**: `dmodel(x,θ)` returns ∂model/∂θ (rows=ydim, cols=pdim), NOT ∂model/∂x.
6. **`global` scoping in Julia**: Variables assigned inside `try` blocks at top level need `global` prefix (e.g. `global x = ...` inside `try`). Wrap analysis in a `function` to avoid this.
7. **SkipOptim vs SkipOptimAndTests**: Positional bool argument sets both; keyword versions allow separate control.
8. **@mtkcompile reorders states**: Use `@named` or supply `ODEFunction` directly for ODE models.
9. **ProfileLikelihood ODE method (IntegrationParameterProfiles)**: Use `reltol < 1e-5` for reliable results. Keep `γ=nothing` when using AD Hessian.
10. **Confidence regions for dim>2**: Only 2D slices via `Dirs=(i,j,k)` + `N` planes.
11. **ModelMap InDomain**: Must return a scalar ≥ 0 on the valid domain, or a boolean.
12. **Parallel computation**: Must define DataModel on all workers (use `@everywhere` or `sendto`).
13. **DataSet ydim convention**: dims tuple is `(Npoints, xdim, ydim)`. Multi-component data can be passed as flat concatenated vectors with this tuple.
14. **ConfidenceIntervals method**: Use `Tuple(ConfidenceIntervals(...))` to get `[(lo, hi), ...]` pairs. May contain `-Inf`/`+Inf` — preserve these.
15. **Always `using LinearAlgebra`**: Needed for `inv`, `diag`, `eigvals`, etc. when computing CIs manually.
16. **Julia string interpolation indexing**: `"$uncert[i]"` parses as `"$(uncert)[i]"` — prints the whole vector then literal `[i]`. Use `"$(uncert[i])"` or comma syntax: `println("θ[$i] = ", uncert[i])`. This affects `MLEuncert(dm)` output (returns `Vector{Measurement{Float64}}`).
