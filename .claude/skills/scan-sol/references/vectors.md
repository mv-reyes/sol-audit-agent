# Solana Vulnerability Vectors (VS1-VS21)

21 vectors for Solana and Anchor (Rust) programs. Grouped in five classes:

- **A. Account model (VS1-VS8)**: the Solana-specific core. Most real Solana exploits live here.
- **B. Arithmetic (VS9-VS11)**: token math, truncation, rounding.
- **C. Oracle and pricing (VS12-VS13)**: freshness, feed binding, market hours.
- **D. Token program (VS14-VS15)**: Token-2022 extensions, decimals.
- **E. Logic and economics (VS16-VS21)**: boundaries, fees, cranks, vaults, authority, sysvars.

Grep signatures were verified against real program source (raydium-clmm, raydium-cp-swap, manifest, hylo, metadao futarchy; hit counts recorded 2026-10-01). Signatures are triage hints, not proofs. A hit means "look here"; absence of a hit never clears a vector on its own.

Merge/split decisions vs. the raw coverage list:
- Oracle staleness and oracle account substitution are merged into VS12: same grep surface, same confirmation site (oracle account validation plus clock comparison), two confirm modes.
- Divide-before-multiply (VS10) is split from rounding direction (VS11): different root cause, different fix, different incident anchors.
- All other requested items map one-to-one.

---

## A. Account model

