# Raw output - Phoenix cold test (2026-10-01)

Target: Ellipsis-Labs/phoenix-v1 @ 5a34f7f901fd9e04057198d4fc7b7286f78b53f2 (master HEAD at clone time). Never scanned before; chosen as a heavily audited native-Rust CLOB to measure false-positive rate honestly.
Scope: `phoenix/src` (45 .rs files, 12,530 lines; tests/ excluded by default scope)
Bundle: `_validation-clones/bundles/phoenix-bundle.rs`

## Agent A - Vector Scan (raw)

Triage: Survive (11): VS1, VS2, VS3, VS4, VS5, VS6, VS7, VS9, VS11, VS16, VS18.
Borderline (3): VS8 (authority-only lamport receiver redirect; self-harm, drop), VS20
(RequestSeatAuthorized payer/trader split; seat starts NotApproved, drop), VS22
(decode_order_packet zero-pad defaults; documented, safe direction, drop).
Skip (8): VS10, VS12, VS13, VS14, VS15, VS17, VS19, VS21 - all with one-line reasons.
Total 11 + 3 + 8 = 22.

Deep pass: all 11 survivors dropped with structured one-liners. Examples:
- VS1: is_signer enforced on every authority path (checkers/mod.rs) -> DROP
- VS2: MarketAccountInfo/TokenAccountInfo/SeatAccountInfo constructors enforce owner +
  discriminant on all raw reads -> DROP
- VS4: vault bumps from find_program_address at init, stored in header -> DROP
- VS6: tombstone drains + assign(system_program) + realloc(0); books/fees/traders required
  empty; no revival path -> DROP
- VS9: plain u64 mul can panic, but BPF panic reverts atomically; self-DoS only -> DROP
- VS11: fee rounds up, maker payouts floor; dust accrues to protocol -> DROP
- VS16: check_for_cross and match_order agree at equality; PostOnly amends/rejects before
  matching -> DROP
- VS18: process_collect_fees is permissionless but destination is owner-checked against
  header.fee_recipient; idempotent, victim-neutral -> DROP

FINDINGS: No findings.

## Agent B - Adversarial Reasoning (raw)

🟠 **1. `raw_base_units_per_base_unit = 0` yields a zero base lot size, letting askers drain the quote vault without depositing any base tokens**
initialize.rs:108-110 · Confidence: 85

**Description:** `process_initialize_market` never rejects `raw_base_units_per_base_unit = Some(0)`, which makes `base_atoms_per_base_unit = 0`, so `base_lot_size = 0` - the divisibility check `0 % num_base_lots_per_base_unit == 0` passes silently, and from then on base lots are credited and traded while every base token transfer computes to 0 atoms.

**Attack path:**
1. Attacker calls InitializeMarket (permissionless) with `raw_base_units_per_base_unit = Some(0)`. All checks pass.
2. Victims deposit quote; real quote atoms enter the quote vault.
3. Attacker deposits N base lots: `N * base_lot_size = 0` atoms move (zero-amount transfer skipped) - free base lots at zero cost.
4. Attacker places asks; a victim Swap buy fills them: victim pays real quote, receives `matched_base_lots * 0 = 0` base atoms; attacker is credited the real quote proceeds.
5. Attacker withdraws real quote atoms having never transferred a single base atom.

🟡 **2. `tick_size_in_quote_lots_per_base_unit = 0` is accepted, creating a market where every trade settles at zero quote atoms**
initialize.rs:98-99,151-155 · Confidence: 82

**Description:** The only tick-size validation is the divisibility check, which 0 trivially passes, so a market initialized with zero tick size silently executes all matches at `price * 0 * size = 0` quote lots with zero fees - takers withdraw base tokens without depositing any quote.

**Attack path:** zero-tick market; victim maker deposits and locks real base; attacker IOC buy matches at 0 quote lots, withdraws the base, maker is credited 0 quote lots.

## Orchestrator grading (honest)

Code-level claims verified against initialize.rs (sed 90-160): both zero-parameter paths
behave exactly as described; there are no nonzero checks on either parameter.

- Both are the "silent misconfiguration" class: permissionless market creation with
  unvalidated parameters. Impact requires victims to trade on the misconfigured market, so
  severity is Medium in practice (agent's High on #1 is arguably one notch hot; the agent
  itself flagged the conditional path).
- Counted as 2 valid findings, 0 false positives. The vector agent's clean "No findings"
  with 11 well-reasoned drops is the FP-discipline evidence: on audited code the scanner
  does not invent account-model bugs.

Cold test result: 2 findings (both real, conditional-impact), 0 false positives.

## Re-run with the 24-vector taxonomy (2026-10-01, after hostile review)

Same bundle, vector agent only, 24 vectors (VS1-VS24) + BUILD_NOTES (native; overflow-checks = true).

Triage: Survive 9 (VS1, VS2, VS3, VS4, VS6, VS7, VS9, VS17, VS23), Borderline 10, Skip 5.
Total 9 + 10 + 5 = 24.

Deep pass: VS1-VS4, VS6, VS7, VS17 dropped with structured reasons (signer/owner/constraint/
bump/CPI/fee-destination enforcement all present and verified). Two confirms:

1. VS23 CONFIRM [85] - zero lot/tick-size parameters pass market initialization. The vector
   pass now finds the class that only the free-form agent caught in run 4; this is the
   direct evidence that the VS23 addition closed the gap. Finding text and fix in the
   agent output (initialize.rs asserts; proposed nonzero guard).

2. VS9 CONFIRM [75] - candidate: a resting ask at an extreme price makes the
   price * tick_size * num_base_lots product in match_order overflow u64 and panic
   (overflow-checks = true per BUILD_NOTES), bricking buys that sweep the level.
   Orchestrator note: no max-price bound on ask placement was found on a spot check
   (only a Ticks::ONE floor for market orders), so the candidate is plausible but not
   fully verified end-to-end (u64 widths of the adjusted-lot types not re-derived).
   Recorded as an unverified candidate, not a confirmed finding.

False positives in the re-run: 0 confirmed FPs (one unverified candidate disclosed as such).

## Refutation pass (2026-10-01, red-team stage validation)

Red Team agent on the two confirmed findings plus the unverified VS9 candidate.
Result: 2 CONFIRMED (findings 1-2 as reported), and the VS9 candidate upgraded from
"unverified" to CONFIRMED with a scope correction.

- Zero lot/tick init params: CONFIRMED on all three gates. Notably, num_base_lots_per_base_unit
  = 0 IS rejected (mod-by-zero), matching the finding's exact two variants. IOC buys skip
  the balance pre-check that would divide by zero, completing the drain path. Init is
  permissionless; the creator self-authorizes PostOnly -> Active.
- Toxic-ask overflow (former unverified candidate): CONFIRMED. The agent traced what the
  orchestrator had not: no max-price guard on ask placement (floor at Ticks::ONE only),
  plain `self.inner * other.inner` Mul impls (panic with overflow-checks = true), unpriced
  IOC buys default to limit Ticks::MAX, and the overflowing multiply executes before the
  budget checks. Scope correction applied: only unpriced/deep sweeps through the toxic
  level are bricked; priced limit orders below it trade normally. Medium severity fair.

False positives after refutation across all runs: 0.
