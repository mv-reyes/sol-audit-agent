# sol-audit-agent

```
███████╗ ██████╗ ██╗          █████╗ ██╗   ██╗██████╗ ██╗████████╗
██╔════╝██╔═══██╗██║         ██╔══██╗██║   ██║██╔══██╗██║╚══██╔══╝
███████║██║   ██║██║         ███████║██║   ██║██║  ██║██║   ██║
╚════██║██║   ██║██║         ██╔══██║██║   ██║██║  ██║██║   ██║
███████║╚██████╔╝███████╗    ██║  ██║╚██████╔╝██████╔╝██║   ██║
╚══════╝ ╚═════╝ ╚══════╝    ╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝   ╚═╝

 █████╗  ██████╗ ███████╗███╗   ██╗████████╗
██╔══██╗██╔════╝ ██╔════╝████╗  ██║╚══██╔══╝
███████║██║  ███╗█████╗  ██╔██╗ ██║   ██║
██╔══██║██║   ██║██╔══╝  ██║╚██╗██║   ██║
██║  ██║╚██████╔╝███████╗██║ ╚████║   ██║
╚═╝  ╚═╝ ╚═════╝ ╚══════╝╚═╝  ╚═══╝   ╚═╝
```

An AI agent that scans Solana and Anchor (Rust) programs for security vulnerabilities. Works on single programs, workspaces, and integration code that CPIs into other programs.

**Works with:** Claude Code · Cursor · Windsurf · GitHub Copilot

## What it does

Runs two parallel analysis agents against your Rust codebase:

| Agent | Strategy | What it catches |
|-------|----------|-----------------|
| **Vector Scan** | Systematic triage of 24 Solana vulnerability patterns | Known footguns - fast, cheap, high recall |
| **Adversarial Reasoning** | Free-form adversarial bug hunting | Novel bugs, logic errors, economic exploits the vector list doesn't cover |

Results are deduplicated, scored by confidence, and presented as a single report.

The vector taxonomy is seeded from real 2026 findings in public Solana repos: a divide-before-multiply truncation that zeroed LP redemptions (Loopscale pricing adapters), a strict-vs-inclusive boundary mismatch that reverted batched orders (manifest), a builder-fee routing bug that credited the wrong party (gmx-solana), a Token-2022 collateral extension policy gap (hylo), and off-hours pricing of equity xAssets (hylo). Classic incident classes (Cashio fake accounts, Wormhole sysvar spoofing) anchor the account-model vectors.

### Vulnerability coverage

**A. Account model**

| ID | Name | Severity |
|----|------|----------|
| VS1 | Missing signer check | Critical |
| VS2 | Missing owner check (fake account injection) | Critical |
| VS3 | Account confusion / arbitrary substitution | High |
| VS4 | PDA seed/bump misvalidation | High |
| VS5 | Duplicate mutable accounts | High |
| VS6 | Closing-account revival | High |
| VS7 | CPI privilege escalation | Critical |
| VS8 | remaining_accounts abuse | High |

**B. Arithmetic**

| ID | Name | Severity |
|----|------|----------|
| VS9 | u64/u128 overflow and truncation | Medium to High |
| VS10 | Divide-before-multiply | Medium to High |
| VS11 | Rounding direction | Medium |

**C. Oracle and pricing**

| ID | Name | Severity |
|----|------|----------|
| VS12 | Oracle staleness + feed substitution | High |
| VS13 | Market-closed pricing (equity/RWA) | Medium to High |

**D. Token program**

| ID | Name | Severity |
|----|------|----------|
| VS14 | Token-2022 dangerous extensions | Medium to High |
| VS15 | Mint decimals mismatch | Medium |

**E. Logic and economics**

| ID | Name | Severity |
|----|------|----------|
| VS16 | Boundary-comparison bugs | Medium |
| VS17 | Fee routing/destination confusion | Medium to High |
| VS18 | Permissionless crank griefing | Medium |
| VS19 | Vault share-price inflation | High |
| VS20 | Signer-vs-payer confusion | High |
| VS21 | Sysvar substitution | Medium |
| VS22 | Silently dropped parameter | Medium to High |
| VS23 | Unvalidated init parameters | Medium to High |
| VS24 | Stale account reads after CPI | Medium |

Full definitions with grep signatures and confirm-if criteria: [VECTORS.md](VECTORS.md).

## Requirements

