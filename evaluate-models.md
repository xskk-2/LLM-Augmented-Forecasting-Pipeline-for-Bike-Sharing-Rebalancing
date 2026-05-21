---
name: evaluate-models
description: >
  Use when evaluating trained models on the test split and peak-hour subset.
  Triggers on: RMSE/MAE/R2 reporting, overestimation analysis, residual plotting.
mode: organisational
---

# Evaluate Models Skill

## Role
Organisational skill. Pure Python, no LLM call.

## When to Use
Invoke immediately after train-models has written all fitted models to state.
Triggers on: trained_models available in state, metric reporting needed,
evaluation visualisations required, model comparison about to begin.

## How to Execute
1. Read trained models from state["trained_models"].
2. Predict on X_test and compute full-set metrics per model:
   - rmse, mae, r2, overestimation_rate
3. Compute peak-hour subset metrics (is_peak == 1):
   - peak_rmse, peak_mae, peak_overestimation_rate
4. Generate diagnostic plots for each model:
   - Predicted vs Actual scatter plot
   - Residual plot (Pred - Actual)
5. Save figure path into state.

## Inputs from agent state
- trained_models (Dict[str, model])
- X_test (pd.DataFrame)
- y_test (pd.Series)

## Outputs to agent state
- evaluation_results (Dict[str, Dict[str, float]])
- eval_plot_path (str)

## Output format
metrics dictionary + local image path

## Notes
- If no peak rows exist, peak metrics should be NaN.
- Overestimation rate = mean(prediction > actual).
