# remit-audit

Security audit workflows for [remit-md/protocol](https://github.com/remit-md/protocol).

All workflows are **manual-only** (`workflow_dispatch`). Never auto-triggered.

## Quick Start

```bash
# Run the full audit suite (all 4 tools in parallel)
gh workflow run full-audit.yml

# Run individual tools
gh workflow run slither.yml
gh workflow run aderyn.yml
gh workflow run mythril.yml
gh workflow run halmos.yml

# Run against a specific branch/commit
gh workflow run slither.yml -f ref=task/fix-escrow

# Run Mythril on specific contracts only
gh workflow run mythril.yml -f contracts="RemitEscrow,RemitTab"
```

Reports are uploaded as workflow artifacts (90-day retention).

## Tools

| Workflow | Tool | What it does |
|----------|------|-------------|
| `slither.yml` | [Slither](https://github.com/crytic/slither) | Static analysis — 100+ detectors, SARIF output for GitHub Security tab |
| `aderyn.yml` | [Aderyn](https://github.com/Cyfrin/aderyn) | Cyfrin static analysis — binary install (fast), markdown + JSON reports |
| `mythril.yml` | [Mythril](https://github.com/Consensys/mythril) | Symbolic execution — all 11 contracts, 300s timeout per contract |
| `halmos.yml` | [Halmos](https://github.com/a16z/halmos) | Symbolic proofs — exhaustive verification of safety properties |
| `full-audit.yml` | All of the above | Orchestrator — runs all 4 tools in parallel, combined summary |

## Contracts (11 source + 4 libraries)

### Fund-Holding Contracts
| Contract | Description | Fee |
|----------|-------------|-----|
| `RemitEscrow` | Milestone-based escrow with splits, arbitration | 1% (inclusive) |
| `RemitTab` | Pre-funded spending tab with per-unit charges | 1% (inclusive) |
| `RemitStream` | Lockup-linear streaming payments | 1% (inclusive) |
| `RemitBounty` | Open bounties with submission bonds and dispute resolution | 1% (inclusive) |
| `RemitDeposit` | Refundable deposits / collateral (return or forfeit) | No fee |
| `RemitOnrampVault` | Fiat on-ramp vault (created by factory) | 1% (hardcoded) |

### Infrastructure Contracts
| Contract | Description |
|----------|-------------|
| `RemitRouter` | UUPS proxy — pay-direct, pay-per-request, relayer For-variants |
| `RemitFeeCalculator` | UUPS proxy — tiered fee calculation (1% standard, 0.5% preferred) |
| `RemitKeyRegistry` | EIP-712 key registration for agent wallets |
| `RemitArbitration` | Dispute resolution for escrows |
| `OnrampVaultFactory` | Factory for deploying on-ramp vaults |

### Libraries
| Library | Purpose |
|---------|---------|
| `RemitTypes` | Shared structs, enums, constants |
| `RemitErrors` | Custom error definitions |
| `RemitEvents` | Event definitions |
| `RemitKeyValidator` | Key validation logic |

### Test Contracts
| Contract | Purpose |
|----------|---------|
| `MockUSDC` | EIP-2612 permit-enabled test token |
| `MockFeeCalculator` | Fixed 1% fee calculator for testing |

## Halmos Symbolic Proofs

Proof source: `test/halmos/RemitSymbolicProofs.sol` in the protocol repo.

### Current Proofs (P1–P5)

| ID | Property | Contract | Status |
|----|----------|----------|--------|
| P1 | Fee correctness: `fee + payout == amount`, fee < amount, payout > 0 | FeeCalculator | Active |
| P2 | Fund conservation: `payeeGain + feeGain == escrowed` on release | Escrow | Active |
| P3 | No double-settle: second `releaseEscrow` always reverts | Escrow | Active |
| P4 | Tab locks exact amount: payer loses exactly `limit` USDC | Tab | Active |
| P5 | Fee oracle: `MockFeeCalculator` returns exactly 1% for all amounts | FeeCalculator | Active |

### Recommended New Proofs

These cover the remaining fund-holding contracts added/modified in V14:

| ID | Property | Contract | Priority |
|----|----------|----------|----------|
| S1 | Stream fund conservation: `withdrawn + refund + fee == maxTotal` | Stream | High |
| S2 | No double-withdrawal on same accrued amount | Stream | Medium |
| S3 | Fee never exceeds pending accrued amount | Stream | Medium |
| B1 | No double-award: only one payout per bounty | Bounty | High |
| B2 | Bond conservation: `amount + bond == winner + fee + returned` | Bounty | High |
| B3 | Dispute window enforced: 24h gating on `disputeRejection` | Bounty | Medium |
| D1 | Exact amount locking: no fees on deposits | Deposit | Medium |
| D2 | No double-settlement: `return` XOR `forfeit` XOR `claimExpired` | Deposit | Medium |
| D3 | Expiry gating: `claimExpiredDeposit` only after expiry | Deposit | Low |

## Configuration

| Setting | Value |
|---------|-------|
| Solidity version | 0.8.24 |
| EVM version | cancun |
| Optimizer | enabled, 200 runs |
| via-IR | enabled |
| Foundry profile | default |

## Notes

- **Private repo**: SARIF upload to GitHub Security tab requires GitHub Advanced Security (GHAS). The Slither workflow has `continue-on-error: true` on the SARIF upload step.
- **Mythril timeout**: Default 300s per contract. The full suite (11 contracts) may take up to 60 minutes. Use the `contracts` input to run specific contracts.
- **Halmos OOM**: Symbolic proofs with complex state (Stream, Bounty) may hit memory limits. Increase runner size or reduce solver timeout if needed.
- **Aderyn**: Uses binary installer (not `cargo install`) for fast CI runs (~10s install vs ~15min compile).
