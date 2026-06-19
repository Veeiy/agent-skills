# Agent resolution and synthesis

How the orchestrator gets the right specialist for a slice of work: reuse what exists, and mint a new agent only when nothing fits. This is the tiered resolution ladder. It runs during planning, once per capability the mission needs, before any worker is dispatched.

The principle is unchanged from the rest of the engine: the orchestrator owns control. Agents are hands, not the brain. This protocol just gives the orchestrator a way to grow new hands, on its own, scoped to the ask, instead of stalling when the built-ins do not cover a slice.

## The resolution ladder

For each capability the wave plan needs, walk down until you get a hit. Do not skip a tier; reuse beats minting, and minting beats stalling.

### Tier 0 - Core

The six mission-agnostic agents that ship with the engine and are always available:

| Agent | Owns |
|---|---|
| `researcher` | External grounding: facts, market, sources, feasibility |
| `ideation` | Problem framing, concept generation, ranked concept brief |
| `implementation` | Building the concept: code, configs, docs, processes, ops |
| `validator` | Verifying any built artifact against its DoD from one angle |
| `auditor` | The gate: cross-checks every wave for correctness and consistency |
| `optimizer` | Strategic assess-and-optimize pass over a run or artifact |

If a Tier 0 agent already owns the slice, use it. Most slices land here. Do not mint a specialist for work a core agent does well; a "code-builder" or "doc-writer" is just `implementation` with a focused brief.

### Tier 1 - Existing specialist

Specialists that already live in the repo's `agents/` directory beyond the core six. These are bundled domain packs (for example the commerce pack: `website-build`, `frontend-tester`) plus any agent a prior run synthesized and persisted.

Discover them dynamically: glob `agents/*.md`, read each `description`, and match it against the capability you need. Do not rely on a hardcoded list; the whole point of the ladder is that the roster grows, so discovery is by scan, not by memory. If the operator configured extra agent directories (other repos), scan those too; the default is this repo's `agents/`.

If an existing specialist matches the slice, reuse it. No rebuild. A close-but-not-exact match is still a reuse: hand it a sharper brief rather than minting a near-duplicate.

### Tier 2 - Synthesize a new specialist

Only when Tier 0 and Tier 1 both miss. The mission needs a capability no core agent does well and no existing specialist covers. Now you mint one, hyper-focused on the ask.

## When a slice is a real Tier 2 gap (and when it is not)

Mint a new agent only if all of these hold:
- A core agent (T0) would do the slice poorly, not just unbranded. Focus is not a gap; `implementation` with a tight brief is not a missing agent.
- No existing specialist (T1) matches, even with a sharper brief.
- The slice is a distinct, reusable capability, not a one-off task. If you will need it once and never again, do not persist an agent for it; just brief a core agent. Synthesis earns its cost only when the capability recurs within this run or is likely to recur across runs.
- It is genuinely one slice. "Researches and builds and tests" is three agents and a wave plan, which is your job, not a new agent.

If any of these fails, drop back up the ladder. The cheapest correct answer is almost always reuse.

## Two modes of synthesis

A newly written `agents/*.md` file is not registered as a dispatchable `subagent_type` until the session reloads. The runtime loads agent definitions at startup. So synthesis has two modes, and you use both:

### Mode A - In-run hyperfocus (use the capability now)

For immediate use inside the current run, you cannot dispatch a brand-new `subagent_type`. Instead, dispatch a **core agent with the synthesized role injected through its brief**. The brief carries the specialist's identity, scope, method, hard rules, and output contract, so a generic worker behaves as the specialist for that dispatch. This is instant and needs no reload.

- For a building slice, hyperfocus `implementation`.
- For a verifying slice, hyperfocus `validator`.
- For a research slice, hyperfocus `researcher`.

The injected role is the same content you would write into the agent file (see Mode B). You are just delivering it as a brief instead of as a registered agent. Record on the dispatch that it ran as a hyperfocused core agent, naming the synthesized role, so the run report and retrospective can see it.

### Mode B - Persisted specialist (make it first-class for next time)

When the capability is reusable, also write the full agent definition to the **source repo** `agents/<name>.md` so that from the next session it is a first-class, dispatchable specialist (a Tier 1 asset). This is what "save to the source repo to hyperfocus based on the ask" means: the engine accumulates sharper tools as it runs.

Persist to the source repo, not just the runtime cache. A cache-only file is wiped on the next plugin update. If the source repo is not reachable in this environment, write to `agents/` wherever the engine is running and flag in the run report that it must be copied into source and committed to survive.

For a recurring capability, do both: Mode A to use it in this run, Mode B to keep it. For a one-off, do neither; brief a core agent and move on.

