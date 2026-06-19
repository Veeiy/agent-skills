# Mission Blueprint

The one input that makes the engine venture-agnostic. Fill this out (or let the orchestrator fill it from a request) before any agent is dispatched. The orchestrator will not run on a vague blueprint; it will sharpen the objective and definition of done with you first.

---

## Mission name
<short slug-able name, e.g. "Reorder product launch">

## Objective
<one or two sentences, measurable. "Launch a single white-label product sold through a partner channel and a test-mode storefront, validated by 3 user personas, for under $X." Not "do the thing.">

## Definition of done
List the concrete things that must be true for this run to be finished. Each becomes a tracked item in run-state.
- [ ] <e.g. winning concept selected and audited>
- [ ] <e.g. fulfillment/ops process built and staged>
- [ ] <e.g. test-mode storefront live with working click-through>
- [ ] <e.g. 3 persona tests run, findings synthesized, ship/iterate decision made>
- [ ] <e.g. run report delivered with the exact go-live actions left for the operator>

## Constraints
- **Budget cap:** $<number or "none"> (agents score viability against this; nothing is spent regardless)
- **Deadline:** <date or "none">
- **Brand tokens:** <name / colors / voice, or "to be created in run">
- **Compliance / hard rules:** <free field, fill with whatever applies to this mission, e.g. "cosmetic claims only no health claims", "no em dashes", "platform = X". These are read verbatim by every agent.>

## Autonomy (Dial 1: execution)
<FULL | CHECKPOINT | MANUAL>
- FULL = run the whole loop hands-off, auto-pick concepts, auto-retry blocks, auto-iterate. (Default.)
- CHECKPOINT = stop and summarize after each wave gate.
- MANUAL = stop before every dispatch.

## External authority (Dial 2: blast radius)
<Tier 1 | Tier 2 | Tier 3>  (default Tier 1)
- Inherits ~/Vault/Vantage-OS/permissions.md if present in the operator's environment; otherwise defaults to Tier 1.
- Tier 1 = build to staging, no external sends/posts/money/signatures.
- Independent of the autonomy dial. FULL autonomy at Tier 1 is the normal fast-and-safe default.

## Run budget (stop conditions)
Caps that keep an autonomous run from looping or fanning out without bound. These are the operator's runaway-API guardrails: set them here and the engine cannot exceed them. When any cap is hit, the orchestrator stops cleanly, sets status to `budget-exhausted`, writes the run report with what is done and what remains, and surfaces it to the operator.
- **Max total dispatches:** 30. The single global ceiling on agent dispatches across the whole run, checked before EVERY dispatch (wave workers, auditor gates, BLOCK retries, on-demand research, synthesis). Every dispatch is a paid API call, so this one number is your hard cap against runaway cost. Lower it for a tight run, raise it for a large mission.
- **Max waves:** 8. Hard ceiling on the number of waves the run will execute.
- **Max iterate loops:** 3. Hard ceiling on test-to-iterate (self-correction) cycles.
- **Max agents synthesized:** 3. Hard ceiling on new specialists the run may mint (Tier 2). At the cap the engine reuses or hyperfocuses existing agents instead of minting more.

These defaults are sane for most missions and can be raised for large missions. The live counters (`dispatches_used`, `waves_used`, `iterate_loops_used`, `agents_synthesized_used`) live in `run-state.json` under `budget`. The one dispatch that does not count against the cap is the single bounded retrospective optimizer pass at the end of the run.

## Self-bootstrapping
If no permissions file or operator config is found in the environment, the engine defaults to FULL execution autonomy at Tier 1 with the universal safety floor, and reads all hard rules from this blueprint. Nothing else is required to start a run.

## Safety floor (always on, not configurable)
The run will pause and hand these to the operator no matter what the dials say:
- Moving money, transacting, placing orders against a payment method.
- Entering credentials, card/bank/gov numbers, passwords, API keys.
- Permanently deleting data; changing access controls or security settings.
- Anything irreversible and externally visible not explicitly requested.

## Pre-supplied inputs (optional)
If a phase is already done, drop its output here so the orchestrator can skip that wave.
- **Chosen concept (skips ideation):** <...>
- **Existing economics:** <price / margin / MOQ / cost>
- **Existing brand/site assets:** <paths>

## Notes for the orchestrator
<anything else: known risks, who the real testers should be later, integration with other systems, etc.>
