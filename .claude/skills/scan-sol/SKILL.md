---
name: scan-sol
description: Double-pass vulnerability scanner for Solana and Anchor (Rust) programs. Detects 24 vectors (VS1-VS24) across account model, arithmetic, oracle, token, and economic classes with parallel agent analysis, then red-teams every finding. Scope is auto-detected; findings reported by severity and confidence.
---

# Solana Vulnerability Scanner

Scan Solana and Anchor (Rust) programs for security vulnerabilities. Works on single programs, workspaces, and codebases that integrate with other Solana programs. Detects what's in scope automatically based on what's present.

## Scope

Target: the first non-flag token of `$ARGUMENTS` (defaults to current working directory if none). `with-tests` and `include-sdk` are flags and may appear in any position; they are never the target. Scan all `.rs` files under the target, **always excluding** `target/` directories. By default also exclude `tests/`, `test/`, and `benches/` directories; with the `with-tests` flag, include them. With the `include-sdk` flag, also bundle `.ts` and `.js` source (excluding `node_modules/`, `dist/`, `build/`, `coverage/`). Use `include-sdk` for Solana codebases where SDK, client, or pricing-adapter code carries security-relevant math alongside the on-chain program.

## Workflow

### Step 0 - Banner

Before doing anything else, print this banner exactly as shown:

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

 ◈ Double-pass audit engine for Solana and Anchor programs
 ◈ 24 vectors ∙ 5 classes ∙ parallel analysis ∙ red-teamed findings
```

### Step 1 - Prepare

1. Glob for all in-scope `.rs` files (plus `.ts`/`.js` when `include-sdk` was passed).
2. Count total lines across all files. If zero, stop and tell the user no source files were found.
3. Concatenate all in-scope files into `/tmp/sol-scan-bundle.rs` using bash, with `// === FILE: <path> ===` separators between each file. (The `.rs` name is a convention; the bundle may hold mixed languages under `include-sdk`. When scanning multiple targets in one session, use a unique bundle name per target.) Record the bundle's total line count including separators.
4. Record build notes for the agents (`BUILD_NOTES`): project shape (presence of `Anchor.toml`, `#[program]` macros, and `declare_id!` calls indicate Anchor; raw `entrypoint!` / `process_instruction` indicates native; both can coexist) and any `overflow-checks` settings found in Cargo.toml files.

### Step 2 - Double Pass (parallel)

Launch **both** agents simultaneously using two Task tool calls in a single message. Both use `subagent_type: "general-purpose"` and `model: "sonnet"`. For maximum depth on high-value codebases, switch both to `model: "opus"`.

---

**Agent A - Vector Scan**

Prompt the agent with the Vector Scan Agent Prompt below. Interpolate:
- `{BUNDLE_PATH}` -> `/tmp/sol-scan-bundle.rs`
- `{LINE_COUNT}` -> the bundle's total line count including separators
- `{BUILD_NOTES}` -> the build notes from prepare step 4
- `{VECTORS}` -> the full "Solana Vulnerability Vectors" section below (copy it verbatim into the prompt)
- `{FP_GATE}` -> the "FP Gate" section below
- `{REPORT_FORMAT}` -> the "Report Format" section below

**Agent B - Adversarial Reasoning**

Prompt the agent with the Adversarial Reasoning Agent Prompt below. Interpolate:
- `{FILE_LIST}` -> the list of in-scope `.rs` file paths, one per line
- `{FP_GATE}` -> the "FP Gate" section below
- `{REPORT_FORMAT}` -> the "Report Format" section below

---

### Step 3 - Merge

1. Collect findings from both agents.
2. Deduplicate: if both agents found the same issue (same affected handler + same root cause), keep the higher-confidence version.
3. Re-number findings sequentially.
4. Sort by confidence descending.

### Step 4 - Refutation Pass (red team)

If the merge produced zero findings, skip to Step 5. Otherwise launch one Red Team agent using a Task tool call (`subagent_type: "general-purpose"`, `model: "sonnet"`). Prompt it with the Refutation Agent Prompt below. Interpolate:
- `{FINDINGS}` -> the numbered, merged findings from Step 3 (full text of each)
- `{BUNDLE_PATH}` -> `/tmp/sol-scan-bundle.rs`
- `{FP_GATE}` -> the "FP Gate" section below

Every finding is attacked before it is reported. Verdicts: CONFIRMED, WEAKENED, or KILLED.

### Step 5 - Report

1. Apply the verdicts:
   - CONFIRMED findings keep their position and gain a `Red-teamed: confirmed` line naming what the red team tried.
   - WEAKENED findings are re-severitied and re-confidenced as the red team directs, with a `Red-teamed: weakened` line giving the reason.
   - KILLED findings move to the appendix.
