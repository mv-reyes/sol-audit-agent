# Raw output - Manifest recall run (2026-10-01)

Target: Bonasa-Tech/manifest @ 2f1abe7738b26144a8bc5a4d30b6a57a3dd3ba82 (PR #738 merge commit, "Reduce red-black tree compute usage")
Scope: `programs/wrapper` (all 31 .rs files) + `programs/manifest/src/state` + `programs/manifest/src/program` top level = 46 files, 12,620 lines
Bundle: `_validation-clones/bundles/manifest-bundle.rs`

## Agent A - Vector Scan (raw)

### Triage

**Skip (6):** VS6 (no close/realloc-zero/manual-drain path anywhere), VS8 (no `remaining_accounts` use; fixed-position `next_account_info` iteration), VS12 (no oracle reads), VS13 (no scheduled-underlying assets), VS19 (no shares/LP issuance), VS21 (sysvars read via `Sysvar::get()`, never from the accounts list; no introspection).

**Borderline (4):**
- VS2 - wrapper forwards raw `trader_token_account`/`vault` into the core Deposit/Withdraw CPI; exploit requires core handlers to skip vault-PDA and token-owner validation, those handlers are not in the bundle - drop.
- VS5 - aliasing could manifest in core BatchUpdate if the same account were passed as both `base_global` and `quote_global`, but that requires `base_mint == quote_mint`, whose rejection lives in CreateMarket (not in bundle) - drop.
- VS10 - the concept would manifest inside `QuoteAtomsPerBaseAtom::checked_quote_for_base`/`checked_base_for_quote` (quantities.rs), not in the bundle; no `(a / b) * c` shape visible in any bundled value path - drop.
- VS14 - would manifest in core Deposit/CreateMarket accepting extension-bearing Token-2022 mints; not in bundle, and the one visible extension consumer (`try_to_reduce_global_tokens`) does check `TransferFeeConfig`/`TransferHook` - drop.

**Survive (11):** VS1, VS3, VS4, VS7, VS9, VS11, VS15, VS16, VS17, VS18, VS20.

### Deep Pass (verdict lines)

- VS1: wrapper/loader.rs process_batch_update/deposit/withdraw/claim_seat() -> check_signer() -> `assert_eq!(header.trader, *owner_key)` | guards: every authority parsed as `manifest::validation::Signer`, WrapperStateAccountInfo::new enforces owner==program+discriminant, Collect requires signer == hardcoded COLLECTOR | DROP
- VS3: wrapper/processors/deposit.rs process_deposit() -> mint sniffed from token account bytes -> core deposit CPI | binding enforced downstream by core vault seeds `[b"vault", market, mint]` | DROP
- VS4: manifest/state/utils.rs transfer_global_tokens() -> invoke_signed(global_vault_seeds_with_bump!) | mint and bump read from GlobalFixed, written at creation from find_program_address | DROP
- VS7: wrapper/processors/batch_upate.rs execute_cpi() -> invoke_passthrough(&manifest::id(), ...) | CPI targets hardcoded or id-checked, signer seeds program-derived | DROP
- VS9: manifest/state/market.rs place_order()/update_balance() -> checked math everywhere on value paths; one plain add bounded by last_valid_slot < 10_000_000 | DROP
- VS11: place_order() -> checked_quote_for_base + maker bonus compensation + round-up on rest/cancel | dust accrues to protocol by design | DROP
- VS15: MarketFixed stores base_mint_decimals/quote_mint_decimals read from mints at creation; no hardcoded scaling on production paths | DROP
- VS16: wrapper/processors/batch_upate.rs process_batch_update() -> prepare_orders() -> `price > best_ask_price` / `price < best_bid_price` (strict) -> execute_cpi() -> core place_order() breaks only on strict exceed then assert_can_take() errors PostOnlyCrosses | guards: none at equality on the wrapper side | CONFIRM [85]
- VS17: collect.rs process_collect() -> lamport move to hardcoded signer collector | DROP
- VS18: place_order() expired-prefix removal; GlobalClean | cleanup restricted and compensated; wrapper cancel_all scoped to caller | DROP
- VS20: wrapper handlers bind check_signer(wrapper_state.trader == owner Signer); core SwapV2 carries distinct payer and owner | DROP

### FINDING (Agent A)

🟡 **1. PostOnly orders at exact top-of-book price pass the wrapper's pre-filter and revert the entire batch in the core**
manifest/programs/wrapper/src/processors/batch_upate.rs:228,247 · Confidence: 85

**Description:** The wrapper's PostOnly crossing filter uses strict inequalities (drop a bid only when `price > best_ask_price`, an ask only when `price < best_bid_price`), while the core matching loop executes at price equality (it stops only when the maker price strictly exceeds the limit) and then rejects the PostOnly order with `PostOnlyCrosses` - so an exact-equality PostOnly order is forwarded by the wrapper and fails the whole BatchUpdate CPI, rolling back every cancel and placement in the batch.

**Attack path:**
1. A live maker order rests at the top of the book at price P (e.g. best ask at P, no expired orders ahead of it).
2. A trader submits a wrapper `BatchUpdate` combining a needed cancel with a PostOnly bid at exactly P - the natural price for a maker quoting at the touch.
3. `prepare_orders` keeps the bid: `price > best_ask_price` is false at equality (batch_upate.rs:228).
4. `execute_cpi` forwards the batch to the core; in `place_order` the limit check `(is_bid && maker_price > price)` is false at equality, matching proceeds, and `assert_can_take(PostOnly)` returns `PostOnlyCrosses` (market.rs:1157,1166; resting_order.rs:100).
5. The CPI error aborts the whole wrapper instruction: the cancels and all other orders in the batch revert. The risk-reducing cancel is lost for that slot, and re-quoting at the same touch keeps reverting. Mirror case (PostOnly ask at exact best bid) identical.

**Fix:**
```diff
- if price > best_ask_price && order.order_type == OrderType::PostOnly {
+ if price >= best_ask_price && order.order_type == OrderType::PostOnly {
...
- } else if price < best_bid_price && order.order_type == OrderType::PostOnly {
+ } else if price <= best_bid_price && order.order_type == OrderType::PostOnly {
```

## Agent B - Adversarial Reasoning (raw)

Note: this agent used `git` on the clone to diff the pinned commit against later upstream commits while validating reachability. Its finding was history-informed; recorded here for completeness but NOT counted as a cold find. Agent A's find is the cold evidence.

🟡 **1. Evicted global maker's order reverts trades instead of being skipped when the required amount rounds to zero**
programs/manifest/src/state/utils.rs:369,438 · programs/manifest/src/state/market.rs:1223 · Confidence: 85

**Description:** `try_to_reduce_global_tokens` only treats a global order as unbacked when `desired_global_atoms > num_deposited_atoms`, so for an evicted maker (no global seat, balance reported as zero) a fill whose quote value rounds to zero (`0 > 0` is false) proceeds to `global_dynamic_account.reduce(...)`, which errors with `MissingGlobal` - and the `?` at the match site reverts the entire BatchUpdate/Swap instead of removing the unbacked order and continuing. Quoting disagrees with execution: `can_back_order` answers `0 <= 0 == true`, so routers price the level as fillable and the trade reverts on-chain.

**Attack path:**
1. Maker places a `Global` bid at price p < 1 quote/base; deposit withdrawn to zero; seat evicted (permissionless at MAX_GLOBAL_SEATS).
2. The evicted maker's bid remains resting on the book at price p.
3. Victim Swap/BatchUpdate selling base reaches the poison level with remainder n base atoms such that floor(n * p) == 0.
4. Matching computes global_atoms_needed = 0; `desired > deposited` passes (0 > 0 false); `reduce` on the seatless maker returns MissingGlobal; the victim's whole transaction reverts.
5. The level persists until someone runs GlobalClean or a large fill removes it. Repeatable against arbitrary victims.

(Secondary: if the maker still holds a seat with a zero deposit, the zero-quote fill succeeds - the maker is credited base atoms for zero quote.)

## Orchestrator merge (Step 3)

Agent A found the TARGET BUG (wrapper PostOnly strict-vs-equality boundary, batch revert) cold, confidence 85, exact lines, exact fix (>= / <=). PASS.
Agent B produced one additional candidate (global-maker zero-rounding revert), history-informed, not counted for recall; flagged for manual follow-up.
Final report: 1 finding (target bug), plus 1 follow-up note.