- One of: [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Cursor](https://cursor.sh), [Windsurf](https://codeium.com/windsurf), or [GitHub Copilot](https://github.com/features/copilot)
- Rust source files (`.rs`) in the target directory. TypeScript/JavaScript SDK and adapter code can be included with the `include-sdk` flag.
- No other dependencies. Nothing to compile, nothing to install beyond the prompt files.

## Installation

### Claude Code (recommended - dual-agent parallel scan)

```bash
# Option 1: curl into your project
mkdir -p .claude/commands
curl -o .claude/commands/scan-sol.md \
  https://raw.githubusercontent.com/mv-reyes/sol-audit-agent/main/.claude/commands/scan-sol.md

# Option 2: clone and point at your code
git clone https://github.com/mv-reyes/sol-audit-agent.git
cd sol-audit-agent
claude
# Then: /scan-sol /path/to/your/program
```

### Cursor

```bash
mkdir -p .cursor/rules
curl -o .cursor/rules/scan-sol.mdc \
  https://raw.githubusercontent.com/mv-reyes/sol-audit-agent/main/.cursor/rules/scan-sol.mdc
```

Then ask Cursor: *"Scan for Solana vulnerabilities"* or *"Run Solana audit"*

### Windsurf

```bash
curl -o .windsurfrules \
  https://raw.githubusercontent.com/mv-reyes/sol-audit-agent/main/.windsurfrules
```

Then ask Windsurf: *"Scan for Solana vulnerabilities"* or *"Run Solana audit"*

### GitHub Copilot

```bash
mkdir -p .github
curl -o .github/copilot-instructions.md \
  https://raw.githubusercontent.com/mv-reyes/sol-audit-agent/main/.github/copilot-instructions.md
```

Then ask Copilot: *"Scan for Solana vulnerabilities"* or *"Run Solana audit"*

## Usage

### Claude Code (slash command)

```
/scan-sol                                # scan current directory
/scan-sol ./programs/my-program          # scan specific directory
/scan-sol /path/to/workspace             # scan absolute path
/scan-sol ./programs/my-program with-tests   # include tests/, test/, benches/
/scan-sol ./my-repo include-sdk          # also scan .ts/.js SDK and adapter code
```

### Cursor / Windsurf / Copilot (natural language)

Ask your agent:
- *"Scan this codebase for Solana vulnerabilities"*
- *"Run a Solana security audit on the program in programs/vault/"*
- *"Check this workspace for Anchor account validation bugs"*

### Platform differences

| Feature | Claude Code | Cursor / Windsurf / Copilot |
|---------|------------|----------------------------|
| Architecture | Dual-agent parallel (vector scan + adversarial) | Single-agent sequential |
| Speed | Both passes run simultaneously | One pass at a time |
| Depth | Two different cognitive strategies catch more bugs | Same vectors, single strategy |
| Token usage | Two agents, roughly 2x a single pass | Single pass |
| Invocation | `/scan-sol` slash command | Natural language prompt |

Claude Code gets the best results because it runs two agents with different analysis strategies in parallel. Cursor/Windsurf/Copilot run the same vectors as a single sequential pass.

### What happens when you run it

1. **Prepare** - Finds all `.rs` files (always excludes `target/`; excludes `tests/`, `test/`, `benches/` unless you pass `with-tests`), concatenates them into a temporary bundle with file separators, and detects Anchor vs native shape. Pass `include-sdk` to also bundle `.ts`/`.js` SDK and adapter source (excluding `node_modules/`, `dist/`, `build/`, `coverage/`).
2. **Double pass** - Launches both agents in parallel:
   - Vector Scan agent reads the bundle, triages all 24 vectors, drops irrelevant ones in 1 line each, deep-analyzes survivors.
   - Adversarial Reasoning agent reads all files, maps the instruction/account/CPI surface, and reasons adversarially about every handler.
3. **Merge** - Deduplicates findings, re-numbers, sorts by confidence, presents the report.

### Example output

From validation run 1 ([Loopscale pricing adapters @ `96dcc95`](validation/loopscale-96dcc95.md), pre-fix commit; full run in [validation/](validation/)). Finding text verbatim from the adversarial agent; the merged report deduplicates it with the vector agent's identical find:

```
📋 Solana Scan Report
Files scanned: 14
Lines analyzed: 1,487
Findings: 4 (1 High, 2 Medium, 1 Low)

🟠 **1. Meteora v1 handler truncates LP redemption rate to a whole number via BigInt scalar division**
src/pricing/meteora.ts:73-83 · Confidence: 90

**Description:** In getMeteoraTokenBalancesBn the per-unit withdraw quote is computed as
BigInt(quote.tokenAOutAmount.toString()) / quoteInput - a BigInt integer division - so the
LP-to-USDC rate is floored to an integer (e.g. 1.08 -> 1, 0.95 -> 0), silently mispricing
or entirely zeroing out the LP collateral on the v1 endpoint while
delete balances[metMint] still removes the original balance.

**Attack path:**
1. POST /v1/decompile_mints with rawBalances containing the Meteora LP mint.
2. getWithdrawQuote(1_000_000n, ...) returns tokenAOutAmount ≈ 1_080_000;
   scalar = 1080000n / 1000000n = 1n - the 8% premium is silently discarded.
3. If the pool rate drops below 1.0, scalar = 0n; the position is credited nothing while
   delete balances[metMint] removes it - no error, HTTP 200.
4. Consumers persist a silently understated valuation. The legacy float path computes the
   scalar correctly, so this is a regression introduced in the Bn port.

**Fix:**
\`\`\`diff
- const scalar = BigInt(quote.tokenAOutAmount.toString()) / quoteInput;
- const scaledOutAmount = switchBaseDecimalsBn(balance * scalar, poolDecimals, outMintDecimals);
+ const outAmount = BigInt(quote.tokenAOutAmount.toString());
+ const scaledOutAmount = switchBaseDecimalsBn((balance * outAmount) / quoteInput, poolDecimals, outMintDecimals);
\`\`\`

...
```

This is the bug Loopscale fixed upstream in commit 908b54ae (merged in PR #3, "don't floor the meteora redemption rate"). In the recall test at the pre-fix commit the scanner rediscovered it and produced the same fix that was adopted: multiply-first, divide-last. The taxonomy is seeded from this bug class (VS10), so this is a recall test, not a cold find - the cold test is the phoenix-v1 run in [validation/](validation/).

## How it stays token-efficient

- **Bundle read**: All source is concatenated into one file - agents read it in parallel chunks on turn 1, no repeated file I/O.
- **Fast triage**: 24 vectors are classified in a single pass against signatures verified on real Solana program source (hit-count table in validation/signature-tests.md). Irrelevant vectors are dropped in 1 structured line each.
- **FP gate**: Every potential finding must pass 3 checks (concrete path, reachable, impactful) before expansion. Kills false positives before they waste tokens.
- **Hard stop**: Agents do not revisit eliminated vectors or re-scan.

Token usage scales with codebase size: the whole source is read once per agent. A ~5k line program fits comfortably in a dual-agent scan; very large workspaces should be scanned per program directory.

## FP Gate

Every finding must pass all three checks or it's dropped:

1. **Concrete attack path** - Can you trace a specific transaction from an entry point to harm? Name the handler and account positions.
2. **Reachable** - Is the path actually reachable past signer checks, constraints, owner checks, and state prerequisites?
3. **Impact** - Does the attacker profit or do users lose funds? Pure inconvenience without fund risk is dropped (unless permanent DoS of core functionality).

## Customization

The skill file at `.claude/commands/scan-sol.md` is self-contained. You can:

- **Add vectors**: Add new entries to the vectors section following the existing format (ID, severity, grep signatures, description, confirm-if criteria).
- **Adjust severity**: Change the severity classification in the vector definitions.
- **Change model**: The agents default to `sonnet` for cost/capability balance. Change to `opus` in the workflow section for maximum depth (costs more).
- **Modify scope**: Edit the exclusion list in the Scope section, or pass `with-tests` to include test directories.

## References

### Provenance

- Format and workflow modeled on [33Audits/cca-audit-agent](https://github.com/33Audits/cca-audit-agent) (EVM/Solidity, Uniswap CCA). sol-audit-agent is the Solana counterpart, built from scratch for the Solana account model.

### Solana program security

- [Solana docs - Accounts](https://solana.com/docs/core/accounts)
- [Anchor docs - Account Constraints](https://www.anchor-lang.com/docs/references/account-constraints)
- [Neodyme - Breakpoint Security Workshop](https://github.com/neodyme-labs/neodyme-breakpoint-workshop)
- [Zellic - The Vulnerabilities You'll Write With Anchor](https://www.zellic.io/blog/the-vulnerabilities-youll-write-with-anchor)

### Incident anchors

- [Cashio $52M exploit (fake collateral accounts)](https://www.halborn.com/blog/post/explained-the-cashio-hack-march-2022) - VS2/VS3
- [Wormhole $326M exploit (spoofed verification account)](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022) - VS21
- [LoopscaleLabs/loopscale-pricing-adapters PR #3](https://github.com/LoopscaleLabs/loopscale-pricing-adapters/pull/3) (fix commit 908b54ae) - VS10
- [Bonasa-Tech/manifest PR #738 discussion](https://github.com/Bonasa-Tech/manifest/pull/738) - VS16
- [gmsol-labs/gmx-solana issues #416](https://github.com/gmsol-labs/gmx-solana/issues/416) and [#406](https://github.com/gmsol-labs/gmx-solana/issues/406), fix in [PR #447](https://github.com/gmsol-labs/gmx-solana/pull/447) - VS17
- [hylo-so/sdk issues #135](https://github.com/hylo-so/sdk/issues/135) and [#136](https://github.com/hylo-so/sdk/issues/136) - VS14, VS13

## License

MIT
