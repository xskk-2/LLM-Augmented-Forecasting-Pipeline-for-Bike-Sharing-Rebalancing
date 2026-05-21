---
name: dispatch-advisor
description: >
  Use when a predicted bike availability figure has been produced by run-inference
  and an operations team needs to know what action to take at the station.
  Triggers on: demand forecast received, station health evaluation, routing decision,
  inventory triage, bike rebalancing, surge event handling.
mode: llm-agent
---

# Dispatch Advisor Skill

## Role
LLM agent skill. Uses GPT-4-turbo with two pre-fetched tool results —
adj_stations_tool and event_calendar_tool — to evaluate station health
and produce a routing decision: REBALANCE, ESCALATE, or MONITOR.

## When to Use
Invoke immediately after run-inference has written a numeric demand
prediction to agent state. This skill bridges the gap between a raw model
forecast and an actionable field instruction for a non-technical dispatcher
or operations manager.

## How to Execute
1. Read prediction, inference_station_id, capacity_map, and
   use_asymmetric_loss from state.
2. Derive current_inventory.
3. Compute availability_ratio = current_inventory / capacity.
4. Call adj_stations_tool(station_id) to discover nearby surplus stations.
5. Call event_calendar_tool(location, date) to check for surge-risk events.
6. Invoke the LLM with station context, pre-fetched tool results, and decision rules:
   - availability_ratio < 0.20 → ESCALATE (critically low, manual override needed)
   - prediction > current_inventory × 1.5 → REBALANCE (demand surge forecast)
   - otherwise → MONITOR (steady state, no immediate action)
7. Parse the agent's structured response to extract dispatch_decision and
   dispatch_reasoning.
8. Write both fields back to agent state for downstream terminal nodes.

## Decision Rules
| Condition                                              | Decision  |
|--------------------------------------------------------|-----------|
| availability_ratio < 0.20 (critically low stock)      | ESCALATE  |
| prediction > current_inventory × 1.5 (forecast surge) | REBALANCE |
| Neither condition met                                  | MONITOR   |

## Inputs from Agent State
| Field                 | Type           | Source              |
|-----------------------|----------------|---------------------|
| prediction            | float          | run-inference       |
| prediction_confidence | str            | run-inference       |
| confidence_note       | str            | run-inference       |
| inference_station_id  | str            | Gradio / user input |
| capacity_map          | Dict[str, int] | preprocess-data     |
| use_asymmetric_loss   | bool           | Gradio / user input |

## Outputs to Agent State
| Field              | Type | Description                                              |
|--------------------|------|----------------------------------------------------------|
| dispatch_decision  | str  | One of: "REBALANCE", "ESCALATE", "MONITOR"               |
| dispatch_reasoning | str  | Plain-language explanation (2-4 sentences) for dispatcher|

## Output Format
{
  "dispatch_decision": "REBALANCE",
  "dispatch_reasoning": "The model predicts 23 bikes needed in the next 30 minutes
    but only 15 are currently available. Station JC105 has a surplus and is 300m away.
    A bike transfer is recommended before the morning rush peak."
}

## Routing Logic (dispatch_router)
The conditional edge dispatch_router reads state["dispatch_decision"] and
routes to one of three terminal nodes:
- "REBALANCE" → rebalance node
- "ESCALATE"  → escalate node
- default     → monitor node

## Notes
- The event_calendar_tool uses deterministic seed generation (f"{date}_{location}")
  so the same query always returns the same event status, ensuring reproducibility.
- If capacity_map does not contain inference_station_id, capacity defaults to 30.
- If the LLM fails to produce a parseable decision, default to "MONITOR"
  to avoid false escalations.
- prediction_confidence informs the reasoning text but does not override the
  rule-based routing logic.
