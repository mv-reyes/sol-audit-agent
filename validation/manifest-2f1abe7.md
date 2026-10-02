# Manifest recall run - full pipeline (2026-10-01, current tool)

Target: Bonasa-Tech/manifest @ 2f1abe7738b26144a8bc5a4d30b6a57a3dd3ba82 (PR #738 merge commit)
Scope: `programs/wrapper` + `programs/manifest/src/state` + `programs/manifest/src/program` top level (46 files, 12,620 lines)
Bundle: `_validation-clones/bundles/manifest-bundle.rs`
Pipeline: 24-vector taxonomy, BUILD_NOTES (native pinocchio; overflow-checks = true), dual finder agents, merge, refutation agent.
Earlier runs (21-vector taxonomy, no refutation) are preserved in git history.

## Finder output (merged)

Vector agent: 8 Survive, 10 Borderline, 6 Skip (24 total). 1 confirm.
Adversarial agent: 1 finding (critical-class candidate, self-flagged uncertainty).
Merge:

1. (Medium, 90) Wrapper PostOnly filter strict inequality reverts batch at exact touch - batch_upate.rs:228,247 (bundle 1006,1025)
2. (Critical, 60) Stale wrapper entry over freed-but-intact core block -> double refund - shared.rs:415-476

## Refutation verdicts

- FINDING 1: CONFIRMED | Killed on all three gates and failed. Wrapper filters only strict
  crossings; core matching stops only when maker price is strictly worse, so equality
  proceeds to assert_can_take -> PostOnlyCrosses; the error propagates out of the single
  BatchUpdate CPI, reverting cancels too. Price representations identical on both sides,
  so equality is preserved exactly. Medium fair (availability, not fund loss; a griefer
  can revert in-flight cancel+replace batches by resting at the bot's exact price).
  Fix >= / <= correct.
- FINDING 2: KILLED | The premise "freed block retains old RestingOrder bytes" is false:
  hypertree FreeList::add zeroes the payload (free_list.rs:51-56), so the block reads
  seq=0, atoms=0, and the wrapper's gone-check catches exactly this (bundle 2166-2170,
  with a comment documenting the zeroed-block case). Even in the worst corner, the core's
  hinted cancel computes the refund from get_num_base_atoms(), and a zeroed block refunds
  0. Both of the finding's self-noted out-of-bundle dependencies resolve against it.

## Final report

```
📋 Solana Scan Report
Files scanned: 46
Lines analyzed: 12,620
Findings: 1 (1 Medium)
Red team: 1 confirmed, 0 weakened, 1 killed

Appendix - Rejected by red team:
- "Stale wrapper entry enables double refund" (Critical, conf 60): killed - hypertree
  FreeList::add zeroes freed blocks; the wrapper's gone-check and the core's refund math
  both handle the zeroed case. Phantom Critical, correctly rejected.
```

TARGET BUG (PostOnly strict-vs-equality boundary): found cold, CONFIRMED by the red team.
The red team also killed a phantom Critical that earlier grading had recorded as a real
candidate - the refutation stage working as designed.
