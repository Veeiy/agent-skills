# Self-Improvement Loop

How the orchestrator gets better at orchestrating over time. The engine already has a tight intra-run quality loop (dispatch, gate, iterate). What this adds is memory ACROSS runs, so a mistake the auditor caught last week does not get made again this week.

This is the scaffold: a structured, persistent place to store what the engine learns, plus the two steps that write to it and read from it.

## The two loops

**Fast loop (inside one run), already built:** dispatch a wave, gate it through the auditor, iterate on BLOCKs. This corrects the current run. It does not survive the run.

**Slow loop (across runs), this scaffold:**
```
   load lessons  ->  PLAN  ->  DISPATCH  ->  GATE  ->  ... DoD met
   (reapply)                                              |
        ^                                                 v
        |                                            RETROSPECT
        |                                            (mine the run)
        |                                                 |
        +---------------- LEARN <-------------------------+
                       (write the ledger)
```
Every run starts by reapplying what prior runs learned, and ends by mining itself for new lessons. The lessons ledger is the thing that persists between the two.

**Meta loop (improving the engine itself):** when a lesson is not about a mission but about the engine's own playbook, the retrospective writes a staged improvement proposal. The operator reviews it and applies it to the engine source. The engine never edits its own files. See "Improving the engine itself" below.

## Where memory lives

Three different lifetimes, three different homes. Do not mix them.

| Store | Lifetime | Home | Holds |
|---|---|---|---|
| run-state.json | one run | the run directory | live state of the current run (already built) |
| Lessons ledger | all runs | operator memory dir | distilled lessons, reapplied to every future run |
| Improvement proposals | until reviewed | operator memory dir | staged edits to the engine itself, awaiting operator review |

**Operator memory directory.** Default root `~/Vault/Vantage-OS/orchestrator-memory/`. If `~/Vault/Vantage-OS/` does not exist, fall back to `./orchestrator-memory/` under the current working folder and tell the operator where it went (the same fallback rule the run directory uses).

It contains:
- `lessons-ledger.json` - the durable lessons. Schema: `protocols/lessons-ledger.schema.json`. Seed from `templates/lessons-ledger.template.json` on first write.
- `improvement-proposals/` - one markdown file per staged engine change, from `templates/improvement-proposal.template.md`.

The ledger lives OUTSIDE the per-run directory (so it survives the run) and OUTSIDE the engine's own files (so the sold, venture-agnostic engine stays clean and every operator's instance learns its own lessons). The engine reads the ledger as an optional input, exactly the way it reads the permissions file: present means use it, absent means run without it.

## Step A: Reapply (at the start of every run)

Right after you load the blueprint and before you plan the DAG:

1. Read `lessons-ledger.json` from the operator memory dir. If it is absent, this is a cold instance with no memory yet; skip this step silently and carry on. The first run never has lessons, and that is normal.
2. Match lessons to this mission. A lesson carries a `trigger` and `tags`. Select the lessons whose trigger fits this blueprint (mission type, whether there is a storefront, whether a price is stated, the compliance lines, the agents this run will use).
3. Rank the matches by `confidence` then relevance, and take the top 7. The cap matters: a brief stuffed with 30 lessons is a brief nobody reads. Quality over coverage.
4. Apply them in two places:
   - **Planning:** lessons whose `applies_to_agents` includes `orchestrator` shape the DAG (a wave shape that wasted dispatches last time, a false parallel that bit you, a missing grounding wave).
   - **Briefs:** every other applicable lesson rides into the relevant agent's brief as a short "Lessons from past runs that apply to you" block. Only give an agent the lessons that touch its slice; the website agent does not need the supplier-research lesson.
5. Record what you applied in run-state under `learning.lessons_applied` (the lesson ids), set each applied lesson's `last_applied`, and increment its `times_applied` in the ledger.

Reapply is the half of the loop that makes the engine actually improve. Writing lessons nobody reapplies is journaling, not learning.

## Step B: Retrospect and learn (at the end of every run)

After the final audit and the run report, before you hand back:

1. **Mine run-state for signal.** Walk the finished `run-state.json` and pull every teachable moment:
   - Every gate `BLOCK`: what did the auditor reject, and was it a one-off or a repeat of a known pattern? Candidate `audit-pattern` lesson.
   - Every dispatch with `attempt` greater than 1: the first brief was thin or wrong. What was missing from it? Candidate `briefing` lesson.
   - Every escalation: did pre-flight miss a capability gap, or did a branch hit the safety floor in a way worth anticipating next time? Candidate `tooling` or `safety` lesson.
   - Budget efficiency: compare `dispatches_used` against DoD items satisfied. Wasted dispatches usually mean a too-wide wave or a false parallel. Candidate `planning` lesson.
   - Cross-agent contradictions the auditor caught (site price not equal to economics, voice not equal to brand): a shared input did not propagate. Candidate `briefing` or `planning` lesson.