```
VS1 - MISSING-SIGNER-CHECK
Severity: Critical
Grep: Signer<'info> | /// CHECK | UncheckedAccount | is_signer | next_account_info

Description:
A handler performs privileged state changes or fund movements keyed to an account that is
never required to sign. Native code: missing is_signer check. Anchor: a /// CHECK or
AccountInfo authority where a Signer<'info> belongs.

Applicability gate:
Any handler that gates on an authority, admin, or owner account.

Inventory:
- Map every account used as an authority: admin checks, owner comparisons, has_one targets.
- For each, confirm it is Signer<'info>, or has #[account(signer)], or an explicit
  require!(account.is_signer) in native code.
- In native handlers, walk the next_account_info chain and confirm is_signer before any
  privileged use.

Report only if ALL true:
- An account authorizes a state change or fund movement.
- Nothing enforces that it signed (no Signer type, no signer constraint, no is_signer check).
- A forged transaction with an arbitrary non-signing pubkey in that position executes the
  privileged path.

Do not report if:
- Anchor Signer<'info> type or signer constraint present.
- Explicit is_signer check present in native code.

Anchor: root cause class in multiple Solana exploits; standard first check in every Solana
audit methodology (Neodyme, Sec3, OtterSec public material).

---

VS2 - MISSING-OWNER-CHECK (fake account injection)
Severity: Critical
Grep: .owner | try_deserialize | AccountInfo<'info> | try_from_slice | /// CHECK

Description:
The program deserializes and trusts account data without verifying the account is owned by
the expected program. An attacker passes a fabricated account, byte-identical layout, owned
by a program they control.

Applicability gate:
Any manual deserialization or /// CHECK account whose data is read and trusted.

Inventory:
- Find all manual deserialization: try_from_slice, try_deserialize_unchecked, unpack.
- Confirm each read path checks account.owner == expected program, or uses Anchor typed
  Account<'info, T> (owner check is automatic there).
- Only raw AccountInfo / UncheckedAccount / /// CHECK paths are suspect.

Report only if ALL true:
- Account data influences logic, accounting, or fund movement.
- No owner check on that account.
- The attacker can fabricate a data-compatible account under a program they control.

Do not report if:
- Anchor typed deserialization is used everywhere on the path.
- Explicit owner check present.

Anchor: Cashio, $52M, March 2022. Infinite mint through unvalidated collateral root of
trust; canonical fake-account incident.

---

VS3 - ACCOUNT-CONFUSION (arbitrary substitution)
Severity: High
Grep: has_one | constraint = | #[account( | associated_token:: | token::mint

Description:
An account of the right type but the wrong instance is accepted: vault A's token account
swapped for vault B's, a user-supplied account where a program-canonical PDA belongs, a
position tied to the wrong owner. The missing piece is a has_one / constraint / address
equality that ties the accounts together.

Applicability gate:
Handlers taking three or more accounts that must reference each other.

Inventory:
- Build the account relationship graph per handler: vault.mint == token.mint,
  position.owner == signer.key(), token_account.owner == vault authority, and so on.
- Anchor: confirm has_one, constraint, and associated_token::* constraints encode the full
  graph.
- Native: confirm explicit key() == comparisons for every edge in the graph.

Report only if ALL true:
- Two accounts must be related for safety.
- The relationship is not constrained anywhere on the path.
- Substitution yields attacker profit or user loss (drain through wrong token account,
  claim against wrong owner, settle against wrong market).

Do not report if:
- The full constraint graph is present, or seeds tie the accounts (PDA derived from the
  other account's key).

Anchor: same family as Cashio; one of the most common findings in Solana program audits.

---

VS4 - PDA-SEED-BUMP-MISVALIDATION
Severity: High
Grep: seeds = | bump | find_program_address | create_program_address

Description:
A PDA is validated against the wrong seeds, a stored bump that was never verified, or a
noncanonical bump. Results: two valid addresses for one logical account, or a same-shape
PDA from a different context passing validation.

Applicability gate:
Any PDA usage.

Inventory:
- For each PDA, list the seeds. Every seed must be a constant or bound to a validated
  account; flag seeds taken from unvalidated input.
- Anchor: seeds = [...] with bare bump re-derives and enforces canonicality; bump = x.stored
  trusts whatever was stored at init. Check the init path verified it.
- Native: create_program_address with an attacker-supplied bump accepts noncanonical bumps.
  Confirm find_program_address or an explicit canonical-bump comparison.

Report only if ALL true:
- Seeds miss a domain separator or bind to unvalidated input, OR a noncanonical bump is
  accepted.
- The collision or substitution is exploitable: two accounts alias one vault, a wrong-user
  PDA passes, a fake "canonical" account validates.

Do not report if:
- Anchor seeds + bare bump, with seeds fully bound to validated accounts.

Anchor: PDA validation classes in Solana developer documentation and audit literature.

---

VS5 - DUPLICATE-MUTABLE-ACCOUNTS
Severity: High
Grep: #[account(mut | require_neq | require_keys_neq | key() != | remaining_accounts

Description:
The same account is passed twice in two mutable positions the handler assumes are distinct
(source and destination, market A and market B). Accounting computed against aliased data
double-counts or state written for one position overwrites the other.

Applicability gate:
Handlers with two or more mutable accounts of the same type.

Inventory:
- List same-type mutable account pairs per handler.
- Confirm require_keys_neq!, require_neq!, or an equivalent inequality constraint for every
  pair the logic assumes distinct.

Report only if ALL true:
- Two mutable positions accept the same account.
- The handler's accounting assumes distinctness.
- Concrete corruption: double-counted debit or credit, overwritten state, bypassed check.

Do not report if:
- Keys-inequality constraints present, or aliasing is harmless by construction.

Anchor: canonical duplicate-mutable-accounts example in Solana developer documentation.

---

VS6 - CLOSING-ACCOUNT-REVIVAL
Severity: High
Grep: close = | realloc | .assign( | set_lamports | close(

Description:
An account is closed (lamports drained) but not rendered unusable within the same
transaction: the discriminator survives, data is not zeroed, or a later instruction refunds
lamports. The closed account revives and passes validation in a subsequent instruction or
transaction. Also covers reinit of a just-closed account inside one transaction.

Applicability gate:
Any close path, realloc-to-zero, or manual lamport drain.

Inventory:
- Anchor close = destination zeroes data and drains at transaction end; check nothing
  refunds lamports to the closed account mid-transaction.
- Manual close: confirm the discriminator is wiped AND data zeroed (or ownership assigned
  away), not just lamports moved.
- Check realloc(0) plus drain patterns for discriminator residue.

Report only if ALL true:
- A close path leaves the account deserializable or validatable within the same transaction
  or after revival.
- A later instruction or transaction reuses it as if live.

Do not report if:
- Anchor close constraint with no mid-transaction refund, or explicit discriminator wipe
  plus zeroing.

Anchor: closing-account revival class documented by Neodyme; recurring in lending and order
book audits.

---

VS7 - CPI-PRIVILEGE-ESCALATION
Severity: Critical
Grep: invoke_signed | invoke( | CpiContext | system_instruction | spl_token::instruction

Description:
A cross-program invocation extends signer privileges (PDA seeds via invoke_signed, or the
program's own authority) to a callee that is attacker-controlled or unvalidated, or builds
seeds or the target program id from user input. The attacker routes the signed CPI through
a malicious program that spends the privilege.

Applicability gate:
Any invoke, invoke_signed, or CpiContext.

Inventory:
- For every CPI: is the target program id hardcoded or validated? Anchor Program<'info, T>
  enforces this; raw AccountInfo program accounts do not.
- For every invoke_signed: are the seeds fully program-controlled, or can caller input shift
  which PDA signs?
- Check remaining_accounts entries used as CPI programs or authorities.

Report only if ALL true:
- Signed authority reaches a program or account set the caller can influence.
- The malicious callee can use that privilege: transfer from a program vault, mint, update
  state.

Do not report if:
- Program id validated or typed, and signer seeds constant or bound to validated accounts.

Anchor: arbitrary-CPI findings across published Solana audits 2022-2025.

---

VS8 - REMAINING-ACCOUNTS-ABUSE
Severity: High
Grep: remaining_accounts | ctx.remaining_accounts

Description:
Handler logic iterates or trusts remaining_accounts without validating count, order,
ownership, or identity. The attacker passes extra, duplicated, or substituted accounts to
skip checks, double-count, or inject fakes. Anchor applies no constraints to
remaining_accounts; every entry needs manual validation.

Applicability gate:
Any use of ctx.remaining_accounts.

Inventory:
- Map how remaining accounts are indexed or paired (fixed offsets? triplets? per-market?).
- Confirm per-entry validation inside the loop: owner, PDA derivation, relationship to the
  fixed accounts.
- Confirm count checks (exact, or multiple-of-N) and no aliasing with the fixed accounts.

Report only if ALL true:
- remaining_accounts drive accounting, CPI, or transfers.
- Validation is missing on identity, owner, count, or aliasing.
- Exploitable consequence: double-counted deposit, skipped market, fake oracle or tick
  account.

Do not report if:
- Every remaining account is validated against fixed accounts or PDAs and counts are
  enforced.

Anchor: standard pattern risk in CLMM and order book programs. Raydium CLMM validates each
remaining tick array by PDA derivation; that is the reference pattern.
```

