# Raw output - GMX recall run (2026-10-01)

Target: gmsol-labs/gmx-solana @ ef6d816596a1 (first parent of PR #447 merge 30459c0e, "feat(sdk/js): expose builder fee instructions in JS SDK")
Scope: `crates/sdk/src/builders` (30 .rs files, 6,739 lines)
Bundle: `_validation-clones/bundles/gmx-bundle.rs`

## What PR #447 fixed (ground truth from the public diff)

1. `CreateOrderParams.nonce` was silently dropped (`self.nonce.unwrap_or_else(generate_nonce)`
   ignored `params.nonce`); orders landed at addresses callers never chose, breaking
   follow-on `set_builder_fee` built against the caller-chosen address.
2. Increase orders hardcoded the final-output-token escrow slot to `None`; the fix passes
   `self.receive_token...` through so builder fees have a settlement path.
3. Stale-hint documentation/tests on `SettleBuilderFeeHint`.

## Run 1 (21-vector taxonomy) - MISS

Both agents returned "No findings." The vector agent triaged cleanly (17 Skip, 4 Borderline,
0 Survive) with good FP discipline, but no vector covered "caller-settable parameter silently
dropped / destination field hardcoded". Kill condition triggered -> taxonomy fixed:

- Added VS22 SILENTLY-DROPPED-PARAMETER.
- Sharpened VS17 FEE-ROUTING-CONFUSION with a hardcoded-destination confirm mode.

## Run 2 (22-vector taxonomy) - Agent A Vector Scan (raw)

Triage: Survive (3): VS10, VS17, VS22. Borderline (5): VS9, VS12, VS15, VS18, VS20 - all
dropped with concrete reasons. Skip (14). Total 3 + 5 + 14 = 22.

Deep pass:
- VS10: liquidity_provider.rs GT reward estimate -> DROP (off-chain display math, mirrors
  on-chain formula, no fund movement).
- VS17: builder_fee.rs + settle_builder_fee.rs claim vaults derived as builder User
  Account ATA, pinned by regression tests -> DROP (destinations correctly bound).
- VS22: create.rs CreateOrder::into_atomic_group() -> increase arms set receive_token =
  None -> receive_token_escrow = None -> final_output_token: Some(collateral) but
  final_output_token_escrow: None | guard: none | CONFIRM [80]. FP gate: concrete path
  (MarketIncrease/LimitIncrease with fee output mint but no escrow; SettleBuilderFeeHint
  errors), reachable (normal user path), impact (builder fees recorded on increase orders
  have no settlement path; stuck/unpayable value).

FINDING:

🟠 **1. Increase orders are built with `final_output_token` set but `final_output_token_escrow: None`, leaving builder fees with no settlement path**
gmx-solana/crates/sdk/src/builders/order/create.rs (bundle lines 4394-4404, 4433-4435, 4532-4536) · Confidence: 80

**Description:** For `MarketIncrease`/`LimitIncrease` orders the builder sets the order's
`final_output_token` to the collateral token but always passes `None` for
`final_output_token_escrow`, so any builder fee checkpointed onto the order can never be
settled out of an escrow (`SettleBuilderFeeHint` errors with "order has a non-zero builder
fee amount but no final output token escrow").

**Attack path:**
1. Caller builds a MarketIncrease order via CreateOrder::into_atomic_group; the match arm
   yields receive_token = None, so receive_token_escrow is None.
2. The instruction is built with final_output_token: Some(...) but
   final_output_token_escrow: None - the order records a fee output mint with no escrow.
3. set_builder_fee succeeds because the final output token is initialized, checkpointing a
   non-zero builder fee onto the order.
4. After execution, settle_builder_fee cannot be built/executed (ERR_NO_ESCROW): the
   recorded builder fee has no source escrow, so the builder is never paid.

**Fix:** pass `self.receive_token...` through as the escrow token instead of hardcoded None
(matches the upstream fix in PR #447 verbatim).

## Run 2 - Agent B Adversarial Reasoning (raw)

No findings. Candidates evaluated and dropped at the FP gate, all with concrete reasons:
swap_path.len() as u8 truncation (unreachable: tx account cap), GT reward using local
SystemTime (display-only, no impact), CloseOrder hint-supplied accounts (guarded on-chain,
out of scope), legacy-SPL token::ID hardcoding vs Token-2022 (reverts, not silent),
PreparePosition compute budget (out-of-scope dependency). PDA derivations cross-checked
self-consistent across all builders; signer model coherent.

## Orchestrator assessment

TARGET BUG CLASS (builder-fee routing break fixed in PR #447) FOUND cold by the vector
agent, exact file/lines, fix identical to upstream. PASS (run 2).
Partial miss recorded honestly: the `params.nonce` drop (also fixed in #447) was not
promoted as a separate finding; VS22 covers it but the agent confirmed only the escrow
variant. The adversarial agent found nothing - acceptable given the bundle is off-chain
builder code and its FP-gate reasoning was sound.
