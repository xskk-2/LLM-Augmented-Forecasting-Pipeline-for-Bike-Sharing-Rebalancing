---
name: monitor
description: >
  Use when dispatch-advisor has determined that a station is operating within
  normal parameters and no immediate intervention is required.
  Triggers on: dispatch_decision == "MONITOR", demand forecast within acceptable
  inventory range, steady-state station health.
mode: organisational
---

# Monitor Skill

## Role
Organisational skill. Terminal node. Produces a plain-language station health
summary confirming that no action is needed at this time, and provides the
dispatcher with a situational awareness snapshot for the next 30-minute window.
Ends the pipeline by writing business_output to state.

## When to Use
Invoked automatically by dispatch_router when dispatch_decision equals
"MONITOR" — the default outcome when neither a rebalancing threshold nor a
critical inventory threshold has been breached. This is the most common terminal
state under normal operating conditions.

## How to Execute
1. Read dispatch_reasoning, prediction, inference_station_id,
   prediction_confidence, confidence_note, selected_model from state.
2. Compute projected_remaining = current_inventory - prediction (clamped to 0).
3. Determine next-check urgency from prediction_confidence:
   - High   → Next check in 30 minutes
   - Medium → Next check in 15 minutes
   - Low    → Next check in 10 minutes (uncertainty is high)
4. Generate a concise station health digest in plain English.
5. Write the digest to state["business_output"].

## Inputs from Agent State
| Field                 | Type  | Source              |
|-----------------------|-------|---------------------|
| dispatch_reasoning    | str   | dispatch-advisor    |
| dispatch_decision     | str   | dispatch-advisor    |
| prediction            | float | run-inference       |
| prediction_confidence | str   | run-inference       |
| confidence_note       | str   | run-inference       |
| inference_station_id  | str   | Gradio / user input |
| selected_model        | str   | select-model        |
| selection_reason      | str   | select-model        |

## Outputs to Agent State
| Field           | Type | Description                                              |
|-----------------|------|----------------------------------------------------------|
| business_output | str  | Plain-language station health digest for Gradio display  |

## Output Format
✅ STATION STATUS: NO ACTION REQUIRED
Station: {station_id}
Current Stock: Normal
Forecast Demand: {prediction:.0f} bikes in next 30 minutes
Projected Remaining After Demand: {projected_remaining:.0f} bikes
Confidence: {confidence} — {confidence_note}
Next Recommended Check: {30 / 15 / 10} minutes
Context: {dispatch_reasoning}
Powered by: {selected_model}

## Notes
- MONITOR is the most frequent terminal outcome. The Gradio interface renders
  this with a neutral green indicator (vs amber for REBALANCE, red for ESCALATE).
- projected_remaining gives the dispatcher advance notice of whether the station
  may need attention in the next check cycle.
- When prediction_confidence is "Low", the next-check interval is shortened to
  10 minutes even in MONITOR state, because high uncertainty may mean the
  steady-state assessment is wrong.
- This node is pure Python formatting — no LLM or external tool call is made.
  The routing decision has already been made by dispatch-advisor.
- This node writes to business_output and terminates at END.