---

## B. Arithmetic

```
VS9 - ARITHMETIC-OVERFLOW
Severity: Medium (handler DoS) to High (silent corruption with wrapping enabled)
Grep: checked_add|checked_sub|checked_mul|checked_div | as u64 | as u128 | saturating_ | .unwrap()

Description:
Token math overflows u64/u128. Release BPF panics on overflow by default, so a reachable
overflow is a permanent handler DoS; if the build enables wrapping, it is silent value
corruption. Also covers narrowing casts (as u64 on a u128 intermediate) that silently
truncate.

Applicability gate:
Any arithmetic on token amounts, shares, prices, or timestamps.

Inventory:
- Find mul/add/sub on amounts: plain operators, checked_* with unwrap (panics = DoS), or
  saturating_* (silently wrong on debit paths).
- Find `as u64` / `as u128` casts on computed values; confirm the source range.
- Check Cargo.toml for overflow-checks settings.

Report only if ALL true:
- A reachable computation overflows or truncates with realistic magnitudes (quantify the
  input that triggers it).
- Consequence: permanent DoS of a path users need, or silent value corruption.

Do not report if:
- Checked math with error handling everywhere on the path, or values are structurally
  bounded below overflow.

Anchor: standard Solana arithmetic class.

---

VS10 - DIVIDE-BEFORE-MULTIPLY
Severity: Medium to High
Grep: regex (\w+)\s*/\s*[\w.()]+\s*\* | checked_div | floor

Description:
Rate or share math divides before it multiplies, so integer truncation zeroes or shrinks
the intermediate before scaling. A redemption or conversion returns 0 or dust although the
true value is nonzero. Nothing reverts; the math just answers wrong.

Applicability gate:
Any (a / b) * c shape in value math, especially rate-then-amount conversions, including the
two-statement form (rate stored after division, multiplied later).

Inventory:
- Find division-then-multiplication chains, direct or via an intermediate variable.
- Check whether .floor() or an integer cast amplifies the truncation.
- Quantify: for what input range does the result floor to zero, and is that range reachable
  by normal users?

Report only if ALL true:
- Division precedes multiplication on a value path.
- Reachable inputs produce material loss (zeroed or shrunken redemption, underpayment) or a
  downstream revert-DoS.

Do not report if:
- Multiply-first, divide-last ordering, or wide intermediates (u128) with documented,
  protocol-favorable rounding.

Anchor: LoopscaleLabs/loopscale-pricing-adapters PR #3 (2026). Meteora redemption rate
computed divide-before-multiply and floored; LP redemption under par returned zero. Fix
adopted upstream: multiply-first, divide-last.

---

VS11 - ROUNDING-DIRECTION
Severity: Medium
Grep: div_ceil | try_round | RoundUp|RoundDown | floor | ceil

Description:
Value flows round in the caller's favor where they should round in the protocol's. Deposits
must round shares down, withdrawals must round assets up, fees must round up. Wrong
direction is dust per transaction and a drain at scale, or a solvency invariant broken.

Applicability gate:
Share/asset conversions, fee computation, interest accrual, any split of a quantity.

Inventory:
- For each conversion, determine who absorbs the remainder. Invariant: the protocol keeps
  the dust; the caller can never gain by splitting transactions.
- Check mint and burn paths for floor where ceil is required and vice versa.

Report only if ALL true:
- Rounding favors the transacting user.
- Repeatable extraction exceeds transaction cost, or a solvency invariant breaks.

Do not report if:
- Dust accrues to the protocol, or the per-call gain is bounded below economic viability.

Anchor: standard vault and AMM finding class.
```

