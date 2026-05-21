---
name: run-inference
description: >
  Use when the selected model object needs to produce a numeric demand forecast
  for a specific station. Triggers on: model selected, inference input available,
  predict bikes available at t+1.
mode: organisational
---

# Run Inference Skill

## Role
Organisational skill. Pure Python — no LLM call.

## When to Use
Invoke immediately after select-model has written selected_model_object to state.
This node runs the trained model on a real input row and produces a numeric
prediction that downstream agent skills can act on.

## How to Execute
1. Read selected_model_object, X_test, feature_names, lat_map, lon_map from state.
2. Look up inference_station_id in lat_map/lon_map to get target (lat, lon).
3. Spatial search: find the closest X_test row to target coordinates via Euclidean distance.
4. Temporal alignment: filter matched rows by inference_hour, inference_dow, inference_month;
   fall back to latest spatial match if no exact temporal match exists.
5. Call model.predict(sample_features) on the matched row and clamp result to >= 0.
6. Read rolling_std_6 from the matched row to compute prediction confidence:
   - rolling_std_6 < 2  → High confidence
   - rolling_std_6 < 4  → Medium confidence
   - rolling_std_6 >= 4 → Low confidence
5. Write prediction, prediction_confidence, and confidence_note to state.

## Inputs from Agent State
| Field                  | Type            | Source              |
|------------------------|-----------------|---------------------|
| selected_model_object  | Any             | select-model        |
| X_test                 | DataFrame       | preprocess-data     |
| feature_names          | List[str]       | preprocess-data     |
| lat_map                | Dict[str,float] | preprocess-data     |
| lon_map                | Dict[str,float] | preprocess-data     |
| capacity_map           | Dict[str,int]   | preprocess-data     |
| inference_station_id   | str             | Gradio / user input |
| inference_hour         | int             | Gradio / user input |
| inference_dow          | int             | Gradio / user input |
| inference_month        | int             | Gradio / user input |

## Outputs to Agent State
| Field                 | Type  | Description                                      |
|-----------------------|-------|--------------------------------------------------|
| prediction            | float | Predicted bikes available at t+1 (clamped >= 0)  |
| prediction_confidence | str   | One of: "High", "Medium", "Low"                  |
| confidence_note       | str   | Plain-language explanation of confidence level   |

## Output Format
Numeric float written directly to state["prediction"].
Example: prediction=12.47, prediction_confidence="High",
confidence_note="Consistent demand pattern over the past 6 hours."

## Notes
- Uses spatial-temporal lookup in X_test as a proxy for real-time inference.
  In production this would be replaced by a live station telemetry feed.
- Fallback prediction of 5.0 bikes is used if model or test data is missing.
- rolling_std_6 measures demand variability over the last 6 time steps (90 min).
  High variability reduces confidence and triggers shorter recheck intervals downstream.
