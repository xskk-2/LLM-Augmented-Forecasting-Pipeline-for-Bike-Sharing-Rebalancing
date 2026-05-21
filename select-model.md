---
name: select-model
description: >
  Use when selecting the final model under asymmetric business risk.
  Triggers on: baseline filtering, overestimation minimization, fallback decisions, business explanation generation.
mode: llm
---

# Select Model Skill

## Role
Hybrid skill: deterministic model selection logic + LLM business summary.

## When to Use
Invoke immediately after evaluate-models has written evaluation_results to state.
Triggers on: model metrics available, final model selection needed,
business justification for model choice required.

## How to Execute
1. Measure inference speed (ms per 1000 rows, averaged across 5 runs).
2. Build symmetric baseline from alpha=1.0 models:
   - baseline_rmse = minimum RMSE among models tagged with _a1.0
3. Apply RMSE filter:
   - keep models with RMSE <= baseline_rmse * 1.15
4. Primary selection rule:
   - if filtered set contains any model at the highest alpha in alpha_grid, choose
     the model with the lowest overestimation_rate among all filtered models
5. Fallback rule:
   - if no highest-alpha model survives filtering, emit warning and choose the
     model with lowest peak_rmse
6. Secondary fallback:
   - if no alpha=1.0 model exists, emit warning and choose lowest peak_rmse
7. Use OpenAI to generate 2-4 sentence non-technical business explanation.

## Inputs from agent state
- trained_models (Dict[str, model])
- evaluation_results (Dict[str, Dict[str, float]])
- X_test (pd.DataFrame)

## Outputs to agent state
- selected_model (str)
- selected_model_object (Any)
- selection_reason (str)

## Output format
final selected model id + short business rationale

## Notes
- Model name suffix format uses _a{alpha}, e.g. xgboost_a3.0.
- Fallback warning should clearly indicate which max alpha value had no survivors.