---

## C. Oracle and pricing

```
VS12 - ORACLE-TRUST (staleness + substitution)
Severity: High
Grep: pyth | switchboard | get_price_no_older_than | get_price_unchecked | pyth_solana_receiver_sdk | load_checked | PriceFeed | staleness | max_age | maximum_age | oracle_interval

Description:
The program prices from an oracle without enforcing freshness against Clock, or without
binding the oracle account to the expected feed. get_price_unchecked and a missing max_age
are the staleness mode; an unvalidated account in the feed position is the substitution
mode. One vector, two confirm modes, same confirmation site.

Applicability gate:
Any price, rate, or collateral valuation read from an oracle account.

Inventory:
- Identify every oracle read: which SDK call, what age bound, what confidence handling.
- Confirm the oracle account is constrained: address equality with the expected feed, PDA
  derivation, or has_one binding.
- Confirm freshness: get_price_no_older_than(clock, max_age) or an equivalent manual
  now - publish_time check with a sane bound.

Report only if ALL true (either mode):
- Staleness: no effective age bound, and a stale price moves value.
- Substitution: the oracle account is not bound, and an attacker-supplied feed moves value.

Do not report if:
- Freshness is enforced AND the feed account is bound.

Anchor: oracle staleness and substitution classes recur across Solana lending and perp
audits. Pyth pull receiver enforces age only through the no_older_than API.

---

VS13 - MARKET-CLOSED-PRICING
Severity: Medium to High
Grep: unix_timestamp | 86400 | weekday | day_of_week | chrono | market_hours | trading_session

Description:
An equity, forex, or commodity-linked asset is priced 24/7 from a last-trade oracle while
the underlying venue is closed (nights, weekends, holidays). The price is fresh by the
clock and stale by the market; the protocol is arbitraged at a known-bad rate until the
venue reopens.

Applicability gate:
Any mint, redeem, borrow, or settle path pricing an asset whose underlying trades on a
schedule. Niche: expect few or no grep hits outside RWA code.

Inventory:
- Is there market-hours awareness: session calendar, weekend and halt handling, or a
  conservative off-hours mode?
- What bounds the protocol when the venue is shut: wider bands, paused mint/redeem, a
  next-open reference price?

Report only if ALL true:
- The protocol prices, mints, redeems, or settles a scheduled-underlying asset around the
  clock from a spot oracle.
- No off-hours guard exists.
- A profitable arbitrage window exists (weekend gap risk against a fixed redemption rate).

Do not report if:
- Off-hours logic is present, or the asset class trades 24/7.

Anchor: hylo-so/sdk issue #136 (2026). Off-hours pricing for equity xAssets.
```

