---
description: "Run the Vantage Orchestrator engine (parallel wave-based multi-agent DAG). Reads the conversation for context; a typed mission is optional."
argument-hint: "[optional: objective + definition of done; dials like autonomy / external tier / budget caps]"
---

Run the **Vantage Orchestrator** engine.

Invoke the `orchestrator` skill from the **vantage-orchestrator** plugin (the engine in this repo), not the bundled `anthropic-skills` orchestrator. Then follow that skill's run loop end to end: build or load the mission blueprint, plan the wave DAG, resolve each slice through the tiered agent ladder (reuse a core agent, reuse an existing specialist, or synthesize one), dispatch each wave in parallel, gate every wave through the auditor, and stop cleanly when the definition of done is met or the first run-budget cap is hit.

**Determine the mission from context, not from required input.** A typed mission is optional. If one is given below, use it. If nothing is given, read the current conversation to understand what the operator needs and why this run is being kicked off, and draft the mission from that. Either way, sharpen the objective and definition of done with the operator before dispatching anything; do not run on a vague mission.

Hold the run-budget caps as a hard ceiling (defaults: 30 total dispatches, 8 waves, 3 iterate loops, 3 synthesized agents) and treat the safety floor as non-negotiable.

Optional mission and dials (may be empty, in which case use the conversation above):

$ARGUMENTS
