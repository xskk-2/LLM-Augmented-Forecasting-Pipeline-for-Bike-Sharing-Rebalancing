---
name: train-models
description: >
  Use when training candidate models on preprocessed Citi Bike data.
  Triggers on: model training, fit regressor, quantile regressor, XGBoost, LightGBM, Ridge.
mode: organisational
---

# Train Models Skill

## Role
Organisational skill. Pure Python, no LLM call.

## When to Use
Invoke immediately after preprocess-data has written X_train and y_train to state.
Triggers on: training data available in state, candidate models need to be fitted,
asymmetric or symmetric loss configuration required.

## How to Execute
1. Read use_asymmetric_loss and alpha_grid from state.
2. If use_asymmetric_loss is True:
   a. For each alpha in alpha_grid, fit XGBRegressor and LGBMRegressor with a
      custom asymmetric gradient objective (overestimation penalised alpha x more).
   b. Fit one QuantileRegressor with q = 1 / (1 + alpha) for the primary alpha.
   c. Key name format: xgboost_a{alpha}, lightgbm_a{alpha}, ridge (quantile).
3. If use_asymmetric_loss is False:
   a. Fit one Ridge regressor, one XGBRegressor, one LGBMRegressor with default MSE.
   b. Key names: ridge, xgboost_a1.0, lightgbm_a1.0.
4. Attempt GPU training (cuda/gpu); fall back to CPU silently on failure.
5. Write all fitted model objects to state["trained_models"].

### Candidate Models
| Key               | Class               | Loss mode          |
|-------------------|---------------------|--------------------|
| ridge             | Ridge / QuantileReg | MSE or quantile    |
| xgboost_a{alpha}  | XGBRegressor        | Asymmetric or MSE  |
| lightgbm_a{alpha} | LGBMRegressor       | Asymmetric or MSE  |

### Hyperparameters (shared across XGB and LGBM)
- n_estimators=500, learning_rate=0.03, max_depth=6
- subsample=0.8, colsample_bytree=0.8, random_state=42

## Inputs from agent state
- X_train (DataFrame)   — from preprocess-data
- y_train (Series)      — from preprocess-data
- use_asymmetric_loss (bool)   — from initial invocation state
- alpha_grid (List[float])     — from initial invocation state

## Outputs to agent state
- trained_models (Dict[str, model]) — all fitted model objects keyed by name

## Output format
Python dict stored directly in state["trained_models"].
Example keys: {"ridge": Ridge, "xgboost_a1.0": XGB, "xgboost_a3.0": XGB,
               "lightgbm_a1.0": LGBM, "lightgbm_a3.0": LGBM}

## Notes
- The _a{alpha} suffix convention is required by select-model to identify the
  symmetric baseline (alpha=1.0) and the highest-penalty variant.
- GPU training is attempted silently and falls back to CPU; no manual flag needed.
- No evaluation happens here; all metric computation is deferred to evaluate-models.