---

## D. Token program

```
VS14 - TOKEN2022-DANGEROUS-EXTENSIONS
Severity: Medium to High
Grep: token_2022 | Token2022 | TOKEN_2022 | ExtensionType | TransferHook | PermanentDelegate | ConfidentialTransfer | Pausable

Description:
A program accepts Token-2022 mints without an extension policy. Extensions change transfer
semantics: TransferFeeConfig drifts internal accounting, TransferHook can fail or callback,
PermanentDelegate can claw back user funds, Pausable can freeze flows,
ConfidentialTransferMint makes balances opaque, DefaultAccountState::Frozen bricks new
accounts. Plain-SPL assumptions silently break.

Applicability gate:
Any program accepting arbitrary or weakly-vetted Token-2022 mints as collateral, deposits,
or liquidity.

Inventory:
- Does the program read the mint's extension set, or assume plain SPL semantics?
- For each accepted mint class, check which extensions break invariants: transfer fees
  (accounting drift), hooks (DoS or reentrancy via hook program), permanent delegate
  (clawback), pausable (freeze), confidential mint (opaque balances), default-frozen
  accounts.
- Is there a whitelist or extension policy enforced in code?

Report only if ALL true:
- Arbitrary or weakly-vetted Token-2022 mints are accepted.
- At least one dangerous extension is not excluded by code.
- Concrete harm: accounting drift, stuck funds, clawback, hidden balances, bricked accounts.

Do not report if:
- An extension policy is enforced in code, or only plain spl-token mints are accepted.

Anchor: hylo-so/sdk issue #135 (2026). Collateral mint carried PermanentDelegate, Pausable,
TransferHook (null program, live authority), and ConfidentialTransferMint under an issuer
key.

---

VS15 - MINT-DECIMALS-MISMATCH
Severity: Medium
Grep: .decimals | 10u64.pow | 10_u64.pow | 1_000_000_000 | 1_000_000

Description:
The program hardcodes a decimal assumption (6 or 9) or combines raw amounts from two mints
without scaling by the decimals difference. Deposits, withdrawals, and prices misvalue by
orders of magnitude for nonstandard mints.

Applicability gate:
Any math combining amounts from two mints, or price and rate scaling.

Inventory:
- Find hardcoded scaling constants (1e6, 1e9, 10^9); check them against the actual
  mint.decimals of every accepted mint.
- Find cross-mint math without decimal normalization.

Report only if ALL true:
- A mint is accepted whose decimals differ from the assumption.
- The misvaluation is material and reachable.

Do not report if:
- Decimals are read from the mint account and used for all scaling, or the mint set is
  fixed and verified.

Anchor: recurring Solana finding class wherever USDC (6) and SOL-wrapped or custom (9)
mint assumptions mix.
```

