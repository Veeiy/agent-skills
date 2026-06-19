# Wave Planning

How the orchestrator turns a mission into a dependency graph and slices that graph into parallel waves. This is the planning step that happens before any agent is dispatched.

## Step 1: Decompose into units of work

Read the blueprint's objective and definition of done. List every discrete unit of work needed to satisfy the DoD. Be concrete. "Build the website" is not a unit; "build PDP", "wire test-mode checkout", "write launch-QA checklist" are units.

## Step 2: Draw dependencies

For each unit, ask: what must exist before this can start? Draw the edges.

The most common real dependencies (the commerce-flavored ones are examples; the pattern generalizes):
- Research grounds ideation. Run `researcher` in Wave 0 so ideation reasons from real facts and constraints. Different research questions (market vs suppliers vs regulatory, or prior-art vs API-docs vs constraints) are independent and run in parallel.
- Concept or approach must be chosen before anything is built. (Ideation gate -> everything.)
- Shared inputs that the build depends on must exist first: brand tokens (name, colors, voice) before website copy or packaging; economics (price, margin, MOQ) before a PDP price or checkout config; a chosen schema or interface before the code that targets it.
- The build must exist before it can be verified: the site before frontend testing, the code before its tests run, the report before claim-checking.

## Step 3: Detect FALSE dependencies

This is where parallelism is won or lost. A false dependency is two units that feel sequential but are not actually coupled.

- Implementation (fulfillment process, ops, back-of-house) and Website Build share the concept and the economics, but neither needs the other's output. They are a parallel wave, not two sequential ones.
- Different ideation frames ("solve frizz" vs "solve scalp health") are independent explorations. Run them at once.
- Different test personas are independent. Run them at once.
- Accessibility audit and usability test can run against the same built site at the same time.

If two units only share an INPUT (not an output of one being the input of the other), they are parallel. Shared input, not shared output, is the test.

## Step 4: Resolve the agent for each unit

Before you group waves, decide who runs each unit. Walk the tiered ladder per capability: Tier 0 core, then Tier 1 existing specialist (scan `agents/`), then Tier 2 synthesize. This is what keeps the plan honest: every unit names a real, reachable agent, and any gap is found here, in planning, not three waves deep. Full method: `references/agent-synthesis.md`.

## Step 5: Group into waves

A wave is the set of all units whose dependencies are already satisfied. Assign each unit to the earliest wave it can legally enter.

The shape of the waves comes from the mission archetype, not from a single fixed template. Below is the product-launch archetype as a worked example, then the catalog of other archetypes, then how to adapt.

### The product-launch archetype

A consumer product or storefront launch collapses to this.

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

### Other archetypes

The machinery (parallel build, gate, iterate) is the same; what changes is which agents fill the build and test waves. The commerce pack (`website-build`, `frontend-tester`) only appears when the mission is a storefront. Everywhere else, `implementation` builds and `validator` verifies.

**Ship-code (feature, fix, migration, tool):**
```
Wave 0  researcher[prior art / API docs / constraints]   (optional)
  [GATE]
Wave 1  ideation[design options]   (skip if the approach is already decided)
  [GATE: pick the approach]
Wave 2  implementation[code]   implementation[tests/fixtures]   (parallel if independent)
  [GATE: auditor checks correctness + consistency]
Wave 3  validator[functional]  validator[edge cases]  validator[spec conformance]
  [GATE: ship / iterate / block]
Wave 4  implementation[fix the failing angles]   ->   re-validate the deltas
```

**Research / analysis (report, brief, recommendation):**
```
Wave 0  researcher[question A]  researcher[question B]  researcher[question C]   (parallel)
  [GATE: auditor checks sourcing + grounding]
Wave 1  implementation[synthesize the findings into the deliverable]
  [GATE]
Wave 2  validator[claim-grounding: every number sourced, arithmetic holds]  validator[completeness vs DoD]
  [GATE: ship / iterate / block]
Wave 3  implementation[fix flagged claims]   ->   re-validate
```

**Ops / process (workflow, SOP, runbook, automation):**
```
Wave 0  researcher[requirements / compliance lines]   (optional)
Wave 1  ideation[process options]   (skip if the process is mandated)
  [GATE: pick the process]
Wave 2  implementation[workflow doc + configs + automation]   (parallel units)
  [GATE: auditor checks the process is complete and internally consistent]
Wave 3  validator[dry-run the process]  validator[failure modes / what breaks under load]
  [GATE: ship / iterate / block]
Wave 4  implementation[fix gaps]   ->   re-validate
```

If a mission fits none of these, build its shape from first principles using Steps 1 to 5; the archetypes are starting points, not a closed set. A mission whose recurring need matches no existing agent is exactly where the resolution ladder synthesizes a new specialist (Step 4).

## Step 6: Right-size the waves

- Too wide (8 agents at once): briefs degrade, integration gets sloppy. Split the fan-out across two waves.
- Too narrow (everything sequential): you lost the point of the engine. Re-check Step 3 for false dependencies.
- Sweet spot: 2 to 4 agents per build/test wave.

## Step 7: Record the plan

Before dispatching, write the planned waves into `run-state.json` under `waves`, each with its agents and their dependency reasons. This is what makes the run resumable and what the final report explains back to the operator. A plan that only lives in the orchestrator's head dies when the session ends.

## Gating cadence

- Gate after every wave that produces build or concept artifacts.
- You may skip a gate only after a pure-iterate wave whose deltas were already scoped by a prior auditor BLOCK or ITERATE verdict, and even then prefer a quick delta re-check.
- The gate is not optional overhead. It is the mechanism that lets you run fast and parallel without shipping contradictions.
