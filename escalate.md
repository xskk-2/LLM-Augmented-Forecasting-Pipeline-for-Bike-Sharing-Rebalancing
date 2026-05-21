---
name: escalate
description: >
  Use when dispatch-advisor has determined that a station has critically low
  bike availability (below 20% of capacity) and the situation exceeds automated
  rebalancing thresholds. Triggers on: dispatch_decision == "ESCALATE",
  availability_ratio < 0.20, potential service failure risk.
mode: llm-agent
---

# Escalate Skill

## Role
LLM agent skill. Terminal node. Generates a critical alert report for a senior
operations manager or on-call supervisor. Escalation indicates that automated
rebalancing is insufficient and human judgement or emergency response is needed.
Ends the pipeline by writing business_output to state.

## When to Use
Invoked automatically by dispatch_router when dispatch_decision equals
"ESCALATE". Occurs when availability_ratio < 0.20 — a service failure risk
that cannot be resolved by a simple bike transfer alone.

## How to Execute
1. Read dispatch_reasoning, prediction, inference_station_id,
   prediction_confidence, confidence_note, selected_model from state.
2. Determine severity from prediction_confidence:
   - High confidence + critically low inventory → Severity: CRITICAL
   - Medium confidence + critically low inventory → Severity: HIGH
   - Low confidence + critically low inventory  → Severity: ELEVATED (verify first)
3. Generate a structured escalation alert naming the station, current stock,
   forecast demand, shortfall, and recommended supervisor action.
4. Write the complete alert to state["business_output"].

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
| business_output | str  | Structured escalation alert for Gradio display           |

## Output Format
🚨 ESCALATION ALERT — CRITICAL STATION STATUS
Station: {station_id}
Severity: {CRITICAL / HIGH / ELEVATED}
Current Stock: CRITICALLY LOW (below 20% capacity)
Forecast Demand: {prediction:.0f} bikes needed in next 30 minutes
Shortfall Estimate: {shortfall} bikes
Recommended Action: Contact on-call supervisor immediately. Emergency restock required.
Context: {dispatch_reasoning}
Prediction Confidence: {confidence} — {confidence_note}
⚠️ This alert requires human review. Do not rely solely on automated routing.
Powered by: {selected_model}

## Notes
- Escalation is the highest-priority terminal state. The Gradio interface should
  render this output with a visible red/warning indicator.
- This node surfaces the problem for human action and does not attempt to solve it.
  Critically low availability may have causes (vandalism, blocked docks, system outage)
  that a rebalancing order cannot fix.
- confidence_note is displayed verbatim so the supervisor can judge whether to trust
  the alert or verify on the ground first.
- This node writes to business_output and terminates at END.
