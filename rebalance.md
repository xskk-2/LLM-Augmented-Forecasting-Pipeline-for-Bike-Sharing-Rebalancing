---
name: rebalance
description: >
  Use when dispatch-advisor has determined that a station faces an imminent
  demand surge and bikes must be transferred from a nearby surplus station.
  Triggers on: dispatch_decision == "REBALANCE", forecast demand exceeds
  current inventory by more than 50%.
mode: llm-agent
---

# Rebalance Skill

## Role
LLM agent skill. Terminal node. Generates a concrete rebalancing work order
in plain business language for a field dispatcher or operations team member
to act on immediately. Ends the pipeline by writing business_output to state.

## When to Use
Invoked automatically by dispatch_router when dispatch_decision equals
"REBALANCE". Do not invoke directly — this node is wired as a terminal
conditional branch in the LangGraph pipeline.

## How to Execute
1. Read dispatch_reasoning, prediction, inference_station_id,
   prediction_confidence, selected_model, and selection_reason from state.
2. Construct a dispatcher-facing work order that includes:
   - The station needing bikes.
   - The nearest surplus source station (from dispatch_reasoning context).
   - The number of bikes to transfer (derived from prediction − current_inventory).
   - Time urgency based on prediction_confidence:
     High → act within 10 min / Medium → act within 20 min / Low → verify first.
3. Format the output as a structured plain-English field instruction.
4. Write the final result to state["business_output"].

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
| business_output | str  | Complete plain-language rebalancing work order for Gradio|

## Output Format
🚲 REBALANCING WORK ORDER
Station: {station_id}
Action Required: Transfer {n} bikes from surplus station {source_id}
Urgency: {High / Medium / Low} — act within {10 / 20 / review} minutes
Reason: {dispatch_reasoning}
Prediction Confidence: {confidence} — {confidence_note}
Powered by: {selected_model}

## Notes
- A business user reading this output should need no knowledge of the ML model,
  probability scores, or LangGraph internals.
- The number of bikes to transfer should be clamped to physical vehicle capacity
  and the surplus available at the source station.
- This node writes to business_output and terminates at END.
- The Gradio interface displays business_output as the primary result panel.
