# Phoenix cold test - full pipeline (2026-10-01, current tool)

Target: Ellipsis-Labs/phoenix-v1 @ 5a34f7f901fd9e04057198d4fc7b7286f78b53f2 (master HEAD at clone time). Never scanned before; chosen as a heavily audited native-Rust CLOB to measure false-positive rate honestly.
Scope: `phoenix/src` (45 .rs files, 12,530 lines; tests/ excluded by default scope)
Bundle: `_validation-clones/bundles/phoenix-bundle.rs`
Pipeline: 24-vector taxonomy, BUILD_NOTES (native; overflow-checks = true), dual finder agents, merge, refutation agent.
Earlier runs (including the taxonomy-gap discovery that created VS23) are preserved in git history.

## Finder output (merged)

Vector agent: 16 Survive, 0 Borderline, 8 Skip. 1 confirm (VS23).
Adversarial agent: 4 findings (zero-param variants + three wrapping-arithmetic claims).
Merge (dedupe: both agents found the zero-param class; kept conf 88):

1. (High, 88) Permissionless InitializeMarket accepts zero lot/tick parameters - initialize.rs:98-160
2. (Critical, 85) Deposit lot-to-atom conversion wraps, unbacked lots drain vaults - deposit.rs:69-70
3. (High, 80) Bid collateral computation wraps, maker buys base for ~nothing - fifo.rs:1048-1051
4. (Medium, 60) InitializeMarket decimal/scaling arithmetic wraps - initialize.rs:108-112

Findings 2-4 all rest on one premise: that the plain u64 multiplies in quantities.rs
(`allow_multiply!`, `self.inner * other.inner`) WRAP silently.

## Refutation verdicts

- FINDING 1: CONFIRMED | No nonzero assert exists; 0 % x == 0 passes for both zero
  variants; zero base lot credits unbacked lots (deposit credits lots, zero-amount
  transfer skipped) that drain victim quote on match; zero tick makes all bids free.
  Caveat recorded: victims must opt into the attacker's crafted market (honeypot
  precondition); the missing-validation path holds regardless.
- FINDING 2: KILLED | phoenix/Cargo.toml sets overflow-checks = true (BUILD NOTES;
  verified at Cargo.toml:26). The multiply PANICS; the transaction reverts atomically,
  reverting the lot credit. No unbacked lots, no drain. The silent-wrap premise is false.
- FINDING 3: KILLED | Same false premise: the lock/match/unlock/evict products panic on
  overflow; the tx reverts. Impact reduces to self-DoS of the attacker's own order.
- FINDING 4: KILLED | Same false premise: 10u64.pow(decimals) * raw factor panics during
  init; the market is never created.

## Final report

```
📋 Solana Scan Report
Files scanned: 45
Lines analyzed: 12,530
Findings: 1 (1 High)
Red team: 1 confirmed, 0 weakened, 3 killed

Appendix - Rejected by red team:
- "Deposit conversion wraps, vault drain" (Critical, conf 85): killed - overflow-checks = true,
  panic not wrap; atomic revert.
- "Bid collateral wraps" (High, conf 80): killed - same premise.
- "Init decimals wrap" (Medium, conf 60): killed - same premise.
```

Cold test outcome: 1 real finding confirmed (conditional impact: honeypot precondition),
3 phantom Critical/High claims killed. Without the refutation stage this report would
have shown a fabricated Critical vault-drain on a heavily audited codebase - exactly the
kind of false positive that burns a tool's credibility. This run is the evidence for why
the red-team stage exists.
