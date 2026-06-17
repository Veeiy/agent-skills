# Engine Improvement Proposal: record and self-check dispatch_mode per wave

A staged change to the orchestrator engine, raised by the golden-run retrospective. NOT applied. Brad reviews and applies.

- **Proposal id:** 2026-06-16-record-dispatch-mode
- **Raised by run:** 2026-06-16-ferry-landing
- **Promoted from lesson:** L-0001
- **Status:** awaiting-review

## Target
- **File:** skills/orchestrator/SKILL.md (the "After a wave returns" / run-loop step 3) and protocols/run-state.schema.json (wave object)
- **Section:** dispatch + state-update step; wave schema

## The change
**Before:** the orchestrator marks a wave's dispatches complete in run-state with no record of whether they were emitted in one message. run-state stores the planned DAG (parallel), so a wave that was actually run serially still looks parallel in state.

**After:** add a `dispatch_mode` field to each wave (`parallel | serial`) and a one-line self-check in the run loop: before writing run-state for a wave with more than one dispatch, assert the dispatches were emitted in a single assistant message and record the result. If a multi-dispatch wave was emitted serially, record `dispatch_mode: serial` and flag it for the retrospective.

## Why
- **Evidence:** run 2026-06-16-ferry-landing serialized waves 2 and 3 (one dispatch per message), losing parallelism. run-state recorded them as parallel, so no gate caught it; only the end-of-run optimizer did.
- **Pattern:** this is the exact silent failure parallel-dispatch.md line 3 warns about. Reminders to "batch" decay; a recorded, gradeable field does not.
- **Expected effect:** the engine's core capability stops being able to silently no-op. Every retrospective can see and grade execution fidelity.

## Risk and blast radius
- **What this touches:** the run loop and the run-state schema. Additive (new optional field), no behavior change to agents.
- **Reversibility:** drop the field and the check.
- **Watch after applying:** that the self-check does not nag on legitimately single-dispatch waves (gates, solo waves).

## Operator decision
- [ ] Apply as written
- [ ] Apply with edits
- [ ] Reject
