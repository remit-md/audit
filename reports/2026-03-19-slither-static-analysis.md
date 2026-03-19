# Remit Protocol — Slither Static Analysis Report

**Date:** 2026-03-19
**Tool:** Slither v0.10.x (all detectors enabled)
**Protocol commit:** `3edac2f` (remit-md/protocol, main)
**CI run:** [23286587611](https://github.com/remit-md/audit/actions/runs/23286587611)
**Runner:** self-hosted (remit-ci, cx33)

---

## Executive Summary

Slither identified **106 findings** across 11 source contracts:

| Severity | Count | Real Issues | False Positives |
|----------|-------|-------------|-----------------|
| High | 10 | 0 (see note) | 10 |
| Medium | 29 | 2 classes | 27 |
| Low | 48 | 0 | 48 |
| Informational | 19 | 0 | 19 |

**Two actionable items** were identified, both affecting immutable (non-upgradeable) contracts. These must be resolved before mainnet deployment.

---

## Contracts in Scope

### Fund-Holding (Immutable)
| Contract | Lines | Fee | Upgradeable |
|----------|-------|-----|-------------|
| RemitEscrow | ~770 | 1% inclusive | No |
| RemitTab | ~450 | 1% inclusive | No |
| RemitStream | ~370 | 1% inclusive | No |
| RemitBounty | ~540 | 1% inclusive | No |
| RemitDeposit | ~310 | None | No |
| RemitOnrampVault | ~120 | 1% hardcoded | No |

### Infrastructure
| Contract | Upgradeable |
|----------|-------------|
| RemitRouter (UUPS) | Yes |
| RemitFeeCalculator (UUPS) | Yes |
| RemitKeyRegistry | No |
| RemitArbitration | No |
| OnrampVaultFactory | No |

---

## Finding 1 — Missing Session Key Validation on Relayer Variants

| Field | Value |
|-------|-------|
| Slither check | `arbitrary-send-erc20` |
| Slither severity | High |
| Assessed severity | **Medium** (design gap) |
| Confidence | High |
| Affected contracts | RemitBounty, RemitDeposit |
| Post-deploy fixable | **No** (immutable contracts) |
| Recommendation | **Fix before mainnet** |

### Description

All `*For()` relayer functions use the `onlyAuthorizedRelayer` modifier, which restricts calls to server-approved relayer addresses. However, three functions skip the secondary `RemitKeyValidator._validateAndRecord()` check that enforces session key delegation, spending limits, and expiry:

| Contract | Function | Relayer Gate | Key Validation |
|----------|----------|:------------:|:--------------:|
| RemitRouter | `payDirectFor` | Yes | Yes |
| RemitRouter | `payPerRequestFor` | Yes | Yes |
| RemitEscrow | `createEscrowFor` | Yes | Yes |
| RemitTab | `openTabFor` | Yes | Yes |
| RemitStream | `openStreamFor` | Yes | Yes |
| **RemitBounty** | **`postBountyFor`** | Yes | **No** |
| **RemitBounty** | **`submitBountyFor`** | Yes | **No** |
| **RemitDeposit** | **`lockDepositFor`** | Yes | **No** |

### Impact

Today the relayer is a single server key controlled by the protocol team, so risk is minimal. However:

- If the relayer key is compromised, an attacker could call `postBountyFor(victim, ...)` to drain any wallet that has approved the Bounty contract, bypassing the victim's session key spending limits.
- If third-party relayers are ever onboarded, they could operate outside the session key delegation model for bounty and deposit operations.
- The inconsistency across contracts creates a false sense of security — developers may assume all `For` variants enforce session key validation.

### Remediation

Add `_validateAndRecord()` calls to all three functions:

```solidity
// RemitBounty.postBountyFor — add after onlyAuthorizedRelayer check
RemitKeyValidator._validateAndRecord(
    keyRegistry, poster, amount, RemitTypes.PaymentType.BOUNTY
);

// RemitBounty.submitBountyFor — add after onlyAuthorizedRelayer check
RemitKeyValidator._validateAndRecord(
    keyRegistry, submitter, bounty.submissionBond, RemitTypes.PaymentType.BOUNTY
);

// RemitDeposit.lockDepositFor — add after onlyAuthorizedRelayer check
RemitKeyValidator._validateAndRecord(
    keyRegistry, depositor, amount, RemitTypes.PaymentType.DEPOSIT
);
```

### References

- `src/RemitBounty.sol` lines 362-398 (`postBountyFor`), 407-435 (`submitBountyFor`)
- `src/RemitDeposit.sol` lines 172-197 (`lockDepositFor`)
- `src/libraries/RemitKeyValidator.sol` lines 22-45 (`_validateAndRecord`)

---

## Finding 2 — CEI Pattern Violation via KeyRegistry External Call

| Field | Value |
|-------|-------|
| Slither check | `reentrancy-no-eth` |
| Slither severity | Medium |
| Assessed severity | **Low** (theoretical) |
| Confidence | Medium |
| Affected contracts | RemitEscrow, RemitTab, RemitStream, RemitBounty, RemitDeposit |
| Post-deploy fixable | **No** (immutable contracts) |
| Recommendation | **Fix before mainnet** |

### Description

Eight functions call `RemitKeyValidator._validateAndRecord()` which makes an external call to the `keyRegistry` contract, then write state variables after the call returns. This violates the Checks-Effects-Interactions (CEI) pattern.

Affected functions (both regular and `For` variants):
- `RemitEscrow.createEscrow` / `createEscrowFor`
- `RemitTab.openTab` / `openTabFor`
- `RemitStream.openStream` / `openStreamFor`
- `RemitBounty.postBounty`
- `RemitDeposit.lockDeposit`

### Impact

If the `keyRegistry` contract were compromised or malicious, it could re-enter any of these functions during the `_validateAndRecord` callback, potentially:
- Creating duplicate escrows/tabs/streams with the same ID
- Writing to state before the initial function completes

Practical risk is very low because:
1. `keyRegistry` is an immutable contract deployed by the protocol team
2. All fund-holding functions have `nonReentrant` guards on the USDC transfer step
3. Duplicate IDs are caught by existence checks

However, the pattern is a best-practice violation that could become exploitable if the architecture changes.

### Remediation

**Option A (preferred):** Reorder to move `_validateAndRecord()` after state writes but before token transfers. This preserves CEI without adding gas overhead:

```solidity
// Current (CEI violation):
_validateAndRecord(keyRegistry, payer, amount, type);  // external call
_escrows[id] = Escrow{...};                            // state write
usdc.safeTransferFrom(payer, address(this), amount);   // transfer

// Fixed:
_escrows[id] = Escrow{...};                            // state write first
_validateAndRecord(keyRegistry, payer, amount, type);   // then external call
usdc.safeTransferFrom(payer, address(this), amount);   // then transfer
```

**Option B:** Add `nonReentrant` modifier to all affected functions (not just the settlement/release functions that already have it).

### References

- `src/RemitEscrow.sol` lines 125, 225 (createEscrow/For)
- `src/RemitTab.sol` lines 109, 191 (openTab/For)
- `src/RemitStream.sol` lines 95, 195 (openStream/For)
- `src/RemitBounty.sol` line 130 (postBounty)
- `src/RemitDeposit.sol` line 81 (lockDeposit)

---

## Suppressed Findings — False Positives

The following findings were reviewed and determined to be false positives or acceptable behavior. These should be added to a Slither triage configuration to reduce noise in future runs.

### HIGH: `arbitrary-send-erc20` (10 findings)

All 10 HIGH findings flag `*For()` relayer functions for using `transferFrom(parameter, ...)` where the `from` address is a function parameter. This is by design — the relayer pattern requires the `from` to be the real payer, not `msg.sender`. All functions are gated by `onlyAuthorizedRelayer`. USDC `safeTransferFrom` requires explicit prior approval by the payer, providing a second layer of protection.

**Verdict:** False positive. Relayer architecture is intentional.

### MEDIUM: `divide-before-multiply` (3 findings)

All three findings are in `RemitFeeCalculator._getMonthKey()` which implements the Hinnant civil date algorithm for calendar month calculation. The divide-then-multiply pattern is the correct implementation of this well-known algorithm — the integer division is intentional truncation for era/year computation, not a precision bug.

**Verdict:** False positive. Algorithm is mathematically proven correct.

### MEDIUM: `incorrect-equality` (4 findings)

Flags `== 0` comparisons on USDC balances and stream amounts. Slither warns about strict equality because miners can manipulate ETH balances, but these compare ERC-20 token amounts which are transferred exactly. `amount == 0` and `balance == 0` are the correct checks.

**Verdict:** False positive. USDC transfers are exact; no miner manipulation vector.

### MEDIUM: `uninitialized-local` (14 findings)

Local variables like `fee`, `providerPayout`, `milestoneSum`, `splitSum`, `feeAccumulated`, `validCount` are declared without explicit initialization. Solidity defaults all `uint` types to `0`, which is the correct initial value in all cases (accumulators, running sums, counters).

**Verdict:** False positive. Solidity zero-initialization is correct for all flagged variables.

### LOW: `timestamp` (45 findings)

All findings flag `block.timestamp` usage in time-dependent logic (expiry checks, stream rate calculations, dispute windows). Time-dependent business logic is the core purpose of these contracts. Block timestamp manipulation on Base L2 is limited to ~1 second, making it irrelevant for hour/day-scale timeouts.

**Verdict:** False positive. Timestamp usage is intentional and manipulation-resistant on L2.

### LOW: `reentrancy-benign` (2 findings)

State writes after external calls in `createEscrow`/`createEscrowFor` for non-critical counters (`_escrowParticipations`). No exploitable path exists.

**Verdict:** False positive. Benign state update.

### LOW: `reentrancy-events` (1 finding)

`OnrampVaultFactory.getOrCreate()` emits `VaultCreated` after calling `vault.initialize()`. Event ordering after external calls is standard practice.

**Verdict:** False positive. Standard event emission pattern.

### INFORMATIONAL: `naming-convention` (13 findings)

Parameter names like `_operator` use underscore prefix to avoid shadowing storage variables. This is an intentional Solidity convention.

**Verdict:** False positive. Naming convention is intentional.

### INFORMATIONAL: `unused-state` — `__gap` (2 findings)

`RemitRouter.__gap` and `RemitFeeCalculator.__gap` are UUPS upgrade storage gaps. They are intentionally unused — their purpose is to reserve storage slots for future upgrades.

**Verdict:** False positive. Standard UUPS pattern.

### INFORMATIONAL: `cyclomatic-complexity` (4 findings)

`RemitEscrow.resolveDispute()` has a cyclomatic complexity of 13. Dispute resolution inherently requires many branches (payer/payee wins, partial splits, milestone handling, evidence checks). Splitting would reduce readability without improving safety.

**Verdict:** Acceptable. Complexity is inherent to the domain.

---

## Recommendations

### Pre-Mainnet (P0)

1. **Add `_validateAndRecord()` to `postBountyFor`, `submitBountyFor`, `lockDepositFor`**
   - Estimated effort: 3 lines per function, 9 lines total
   - Requires contract redeploy

2. **Fix CEI ordering in 8 creation functions**
   - Move `_validateAndRecord()` after state writes, before token transfers
   - Requires contract redeploy

### Post-Mainnet (P1)

3. **Add `slither.config.json` to protocol repo** to suppress confirmed false positives:
   - `arbitrary-send-erc20` on all `*For()` functions
   - `divide-before-multiply` on `_getMonthKey()`
   - `incorrect-equality` on USDC balance checks
   - `uninitialized-local` on accumulator variables
   - `timestamp` on all time-dependent logic
   - `unused-state` on `__gap` variables

4. **Run Aderyn, Mythril, and Halmos** for complementary coverage (workflows are configured in this repo, pending first run).

5. **Add Halmos proofs** for Stream (fund conservation), Bounty (no double-award, bond conservation), and Deposit (no double-settlement) — see README for proof roadmap.

---

## Appendix: Finding Counts by Check

| Check | Impact | Count |
|-------|--------|-------|
| `arbitrary-send-erc20` | High | 10 |
| `divide-before-multiply` | Medium | 3 |
| `incorrect-equality` | Medium | 4 |
| `reentrancy-no-eth` | Medium | 8 |
| `uninitialized-local` | Medium | 14 |
| `reentrancy-benign` | Low | 2 |
| `reentrancy-events` | Low | 1 |
| `timestamp` | Low | 45 |
| `cyclomatic-complexity` | Informational | 4 |
| `naming-convention` | Informational | 13 |
| `unused-state` | Informational | 2 |
| **Total** | | **106** |
