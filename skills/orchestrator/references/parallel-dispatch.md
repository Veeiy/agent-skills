# Parallel Dispatch Mechanics

This is the exact mechanic the orchestrator uses to deploy sub-agents simultaneously. Get this wrong and "the agents run in parallel" silently becomes "the agents run one at a time."

## The core rule

The runtime runs `Agent` tool calls concurrently when, and only when, they are emitted in the **same assistant message**.

- One message with N `Agent` calls -> N agents run at the same time. This is one **wave**.
- N messages with one `Agent` call each -> the agents run one after another.

So the orchestrator's job for each wave is: assemble every brief first, then fire them all in a single message.

## A wave, concretely

Say Wave 2 is "build the back-of-house and the storefront at the same time." The orchestrator emits one message containing two calls:

```
Agent(
  subagent_type: "implementation",
  description: "Build product fulfillment process",
  prompt: "<full self-contained brief: concept, economics, constraints, DoD, handoff schema, where to write artifacts>"
)
Agent(
  subagent_type: "website-build",
  description: "Build product storefront",
  prompt: "<full self-contained brief: concept, brand tokens, pages, checkout mode=test, DoD, handoff schema, where to write artifacts>"
)
```

Both run concurrently. Their results return together. The orchestrator then writes both artifacts to the run directory and proceeds to the gate.

## Fan-out: same agent, many briefs

To get N user-testing personas at once, dispatch N `frontend-tester` calls in one message, each with a different persona brief:

```
Agent(subagent_type: "frontend-tester", description: "Persona: busy returning client", prompt: "<persona A brief>")
Agent(subagent_type: "frontend-tester", description: "Persona: price-sensitive shopper", prompt: "<persona B brief>")
Agent(subagent_type: "frontend-tester", description: "Persona: skeptical first-timer", prompt: "<persona C brief>")
```

Same for parallel ideation tracks, or building three independent product concepts at once.

## Briefs must be self-contained

Sub-agents do **not** inherit the orchestrator's conversation. Each starts cold. Every brief must therefore carry, in full:
1. The mission objective and this agent's slice of it.
2. All inputs the agent needs (the winning concept, brand tokens, economics, prior artifacts by path).
3. Constraints and hard rules that apply (budget cap, no health claims, no em dashes, external tier).
4. The definition of done for this specific dispatch.
5. The handoff schema to return (`protocols/handoff-schema.md`).
6. Where to write its artifact in the run directory.

A vague brief is the number one cause of a wasted wave. Spend the tokens to brief well; it is cheaper than a re-dispatch.

## Sizing a wave

- Keep a wave to work that is genuinely independent. If B needs A's output, they are not in the same wave.
- There is no hard cap on agents per wave, but past ~5 concurrent the orchestrator's job of writing good briefs and integrating results gets sloppy. Prefer 2 to 4 per wave; split a big fan-out across two waves if briefs would suffer.
- Mixed-type waves are fine and normal (implementation plus website-build together).

## Gating is sequential by design

The auditor gate is deliberately NOT parallel with the wave it audits. You cannot audit work that does not exist yet. Sequence is: dispatch wave (parallel) -> collect (parallel results) -> dispatch auditor (single) -> act on verdict. The only thing that runs alone is the gate.

## After a wave returns

1. Write each agent's artifact to `artifacts/`.
2. Update `run-state.json`: mark each agent's dispatch complete, link its artifact, record any self-reported blockers.
3. Dispatch the auditor for this wave.
4. Branch on the verdict per the run loop in `SKILL.md`.

## Failure handling inside a wave

If one agent in a parallel wave fails or returns thin output while the others succeed:
- Keep the good artifacts.
- Re-brief and re-dispatch only the failed agent (it can go alone in its own follow-up message, or batched with the next wave's independent work).
- Do not discard a whole wave because one branch stumbled.
