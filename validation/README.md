# Validation runs

Current suite executed 2026-10-01 with the full pipeline: dual finder agents (24-vector
taxonomy, BUILD_NOTES), merge, and the refutation (red-team) stage. Agents were instructed
not to consult git history or files outside the bundle. Earlier runs (21/22-vector
taxonomy, no refutation) are preserved in git history; the current run files below are
the canonical record.

| # | Type | Repo @ commit | Scope | Files / lines | Candidates | Confirmed | Killed | Result |
|---|------|---------------|-------|---------------|------------|-----------|--------|--------|
| 1 | Recall | [LoopscaleLabs/loopscale-pricing-adapters](https://github.com/LoopscaleLabs/loopscale-pricing-adapters) @ `96dcc95` (parent of "don't floor the meteora redemption rate") | `src/` (include-sdk) | 14 / 1,487 | 4 | 4 | 0 | PASS - floor bug found by both finders, CONFIRMED by red team, fix matches upstream |
| 2 | Recall | [Bonasa-Tech/manifest](https://github.com/Bonasa-Tech/manifest) @ `2f1abe7` (PR #738 merge) | `programs/wrapper` + core `state`/`program` | 46 / 12,620 | 2 | 1 | 1 | PASS - PostOnly boundary CONFIRMED; phantom Critical (double refund) killed via hypertree zeroing evidence |
| 3 | Recall | [gmsol-labs/gmx-solana](https://github.com/gmsol-labs/gmx-solana) @ `ef6d816` (parent of PR #447 merge) | `crates/sdk/src/builders` | 30 / 6,739 | 2 | 0 | 2 | See note - no exploitable bug at this rev; both candidates correctly killed |
| 4 | Cold | [Ellipsis-Labs/phoenix-v1](https://github.com/Ellipsis-Labs/phoenix-v1) @ `5a34f7f` | `src/` | 45 / 12,530 | 4 | 1 | 3 | 1 real finding (zero init params) CONFIRMED; 3 phantom wrap-arithmetic claims killed via overflow-checks = true |

## Command shape used per run

```
/scan-sol <target>            # runs 2, 3, 4
/scan-sol <target> include-sdk  # run 1 (TypeScript pricing adapters)
```

## Notes

- **Refutation stage evidence.** Run 4 is the cleanest demonstration: the adversarial
  finder produced a Critical vault-drain claim built on a silent-wrapping premise;
  overflow-checks = true makes it a panic, so the red team killed all three wrap claims.
  Without refutation the report would have shown a fabricated Critical on audited code.
- **Run 3 reclassification (honest).** Earlier grading called the gmx escrow candidate a
  true positive. The red team killed it, and the kill is correct: the SDK refuses to
  checkpoint builder fees on escrow-less orders (set_builder_fee.rs:122-130), so no fee
  can be stranded; upstream's fix comment agrees ("left unset by default so existing
  callers are unaffected"). The actual #447-class bug at this rev (dropped `params.nonce`)
  was missed by all three finder runs - a genuine recall gap, recorded as such.
- **Run 3 variance.** The escrow candidate was found in 2 of 3 finder runs. Finder output
  is non-deterministic; repeated runs of the same target can disagree.
- **False positives surviving refutation across the suite: 0.** Candidates killed: 6
  (all with cited evidence in the run files).

## Supporting artifacts

- [signature-tests.md](signature-tests.md) - grep-signature hit counts over five real
  Solana codebases (raydium-clmm, raydium-cp-swap, manifest, hylo, metadao).
