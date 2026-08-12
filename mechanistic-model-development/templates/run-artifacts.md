# Run-Repository Artifact Templates

Copy and complete these files in each model-development repository. Keep prose terse; keep numerical outputs in `results/`.

## `STATUS.md`

```markdown
# Run Status

- **State:** `IN_PROGRESS` | `COMPLETE` | `PARTIAL` | `BLOCKED_INPUT`
- **Current model commit:** `<git commit>`
- **Last completed phase:** `A | B | C`
- **Budget:** `<limit>`; consumed `<amount>`
- **Reproduce:** `julia --project=. --startup-file=no run.jl`
- **Last successful command:** `<command>`

## Conclusion
<One or two sentences. State plainly if fit, local/numerical identifiability, formal structural identifiability, or practical UQ remains incomplete.>

## Next action
`<exact command or required user clarification>`
```

## `INPUT_INTERPRETATION.md`

```markdown
# Input Interpretation

## Data source
- **Source / immutable copy:** `data/input.csv`
- **Rows used / excluded:** `<details and reason>`

## Column roles
| CSV column | Role | Units | Model mapping / transformation | Notes |
|---|---|---|---|---|
| `<column>` | `time/input/output/condition/replicate` | `<unit>` | `<mapping>` | `<note>` |

## Model and observation assumptions
- **Model type:** `ODE/state-space` | `static nonlinear/algebraic`
- **Observed quantity / state mapping:** `<details>`
- **Initial-condition convention:** `<details or N/A>`
- **Parameter conventions:** `<names, units, transforms, bounds>`

## Error model
- **Model:** `<user specified | independent additive Gaussian / OLS default>`
- **Absolute standard deviation / scaling:** `<source or estimated plug-in value>`
- **Limitation:** `<state whether measurement uncertainty was supplied>`

## Material ambiguities resolved
- `<ambiguity and basis>`
```

## `MODELING_LOG.md`

```markdown
# Modeling Log

| Commit | Phase | Change / hypothesis | Evidence or purpose | Outcome | Status |
|---|---|---|---|---|---|
| `<hash>` | A | Baseline model | `<scientific starting point>` | `<runnable baseline>` | retained |
```

## `INPUT_CLARIFICATIONS.md`

```markdown
# Input Clarifications Required

Calibration has not started because the following ambiguity materially changes the inference problem.

| Required clarification | Why it changes the model/inference | Needed answer |
|---|---|---|
| `<item>` | `<impact>` | `<specific response>` |

**Status:** `BLOCKED_INPUT`
```