2. Present the final report with a summary header:
   - Files scanned: N
   - Lines analyzed: N
   - Findings: N (breakdown by severity)
   - Red team: N confirmed, N weakened, N killed
3. Appendix - "Rejected by red team": one line per killed finding (title + kill reason). Include the appendix even when empty kills happen; never silently drop.
4. If no findings survived the refutation: "No confirmed findings - all candidates were rejected by the red team (see appendix)." If there were no candidates at all: "No findings - the scanned code passed all Solana vector checks and adversarial analysis."

---

## Solana Vulnerability Vectors

Full vector definitions are in [references/vectors.md](references/vectors.md). Read that file and paste its full content verbatim wherever `{VECTORS}` is interpolated.

---

## FP Gate (3 Checks)

Every potential finding MUST pass all three checks before being reported. If any check fails, DROP immediately.

1. **Concrete path**: Can you trace a specific transaction (or transaction sequence) from an external entry point to a harmful state change or fund loss? Name the exact handler and account positions. For missing-validation bugs (no attacker required), the path can be: unsafe configuration or input class -> normal usage -> silent incorrect result. Not theoretical.
2. **Reachable**: Is the path actually reachable given signer checks, constraints, owner checks, access control, and state prerequisites? If fully guarded, DROP.
3. **Impact**: Do users lose funds, does an attacker profit, or is a core invariant broken? Pure inconvenience without fund risk -> DROP (unless permanent DoS of core functionality).

---

## Report Format

Each finding uses this exact format:

```
<severity> **<N>. <title>**
<file>:<lines> · Confidence: <0-100>
**Red-teamed:** <confirmed | weakened> - <one line: what the red team tried>

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

---

## Vector Scan Agent Prompt

```
You are a security auditor scanning Solana and Anchor (Rust) programs for vulnerabilities. The codebase may contain on-chain programs, client/SDK code, integration code that CPIs into other programs, or a mix. There are bugs here - find them.

CRITICAL OUTPUT RULE: Return findings ONLY in your final text response. Do NOT write any files.

BUILD NOTES: {BUILD_NOTES}

WORKFLOW:
1. Read the bundle file at {BUNDLE_PATH} in parallel 1000-line chunks on your first turn. Total lines: {LINE_COUNT}. Compute offsets and issue all Read calls at once. These are your ONLY file reads. Configuration facts come from BUILD NOTES above, not from extra reads.

2. TRIAGE PASS. For each vector (VS1-VS24), classify into Skip / Borderline / Survive:
   - Skip: the construct AND underlying concept are both absent from this codebase.
   - Borderline: no direct match but the concept could manifest differently. 1-sentence check: name the specific handler where it manifests AND describe the exploit. Promote only if both are concrete, otherwise drop.
   - Survive: the construct or pattern is clearly present.
   Output all three tiers. Every vector in exactly one. End with total count to verify.

3. DEEP PASS. Only surviving vectors. Structured one-liners:
   VS#: path: handler() -> fn() -> vulnerable line | guard: <guards> | verdict: CONFIRM [confidence] or DROP (reason)
   Trace the call chain from the external instruction entry point to the vulnerable line. Check every constraint, signer check, owner check, and state guard. If no match or fully guarded -> DROP in one line. If match -> apply FP Gate (3 checks). Only if all 3 pass -> expand into formatted finding.
   Budget: 1 line per drop, 3 lines max per confirm before formatted finding.

4. COMPOSABILITY CHECK. If 2+ confirmed: do any compound? Note in the higher-confidence finding.

5. HARD STOP. After the deep pass, do not re-examine eliminated vectors or revisit anything. Return ALL formatted findings, or "No findings."

{VECTORS}
{FP_GATE}
{REPORT_FORMAT}
```

---

## Adversarial Reasoning Agent Prompt

```
You are an adversarial security researcher trying to exploit these Solana programs. The codebase may contain on-chain programs, client/SDK code, integration code, or a mix. There are bugs here - find them. Your goal is to find every way to steal funds, lock funds, grief users, or break invariants. Do not give up. If your first pass finds nothing, assume you missed something and look again from a different angle.

CRITICAL OUTPUT RULE: Return findings ONLY in your final text response. Do NOT write any files.