2. **Get the strategic read.** Dispatch the `optimizer` agent against the finished run (scope = the run directory). It returns the one highest-leverage move and what to cut. Its "the one move" and "cut or stop" lines are often the best lessons in the run. The optimizer is advisory, so you are distilling its read, not actioning it. This single retrospective dispatch is exempt from the mission dispatch cap (it is meta-work, not mission work), but it is bounded to exactly one optimizer call with no iteration. A budget-exhausted run still gets its retrospective; a run that ran out of budget is exactly the kind worth learning from.
3. **Distill to lessons, and dedupe.** Turn the candidates into ledger lessons in imperative voice ("Carry the chosen price into both the website brief AND the economics brief in the same wave; they drift otherwise"). Before writing a new lesson, check the ledger for an existing one with the same trigger. If it exists, do not duplicate it: increment its `times_confirmed` and raise its `confidence`. A lesson the engine relearns is a lesson the engine should trust more.
4. **Write the ledger** to the operator memory dir. Seed it from the template if this is the first run, and increment `runs_seen`.
5. **Write the retrospective** to the run directory as `retrospective.md` (from `templates/retrospective.template.md`), and point run-state `learning.retrospective_path` at it. The retrospective is the human-readable record of what this run taught; the ledger is the machine-readable carry-forward.
6. **Surface it.** In the run report, state how many lessons you applied at the start, and how many you wrote or strengthened. That one line is how the operator sees the engine getting smarter.

## Lesson lifecycle

A lesson is not gospel the moment it is written. It earns trust.

- **Born `low` confidence.** One run taught it. It might be a fluke.
- **Confirmed:** each later run where the lesson was applied and the bad outcome did not recur (or the good one did) increments `times_confirmed` and raises confidence: low to medium at 2, medium to high at 4.
- **Contradicted:** if a lesson was applied and the failure happened anyway, the lesson is not working. Increment `times_contradicted`, drop its confidence, and flag it in the retrospective for the operator to rewrite or retire. Do not silently keep trusting a lesson that keeps failing.
- **Dormant:** a lesson that has not matched a trigger in many runs is not wrong, just unused. Keep it, deprioritize it. Do not delete on age alone.
- **Promoted:** a `high` confidence lesson that is about the engine's own playbook rather than a single mission graduates into an improvement proposal (next section). Set its `status` to `promoted` and link the proposal.

This is what keeps the ledger from rotting into a pile of stale superstitions. Lessons that pay off get stronger; lessons that lie get caught.

## Improving the engine itself (staged, never auto-applied)

Some lessons are not "this mission needed X." They are "the engine's wave-planning guidance is missing X, and every run rediscovers it." Those should change the engine, not just the ledger.

When the retrospective produces one of these (usually a promoted high-confidence lesson):

1. Write an **improvement proposal** to `improvement-proposals/` in the operator memory dir, from `templates/improvement-proposal.template.md`. It names the exact target file (`SKILL.md`, a reference, an agent file), the precise before/after change, the evidence (which runs and dispatches), and the expected effect.
2. **Surface it in the run report.** "N engine-improvement proposals are waiting in <path>."
3. **Stop there.** The orchestrator does NOT apply the change to its own files.

Why staged and not automatic, even under FULL autonomy:
- This engine is a sold, versioned product. An engine that rewrites its own source diverges the operator's copy from what ships, and one bad self-edit can corrupt the engine for every future run. The blast radius is the entire product, not one mission.
- Editing your own operating instructions is irreversible and not what the operator asked for on this run. That is squarely inside the universal safety floor (irreversible, not explicitly requested). FULL autonomy does not move the floor.

So the engine improves at two speeds, both real: the ledger makes it smarter every single run with zero human input, and the engine source improves through proposals the operator reviews and applies. The first is automatic because it is reversible and scoped to one operator. The second is gated because it is neither.

If the operator reviews a proposal and says apply it, applying it is then an explicit, requested edit and you may make it like any other file change.

## Bootstrapping

- No memory dir yet: the reapply step finds no ledger and skips. The retrospective step creates the dir and seeds the ledger from the template. By the second run there is memory to reapply.
- No Vault: fall back to `./orchestrator-memory/` and say so, same as run directories.
- A sold or fresh instance therefore needs nothing configured. It runs cold on run one and is already learning by run two.

## What this is NOT

- Not a place to dump raw run logs. The run directory already holds those. The ledger holds distilled, reusable lessons only.
- Not auto-modification of the engine. See above. Proposals are reviewed.
- Not a memory of mission content. Do not store a venture's concept, prices, or copy in the ledger. Store how to orchestrate better, not what one mission contained. Mission content lives in that mission's run directory.