---

## E. Logic and economics

```
VS16 - BOUNDARY-COMPARISON
Severity: Medium
Grep: require_gte! | require_lte! | require_gt! | require_lt! | require!( | PostOnly | post_only

Description:
Strict versus inclusive comparison mismatch at a threshold between two components: wrapper
versus core, precheck versus execution. Values at exact equality pass one side and fail the
other, causing reverts, missed fills, or unintended execution at the boundary.

Applicability gate:
Any threshold logic mirrored in two places, price or tick comparisons, limit checks,
post-only or reduce-only filters.

Inventory:
- Find paired comparisons on the same quantity in different modules; check operator
  agreement (> and < versus >= and <=).
- Check revert-on-boundary paths inside batched instructions: one boundary leg fails the
  whole transaction.

Report only if ALL true:
- Comparison semantics differ at equality between check and execution, or wrapper and core.
- The boundary value is naturally reachable: exact top of book, exact threshold, tick
  boundary.
- Consequence: transaction revert (batched operations lost), stuck state, or unintended
  match.

Do not report if:
- A single source of truth owns the comparison, or equality is handled identically on both
  sides.

Anchor: Bonasa-Tech/manifest PR #738 discussion (2026). Wrapper PostOnly filter used strict
inequalities while core matching executes at exact equality; a PostOnly order at exact
top-of-book reverted the entire batch.

---

VS17 - FEE-ROUTING-CONFUSION
Severity: Medium to High
Grep: fee_account | fee_recipient | fee_destination | treasury | referr | builder_fee | builder | transfer( | transfer_checked(

Description:
Fee credit or debit destinations resolve from the wrong field, a field shared across two
roles, or a user-supplied account with no binding. Fees are credited to the wrong party,
self-referral captures protocol fees, or fee accounts are swapped between roles.

Applicability gate:
Any fee, referral, commission, or builder-fee accounting with two or more destination
roles.

Inventory:
- Map every fee flow: who is credited, which account field supplies the destination, who
  can set that field.
- Check for one field used in two roles (referrer and builder, payer and beneficiary) and
  for self-dealing bindings (payer == fee recipient).

Report only if ALL true:
- A fee destination resolves to an unintended or attacker-chosen party.
- Value actually moves; not accounting cosmetics.

Do not report if:
- Destinations are bound by constraint or PDA, and role fields are distinct.

Anchor: gmsol-labs/gmx-solana issues #416 and #406 (2026). Builder-fee routing credited the
wrong party, self-referral class; fix and regression tests in PR #447.

---

VS18 - PERMISSIONLESS-CRANK-GRIEFING
Severity: Medium
Grep: crank | keeper | fn settle | fn liquidate | fn expire | fn resolve | permissionless

Description:
A permissionless crank, settle, liquidate, or expire instruction is invoked to grief:
forced settlement at bad prices, premature expiry, attacker-chosen ordering across cranked
accounts, or spam that blocks dependent user actions.

Applicability gate:
Any handler intentionally callable by anyone that advances protocol state.

Inventory:
- List permissionless state-advancing handlers. What protects victims: price bounds, time
  gates, grace periods, victim consent?
- Can the crank be called with attacker-chosen subsets or orderings (via remaining_accounts)
  to skip protections?

Report only if ALL true:
- The crank is callable by anyone with attacker-chosen timing or subset.
- A victim loses value or a needed action is durably blocked; beyond pure gas costs.

Do not report if:
- Crank effects are idempotent and victim-neutral, or bounded by oracle and time guards.

Anchor: keeper-griefing class in perp and lending audits.

---

VS19 - VAULT-SHARE-INFLATION
Severity: High
Grep: total_shares | total_assets | total_deposits | lp_supply | shares | lp_token_amount | lp_mint

Description:
First-deposit or empty-pool share-price manipulation. The attacker deposits dust, donates
directly to the vault to inflate the exchange rate, and later depositors' share
calculations round to zero; their funds accrue to the attacker.

Applicability gate:
Any vault, pool, or lending market issuing shares or LP tokens against deposited assets.

Inventory:
- First-deposit path: is there a minimum liquidity lock, dead shares, or a virtual
  shares/assets offset?
- Donation surface: can tokens reach the vault outside the deposit instruction (direct
  transfer to the vault token account)?
- Rounding: do share calculations floor in the protocol's favor on deposit?

Report only if ALL true:
- The exchange rate is manipulable by the first depositor or a donator.
- A later depositor loses material value to the manipulation.

Do not report if:
- Minimum-liquidity lock, virtual offsets, or share math makes the attack unprofitable.

Anchor: ERC-4626 inflation attack class; applies identically to Solana vault and LP share
math.

---

VS20 - SIGNER-PAYER-CONFUSION
Severity: High
Grep: payer: Signer | pub payer | has_one = authority | Signer<'info>

Description:
The handler uses the fee payer as the authorization identity when payer and owner differ
(relayers, sponsored transactions, bundled instructions). Anyone paying becomes the
authority, or the owner's intent is bypassed by having someone else submit.

Applicability gate:
Handlers with both payer and authority/owner semantics, init paths, sponsored or relayed
flows.

Inventory:
- For each privileged action: which account authorizes? Flag any path where payer stands in
  for authority.
- Check init paths where payer becomes the stored authority by default.

Report only if ALL true:
- Authorization identity collapses to the payer (or the reverse).
- A third party can sponsor or relay to satisfy the check without the resource owner's key.

Do not report if:
- A distinct authority signer with has_one binding authorizes the action.

Anchor: live pattern in 2026 codebases with sponsored transactions and relayer flows.

---

VS21 - SYSVAR-SUBSTITUTION
Severity: Medium
Grep: sysvar | Sysvar | next_account_info | from_account_info | load_instruction_at | load_current_index

Description:
A sysvar or instructions account is taken from the accounts list without checking it is
the real sysvar address. The attacker passes a fabricated account with forged rent, clock,
or instruction data to defeat checks. Includes introspection flows that read the
instructions sysvar without an id check.

Applicability gate:
Native code reading sysvars from the accounts list; any instruction introspection.

Inventory:
- Confirm each sysvar account is validated against sysvar::<name>::id(), or accessed via
  Anchor typed Sysvar<'info, T> (id enforced).
- For introspection: confirm the instructions sysvar id is checked before
  load_instruction_at.

Report only if ALL true:
- An unvalidated sysvar or instructions account is read.
- Forged data defeats a check or changes economics.

Do not report if:
- Typed sysvar or explicit id check present on every read.

Anchor: Wormhole, $326M, February 2022. Spoofed signature-verification account in the
fake-sysvar family enabled forged guardian approvals.
```

---

## Cross-reference: required coverage

| Required item | Vector |
|---|---|
| missing signer check | VS1 |
| missing owner check (fake accounts) | VS2 |
| arbitrary account substitution / confusion | VS3 |
| PDA seed/bump misvalidation | VS4 |
| duplicate mutable accounts | VS5 |
| closing-account revival/reinitialization | VS6 |
| CPI privilege escalation | VS7 |
| remaining_accounts abuse | VS8 |
| u64/u128 overflow | VS9 |
| divide-before-multiply | VS10 |
| rounding direction | VS11 |
| oracle staleness | VS12 (mode 1) |
| oracle account substitution | VS12 (mode 2) |
| off/market-closed pricing | VS13 |
| Token-2022 dangerous extensions | VS14 |
| mint/decimals mismatch | VS15 |
| boundary-comparison bugs | VS16 |
| fee routing/destination confusion | VS17 |
| permissionless crank griefing | VS18 |
| vault share-price inflation | VS19 |
| signer-vs-payer confusion | VS20 |
| sysvar substitution | VS21 |
