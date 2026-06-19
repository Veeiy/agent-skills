# Handoff Schema

The contract every agent uses to return work to the orchestrator. A consistent envelope is what lets the orchestrator integrate parallel results and lets the auditor cross-check across agents. Every agent's body content differs (see each agent file), but the envelope is the same.

## The envelope

Every agent returns:

```
=== VANTAGE HANDOFF ===
run_id:        <YYYY-MM-DD-mission-slug>
wave:          <integer>
agent:         <researcher | ideation | implementation | validator | auditor | optimizer | website-build | frontend-tester | a synthesized kebab-case specialist name>
dispatch_id:   <wave>.<agent>.<instance>   e.g. 2.website-build.1
status:        <complete | partial | blocked>
artifact_path: <relative path under the run's artifacts/ where this output is written>
=======================

<agent-specific body, per the agent's own output format>

--- ESCALATIONS ---
<none | a list of items that hit the safety floor, exceeded the tier, or need the operator. Empty section is allowed but the header must be present.>
```

## Field rules

- **run_id** ties the artifact to its run directory. Always echo the one the orchestrator gave you.
- **dispatch_id** is `wave.agent.instance`. Instance distinguishes parallel siblings of the same type (e.g. `1.ideation.1`, `1.ideation.2`, `1.ideation.3`). The orchestrator assigns these in the brief.
- **status:**
  - `complete` - the dispatch's definition of done is met.
  - `partial` - some done, some blocked on a missing input; the body says which.
  - `blocked` - could not meaningfully proceed; the ESCALATIONS section says why.
- **artifact_path** - where you wrote the actual deliverable files (code, site, docs). The handoff text summarizes; the artifact_path holds the real thing.
- **ESCALATIONS** - the channel for anything the agent cannot do under the safety floor or current tier. The orchestrator reads this to decide what to surface to the operator. Never silently drop a blocked item; escalate it here.

## Why a fixed envelope

In a parallel engine the orchestrator receives several handoffs at once. The fixed envelope lets it:
1. Slot each result into the right wave and dispatch slot in `run-state.json` without guessing.
2. Hand the auditor a clean set of artifacts to cross-check (the auditor reads `agent` and `artifact_path` to know what it is comparing).
3. Detect a `blocked` or `partial` status immediately and trigger a re-dispatch or escalation without parsing prose.

## Orchestrator-side handling

On receiving a handoff, the orchestrator:
1. Writes the body + artifact to `artifacts/<artifact_path>`.
2. Updates `run-state.json`: sets the dispatch's `status`, links `artifact_path`, copies any `escalations`.
3. If `status` is `blocked` or `partial`, decides: re-dispatch with a fixed brief, or escalate to the operator if it is a safety-floor item.
4. Once all dispatches in the wave are in, dispatches the auditor with every artifact_path from the wave.
