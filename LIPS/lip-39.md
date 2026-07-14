---
lip: 39
title: Simplified Negative Rebase Sanity Check
status: WIP
author: George Avsetsin
discussions-to: <research.lido.fi thread to be created>
created: 2026-07-14
updated: 2026-07-14
---

# Simplified Negative Rebase Sanity Check

## Simple Summary

Replace the stateful 36-day negative CL rebase allowance with a committee trigger at zero and a configurable per-report hard cutoff. Reports with no detected negative CL rebase are unchanged.

## Abstract

[LIP-23](./lip-23.md) introduced a CL-spec-derived negative CL rebase allowance and a second-opinion interface. [LIP-35](./lip-35.md) adapted the same design to MaxEB accounting but kept its 36-day report ledger and left the second-opinion provider unset.

This proposal simplifies both parts. It replaces the modeled allowance with two thresholds: zero for committee involvement and a configurable per-report hard cutoff. It also makes a DAO-approved committee, independent from the accounting-oracle members, the active second opinion. The negative-rebase check no longer needs report history or migration, and a detected negative CL rebase can no longer pass on the accounting-oracle report alone.

## Motivation

### Current design: calibrated but stateful

The current checker allows a cumulative negative CL rebase of about 3.6% over a window of up to 36 days before requiring a second opinion. The limit is calibrated to a selected natural penalties-and-slashing envelope, not every possible CL loss ([LIP-35](./lip-35.md#ao-cl-balance-decrease), [SRv3 parameter research](https://hackmd.io/0ePemSJtQA6S5NDK2M3mww)). Maintaining it requires stored report history, a sliding-window calculation, migration at checker replacement, and parameter revisions when CL rules change.

With no second-opinion provider configured, a loss within the modeled allowance can be submitted by the accounting oracle alone, while a larger loss reverts.

### Proposed design: two simple thresholds

The proposal stops trying to classify losses from CL rules:

- non-negative CL rebase: pass;
- negative CL rebase within the decrease allowed by `maxCLBalanceDecreaseBP`: require the committee;
- loss above that limit: reject.

The zero threshold removes the modeled oracle-only allowance. As of July 2026, Lido has never had a negative stETH rebase. The proposed check is stricter because it excludes EL rewards, so a small genuine CL-side loss may still require the committee. This deliberate false positive is safer than automatically accepting an unusual loss.

The proposed initial 5% hard cutoff adds modest headroom over the [current penalty-derived 3.6% 36-day allowance](./lip-35.md#ao-cl-balance-decrease), reducing the need to retune the limit after CL parameter changes. The wider cutoff does not increase what the accounting oracle can submit alone: every detected negative CL rebase requires the committee. It also improves liveness during Dual Governance: a genuine loss within the cutoff can be confirmed without a DAO proposal while Rage Quit is active. A loss above the cutoff still requires DAO action.

### Make the second opinion operational

The [SP1 oracle](https://research.lido.fi/t/zk-lido-oracle-powered-by-succinct/5747) was completed and deployed on mainnet in advisory mode, but never connected as the protocol's second opinion. It was later [updated for Electra](https://github.com/lidofinance/sp1-lido-accounting-zk/commit/07560d2ca75bde9ec85094212d454bd9a1f3ef90). A separate [RISC Zero/Boundless implementation](https://research.lido.fi/t/proposal-for-a-lido-accounting-oracle-second-opinion/9534/8) produced working mainnet test reports, but its external audit and production deployment remained outstanding. Its integration was [publicly paused](https://research.lido.fi/t/proposal-for-a-lido-accounting-oracle-second-opinion/9534/9). Both implementations show the maintenance cost of this approach. A CL proof tracks fork-specific state and proof formats, so Ethereum upgrades require development, testing, and renewed audit work.

[LIP-23](./lip-23.md#second-opinion) explicitly allowed a third-party oracle committee or multisig as an alternative. This proposal uses that option and makes the second opinion part of the active report path.

## Specification

### Overview

```python
principalCLBalance = previousCLBalance + deposits
totalPostCLBalance = postCLValidatorsBalance + postCLPendingBalance
accountedPostCLBalance = totalPostCLBalance + withdrawalsVaultTransfer
clRebase = accountedPostCLBalance - principalCLBalance
```

- `clRebase >= 0`: pass without a second opinion;
- `clRebase < 0` and the loss does not exceed the decrease allowed by `maxCLBalanceDecreaseBP`: require the committee;
- the loss exceeds that allowance: revert before calling the committee.

### Rebase calculation

The check limits the CL-side effect of the report on protocol accounting:

- `previousCLBalance` and `totalPostCLBalance` include validator and pending-deposit balances;
- `deposits` is protocol-known ETH moved into the CL;
- `withdrawalsVaultTransfer` is ETH actually taken from the Withdrawal Vault in this report. It is calculated onchain by the existing rebase limiter and cannot exceed the reported vault balance, which the existing sanity check requires not to exceed the live vault balance.

Using the actual transfer avoids treating the CL-to-vault flow as a loss without storing the previous vault balance. A CL loss covered by real ETH transferred from the vault does not trigger the committee. This is intentional: the check limits the report's impact on stETH accounting, not raw CL performance in isolation.

`reportReference` is the Accounting Oracle's identifier for the report being processed. In the current implementation it is the reference slot. The pseudocode also uses the report's `withdrawalVaultBalance`:

```python
if clRebase >= 0:
    return

negativeCLRebase = -clRebase
maxDecrease = principalCLBalance * maxCLBalanceDecreaseBP / 10_000

if negativeCLRebase > maxDecrease:
    revert IncorrectCLBalanceDecrease

if secondOpinionOracle == address(0):
    revert SecondOpinionOracleNotSet

_askSecondOpinion(reportReference, totalPostCLBalance, withdrawalVaultBalance)
```

The cutoff is based on the principal CL balance before the reported loss. It is checked before the external call and cannot be overridden by the committee.

### Second opinion

The provider can be a small contract controlled by the second-opinion committee:

```solidity
struct Report {
    bool exists;
    uint256 clBalanceGwei;
    uint256 withdrawalVaultBalanceWei;
}

address committee;
mapping(uint256 => Report) reports;

// Called by the second-opinion committee. Calling it again replaces the attestation.
function setReport(
    uint256 reportReference,
    uint256 clBalanceGwei,
    uint256 withdrawalVaultBalanceWei
) external {
    require(msg.sender == committee);
    reports[reportReference] = Report(true, clBalanceGwei, withdrawalVaultBalanceWei);
}

// Called by the Sanity Checker while processing the accounting report.
function getReport(uint256 reportReference) external view returns (
    bool success,
    uint256 clBalanceGwei,
    uint256 withdrawalVaultBalanceWei
) {
    Report memory report = reports[reportReference];
    return (report.exists, report.clBalanceGwei, report.withdrawalVaultBalanceWei);
}
```

The report passes if:

- an attestation for `reportReference` is ready;
- the second-opinion total CL balance is not below the reported value;
- the difference does not exceed `clBalanceOraclesErrorUpperBPLimit` of the second-opinion value;
- the reported Withdrawal Vault balance equals the committee value.

The committee publishes the exact values it verified, and `clBalanceOraclesErrorUpperBPLimit` is initially zero. The reported and committee CL balances must therefore match. A mismatch reverts the report and must be investigated before the incorrect report or attestation is replaced.

[LIP-23](./lip-23.md#matching) used a one-sided upper margin because a ZK oracle that identifies Lido validators by withdrawal credentials can include validators not deposited by Lido. The comparison remains one-sided: the second-opinion value may be slightly above the reported value, but never below it. A non-zero margin should be enabled only when the active provider's methodology requires it.

`clBalanceGwei` and `totalPostCLBalance` both mean the total CL balance, including pending deposits.

The Withdrawal Vault value is matched only in the negative CL rebase branch. The exact match prevents the accounting oracle from understating ETH held in the vault, without adding this check to normal reports.

### Committee and provider

The provider is a small contract controlled by a dedicated committee multisig. The committee can publish or replace an attestation for a `reportReference`.

The committee cannot submit an accounting report or override the hard cutoff. Its members must be independent from the accounting-oracle members. The DAO approves the multisig address and its initial membership and threshold. The committee's off-chain process must independently derive the total CL balance and Withdrawal Vault balance for the report reference. The provider contract only stores the resulting attestation.

The checker retains `setSecondOpinionOracleAndCLBalanceUpperMargin(...)` and `SECOND_OPINION_MANAGER_ROLE`. The provider may also be changed through the existing bulk setter.

### Configurable limits

The proposal keeps the existing limit surface:

- `maxCLBalanceDecreaseBP` in `LimitsList`;
- `getMaxCLBalanceDecreaseBP()`;
- `setMaxCLBalanceDecreaseBP(uint256)`;
- the existing bulk `setOracleReportLimits(...)` setter;
- `MaxCLBalanceDecreaseBPSet`;
- `MAX_CL_BALANCE_DECREASE_MANAGER_ROLE`.

The meaning changes from a cumulative 36-day allowance to a per-report hard cutoff. Other limit parameters are unchanged.

The existing `clBalanceOraclesErrorUpperBPLimit`, `CLBalanceOraclesErrorUpperBPLimitSet`, and combined second-opinion setter are retained. The margin is zero for the committee and may be changed together with the provider.

| Parameter                           | Proposed initial value | Meaning                                                                                                        |
| ----------------------------------- | ---------------------: | -------------------------------------------------------------------------------------------------------------- |
| `maxCLBalanceDecreaseBP`            |          `500` BP (5%) | Maximum negative CL rebase accepted in one report with committee attestation, relative to principal CL balance |
| `clBalanceOraclesErrorUpperBPLimit` |                 `0` BP | Maximum amount by which the second-opinion CL balance may exceed the reported value                            |

The initial hard cutoff must be finalized before implementation.

An account holding `MAX_CL_BALANCE_DECREASE_MANAGER_ROLE` can change the cutoff through `setMaxCLBalanceDecreaseBP`.

### Retained vault scalar

The negative-rebase check stores no report state. It uses the `withdrawalsVaultTransfer` already calculated for the current report instead of deriving CL withdrawals from two vault snapshots. The 36-day report ledger and `lastReportTimestamp` are removed.

`lastVaultBalanceAfterTransfer` remains and continues to be updated after every successful report because the separate CL balance increase check (`IncorrectTotalCLBalanceIncrease`) uses it to derive CL withdrawals. The new negative-rebase check does not use this scalar.

No initialization function is added. The scalar starts at zero, so the first report may count the outgoing checker's remaining vault baseline as new CL withdrawals. This can cause a false revert in the unchanged CL balance increase check, but cannot make that check more permissive.

### Removal of migration-only state

The current `isPostMigrationFirstReportDone` flag only disables the module balance increase check for the first report after the SRv3 migration. It does not initialize or modify module or Staking Router state. The replacement checker is activated after that migration and its first accounting report have completed, so the flag and its one-report bypass are removed. The module balance increase check runs for every report.

### Upgrade path

1. The DAO approves the committee, its threshold, and the initial cutoff.
2. The committee multisig, provider, and checker are deployed. Existing limits are copied, `maxCLBalanceDecreaseBP` is set to the approved value, and `clBalanceOraclesErrorUpperBPLimit` is set to zero.
3. One DAO vote configures the provider and switches `LidoLocator` to the new checker. The provider is set before the checker becomes active.
4. No negative-rebase ledger state is migrated or initialized. `lastVaultBalanceAfterTransfer` starts from zero as described above.

### Removed code and parameters

The following are removed:

- `ReportData`, `reportData`, `getReportDataCount()`, `lastReportTimestamp`, and `CL_BALANCE_WINDOW`;
- `_calcWindowDiff`, `_findWindowBaselineIndex`, and `_addReportData`;
- `migrateBaselineSnapshot()`;
- `BaselineSnapshotMigrated`, `MigrationAlreadyDone`, and `UnexpectedLidoVersion`;
- `NegativeCLRebaseAccepted`.

### Backward compatibility

`getOracleReportLimits()` keeps the same tuple shape. `maxCLBalanceDecreaseBP` and `clBalanceOraclesErrorUpperBPLimit` stay in the same positions and types. The accounting daemon does not need to change its report calculation.

The second-opinion ABI is simplified because no provider is currently connected: unused validator-count values are removed, and the parameter is named `reportReference`. No provider migration is needed. The new checker has a new address and ABI.

`reportData` and `getReportDataCount()` are removed from the ABI. The accounting daemon does not use them.

## Security Considerations

The proposal replaces a model of plausible CL losses with two explicit security boundaries:

- With the initial zero second-opinion margin, a negative CL rebase detected by the accounting-impact check requires an exact committee attestation; the current modeled allowance is removed.
- The recovery corridor for a genuine loss is increased up to the hard cutoff. During Dual Governance, the committee can attest directly without waiting for a DAO proposal.

The zero threshold applies after adding the ETH actually transferred from the Withdrawal Vault, including balance left there after earlier reports. A compromised accounting oracle can offset a reported CL loss only with ETH that is present in the vault and transferred into protocol accounting. It cannot fabricate that balance beyond the live vault balance. This limited corridor is accepted to keep the check stateless.

The hard cutoff is per report, not cumulative. If both reporting groups are compromised, they can repeat a within-cutoff loss in later reports. This risk is accepted to avoid restoring stateful loss accounting; separation of the two groups and monitoring are the controls against repetition.

If the DAO later enables a non-zero second-opinion margin for a ZK provider, the accounting oracle can under-report CL balance within that margin even when the provider is honest. This is the price of supporting a withdrawal-credentials-based upper bound. The margin and provider must therefore be changed together, and the hard cutoff remains the outer bound.

The check limits the combined effect of the reported CL balance and the actual Withdrawal Vault transfer, not raw CL performance or the final stETH rebase. EL rewards and other report components are outside this calculation. Reports that the accounting-impact calculation classifies as non-negative remain outside this check.

## Failure Modes

**Provider missing or committee unavailable.** A genuine negative CL rebase report fails closed and is retried until the frame deadline. If the frame is missed, withdrawals and the rebase are deferred.

**Loss above the cutoff.** The report reverts even with a correct committee attestation. The DAO must change the cutoff or replace the checker. This may be delayed during Dual Governance Rage Quit.

**Missed reports or non-finality.** The next report covers a longer period and may exceed the configured cutoff even when correct. It then reverts until the DAO raises the cutoff or replaces the checker.

**First report after the checker switch.** Starting `lastVaultBalanceAfterTransfer` from zero makes the unchanged CL balance increase check more conservative. If the outgoing checker left a significant vault baseline, the first report may revert. This must be checked in a fork simulation before activation.

## Links

- [LIP-23: Negative rebase sanity check with second opinion](./lip-23.md)
- [LIP-35: Staking Router v3](./lip-35.md)
- [SRv3 sanity-check parameter research](https://hackmd.io/0ePemSJtQA6S5NDK2M3mww)
- [ZK Lido Oracle powered by Succinct](https://research.lido.fi/t/zk-lido-oracle-powered-by-succinct/5747)
- [RISC Zero/Boundless second-opinion update](https://research.lido.fi/t/proposal-for-a-lido-accounting-oracle-second-opinion/9534/8)
- [Accounting Oracle second-opinion pause](https://research.lido.fi/t/proposal-for-a-lido-accounting-oracle-second-opinion/9534/9)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
