# Adding a new agent

How to add a custom specialist the orchestrator can fan out, scope, and gate exactly like the built-in core.

There are two paths to a new agent, and they share this same contract:
- **Operator-driven (this doc):** you decide a specialist is worth having and add it deliberately, by hand, as a permanent part of the roster.
- **Orchestrator-driven (autonomous):** during a run the tiered resolution ladder hits a real Tier 2 gap and the engine mints the agent itself, hyper-focused on the ask, then persists it so it becomes reusable. That path is `references/agent-synthesis.md`, including the gap test, the two synthesis modes, and the self-edit carve-out that makes it safe. Read that if you want the engine to grow its own roster; read on here for the manual path.

## The principle

The orchestrator owns control. It scopes the mission, plans the waves, fans out specialists based on what the mission actually needs, loops the work through the auditor to catch gaps, and routes fixes back to whoever produced them. The agents are the hands, not the brain. A new agent is just a new capability the orchestrator can reach for. For that to work, the agent has to fit the loop: two hard requirements, then a few choices.

## Two hard requirements

1. **A sharp `description` (this is the fan-out trigger).** The orchestrator reads the description to decide when to dispatch the agent, so it has to name the single slice the agent owns and the concrete triggers that should summon it. Vague descriptions never get dispatched. Include 2-3 `<example>` blocks; they teach the orchestrator when to fan it out.
2. **Return the handoff envelope** (`protocols/handoff-schema.md`). Every agent returns the same envelope: run_id, wave, agent, dispatch_id, status, artifact_path, then its body, then ESCALATIONS. That envelope is what lets the auditor gate the work and the orchestrator slot it into run-state. An agent that returns loose prose cannot be gated or integrated, so it breaks the loop.

## Three choices

3. **Model.** `haiku` for wide, mechanical fan-out (many cheap instances, like persona testing). `inherit` for reasoning-heavy work. Keep the auditor on a strong model regardless.
4. **Tools.** Omit the `tools:` line to inherit all tools; add an allowlist to restrict it. Leave the Agent/Task tool out so the worker stays flat and cannot spawn its own sub-agents. The built-in `Explore` agent does exactly this. If a worker needs more work done, it escalates and the orchestrator fans out; the worker does not.
5. **Keep it one slice.** One agent, one job. If you find yourself writing an agent that "researches and builds and tests," that is three agents and a wave plan, which is the orchestrator's job, not the agent's.

## Register it so the orchestrator knows it exists

Dropping a well-formed file in `agents/` is enough for the orchestrator to find and use it: Tier 1 resolution is a glob-and-match over `agents/*.md` descriptions, and the `agent` field in both schemas accepts any kebab-case name, so a new agent is discoverable and its dispatches validate with no further edits. That is the minimum, and it is all a synthesized agent needs.

To **promote** an agent into a standard, documented part of the roster (recommended for one you added deliberately and will rely on), also:

1. **Roster:** add a row to the right tier table in `SKILL.md` (## The roster you command) with the agent's call and what it owns.
2. **Wave shape (optional):** if the new agent changes a canonical wave shape, add a note to `references/wave-planning.md`.
3. **Enums (optional, for legibility):** the schemas already validate any kebab-case name, so no edit is required; add the name to the lists in `protocols/handoff-schema.md` and `protocols/run-state.schema.json` only if you want it documented there.

## The template

Copy `references/agent.template.md` into `agents/<agent-name>.md` and fill it in.

## Want it callable directly too?

The pattern is the `optimizer`: it is both a skill you invoke yourself and an agent the orchestrator dispatches, sharing one brain. To make a dual-use agent, add a thin `skills/<name>/SKILL.md` pointing at the same method alongside the `agents/<name>.md`. Most workers do not need this; add it only for agents you also want to run standalone.

## Ship it

A new agent has to live in the source repo, not just the runtime cache. Editing only the cache is wiped on the next plugin update. Add the file in the source repo, commit, push, bump the version in `.claude-plugin/plugin.json`, and update the installed plugin to pick it up.

## What not to do

- Do not put the template, this guide, or any non-agent file in `agents/`. Everything there is loaded as a live agent. Templates and guides live under `references/`.
- Do not give a worker the dispatch tool without a depth cap and a run budget. Uncapped nesting escapes the auditor and burns tokens. Centralize dispatch in the orchestrator.
- Do not skip the handoff envelope. It is the contract the whole scope-dispatch-gate loop runs on.
