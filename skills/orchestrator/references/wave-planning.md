# Wave Planning

How the orchestrator turns a mission into a dependency graph and slices that graph into parallel waves. This is the planning step that happens before any agent is dispatched.

## Step 1: Decompose into units of work

Read the blueprint's objective and definition of done. List every discrete unit of work needed to satisfy the DoD. Be concrete. "Build the website" is not a unit; "build PDP", "wire test-mode checkout", "write launch-QA checklist" are units.

## Step 2: Draw dependencies

For each unit, ask: what must exist before this can start? Draw the edges.

The most common real dependencies:
- Research grounds ideation. Run `researcher` in Wave 0 so ideation reasons from real prices and constraints. Different research questions (market vs suppliers vs regulatory) are independent and run in parallel.
- Concept must be chosen before anything is built. (Ideation gate -> everything.)
- Brand tokens (name, colors, voice) must exist before website copy and before packaging.
- Economics (price, margin, MOQ) must exist before the PDP can state a price and before checkout config.
- The built site must exist before frontend testing can run against it.

## Step 3: Detect FALSE dependencies

This is where parallelism is won or lost. A false dependency is two units that feel sequential but are not actually coupled.

- Implementation (fulfillment process, ops, back-of-house) and Website Build share the concept and the economics, but neither needs the other's output. They are a parallel wave, not two sequential ones.
- Different ideation frames ("solve frizz" vs "solve scalp health") are independent explorations. Run them at once.
- Different test personas are independent. Run them at once.
- Accessibility audit and usability test can run against the same built site at the same time.

If two units only share an INPUT (not an output of one being the input of the other), they are parallel. Shared input, not shared output, is the test.

## Step 4: Group into waves

A wave is the set of all units whose dependencies are already satisfied. Assign each unit to the earliest wave it can legally enter.

### The canonical shape

Most build missions collapse to this. Start here and adapt.

```
Wave 0  RESEARCH / GROUNDING (parallel questions) - optional but recommended
        researcher[market]  researcher[suppliers]  researcher[regulatory]
   |
  [GATE: auditor sanity-checks sourcing -> facts feed Wave 1]
   |
Wave 1  IDEATION (parallel frames, each handed the research)
        ideation[frameA]  ideation[frameB]  ideation[frameC]
   |
  [GATE: auditor scores concepts -> orchestrator picks winner]
   |
Wave 2  BUILD (parallel, independent)
        implementation[process/ops]   website-build[storefront]
   |
  [GATE: auditor checks cross-consistency: site price == economics, voice == brand]
   |
Wave 3  TEST (parallel personas)
        frontend-tester[A]  frontend-tester[B]  frontend-tester[C]
   |
  [GATE: auditor synthesizes -> ship / iterate / block]
   |
Wave 4  ITERATE (only the branches that need it)
        website-build[fix nav]   implementation[fix returns flow]
   |
  [GATE: auditor re-checks the deltas]
   |
  FINAL AUDIT + RUN REPORT + DELIVER
```

### When the shape changes

- **Pure-build mission, concept already chosen:** skip Wave 1. Start at Wave 2. Record in run state that ideation was pre-supplied.
- **Ideation-only mission:** run Wave 1 plus its gate, deliver the ranked concept brief, stop.
- **No website needed:** drop `website-build` from Wave 2; implementation may itself fan out into parallel implementation units.
- **Brand work is a real dependency:** if the concept needs a name/voice/colors before the site can be built, insert a short brand wave between ideation and build, or hand brand work to an `implementation` instance and gate it before the website wave. Do not let the website agent invent brand tokens that the rest of the run then contradicts.

## Step 5: Right-size the waves

- Too wide (8 agents at once): briefs degrade, integration gets sloppy. Split the fan-out across two waves.
- Too narrow (everything sequential): you lost the point of the engine. Re-check Step 3 for false dependencies.
- Sweet spot: 2 to 4 agents per build/test wave.

## Step 6: Record the plan

Before dispatching, write the planned waves into `run-state.json` under `waves`, each with its agents and their dependency reasons. This is what makes the run resumable and what the final report explains back to the operator. A plan that only lives in the orchestrator's head dies when the session ends.

## Gating cadence

- Gate after every wave that produces build or concept artifacts.
- You may skip a gate only after a pure-iterate wave whose deltas were already scoped by a prior auditor BLOCK, and even then prefer a quick delta re-check.
- The gate is not optional overhead. It is the mechanism that lets you run fast and parallel without shipping contradictions.
