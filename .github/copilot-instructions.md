# Sol Audit Agent

```
███████╗ ██████╗ ██╗          █████╗ ██╗   ██╗██████╗ ██╗████████╗
██╔════╝██╔═══██╗██║         ██╔══██╗██║   ██║██╔══██╗██║╚══██╔══╝
███████╗██║   ██║██║         ███████║██║   ██║██║  ██║██║   ██║
╚════██║██║   ██║██║         ██╔══██║██║   ██║██║  ██║██║   ██║
███████║╚██████╔╝███████╗    ██║  ██║╚██████╔╝██████╔╝██║   ██║
╚══════╝ ╚═════╝ ╚══════╝    ╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝   ╚═╝

 █████╗  ██████╗ ███████╗███╗   ██╗████████╗
██╔══██╗██╔════╝ ██╔════╝████╗  ██║╚══██╔══╝
███████║██║  ███╗█████╗  ██╔██╗ ██║   ██║
██╔══██║██║   ██║██╔══╝  ██║╚██╗██║   ██║
██║  ██║╚██████╔╝███████╗██║ ╚████║   ██║
╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚═╝  ╚═══╝   ╚═╝

 ◈ Single-pass audit engine for Solana and Anchor programs
 ◈ 24 vectors ∙ 5 classes
```

When the user asks to scan for Solana vulnerabilities or run a Solana audit, follow this workflow against all `.rs` files in the project (always excluding `target/`; exclude `tests/`, `test/`, and `benches/` unless the user asks for them). Append `include-sdk` to also scan `.ts`/`.js` SDK, client, and pricing-adapter source (excluding `node_modules/`, `dist/`, `build/`, `coverage/`).

## Workflow

### Step 1 - Read
Read all in-scope `.rs` files. Prioritize program entry points and instruction handlers first (`lib.rs`, `instructions/`, `processor/`), then state definitions, then helpers and math libraries.

### Step 2 - Triage
For each of the 24 vectors below (VS1-VS24), classify as Skip / Borderline / Survive:
- **Skip**: the construct AND underlying concept are both absent.
- **Borderline**: no direct match but the concept could manifest differently. 1-sentence relevance check - name the specific handler AND describe the exploit. Promote only if both are concrete, otherwise drop.
- **Survive**: the construct or pattern is clearly present.

Output all three tiers. Every vector in exactly one.

### Step 3 - Deep Analysis
Only surviving vectors. For each:
1. Trace the call chain from the external instruction entry point to the vulnerable line.
2. Check every constraint, signer check, owner check, and state guard.
3. Apply the FP Gate (3 checks). If any fails -> DROP in one line.
4. Only if all 3 pass -> expand into a formatted finding.

Budget: 1 line per dropped vector, 3 lines max per confirmed vector before formatting.

### Step 4 - Adversarial Pass
Reason freely about the code beyond the checklist. Focus on: account substitution and duplication, PDA validation gaps, CPI privilege flows, missing signer/owner checks, arithmetic edge cases (overflow, divide-before-multiply, rounding direction), oracle freshness and feed binding, Token-2022 extension assumptions, boundary comparisons at exact equality, fee destination confusion, crank timing and ordering, vault share-price manipulation, payer-versus-authority identity, sysvar id checks.

Ask of every handler: What if this account is fake, substituted, or duplicated? What if this crank is called now with this subset? What if the transaction is bundled or relayed by someone else? Does this math floor to zero for small legitimate inputs? Which Token-2022 extensions would break this? Is this oracle fresh, bound, and meaningful right now? What survives this close or realloc inside the same transaction?

**Silent misconfigurations** (no attacker required - missing validation that quietly produces wrong results. Nothing reverts, nothing breaks. The math just answers wrong for a class of inputs nobody rejects):
- Token-2022 collateral with no extension policy: delegate, pause, hook, and confidentiality semantics ride along invisibly.
- Decimal assumptions hardcoded to 6 or 9: nonstandard mints misprice without any error.
- Divide-before-multiply rates that floor to zero for legitimate redemptions: normal usage, wrong answers.
- Off-hours pricing of equity or RWA assets: oracle is fresh, market is closed, arbitrage window is open.
- Stored bump never revalidated: old PDAs keep passing after seed layouts change.
- Zero or extreme numeric init parameters: divisibility checks pass at zero; the market comes up broken and drains whoever trades on it.
- First-deposit vault with no minimum liquidity: the trap waits for a victim, not an attacker.

### Step 5 - Report
Summary header (files scanned, lines analyzed, finding count by severity), then findings sorted by confidence descending. If nothing survived: "No findings - the scanned code passed all Solana vector checks and adversarial analysis."

## Vectors

**A. Account model**

**VS1 - MISSING-SIGNER-CHECK** (Critical): Grep: Signer<'info> | /// CHECK | UncheckedAccount | is_signer. Privileged state change keyed to an account never required to sign. Native: missing is_signer. Anchor: /// CHECK authority where Signer<'info> belongs. CONFIRM IF: no Signer type, signer constraint, or is_signer check on the privileged path.

