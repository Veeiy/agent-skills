# Vantage Orchestrator

A venture-agnostic autonomous build engine for [Claude Code](https://docs.claude.com/en/docs/claude-code). You hand it a mission, it plans that mission into a wave-based DAG, deploys a team of specialist sub-agents in parallel, gates every wave through an auditor, and drives the work from research all the way to a defined finish line. It runs autonomously under dials you control, with a hard safety floor and a run budget that stops it cleanly before it can spin out. Nothing in it is tied to one business: the entire run is driven by a mission blueprint, so the same engine ships a hair-care storefront, a research report, a landing page, or an internal tool.

## What is in the box

- **Orchestrator skill** (`skills/orchestrator/SKILL.md`): the planner and conductor. Turns a mission into waves, dispatches agents, runs the auditor gates, integrates results, and loops to the definition of done. Learns across runs through a lessons ledger.
- **Optimizer skill** (`skills/optimizer/SKILL.md`): a standalone assess-and-optimize engine callable in any chat. Finds the single highest-leverage move and returns a ranked plan. Advisory by default.
- **A tiered agent roster** (`agents/`). The orchestrator does not have a fixed cast; it walks a ladder to get the right specialist for each slice (see Extending the engine below).
  - **Tier 0, core (always available, mission-agnostic):**
    - **researcher**: gathers facts, market, pricing, regulatory, and source material the rest of the run depends on.
    - **ideation**: turns research into concrete framings, concepts, and options to build against.
    - **implementation**: builds the deliverables: code, specs, workflows, ops docs, data files, checklists.
    - **validator**: verifies any built artifact against its definition of done from one angle (correctness, edge cases, spec conformance, a persona, data quality, accessibility). Mission-agnostic, not just for sites.
    - **auditor**: gates each wave for correctness and completeness and returns PASS, WARN, or BLOCK.
    - **optimizer**: the dispatchable twin of the optimizer skill, for in-run strategic optimization passes.
  - **Tier 1, bundled specialists (loaded only when the mission is in their domain).** The commerce/web pack:
    - **website-build**: builds and fixes the storefront, landing page, or site, including post-audit fixes.
    - **frontend-tester**: runs persona-based user testing against a built storefront UI to surface real-user friction.
  - **Tier 2, synthesized.** When a mission needs a capability no core or bundled agent covers, the engine mints a new specialist hyper-focused on the ask and persists it for reuse.

## How it works

1. **Mission blueprint in.** You fill in a blueprint (`skills/orchestrator/templates/mission-blueprint.template.md`) or just describe the mission. The blueprint carries the goal, the definition of done, the hard rules, and the dials.
2. **Wave-based DAG.** The orchestrator decomposes the mission into ordered waves. Within a wave, independent work runs in parallel; across waves, each wave depends on the gated output of the one before it.
3. **Single-message parallel dispatch.** All agents in a wave are dispatched in one message so they run concurrently, not one after another.
4. **Auditor gates each wave.** When a wave's agents return, the auditor reviews the combined output and returns PASS, WARN, or BLOCK. A BLOCK or WARN sends the branch back for rework, with up to 2 retries per branch.
5. **Loop to done.** The orchestrator integrates passing output, opens the next wave, and repeats until the mission hits its definition of done or the run budget stops it.

## Autonomy model

Two orthogonal dials govern how the run behaves. They are independent: one controls how much the operator is in the loop, the other controls how far outside the workspace the run is allowed to reach.

- **Dial 1, execution autonomy:** how much the run pauses for the operator.
  - **FULL**: runs end to end without stopping.
  - **CHECKPOINT**: pauses at wave boundaries for a go or no-go.
  - **MANUAL**: confirms each dispatch before it happens.
- **Dial 2, external authority:** how far the run can reach outside the workspace.
  - **Tier 1 (default)**: local and in-workspace work only.
  - **Tier 2**: limited external reads and reversible external actions.
  - **Tier 3**: broader external action, still under the safety floor.
- **Non-negotiable safety floor:** regardless of the dials, the run never moves money, never touches credentials or secrets, never performs permanent deletion, never changes access controls, and never takes an irreversible external action that was not explicitly requested.

## Run budget and stop conditions

Every run carries a budget so it cannot loop forever:

- **Max waves**: default 8.
- **Max total dispatches**: default 30.
- **Max iterate loops**: default 3.

When any limit is exhausted, the run stops cleanly, sets its status to `budget-exhausted`, and writes a report covering what was completed, what was gated, and what remains. This is the guardrail that prevents a runaway autonomous loop: the engine always halts on a known boundary and leaves a paper trail instead of burning on.

## The self-improvement loop

The auditor gates keep a single run honest. The self-improvement loop makes the engine better at running the next one. After each run the engine retrospects: it mines its own run state for every gate block, every re-dispatch, every escalation, and every wasted dispatch, dispatches the optimizer for a strategic read, and distills what it learns into a durable lessons ledger. At the start of every future run it reapplies the lessons that match the mission, carrying them into the wave plan and the agent briefs, so a mistake caught last week is not made again this week.

Lessons earn trust over time. A lesson is born low-confidence, rises as later runs confirm it, and falls when a run contradicts it, so the ledger strengthens what pays off and catches what does not. The ledger lives in an operator memory directory outside any single run and outside the engine's own files, so the engine stays venture-agnostic and every operator's instance learns its own lessons.

The engine never edits its own core logic to improve itself: the run loop, the safety floor, the dials, the budget, the schemas, and existing agents' contracts change only through a reviewed improvement proposal. The one bounded exception is additive: when the resolution ladder hits a real gap, the engine may write a new agent file into `agents/`, because that adds a hand without rewriting the machine, is reversible (delete the file), and is gated (the definition is checked, its output is audited). When a lesson is about the engine's own playbook rather than one mission, the retrospective stages an improvement proposal for the operator to review and apply. That keeps self-improvement on the right side of the safety floor: the ledger sharpens every run automatically, the roster can grow on its own, and the engine's core source changes only through a reviewed proposal.

## Model assignment

- **frontend-tester runs on a fast model (haiku).** It is dispatched as a wide, mechanical persona fan-out (many testers, simple personas), so speed and cost matter more than deep reasoning.
- **The reasoning agents inherit the session model.** Researcher, ideation, implementation, validator, website-build, auditor, and optimizer use whatever model the session is running, because their work is judgment-heavy. Drop validator to haiku when you fan it out wide and shallow.
- Operators can change any of this in each agent's frontmatter. A synthesized agent picks its own model at mint time by the same rule.

## Self-bootstrapping

The engine runs out of the box. With no config present, it defaults to **FULL** execution autonomy at **Tier 1** external authority and reads its hard rules straight from the mission blueprint. No setup file is required to start a run.

## Install and usage

1. Install the plugin into Claude Code.
2. Invoke the orchestrator skill and describe the mission, for example: `orchestrate <mission>`.
3. For a structured run, fill in `skills/orchestrator/templates/mission-blueprint.template.md` and point the orchestrator at it.
4. Each run writes its state and artifacts to `orchestrator-runs/<run-id>/`. Runs are resumable: point the orchestrator back at the run directory to pick up where it left off.

The optimizer skill is independent. Summon it in any chat with "optimize this" or "what's the highest-leverage move" to get a ranked, advisory optimization plan.

## Extending the engine

The roster grows two ways, both through the same tiered resolution ladder. For each capability a mission needs, the orchestrator walks: **Tier 0** (a core agent), then **Tier 1** (scan `agents/` for an existing or previously-synthesized specialist), then **Tier 2** (mint a new one). Reuse beats minting, minting beats stalling.

- **You add one (deliberate).** When you know a specialist is worth keeping, add it by hand. An agent is a single markdown file in `agents/` with a sharp when-to-use description (the dispatch trigger) and the shared handoff envelope (what lets the auditor gate it). Copy `skills/orchestrator/references/agent.template.md` and follow `skills/orchestrator/references/add-an-agent.md`.
- **The engine mints one (autonomous).** When the ladder hits a real gap mid-run, the orchestrator synthesizes a specialist hyper-focused on the ask: it dispatches a core agent with the synthesized role injected via its brief so it can use the capability immediately, and writes the full definition into `agents/` so it becomes a first-class specialist next run. The full method, the test for when a gap is real, and the exact line between adding a hand (allowed) and rewriting core logic (operator-gated) live in `skills/orchestrator/references/agent-synthesis.md`.

Discovery is by scan, not a hardcoded list, so a well-formed file in `agents/` is found and reused automatically. Workers stay flat: only the orchestrator dispatches, which keeps every wave inside the audit loop.

## Repo layout

```
.claude-plugin/
  plugin.json
  marketplace.json
agents/                # Tier 0 core (always available) + Tier 1 bundled pack
  researcher.md        # core
  ideation.md          # core
  implementation.md    # core
  validator.md         # core
  auditor.md           # core
  optimizer.md         # core
  website-build.md     # commerce/web pack
  frontend-tester.md   # commerce/web pack
skills/
  orchestrator/
    SKILL.md
    references/        # wave planning, parallel dispatch, self-improvement, agent-synthesis, add-an-agent + template
    templates/         # mission blueprint, run state, lessons ledger, retrospective, improvement proposal
    protocols/         # handoff schema, run-state schema, lessons-ledger schema
  optimizer/
    SKILL.md
    references/        # assessment rubric
```

The lessons ledger and improvement proposals are written at runtime to an operator memory directory outside the repo (default `~/Vault/Vantage-OS/orchestrator-memory/`, with a local fallback), so the engine's own files stay clean.

## License

MIT, see [`LICENSE`](./LICENSE).
