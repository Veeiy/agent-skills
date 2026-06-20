---
description: "Run the Vantage Orchestrator engine on a mission (parallel wave-based multi-agent DAG)"
argument-hint: "<objective + definition of done; optional: autonomy, external tier, budget caps>"
---

Run the **Vantage Orchestrator** engine on the mission below.

Invoke the `orchestrator` skill from the **vantage-orchestrator** plugin (the engine in this repo), not the bundled `anthropic-skills` orchestrator. Then follow that skill's run loop end to end: load or build the mission blueprint, plan the wave DAG, resolve each slice through the tiered agent ladder (reuse a core agent, reuse an existing specialist, or synthesize one), dispatch each wave in parallel, gate every wave through the auditor, and stop cleanly when the definition of done is met or the first run-budget cap is hit.

Hold the run-budget caps as a hard ceiling (defaults: 30 total dispatches, 8 waves, 3 iterate loops, 3 synthesized agents) and treat the safety floor as non-negotiable. If the mission below is thin, sharpen the objective and definition of done with the operator before dispatching anything.

Mission and any operator dials:

$ARGUMENTS
