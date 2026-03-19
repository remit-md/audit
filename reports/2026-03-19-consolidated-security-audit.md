# Consolidated Security Audit Report

**Protocol:** remit.md -- Universal Payment Protocol for AI Agents (USDC on EVM L2)
**Date:** 2026-03-19
**Protocol Commit:** `3edac2f` (remit-md/protocol, main)
**Solidity:** 0.8.24 | EVM Target: Cancun | Optimizer: 200 runs, via-IR
**Scope:** 11 contracts, ~3,230 nSLOC

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Tools and Methodology](#tools-and-methodology)
3. [Findings by Severity](#findings-by-severity)
   - [P0 -- Must Fix Before Mainnet](#p0--must-fix-before-mainnet)
   - [P1 -- Should Fix Before Mainnet](#p1--should-fix-before-mainnet)
   - [P2 -- Nice to Have](#p2--nice-to-have)
4. [Suppressed and False Positives](#suppressed-and-false-positives)
5. [Tooling Issues](#tooling-issues)
6. [Summary Table](#summary-table)

---

## Executive Summary

Four static analysis tools were run against the full contract suite. Two produced actionable results (Slither, Aderyn). Two failed due to tooling constraints (Mythril, Halmos).

Combined results: **106 Slither findings** (10 High, 29 Medium, 48 Low, 19 Informational) and **19 Aderyn findings** (2 High, 17 Low). After triage, the vast majority are false positives or accepted design trade-offs.

**Two P0 issues** require fixes before any mainnet deployment:

1. **Missing session key validation** on 3 `*For()` relayer functions -- a real design gap that bypasses spending limits if the relayer key is compromised.
2. **CEI pattern violations** -- 15 instances across two external-call vectors where state writes occur after calls to protocol-owned contracts.

Both affect immutable (non-proxy) contracts and cannot be patched post-deploy.

---

## Tools and Methodology

| Tool | CI Run | Duration | Result | Findings |
|------|--------|----------|--------|----------|
| Slither | 23286587611 | 1m 0s | SUCCESS | 106 (10H / 29M / 48L / 19I) |
| Aderyn | 23287940532 | 26s | SUCCESS | 19 (2H / 17L) |
| Mythril | 23287903475 | 1m 45s | FAILED | 0 (tooling issue) |
| Halmos | 23287902456 | 22m 30s | FAILED | 0 (OOM on 8GB runner) |

All tools were run in CI (GitHub Actions) on the `remit-audit` repository against protocol commit `3edac2f`.

---

## Findings by Severity

### P0 -- Must Fix Before Mainnet

These findings affect immutable contracts and **cannot be patched post-deploy**.

#### Finding 1: Missing `_validateAndRecord` on 3 For-Variants

| Field | Value |
|-------|-------|
| Sources | Slither HIGH `arbitrary-send-erc20` (confirmed by manual review) |
| Contracts | `RemitBounty` (`postBountyFor`, `submitBountyFor`), `RemitDeposit` (`lockDepositFor`) |
| Impact | P0 -- Critical |
| Post-Deploy Fixable | NO (immutable contracts) |

**Description:**
All `*For()` functions are relayer endpoints that call `transferFrom(parameter, ...)` where the `from` address is the real payer, not `msg.sender`. This is the intended relayer pattern and all 10 Slither HIGH findings on `arbitrary-send-erc20` are by-design for that reason.

However, 3 of the 8 `*For()` variants skip the `RemitKeyValidator._validateAndRecord()` call that the other 5 variants all include. The missing validation means session key spending limits are not enforced on these paths.

**Affected Functions (missing validation):**
- `RemitBounty.postBountyFor()` -- no `_validateAndRecord`
- `RemitBounty.submitBountyFor()` -- no `_validateAndRecord`
- `RemitDeposit.lockDepositFor()` -- no `_validateAndRecord`

**Functions with correct validation (for reference):**
- `RemitRouter.payFor()` -- has `_validateAndRecord`
- `RemitRouter.payWithFeeFor()` -- has `_validateAndRecord`
- `RemitEscrow.createEscrowFor()` -- has `_validateAndRecord`
- `RemitTab.openTabFor()` -- has `_validateAndRecord`
- `RemitStream.createStreamFor()` -- has `_validateAndRecord`

**Impact:**
If the relayer key leaks or third-party relayers are added in the future, session key spending limits are bypassed for bounty posting, bounty submission, and deposit locking. An attacker with the relayer key could drain any wallet that has approved these contracts, up to the full approval amount, without session key rate limiting.

**Recommended Fix:**
Add `_validateAndRecord(keyRegistry, poster/submitter/depositor, amount, PaymentType)` to all 3 functions. Estimated 9 lines of code total.

---

#### Finding 2: CEI Pattern Violations (15 Instances, 2 Vectors)

| Field | Value |
|-------|-------|
| Sources | Slither MEDIUM `reentrancy-no-eth` (8 instances), Aderyn HIGH H-2 (7 instances) |
| Contracts | `RemitEscrow`, `RemitTab`, `RemitStream`, `RemitBounty` |
| Impact | P0 -- Must Fix |
| Post-Deploy Fixable | NO (immutable contracts) |

**Description:**
Two independent reentrancy vectors were identified by different tools:

**Vector A (Slither, 8 instances):** `_validateAndRecord()` makes an external call to `keyRegistry` before state variables are written. Found in all For-variant functions that do call `_validateAndRecord`.

**Vector B (Aderyn, 7 instances):** `feeCalculator.calculateFee()` is called externally before state variables are updated. Found in:
- `RemitBounty`: 3 instances
- `RemitEscrow`: 2 instances
- `RemitStream`: 2 instances

**Impact:**
Both `keyRegistry` and `feeCalculator` are protocol-owned contracts, making exploitation require a compromised protocol component. Real-world risk is very low. However, this violates the Checks-Effects-Interactions pattern and is a defense-in-depth concern -- if either contract were ever compromised or upgraded maliciously, reentrancy would be possible.

**Recommended Fix (choose one):**
- **Option A:** Reorder state writes before external calls in all 15 instances.
- **Option B:** Add OpenZeppelin `ReentrancyGuard` (`nonReentrant` modifier) to all creation/submission functions.

Option B is simpler and provides blanket protection regardless of future code changes.

---

### P1 -- Should Fix Before Mainnet

#### Finding 3: Missing Events on 27 Admin Functions

| Field | Value |
|-------|-------|
| Source | Aderyn L-11 (27 instances) |
| Contracts | Most contracts |
| Impact | P1 -- Should Fix |
| Post-Deploy Fixable | Partial (UUPS proxies only: Router, FeeCalculator) |

**Description:**
Functions that change critical protocol state do not emit events. This includes `authorizeRelayer`, `revokeRelayer`, `authorizeEscrow`, `setKeyRegistry`, `authorizeCaller`, and similar admin operations across multiple contracts.

**Impact:**
- The Ponder indexer cannot track admin configuration changes.
- No on-chain audit trail for privileged operations.
- Monitoring and alerting systems have blind spots for admin actions.

**Recommended Fix:**
Define and emit events for all admin state-changing functions. For immutable contracts, this requires redeployment. For UUPS proxies (Router, FeeCalculator), it can be done via upgrade.

---

#### Finding 4: OnrampVaultFactory Missing Interface Inheritance

| Field | Value |
|-------|-------|
| Source | Aderyn L-7 (1 instance) |
| Contract | `OnrampVaultFactory` |
| Impact | P1 -- Should Fix |
| Post-Deploy Fixable | NO |

**Description:**
`OnrampVaultFactory` does not implement `IOnrampVaultFactory` despite the interface existing. This means the compiler does not enforce that the contract satisfies the interface, allowing signature drift.

**Recommended Fix:**
Add `is IOnrampVaultFactory` to the contract declaration.

---

#### Finding 5: 9 Unused Error Definitions in RemitErrors.sol

| Field | Value |
|-------|-------|
| Source | Aderyn L-15 (9 instances) |
| Contract | `RemitErrors.sol` (library) |
| Impact | P1 -- Should Fix |
| Post-Deploy Fixable | YES (library, does not affect deployed contracts) |

**Description:**
Nine custom errors are defined but never used anywhere in the codebase:
- `EscrowExpired`
- `NonceReused`
- `TabExpired`
- `RateExceedsCap`
- `DisputeWindowClosed`
- `DelegationNotFound`
- `DisputeBondInsufficient`
- `ArbitratorBondInsufficient`
- `PoolTooSmall`

**Impact:**
Dead code that adds confusion. Some of these (e.g., `EscrowExpired`, `TabExpired`) may indicate missing validation paths where these errors should be thrown but are not.

**Recommended Fix:**
Audit each unused error. Either remove it if the check is genuinely unnecessary, or add the missing validation where it should be used.

---

#### Finding 6: Router.setKeyRegistry Missing Zero-Address Check

| Field | Value |
|-------|-------|
| Source | Aderyn L-12 (1 instance) |
| Contract | `RemitRouter` |
| Impact | P1 -- Should Fix |
| Post-Deploy Fixable | YES (UUPS proxy) |

**Description:**
`setKeyRegistry()` accepts any address including `address(0)`. Setting it to zero would break all session key validation across the protocol.

**Recommended Fix:**
Add `require(newKeyRegistry != address(0), "zero address")`.

---

### P2 -- Nice to Have

#### Finding 7: Local Variable Shadowing in Router.initialize()

| Field | Value |
|-------|-------|
| Source | Aderyn L-6 (4 instances) |
| Contract | `RemitRouter` |
| Post-Deploy Fixable | YES (UUPS) |

Local variables `usdc`, `feeCalculator`, `protocolAdmin`, and `feeRecipient` in `Router.initialize()` shadow storage variables of the same name. Rename locals to `_usdc`, `_feeCalculator`, etc.

---

#### Finding 8: ETH Locked in UUPS Proxies

| Field | Value |
|-------|-------|
| Source | Aderyn H-1 (2 instances) |
| Contracts | `RemitRouter`, `RemitFeeCalculator` |
| Post-Deploy Fixable | YES (UUPS) |

These contracts inherit `UUPSUpgradeable` which includes a `receive()` fallback. If ETH is accidentally sent, it is permanently locked. Real risk is very low since these contracts only handle USDC.

**Recommended Fix:**
Add `receive() external payable { revert(); }` to reject ETH, or add a rescue function for the protocol admin.

---

#### Finding 9: Magic Numbers Could Be Named Constants

| Field | Value |
|-------|-------|
| Source | Aderyn L-5 (28 instances) |
| Contracts | Various |
| Post-Deploy Fixable | NO |

Numeric literals like `10_000` (basis points denominator), `86400` (seconds per day), `100` (percentage), and `146097` (Hinnant algorithm constant) could be extracted to named constants in `RemitTypes.sol` for readability.

---

#### Finding 10: Unused Import in IRemitKeyRegistry

| Field | Value |
|-------|-------|
| Source | Aderyn L-16 (1 instance) |
| Contract | `IRemitKeyRegistry` |
| Post-Deploy Fixable | YES |

`RemitTypes` is imported but not used in the interface file.

---

## Suppressed and False Positives

The following findings were triaged and determined to be non-actionable:

| Tool | Check | Count | Reason |
|------|-------|-------|--------|
| Slither | `arbitrary-send-erc20` | 10 | By design -- relayer `transferFrom(payer, ...)` pattern. `onlyAuthorizedRelayer` gated. |
| Slither | `divide-before-multiply` | 3 | Hinnant civil date algorithm in `_getMonthKey()`. Mathematically correct. |
| Slither | `incorrect-equality` | 4 | `== 0` checks on USDC balances. ERC-20 transfers are exact. |
| Slither | `uninitialized-local` | 14 | Accumulators defaulting to `uint256(0)`. Standard Solidity behavior. |
| Slither | `timestamp` | 45 | Time-based logic is core business (streams, escrows, tabs, disputes). |
| Slither | `reentrancy-benign` | 2 | Non-exploitable reentrancy in view-like paths. |
| Slither | `reentrancy-events` | 1 | Event emitted after external call. Standard pattern, no state risk. |
| Slither | `naming-convention` | 13 | Intentional underscore prefix on internal functions and storage gaps. |
| Slither | `unused-state` | 2 | `__gap` arrays for UUPS storage layout compatibility. |
| Slither | `cyclomatic-complexity` | 4 | Dispute resolution logic. Inherent domain complexity. |
| Aderyn | Empty Block | 2 | `_authorizeUpgrade()` body in UUPS. Standard OpenZeppelin pattern. |
| Aderyn | Large Numeric Literal | 15 | USDC decimal constants (`1e6`, etc.). Intentional. |
| Aderyn | Modifier Invoked Once | 1 | `FeeCalculator.onlyAuthorized`. Single use is acceptable. |
| Aderyn | PUSH0 Opcode | 26 | Base supports Shanghai+ opcodes. No compatibility issue. |
| Aderyn | Loop require/revert | 8 | Fail-fast validation inside bounded loops. Intentional. |
| Aderyn | Unspecific Pragma | 22 | `^0.8.24` vs `0.8.24`. Style preference, not a security concern. |
| Aderyn | Centralization Risk | 20 | `onlyOwner` functions. Protocol is admin-controlled by design. |
| Aderyn | Costly Loop Operations | 6 | `SSTORE` in loops for milestones/splits. Bounded by max 10 items. |
| Aderyn | Unused State Variable | 2 | `__gap` arrays. Same as Slither `unused-state`. |

**Total suppressed: 180 findings across both tools.**

---

## Tooling Issues

These are not security findings. They represent gaps in tooling coverage that should be resolved before the next audit cycle.

| Tool | Issue | Root Cause | Recommended Fix |
|------|-------|------------|-----------------|
| Mythril | All 11 contracts failed with "Source not found" | Cannot resolve Foundry `@openzeppelin/` remappings | Pass `--solc-remaps` flag or feed pre-compiled artifacts |
| Halmos | OOM (SIGKILL) during `forge build --ast --extra-output storageLayout` | via-IR compilation requires more than 8GB RAM; cx33 runner insufficient | Use 16GB+ runner (cx41 or equivalent) |

**Note on Halmos:** Five symbolic proofs exist in `test/halmos/RemitSymbolicProofs.sol` (P1-P5) and compile successfully. They simply cannot execute on current CI hardware.

---

## Summary Table

| # | Finding | Severity | Contracts | Post-Deploy Fixable |
|---|---------|----------|-----------|---------------------|
| 1 | Missing `_validateAndRecord` on 3 For-variants | **P0** | Bounty, Deposit | NO |
| 2 | CEI violations (15 instances, 2 vectors) | **P0** | Escrow, Tab, Stream, Bounty | NO |
| 3 | Missing events on 27 admin functions | P1 | Most contracts | Partial (UUPS only) |
| 4 | OnrampVaultFactory missing interface | P1 | OnrampVaultFactory | NO |
| 5 | 9 unused error definitions | P1 | RemitErrors.sol | YES |
| 6 | setKeyRegistry missing zero-address check | P1 | Router | YES (UUPS) |
| 7 | Variable shadowing in Router.initialize | P2 | Router | YES (UUPS) |
| 8 | ETH locked in UUPS proxies | P2 | Router, FeeCalculator | YES (UUPS) |
| 9 | Magic numbers as literals | P2 | Various | NO |
| 10 | Unused import | P2 | IRemitKeyRegistry | YES |

**Bottom line:** 2 P0 findings must be resolved before any mainnet deployment. Both affect immutable contracts (Bounty, Deposit, Escrow, Tab, Stream) and cannot be patched after deploy. The remaining 8 findings are lower severity, with several fixable via UUPS proxy upgrades.
