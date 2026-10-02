# Loopscale recall run - full pipeline (2026-10-01, current tool)

Target: LoopscaleLabs/loopscale-pricing-adapters @ 96dcc95aecab (parent of fix commit 908b54ae, "don't floor the meteora redemption rate")
Scope: `src/` (14 .ts files, 1,487 lines), include-sdk semantics
Bundle: `_validation-clones/bundles/loopscan-bundle.ts`
Pipeline: 24-vector taxonomy, BUILD_NOTES, dual finder agents, merge, refutation agent.
Earlier runs (21/22-vector taxonomy, no refutation) are preserved in git history.

## Finder output (merged)

Vector agent: 17 Skip, 6 Borderline (all dropped with reasons), 1 Survive (VS10).
Adversarial agent: 4 findings.
Merge (dedupe: both agents found the meteora bug; kept conf 95 version):

1. (High, 95) Meteora Bn redemption rate divide-before-multiply, LP collateral zeroed/halved - meteora.ts:79-80 (bundle 664-665)
2. (Medium, 88) Legacy hylo handler NaN-poisons JitoSOL balance without xSOL - hylo.ts:43-47
3. (Medium, 85) Legacy sanctum handler NaN wSOL entry when no wSOL held - sanctum.ts:43-47
4. (Medium, 75) Whirlpool handler silently skips positions on missing/0 decimals - orca.ts:65-68,166-169

## Refutation verdicts

- FINDING 1: CONFIRMED | Kill attempt "rate always >= 1 so flooring is harmless" failed:
  truncating BigInt division misprices any non-integer rate, yields 0n under par, and the
  catch/reportError path never fires because the result is silently wrong, not an exception.
- FINDING 2: CONFIRMED | Kill attempt via the decimals guard failed: xsolDecimals is always
  present (main.ts pushes underlying mints into the decimals list), so the unguarded multiply
  executes with balances[XSOL_MINT] undefined; NaN propagates to a real JitoSOL entry.
- FINDING 3: CONFIRMED | Kill attempt "SOL_MINT always initialized" failed: balances comes
  straight from the request body; undefined + newSol -> NaN -> null. Impact actually
  understated by the finder: the LST's full value vanishes, not just a spurious entry.
- FINDING 4: CONFIRMED | Kill attempt on reachability failed: decimalMap is built only from
  request mints plus fixed underlying lists; whirlpool pool tokens are never added, so the
  skip path is taken and requestErrors stays empty, defeating the fail-loud invariant.

## Final report

```
📋 Solana Scan Report
Files scanned: 14
Lines analyzed: 1,487
Findings: 4 (1 High, 3 Medium)
Red team: 4 confirmed, 0 weakened, 0 killed
```

TARGET BUG (meteora floor, fixed upstream in 908b54ae): found by both finder agents,
CONFIRMED by the red team. Proposed fix matches upstream (multiply-first, divide-last).
False positives surviving refutation: 0.
