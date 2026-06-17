# Run Retrospective: 2026-06-16-ferry-landing

The human-readable record of what this run taught. The machine-readable carry-forward is the lessons ledger at examples/orchestrator-memory/lessons-ledger.json. This was the first real end-to-end run of the engine (a golden run), so the subject is the engine, not the Ferry product.

## Run at a glance
- **Mission:** Ferry landing page + launch one-pager
- **Outcome:** complete (all 6 definition-of-done items met)
- **Waves used / cap:** 5 / 8
- **Dispatches used / cap:** 10 / 30 (plus 1 retrospect optimizer, exempt)
- **Iterate loops used / cap:** 1 / 3

## Lessons reapplied this run
None. This was the first run; no ledger existed at start, so the reapply step found nothing and skipped (the cold-start path, working as designed). The ledger now exists, so future runs will reapply from it.

## Signal mined from this run
- **Gate BLOCKs:** one, the wave-3 test synthesis. It was a test-wave ITERATE stored as BLOCK because of a verdict-vocabulary mismatch (see proposal). It correctly stopped the run from shipping a dead-ended CTA.
- **Re-dispatches (attempt > 1):** one, website-build attempt 2, the iterate pass. Clean: the routed fixes were specific and all landed in one pass.
- **Escalations:** none. The auditor correctly declined to fabricate compliance claims rather than escalating, which was the right call.
- **Budget efficiency:** 10/30 dispatches, 5/8 waves, 1/3 iterate loops. Nothing wasted; every gate changed or confirmed state. The wave-3 gate alone justified the gate apparatus by catching the disabled-CTA dead end before "ship."
- **Cross-agent contradictions:** none in the artifacts, the two parallel builds stayed consistent (the wave-2 gate verified byte-for-byte agreement). But two ENGINE-level gaps surfaced (below).

## Execution gaps surfaced (the gold from a golden run)
1. **Serialized parallelism (highest):** waves 2 and 3 were emitted one dispatch per message, so the builds and the persona tests ran serially, not in parallel. run-state recorded them as parallel, so the failure was invisible to every content gate. Caught only by the optimizer. Promoted to proposal + lesson L-0001.
2. **Verdict vocabulary mismatch:** the test wave produces SHIP/ITERATE/BLOCK but run-state.gate.verdict only allows PASS/WARN/BLOCK. Had to map ITERATE to BLOCK lossily. Raised as a direct proposal.

## Optimizer read
- **The one move:** record `dispatch_mode` (parallel|serial) per wave and self-check it before writing run-state, so the engine's core capability can never silently no-op again.
- **Cut or stop:** nothing in the run was wasted; do not add gates, personas, or wave width. The one thing to stop: shipping the spec with the verdict-vocabulary contradiction unresolved.

## Lessons written or strengthened
| Lesson id | New or strengthened | Category | The lesson |
|---|---|---|---|
| L-0001 | new | planning | Emit every dispatch of a parallel wave in one message; serial emission silently kills parallelism and run-state hides it |
| L-0002 | new | safety | Decline to fabricate trust/compliance claims on a fictional or unverified product, even if a test says it would convert |
| L-0003 | new | briefing | Frontend-testers answer the scripted riskiest-assumption question before reading explanatory copy |
| L-0004 | new | audit-pattern | Small-N persona testing is directional, not market validation; label it so in the synthesis |

## Engine-improvement proposals raised
Staged for operator review in examples/orchestrator-memory/improvement-proposals/, not applied:
- 2026-06-16-record-dispatch-mode.md (the one move)
- 2026-06-16-reconcile-verdict-vocabulary.md

## One-line summary for the run report
Applied 0 lessons (cold start), wrote 4 new ones, raised 2 engine-improvement proposals. The fast loop is proven; the slow loop is now seeded.