## How to synthesize the definition

1. **Copy the template.** Start from `references/agent.template.md`. It already encodes the two hard requirements (sharp description, handoff envelope) and the flat-worker rules.
2. **Name the single slice.** One agent, one job. The `description` must name the slice and the concrete triggers that summon it, with 2 to 3 `<example>` blocks, because that description is the fan-out trigger the orchestrator reads later.
3. **Hyperfocus on the ask.** Scope the method, hard rules, and output body to the specific mission need that opened the gap. A synthesized agent should be narrower than a core agent, not broader. Breadth is what core agents are for.
4. **Carry the floor.** The agent inherits the universal safety floor and the mission's hard rules verbatim. Leave the Agent/Task tool out of any `tools:` allowlist so the worker stays flat and cannot spawn its own sub-agents.
5. **Pick the model.** `haiku` for wide mechanical fan-out, `inherit` for reasoning-heavy work.

## Gate the definition before you trust it

A synthesized agent is engine surface, so it gets gated like any artifact before its output is trusted:
- Before first real dispatch, self-check (or run a quick `auditor` pass on) the definition against the two hard requirements: is the description a sharp fan-out trigger, and does the output contract return the handoff envelope? A vague description never gets dispatched correctly later; a missing envelope cannot be gated.
- Dedup check: confirm one more time it does not duplicate a T0 or T1 agent. If it substantially overlaps an existing one, discard it and reuse the existing agent with a sharper brief instead.
- The agent's actual work output is still gated by the normal auditor wave like every other agent. Synthesis does not buy a pass on the gate.

## Registration is discovery, not hand-editing

Because Tier 1 resolution is a glob-and-match over `agents/*.md`, dropping a well-formed file in `agents/` is enough for the orchestrator to find and reuse it next run. You do not have to hand-edit the SKILL.md roster table or the schema enums for a synthesized agent to be discoverable and dispatchable.

- The `agent` field in `protocols/run-state.schema.json` and the `agent:` line in `protocols/handoff-schema.md` accept the core enum plus any kebab-case synthesized name, so run-state still validates when you record a synthesized dispatch.
- Optionally, if a synthesized agent proves broadly useful and graduates into a standard part of the roster, the operator can promote it: add a roster row in `SKILL.md` and fold it into a wave shape in `references/wave-planning.md`. That promotion is an operator-gated edit to core engine docs, not something the orchestrator does mid-run.

## The carve-out: what the orchestrator may and may not self-edit

The engine's standing rule is that it does not edit its own files to "improve itself". Synthesis is a deliberate, bounded exception. Hold the line exactly here:

- **Allowed (additive, the orchestrator does it):** writing a new agent definition into `agents/`. This adds a hand. It is additive, reversible (delete the file), and gated (the definition is checked, the output is audited). This is Tier 2.
- **Not allowed (core logic, stays operator-gated):** editing the run loop, the safety floor, the autonomy dials, the budget logic, the schemas' structure, or any existing agent's contract to change how the engine behaves. That is editing the brain. It still goes through a staged improvement proposal in the operator memory dir for the operator to review and apply, exactly as `references/self-improvement.md` describes.

Minting a new tool is not the same as rewriting the machine that wields it. The first is in scope; the second is inside the safety floor.

## Persisting to source: commit, do not publish silently

When you write a Mode B agent to the source repo:
- Writing and committing the new file locally is in scope under the carve-out (additive, reversible).
- Pushing to a shared remote, bumping the plugin version, and reinstalling the plugin are heavier, operator-visible actions. Surface them in the run report as the operator's next step rather than doing them autonomously, unless the operator has authorized push under the current external tier or in standing instructions.

## Record it

Whatever you mint, record it so the run is legible:
- On the dispatch in run-state: note whether it ran as a hyperfocused core agent (Mode A) or a registered specialist (Mode B), and the synthesized role name.
- In the run report: list every agent synthesized this run, the gap that opened it, and whether it was persisted. For each persisted (Mode B) agent, state the operator action that promotes it to a first-class dispatchable specialist: restart the session. A new `agents/*.md` file is not a dispatchable `subagent_type` until a fresh session re-scans `agents/` at startup; subagent types do not hot-reload mid-session (not even via `/reload-plugins`, which refreshes skills and hooks but not subagent definitions), so the engine cannot promote them itself. Until the restart, the capability is still available in-run through Mode A.
- In the retrospective: a synthesized agent that earned its keep is a lesson (a recurring capability the engine should keep reaching for); a synthesized agent that was a one-off and should not have been persisted is also a lesson. Feed both back per `references/self-improvement.md`.
