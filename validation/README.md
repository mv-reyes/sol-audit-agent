# Validation runs

All runs executed 2026-10-01 with the dual-agent workflow (vector scan + adversarial
reasoning in parallel, sonnet-class agents, FP gate, merge/dedup). Runs 1 and 2 used the
21-vector taxonomy; run 3 used 22 (VS22 added mid-day via the kill-condition loop); the
current taxonomy is 24 vectors (VS23 from the run-4 cold test, VS24 from hostile review).
Agents were instructed not to consult git history or files outside the bundle. Raw agent
output is in each run file.

| # | Type | Repo @ commit | Scope | Files / lines | Findings | False positives | Result |
|---|------|---------------|-------|---------------|----------|-----------------|--------|
| 1 | Recall | [LoopscaleLabs/loopscale-pricing-adapters](https://github.com/LoopscaleLabs/loopscale-pricing-adapters) @ `96dcc95` (parent of "don't floor the meteora redemption rate") | `src/` (include-sdk) | 14 / 1,487 | 4 | 0 | PASS - both agents rediscovered the floor bug, exact fix |
| 2 | Recall | [Bonasa-Tech/manifest](https://github.com/Bonasa-Tech/manifest) @ `2f1abe7` (PR #738 merge) | `programs/wrapper` + core `state`/`program` | 46 / 12,620 | 1 | 0 | PASS - PostOnly strict-vs-equality boundary, fix proposed (no upstream fix exists to compare) |
| 3 | Recall | [gmsol-labs/gmx-solana](https://github.com/gmsol-labs/gmx-solana) @ `ef6d816` (parent of PR #447 merge) | `crates/sdk/src/builders` | 30 / 6,739 | 1 | 0 | PASS on retry - first run missed; VS22 added; escrow bug found with upstream-identical fix |
| 4 | Cold | [Ellipsis-Labs/phoenix-v1](https://github.com/Ellipsis-Labs/phoenix-v1) @ `5a34f7f` | `src/` | 45 / 12,530 | 2 + 1 re-run | 0 | 2 real zero-parameter gaps (exposed a taxonomy gap -> VS23 added); 24-vector re-run: vector agent confirms VS23 [85], one unverified VS9 candidate disclosed |

## Command shape used per run

```
/scan-sol <target>            # runs 2, 3, 4
/scan-sol <target> include-sdk  # run 1 (TypeScript pricing adapters)
```

## Notes

- Run 3 documents the kill-condition loop: the 21-vector taxonomy missed the builder-fee
  class, VS22 (silently dropped parameter) was added, VS17 sharpened, and the retry found
  the bug. The first run's "No findings" output is preserved in the run file.
- Run 3 partial miss, recorded honestly: the `params.nonce` drop (also fixed in PR #447)
  was not promoted as a separate finding; the escrow variant of the same class was.
- Run 2's adversarial agent inspected git history despite instructions; its extra finding
  was excluded from recall credit. Agent A's find was fully cold.
- Run 4 severities: both findings require victims to interact with a permissionlessly
  created, misconfigured market. Real, but conditional-impact; reported as Medium in
  practice.
