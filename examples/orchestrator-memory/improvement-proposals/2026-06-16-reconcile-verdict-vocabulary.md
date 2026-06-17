# Engine Improvement Proposal: reconcile the gate verdict vocabulary

A staged change to the orchestrator engine, raised by the golden-run retrospective. NOT applied. Brad reviews and applies.

- **Proposal id:** 2026-06-16-reconcile-verdict-vocabulary
- **Raised by run:** 2026-06-16-ferry-landing
- **Promoted from lesson:** none, raised directly from a spec contradiction
- **Status:** awaiting-review

## Target
- **File:** protocols/run-state.schema.json (gate.verdict enum) and SKILL.md + references/wave-planning.md (test-wave gate language)
- **Section:** gate verdict definitions

## The change
**Before:** the test wave's auditor synthesizes `SHIP | ITERATE | BLOCK` (SKILL.md run loop, wave-planning.md "auditor synthesizes -> ship / iterate / block"), but run-state.schema.json `gate.verdict` only allows `PASS | WARN | BLOCK | null`. The two vocabularies do not match.

**After:** pick one and make it consistent. Recommended: extend the run-state `gate.verdict` enum to `PASS | WARN | BLOCK | SHIP | ITERATE | null`, and give ITERATE a distinct gate path from BLOCK (ITERATE = expected, route fixes and re-test; BLOCK = failed, retry within the per-branch cap). Then update wave-planning.md so the gate-skip allowance keys on a real ITERATE, not on a BLOCK that was actually an iterate.

## Why
- **Evidence:** in run 2026-06-16-ferry-landing the wave-3 synthesis verdict was ITERATE, which had to be stored as BLOCK to keep run-state valid, with a warning noting the lossy mapping.
- **Pattern:** every test-style wave re-pays this tax. It is a live spec defect, not a one-run artifact.
- **Expected effect:** the test wave's natural verdict stores losslessly; the iterate path stops being conflated with failure; the gate-skip rule stops being able to fire on a mislabeled block.

## Risk and blast radius
- **What this touches:** the run-state schema and two spec docs. A schema enum widening is backward compatible with existing runs.
- **Reversibility:** revert the enum and doc wording.
- **Watch after applying:** that SHIP/ITERATE are used only on test/synthesis waves and PASS/WARN/BLOCK on build/ideation waves, so the two do not blur.

## Operator decision
- [ ] Apply as written
- [ ] Apply with edits
- [ ] Reject
