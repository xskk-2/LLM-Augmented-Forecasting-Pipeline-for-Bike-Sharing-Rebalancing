---
name: preprocess-data
description: >
  Use when preparing raw Citi Bike CSV data for model training.
  Triggers on: data cleaning, resampling, feature engineering,
  train/test split, missing value imputation.
mode: organisational
---

# Preprocess Data Skill

## Role
Organisational skill. Pure Python,no LLM call.

## When to Use
Invoke at the start of every pipeline run before any model training occurs.
Triggers on: raw Citi Bike CSV data available, data cleaning required,
feature engineering needed, train/test split needed.

## How to Execute
1. Download 5 CSV files from Kaggle (random seed=42)
2. Filter to 2021 only; keep active stations (is_installed=1, is_renting=1)
3. Resample to 30-min intervals per station (mean aggregation); carry lat/lon
4. Drop stations with >30% missing rate after resampling
5. Forward-fill + backward-fill remaining NaNs per station
6. Feature engineering: hour, day_of_week, month, cyclic encodings,
   is_peak, is_weekend, is_covid, lag_1/2/3/6/48/336,
   rolling_mean_6, rolling_std_6, availability_ratio, bikes_current, lat, lon
7. 80/20 chronological split -> write to state

## Inputs from agent state
This is the pipeline entry node. No upstream state inputs are required.
Reads use_asymmetric_loss and alpha_grid from the initial invocation state.

## Outputs to agent state
- X_train, X_test  (pd.DataFrame)
- y_train, y_test  (pd.Series) — target: num_bikes_available at t+1
- feature_names    (List[str])
- capacity_map     (Dict[str, int])
- lat_map, lon_map (Dict[str, float])

## Output format
pandas objects stored directly in state

## Notes
- lag_336 requires 7 days of burn-in per station; first week rows dropped via dropna
- is_covid = 1 for Jan–Jun 2021 (restricted ridership), 0 for Jul–Dec 2021
- bikes_current = lag_0 (current slot observation), renamed to distinguish from target