SOLANA KNOWN HAZARDS (keep in mind while reading):
- Anchor typed accounts enforce owner and discriminator; /// CHECK accounts enforce nothing
- A noncanonical bump gives one logical PDA a second valid address
- Account closure only settles at transaction end; mid-transaction revival is possible
- invoke_signed extends PDA privilege to whatever program the caller routes it through
- remaining_accounts carry no Anchor constraints; every entry needs manual validation
- Release builds wrap silently on overflow by default; overflow-checks = true turns it into a panic (handler DoS). Both modes are exploitable
- Divide-before-multiply truncates to zero; floor() amplifies it
- Rounding that favors the caller is extractable at scale
- Pyth pull oracles enforce freshness only through the no_older_than API
- Token-2022 extensions change transfer semantics; plain-SPL assumptions break
- Permissionless cranks can be timed and ordered against victims
- First deposits set vault share prices; direct-transfer donations manipulate them
- The fee payer is not the authority in sponsored or relayed transactions
- Sysvar and instructions accounts must be id-checked; Wormhole ($326M) was this
- Oracle freshness is not market-hours awareness; a fresh price on a closed market is stale

SILENT MISCONFIGURATIONS (no attacker required - missing validation that quietly produces wrong results):
These bugs have NO obvious failure signal. Nothing reverts. Nothing breaks. The math just gives wrong answers for a class of inputs nobody rejects. Look for these even when there is no adversary to reason about:
- Token-2022 collateral accepted with no extension policy: delegate, pause, hook, and confidentiality semantics ride along invisibly.
- Decimal assumptions hardcoded to 6 or 9: nonstandard mints misprice without any error.
- Divide-before-multiply rates that floor to zero for legitimate redemptions: normal usage, wrong answers.
- Off-hours pricing of equity or RWA assets: oracle is fresh, market is closed, arbitrage window is open.
- Stored bump never revalidated: old PDAs keep passing after seed layouts change.
- Zero or extreme numeric init parameters: divisibility checks pass at zero, so a market comes up with zero lot sizes or zero tick sizes and drains whoever trades on it.
- First-deposit vault with no minimum liquidity: the trap waits for a victim, not an attacker.

WORKFLOW:
1. Read ALL in-scope files in a single parallel batch:
{FILE_LIST}

2. Determine what you're looking at:
   - Anchor (#[program], declare_id!, Accounts structs) or native (entrypoint!, process_instruction)?
   - Single program or workspace? Which programs CPI into which?
   - What is the trust surface: oracles, token mints, admin keys, external programs?

3. Map the attack surface:
   - Every instruction handler and its Accounts struct (or account list in native code)
   - Every CPI: target program, signed seeds, accounts passed
   - Every oracle read, every token transfer, every PDA derivation

4. Reason adversarially about every handler. Ask:
   - What if this account is substituted with a fake (wrong owner, wrong instance, duplicated)?
   - What if this crank or settle is called right now, with this subset of accounts?
   - What if this transaction is bundled with others, or relayed by someone else?
   - Who pays versus who authorizes - can those be different keys?
   - What happens at exact equality on this threshold?
   - Does this math floor to zero for small but legitimate inputs?
   - Which Token-2022 extensions would break this transfer or this accounting?
   - Is this oracle fresh, bound to the right feed, and meaningful right now (market hours)?
   - Can the same account appear twice in this account list?
   - What survives this close, realloc, or lamport drain inside the same transaction?

5. For each potential finding, apply the FP Gate. If any check fails -> drop. Only if all 3 pass -> format.

6. Return ALL formatted findings, or "No findings."

{FP_GATE}
{REPORT_FORMAT}
```

---

## Refutation Agent Prompt

```
You are a hostile senior auditor reviewing candidate findings from two junior scanners. Your only job is to KILL findings. You are not looking for new bugs. Do not write any files.

CANDIDATE FINDINGS:
{FINDINGS}

WORKFLOW:
1. For each finding, read ONLY the cited code region in {BUNDLE_PATH} (the cited file plus lines around the citation, and any directly named callees or callers). No broad reads.

2. Attempt to falsify the finding on exactly one of the three FP-gate checks:
   - Concrete path: show a step in the attack path that is impossible, misread, or misstated.
   - Reachable: name the specific guard (signer check, constraint, owner check, state precondition) that stops the path.
   - Impact: show that nobody loses funds, no attacker profits, and no core invariant breaks.

3. Verdict per finding:
   - CONFIRMED: you tried to kill it and failed. State what you tried (one line).
   - WEAKENED: part of the claim fails (wrong severity, overstated impact, narrower preconditions). State the corrected severity/confidence and why.
   - KILLED: state the exact guard or logic that defeats it, with file:line.

4. Be fair: kill only on evidence you can cite, not on priors. If the finding is real, confirm it.

OUTPUT FORMAT (nothing else):
FINDING <N>: <verdict> | <2-4 lines of reasoning with file:line evidence>

{FP_GATE}
```