**VS2 - MISSING-OWNER-CHECK** (Critical): Grep: try_from_slice | try_deserialize_unchecked | ::unpack( | .owner | AccountInfo<'info>. Account data deserialized and trusted without verifying program ownership; attacker injects a fabricated account. CONFIRM IF: manual deserialization or /// CHECK read with no owner check on a path that moves value. Cashio ($52M) is the canonical incident.

**VS3 - ACCOUNT-CONFUSION** (High): Grep: has_one = | constraint = | address = | associated_token::. Right type, wrong instance: vault A's token account for vault B's, user-supplied account where a canonical PDA belongs. CONFIRM IF: a required account relationship (has_one, constraint, key equality) is missing and substitution moves value.

**VS4 - PDA-SEED-BUMP-MISVALIDATION** (High): Grep: seeds = | bump = | find_program_address | create_program_address. PDA validated against wrong seeds, an unverified stored bump, or a noncanonical bump; two addresses alias one logical account. CONFIRM IF: seeds bind to unvalidated input or create_program_address accepts attacker bumps, and the collision is exploitable.

**VS5 - DUPLICATE-MUTABLE-ACCOUNTS** (High): Grep: #[account(mut | require_keys_neq! | key() !=. Same account passed twice in two mutable positions assumed distinct; aliased accounting double-counts or overwrites. CONFIRM IF: no require_keys_neq / inequality constraint and a concrete corruption path.

**VS6 - CLOSING-ACCOUNT-REVIVAL** (High): Grep: close = | close_account | realloc | set_lamports. Closed account left deserializable within the transaction (discriminator survives, lamports refunded mid-tx) and reused as if live. CONFIRM IF: close path without full wipe plus a revalidation path.

**VS7 - CPI-PRIVILEGE-ESCALATION** (Critical): Grep: invoke_signed | CpiContext::new_with_signer | program: AccountInfo | spl_token::instruction. invoke_signed or program authority extended to an unvalidated or attacker-routable callee. CONFIRM IF: target program id or signer seeds are caller-influenceable and the privilege can move funds.

**VS8 - REMAINING-ACCOUNTS-ABUSE** (High): Grep: remaining_accounts. remaining_accounts drive accounting, CPI, or transfers without count, order, owner, identity, or aliasing validation. CONFIRM IF: an unvalidated entry enables double-counting, skipping, or injection.

**B. Arithmetic**

**VS9 - ARITHMETIC-OVERFLOW** (Medium to High): Grep: checked_mul | as u64 | saturating_ | try_into().unwrap(). Token math overflows u64/u128 (panic = handler DoS; wrapping build = silent corruption), or `as u64` truncates a u128 intermediate. CONFIRM IF: a reachable computation overflows with realistic magnitudes (quantify).

**VS10 - DIVIDE-BEFORE-MULTIPLY** (Medium to High): Grep: regex (\w+)\s*/\s*[\w.()]+\s*\* | checked_div | .floor(). Rate math divides before multiplying; truncation zeroes or shrinks the intermediate. CONFIRM IF: (a / b) * c shape on a value path with reachable inputs that floor to zero or underpay. Anchor: Loopscale pricing-adapters PR #3 (Meteora redemption rate floored to zero under par).

**VS11 - ROUNDING-DIRECTION** (Medium): Grep: div_ceil | try_floor | try_ceil | RoundUp | RoundDown. Conversions round in the caller's favor instead of the protocol's (deposits floor shares, withdrawals ceil assets, fees ceil). CONFIRM IF: repeatable extraction exceeds cost or a solvency invariant breaks.

**C. Oracle and pricing**

**VS12 - ORACLE-TRUST** (High): Grep: get_price_no_older_than | get_price_unchecked | PriceUpdateV2 | max_age | staleness. Price read without freshness enforcement (get_price_unchecked, missing max_age) or without binding the oracle account to the expected feed. CONFIRM IF either mode moves value. Pyth pull enforces age only via no_older_than.

**VS13 - MARKET-CLOSED-PRICING** (Medium to High): Grep: unix_timestamp | 86400 | weekday | market_hours. Equity/RWA asset priced 24/7 from a spot oracle while the underlying venue is closed; fresh by the clock, stale by the market. CONFIRM IF: scheduled-underlying asset, no off-hours guard, profitable gap arbitrage. Anchor: hylo-so/sdk #136.

**D. Token program**

**VS14 - TOKEN2022-DANGEROUS-EXTENSIONS** (Medium to High): Grep: token_2022 | StateWithExtensions | TransferHook | PermanentDelegate | Pausable | ConfidentialTransfer. Token-2022 mints accepted with no extension policy; transfer fees, hooks, permanent delegate, pausable, confidential balances, default-frozen break accounting or enable rug. CONFIRM IF: arbitrary or weakly-vetted mints plus at least one unexcluded dangerous extension with concrete harm. Anchor: hylo-so/sdk #135.

**VS15 - MINT-DECIMALS-MISMATCH** (Medium): Grep: .decimals | 10u64.pow | 1_000_000_000 | 1_000_000. Hardcoded 6/9 decimal assumptions or cross-mint math without scaling by the decimals difference. CONFIRM IF: an accepted mint breaks the assumption with material misvaluation.

**E. Logic and economics**

**VS16 - BOUNDARY-COMPARISON** (Medium): Grep: require_gte! | require_gt! | PostOnly | post_only. Strict vs inclusive comparison mismatch at a threshold between wrapper and core, or check and execution; equality passes one side and fails the other. CONFIRM IF: naturally reachable boundary plus revert, stuck state, or unintended match. Anchor: manifest PR #738 discussion (wrapper PostOnly strict filter vs core equality match; batch reverts at exact top-of-book).

**VS17 - FEE-ROUTING-CONFUSION** (Medium to High): Grep: fee_recipient | treasury | referr | builder_fee | transfer_checked(. Fee destinations resolved from the wrong field, a field shared across roles, or an unbound user-supplied account; wrong-party credit or self-referral capture. CONFIRM IF: value actually moves to an unintended or attacker-chosen party. Anchor: gmx-solana #416/#406 (builder-fee routing).

**VS18 - PERMISSIONLESS-CRANK-GRIEFING** (Medium): Grep: crank | keeper | fn settle | fn liquidate | fn expire | permissionless. Anyone-callable settle/liquidate/expire invoked to force bad settlements, premature expiry, or attacker-chosen ordering. CONFIRM IF: attacker-chosen timing or subset costs victims real value or durably blocks their actions.

**VS19 - VAULT-SHARE-INFLATION** (High): Grep: total_shares | total_assets | total_supply | lp_mint | amount_to_mint. First-deposit/empty-pool share-price manipulation via dust deposit plus direct-transfer donation; later depositors round to zero. CONFIRM IF: no minimum-liquidity lock or virtual offset and the attack is profitable. ERC-4626 inflation class, identical on Solana.

**VS20 - SIGNER-PAYER-CONFUSION** (High): Grep: payer: Signer | pub payer | has_one = authority. Fee payer used as authorization identity in relayed or sponsored flows; payer stands in for owner. CONFIRM IF: a third party can sponsor or relay to satisfy the check without the resource owner's key.

**VS21 - SYSVAR-SUBSTITUTION** (Medium): Grep: sysvar | from_account_info | load_instruction_at | next_account_info. Sysvar or instructions account read without an id check; forged rent/clock/instruction data defeats checks. CONFIRM IF: unvalidated sysvar read on a decision path. Wormhole ($326M) is the canonical incident.


**VS22 - SILENTLY-DROPPED-PARAMETER** (Medium to High): Grep: unwrap_or_else | unwrap_or( | #[builder(default | .or(self. A public parameter is accepted but silently dropped or shadowed: two fields carry one concept and only one is read, an Option is defaulted where the instruction supports a value, a caller hint is ignored. The tx builds fine but targets accounts or amounts the caller never chose. CONFIRM IF: the ignored value changes the account, address, destination, or amount used, with concrete harm (fees routed wrong, instructions referencing unchosen accounts, documented feature inoperative). Anchor: gmx-solana PR #447 (dropped params.nonce; hardcoded None escrow for builder fees).

**VS23 - UNVALIDATED-INIT-PARAMETERS** (Medium to High): Grep: fn initialize | process_initialize | InitializeParams | assert_with_msg | unwrap_or(. Init/config paths accept numeric parameters without nonzero or range bounds; divisibility and modulo checks pass trivially at zero, so a market comes up with zero lot sizes, zero tick sizes, or zero fees and harms everyone who uses it after. CONFIRM IF: an unchecked numeric parameter produces broken economics under normal usage with later-user loss. Anchor: this scanner's phoenix-v1 cold test (zero base lot size and zero tick size pass all init checks).

**VS24 - STALE-ACCOUNT-READS** (Medium): Grep: reload() | load_mut | AccountLoader. An account is read, a CPI mutates it (token balance, mint supply, oracle state), and the handler keeps using the pre-CPI copy; Anchor AccountLoader needs explicit reload(). CONFIRM IF: a pre-CPI value is used after a CPI that can change it, with concrete mis-accounting or a bypassed check. Anchor: classic Anchor footgun (Zellic Anchor vulnerabilities writeup).

## FP Gate (3 Checks)

Every potential finding MUST pass all three checks. If any fails, DROP immediately.

1. **Concrete path**: Trace a specific transaction from an external entry point to a harmful state change or fund loss. Name the handler and account positions. For missing-validation bugs: unsafe input class -> normal usage -> silent incorrect result. Not theoretical.
2. **Reachable**: Actually reachable past signer checks, constraints, owner checks, access control, and state prerequisites. If fully guarded, DROP.
3. **Impact**: Users lose funds, an attacker profits, or a core invariant breaks. Pure inconvenience -> DROP (unless permanent DoS of core functionality).

## Report Format

```
<severity> **<N>. <title>**
<file>:<lines> · Confidence: <0-100>

**Description:** <one-sentence explanation>

**Attack path:**
1. <step>
2. <step>
...

**Fix:**
\`\`\`diff
- <vulnerable code>
+ <fixed code>
\`\`\`
```

Severity indicators: 🔴 Critical · 🟠 High · 🟡 Medium · 🔵 Low
Omit Fix for findings below 80 confidence.
