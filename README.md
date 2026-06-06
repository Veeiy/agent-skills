# Agent Skills

Two standalone [Claude Code](https://docs.claude.com/en/docs/claude-code) skills for high-leverage agentic work. Each is a single, carefully engineered `SKILL.md` — these are working prompts, not pseudocode.

> Genericized for public release: the operator persona is a neutral stand-in for the author.

## `optimizer/` — assess-and-optimize engine

A general-purpose "what's the highest-leverage move?" engine, callable in any chat. It ingests the whole context (conversation, referenced files, project state), assesses real state vs. talked-about state, finds the 80/20, and returns a ranked plan led by the single biggest-gain move. Explicitly a sparring partner, not a cheerleader: it challenges weak logic and names what isn't working. Advisory by default — it recommends, it doesn't take side-effectful actions.

**Triggers:** "optimize", "assess this", "what's the highest-leverage move", "poke holes in this", "where am I leaving value on the table".

## `orchestrator/` — wave-based multi-agent build engine

Plans a mission into a wave-based DAG, deploys specialist sub-agents in parallel, gates each wave through an auditor, and runs autonomously under a configured autonomy tier. The pattern: decompose → parallelize → gate → integrate, with each wave's output validated before the next wave starts.

**Triggers:** "orchestrate", "build this out", "spin up the agents", "run ideation to launch".

## What these demonstrate

- Prompt engineering for **role separation** (optimizer assesses, orchestrator builds, an auditor gates) so each agent has one job and clear boundaries.
- **Autonomy tiers** and explicit guardrails on side-effectful actions.
- Treating context ingestion as a first-class step — read the artifact before judging it, don't assess from the summary.

## License

MIT — see [`LICENSE`](./LICENSE).
