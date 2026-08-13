# AgenticParameterInference

_Skills for automating simple parameter inference tasks with agentic LLMs_



This repository implements a family of skills for the automated development of mathematical models and their calibration to given data via natural language using the [**InformationGeometry.jl**](https://github.com/RafaelArutjunjan/InformationGeometry.jl) package in [julia](https://julialang.org/downloads/).
The goal of these skills is to offload simple repetitive tasks which come up during any parameter inference process to an LLM in order to reduce the time required to set up a mathematical model and reading in data.

While an LLM might not (yet) be able to develop an ideal mechanistic model on its own, entirely end-to-end simply from a file containing data, it can nevertheless follow the simple deterministic workflow of testing different model candidates, reducing away any structural non-identifiabilities and using information criteria such as AIC or BIC to strike a balance between model complexity and goodness-of-fit.


**It should be stressed that this set of skills is still a work-in-progress and will likely change over time.**


## Usage

| Skill | What it does |
|-------|-------------|
| `mechanistic-model-development` | Coordinator that orchestrates the full workflow: project setup → calibration → identifiability/UQ. |
| `ig-julia-project` | Sets up initial Julia project with required packages and interprets input data. |
| `ig-calibration-diagnosis` | Fits candidate models, compares them via AIC/BIC/AICc, and evaluates residual diagnostics. |
| `ig-identifiability-uq` | Checks parameter identifiability and computes profile-likelihood confidence intervals. Guides model reduction if parameters are non-identifiable. |


In its current form, the skills are formatted to be executed by [Hermes Agent](https://github.com/NousResearch/hermes-agent).
However, this "format" mainly pertains to the header of each MD skill file.
Therefore it should be straightforward to adapt the headers of the given skill files to any other harness or environment, such as [opencode](https://github.com/anomalyco/opencode).



