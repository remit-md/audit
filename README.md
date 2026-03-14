# remit-audit

Security audit workflows for [remit-md/protocol](https://github.com/remit-md/protocol).

All workflows are **manual-only** (`workflow_dispatch`). Never auto-triggered.

## Tools

| Workflow | Tool | What it does |
|----------|------|-------------|
| `slither.yml` | [Slither](https://github.com/crytic/slither) | Static analysis (101 detectors) |
| `aderyn.yml` | [Aderyn](https://github.com/Cyfrin/aderyn) | Cyfrin static analysis |
| `mythril.yml` | [Mythril](https://github.com/Consensys/mythril) | Symbolic execution (7 fund-holding contracts) |
| `halmos.yml` | [Halmos](https://github.com/a16z/halmos) | Symbolic proofs (5 properties in `test/halmos/`) |

## Usage

Trigger from the Actions tab, or via CLI:

```bash
# Run Slither against main
gh workflow run slither.yml

# Run against a specific branch/commit
gh workflow run halmos.yml -f ref=task/fix-escrow

# Run Mythril on specific contracts
gh workflow run mythril.yml -f contracts="RemitEscrow,RemitTab"
```

Reports are uploaded as workflow artifacts (90-day retention).

## Halmos Properties

| ID | Property | Contract |
|----|----------|----------|
| P1 | Fee ≤ 1% of amount | FeeCalculator |
| P2 | Fund conservation: payer loss = payee + fee | Router + Escrow |
| P3 | No double-settle on escrow | Escrow |
| P4 | Tab locks exact amount on open | Tab |
| P5 | Fee oracle matches on-chain calculation | FeeCalculator |

Proof source: `test/halmos/RemitSymbolicProofs.sol` in the protocol repo.
