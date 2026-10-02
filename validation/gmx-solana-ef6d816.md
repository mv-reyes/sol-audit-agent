# GMX recall run - full pipeline (2026-10-01, current tool)

Target: gmsol-labs/gmx-solana @ ef6d816596a1 (first parent of PR #447 merge 30459c0e)
Scope: `crates/sdk/src/builders` (30 .rs files, 6,739 lines)
Bundle: `_validation-clones/bundles/gmx-bundle.rs`
Pipeline: 24-vector taxonomy, BUILD_NOTES (off-chain SDK builders; overflow-checks = true), dual finder agents, merge, refutation agent.
Earlier runs (21/22-vector taxonomy, no refutation) are preserved in git history.

## What is actually exploitable at this commit (ground truth)

PR #447's diff shows three change classes: (a) dropped `params.nonce` in CreateOrder
(silently ignored in favor of a random nonce; breaks follow-on `set_builder_fee` built
against the caller-chosen address), (b) increase orders hardcoded the final-output-token
escrow to None, repurposed by the fix as an explicit opt-in ("left unset by default so
existing callers are unaffected" - upstream's own words), (c) stale-hint documentation
and tests. The wrong-party claim-vault routing bug (fixed in #445) is already fixed and
pinned by regression tests at this commit.

## Finder output (merged, run 3 of 3 for this target)

Vector agent: 2 Survive (VS17, VS22), 13 Borderline, 9 Skip. 1 confirm.
Adversarial agent: 1 candidate.
Merge:

1. (High, 80) Increase orders hardcode final-output-token escrow to None, builder fees
   stranded - create.rs:4394-4405,4531-4536 (bundle lines)
2. (Medium, 60) PreparePosition stamps a 15,000 CU budget on increase-order transactions -
   position.rs:13-14,106-107 + create.rs:403-417

Recall variance note: the escrow candidate was found in 2 of 3 finder runs (missed once
with a plausible-sounding but wrong justification). Non-determinism recorded honestly.

## Refutation verdicts

- FINDING 1: KILLED | The impact chain cannot start. Per settle_builder_fee.rs:84 the
  on-chain TokenAndAccount "records both or neither", so an escrow-less order also records
  NO final output token, and the SDK refuses to checkpoint a fee on exactly that order:
  set_builder_fee.rs:122-130 errors "the order's final output token is uninitialized, so
  it cannot carry a builder fee". No fee is ever checkpointed, so no fee can be stranded;
  with builder_fee_amount == 0, settle is a documented no-op. Verified independently by
  the orchestrator against set_builder_fee.rs:122-130. The escrow gap is a feature gap
  (upstream: opt-in, "callers unaffected"), not a fund-loss bug. Correctly killed.
- FINDING 2: KILLED | AtomicGroup::merge SUMS compute budgets
  (gmsol-solana-utils instruction_group.rs:271 + compute_budget.rs:170): the outer group's
  default 200,000 plus PreparePosition's 15,000 yields 215,000 - the 15k is additive
  headroom, not a cap. The transaction does not fail.

## Final report

```
📋 Solana Scan Report
Files scanned: 30
Lines analyzed: 6,739
Findings: 0
Red team: 0 confirmed, 0 weakened, 2 killed

No confirmed findings - all candidates were rejected by the red team (see appendix).

Appendix - Rejected by red team:
- "Increase orders hardcode escrow to None, builder fees stranded" (High, conf 80): killed -
  the SDK refuses to checkpoint fees on escrow-less orders (set_builder_fee.rs:122-130);
  feature gap, not fund loss.
- "PreparePosition 15k CU budget bricks increase orders" (Medium, conf 60): killed -
  AtomicGroup::merge sums budgets (215k total); not a cap.
```

## Honest recall assessment for this target

Earlier grading called the escrow finding the recall true-positive. The refutation stage
proved that grading wrong: by our own FP gate there is no reachable harm at this commit,
and upstream's fix comment says the same ("existing callers are unaffected"). The actual
#447-class bug present here - the dropped `params.nonce` - was NOT found by any of the
three finder runs. That is a genuine finder recall gap on this bug class at this target,
recorded rather than laundered: the pipeline's correct answer at this commit is "no
confirmed findings", and that is what it reports.
