# Raw output - Loopscale recall run (2026-10-01)

Target: LoopscaleLabs/loopscale-pricing-adapters @ 96dcc95aecab (parent of fix commit 908b54ae "don't floor the meteora redemption rate", PR #3)
Scope: `src/` (14 .ts files, 1487 lines), include-sdk semantics
Bundle: `_validation-clones/bundles/loopscan-bundle.ts`

## Agent A - Vector Scan (raw)

## TRIAGE PASS

**Skip** (construct and concept absent - no on-chain program, no account model, no CPI, no vault):
VS1, VS3, VS4, VS5, VS6, VS7, VS8, VS13, VS14, VS16, VS17, VS18, VS19, VS20, VS21

**Borderline** (no direct match, concept could manifest differently):
- VS2 - Off-chain decode without owner check manifests in `deriveStakePoolExchangeRatesBn()` decoding stake-pool accounts from a TOML-fetched address list, but exploitation requires compromising the curated sanctum-lst-list repo, not fabricating an account -> drop.
- VS11 - Rounding manifests in `switchBaseDecimalsBn()`/`toFixed(0)` in every Bn handler, but the rounding error is ~1e-16 relative, far below economic viability -> drop.
- VS15 - Decimals are read on-chain via `getDecimalMap()` everywhere; the flash handler's hardcoded 6-decimal assumption covers a fixed, verified mint set -> drop.

**Survive** (construct clearly present):
VS9 (arithmetic overflow/truncation: `toNumber()`, `parseInt()`, float coercion of BN), VS10 (divide-before-multiply in rate math), VS12 (oracle reads from Pyth Hermes / remote price APIs with no freshness bound)

Total: 15 Skip + 3 Borderline + 3 Survive = 21

## DEEP PASS

VS9: main.ts `/v1/decompile_mints` -> `parseAndConvertWhirlpoolPositionsBn()` -> `attemptParseWhirlpoolPositionsBn()` -> `whirlpoolTokenBalancesBn()` -> `liquidity.toNumber()` (clmm_math.ts:1417) | guard: outer try/catch -> `reportError` (fails request loud); legacy route swallows silently | verdict: CONFIRM [80] - bn.js `toNumber()` throws above 2^53, whirlpool `liquidity` is u128 and routinely exceeds it.

VS9: main.ts `/decompile_mints` (legacy) -> `getXsolBalanceInJitoSol()` -> `xsolPriceInJitoSol * balances[XSOL_MINT]` (hylo.ts:517) with no presence guard; and `deriveStakePoolExchangeRates()` -> `balances[SOL_MINT] + newSol` (sanctum.ts:1164) with undefined LHS | guard: none (Bn variants fixed both) | verdict: CONFIRM [85] - NaN poisoning of aggregate mints.

VS10: main.ts `/v1/decompile_mints` -> `getMeteoraTokenBalancesBn()` -> `scalar = BigInt(quote.tokenAOutAmount) / quoteInput` (meteora.ts:664) -> `balance * scalar` (meteora.ts:665) | guard: none | verdict: CONFIRM [90] - integer divide-before-multiply floors the per-LP quote to whole dollars before scaling by balance.

VS10: legacy `getMeteoraTokenBalances()` scalar division (meteora.ts:627) and legacy sanctum `solRatio = totalLamports / poolTokenSupply` (sanctum.ts:1160) are float divide-before-multiply | DROP (relative error ~1e-16, immaterial; Bn sanctum already mul-then-div).

VS12: `getXsolBalanceInJitoSolBn()` -> Hermes `latest` price, no publish_time/max_age | DROP (off-chain TVL adapter; every source - Hermes, CloudFront, markets API - is a trusted remote endpoint with no age-bound machinery by design, and no on-chain value movement is in scope; fetch failures already fail loud on the v1 route).

Composability: findings 2 and 3 both degrade the same endpoints' output but on independent paths; no chaining amplification.

## FINDINGS (Agent A)

🟠 **1. Meteora LP pricing floored to whole dollars by integer divide-before-multiply**
loopscale/src/pricing/meteora.ts:664-665 (bundle lines) · Confidence: 90

**Description:** `getMeteoraTokenBalancesBn()` computes a per-LP exchange-rate scalar with integer division (`tokenAOutAmount / 1_000_000`) before multiplying by the user's balance, discarding all sub-unit precision of the quote and zeroing it entirely when one LP unit is worth less than 1 USDC base-unit multiple.

**Attack path:**
1. A request to `/v1/decompile_mints` includes the Meteora LP mint `xLebAypjbaQ9tmxUKHV6DZU4mY8ATAAP2sfkNNQLXjf` with any balance.
2. `amm.getWithdrawQuote(1_000_000, 0, USDC)` returns `tokenAOutAmount` in USDC base units for 1e6 LP base units (≈ the dollar value of one LP, e.g. 1.99 USDC -> 1_990_000).
3. `scalar = 1_990_000n / 1_000_000n = 1n` - the LP is priced at $1.00 instead of $1.99 (33% undervaluation); if one LP is worth < $1, `scalar = 0n` and the entire position prices to zero.
4. The mispriced USDC total is returned with HTTP 200 (no error is raised), so consumers persist a silently wrong collateral valuation.

**Fix:**
```diff
- const scalar = BigInt(quote.tokenAOutAmount.toString()) / quoteInput;
- const scaledOutAmount = switchBaseDecimalsBn(balance * scalar, poolDecimals, outMintDecimals);
+ // Multiply first, divide last: preserve quote precision across the full balance.
+ const scaledOutAmount = switchBaseDecimalsBn(
+     (balance * BigInt(quote.tokenAOutAmount.toString())) / quoteInput,
+     poolDecimals,
+     outMintDecimals
+ );
```

🟡 **2. Legacy number handlers poison aggregate balances with NaN (missing zero-balance guards)**
loopscale/src/pricing/hylo.ts:517-521, loopscale/src/pricing/sanctum.ts:1163-1164 (bundle lines) · Confidence: 85

**Description:** On the live legacy `/decompile_mints` route, `getXsolBalanceInJitoSol()` multiplies the Pyth price by `balances[XSOL_MINT]` without checking it exists, and `deriveStakePoolExchangeRates()` adds to `balances[SOL_MINT]` without initializing it, producing NaN that serializes to `null` in the response - silently corrupting the JitoSOL/SOL totals the Bn variants explicitly guard against.

🟡 **3. `liquidity.toNumber()` throws on whirlpool positions with liquidity > 2^53, breaking whirlpool pricing**
loopscale/src/utils/clmm_math.ts:1417-1418 (bundle lines) · Confidence: 80

**Description:** `whirlpoolTokenBalancesBn()` converts the u128 `liquidity` field to a JS number via bn.js `toNumber()`, which throws for any value above 2^53 - a routine magnitude for whirlpool liquidity - causing the v1 route to fail the entire request (500) and the legacy route to silently skip the position, under-reporting its value.

## Agent B - Adversarial Reasoning (raw)

🟠 **1. Meteora v1 handler truncates LP redemption rate to a whole number via BigInt scalar division**
loopscale/src/pricing/meteora.ts:73-83 · Confidence: 90

**Description:** In `getMeteoraTokenBalancesBn` the per-unit withdraw quote is computed as `BigInt(quote.tokenAOutAmount.toString()) / quoteInput` - a BigInt integer division - so the LP-to-USDC rate is floored to an integer (e.g. 1.08 -> 1, 0.95 -> 0), silently mispricing or entirely zeroing out the LP collateral on the v1 endpoint while `delete balances[metMint]` still removes the original balance.

**Attack path:**
1. Normal usage: POST `/v1/decompile_mints` with `rawBalances` containing the Meteora LP mint `xLebAypjbaQ9tmxUKHV6DZU4mY8ATAAP2sfkNNQLXjf` (e.g. balance `5000000` = 5 LP at 6 decimals).
2. `getWithdrawQuote(1_000_000n, ...)` returns `tokenAOutAmount ≈ 1_080_000` (LP trades at ~1.08 USDC); `scalar = 1080000n / 1000000n = 1n` - the 8% premium is silently discarded.
3. If the pool rate ever drops below 1.0 (or the quote returns < 1_000_000), `scalar = 0n`; `switchBaseDecimalsBn(balance * 0n, ...) = 0n` credits nothing while `delete balances[metMint]` removes the collateral - the position vanishes from the response with no error reported (no exception is thrown, so `reportError` never fires and the request returns 200).
4. Consumers (DefiLlama) persist a silently understated TVL. The legacy float path (`getMeteoraTokenBalances`, line 42) computes the scalar correctly, so this is a regression introduced in the Bn port.

**Fix:**
```diff
- const scalar = BigInt(quote.tokenAOutAmount.toString()) / quoteInput;
- const scaledOutAmount = switchBaseDecimalsBn(balance * scalar, poolDecimals, outMintDecimals);
+ const outAmount = BigInt(quote.tokenAOutAmount.toString());
+ const scaledOutAmount = switchBaseDecimalsBn((balance * outAmount) / quoteInput, poolDecimals, outMintDecimals);
```

🟡 **2. Legacy hylo handler poisons the JitoSOL balance with NaN whenever the request contains no xSOL**
loopscale/src/pricing/hylo.ts:43-47 · Confidence: 85

**Description:** `getXsolBalanceInJitoSol` multiplies the Pyth price by `balances[XSOL_MINT]` without an undefined guard, so on every legacy `/decompile_mints` request lacking xSOL it produces `NaN` and overwrites any pre-existing JitoSOL balance (`(balances[JITOSOL_MINT] || 0) + NaN = NaN`), which serializes to `null` in the JSON response.

🔵 **3. Legacy sanctum handler emits a NaN wSOL entry when the wallet holds an LST but no wSOL**
loopscale/src/pricing/sanctum.ts:46-48 · Confidence: 85

**Description:** `deriveStakePoolExchangeRates` accumulates into `balances[SOL_MINT]` with `balances[SOL_MINT] + newSol` instead of `(balances[SOL_MINT] || 0) + newSol`, so any request containing a Sanctum LST but no pre-existing wSOL balance yields `undefined + newSol = NaN` (serialized as `null`) for the wSOL entry.

🔵 **4. xSOL/JitoSOL conversion uses Pyth "latest" price with no staleness or confidence validation**
loopscale/src/pricing/hylo.ts:22-35,62-90 · Confidence: 65

**Description:** Both hylo handlers fetch the Hermes `/v2/updates/price/latest` feed and use `price.price`/`price.expo` without checking `metadata.publish_time` or `price.conf`, so a stale or halted xSOL feed silently prices collateral at an outdated rate.

## Orchestrator merge (Step 3)

Dedup: finding 1 identical in both agents (same function, same root cause) -> keep confidence 90. Agent A finding 2 = Agent B findings 2+3 (same root cause class, two sites) -> merged as one finding with two sites, confidence 85. Agent A finding 3 (whirlpool toNumber) unique, confidence 80. Agent B finding 4 (Pyth staleness) unique, confidence 65, no fix (below 80).

Final: 4 findings. TARGET BUG (Meteora divide-before-multiply floor) FOUND by both agents, cold. PASS.
